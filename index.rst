..
  Technote content.

  See https://developer.lsst.io/docs/rst_styleguide.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

  Use the following syntax for sections:

  Sections
  ========

  and

  Subsections
  -----------

  and

  Subsubsections
  ^^^^^^^^^^^^^^

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-technote-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. TODO: Delete the note below before merging new content to the master branch.

.. note::

   **This technote is not yet published.**

   Notes and recommendations of an investigation into third-party tools for consolidating system deployment and management across LSST enclaves and physical sites

.. Add content here.
.. Do not include the document title (it's automatically added from metadata.yaml).

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :encoding: latex+latin
..    :style: lsst_aa

Hosting, Managing, and Consuming Yum Repos: Pakrat, Katello, and More
=====================================================================

Managed Yum repositories are very important for the sake of
reproducibility and control.

-  The specific nature of the LSST project does not always allow us to
   rebuild nodes (e.g., from xCAT) in order to update them so we must be
   able to apply Yum updates from a controlled source.

-  We need to be able to (re)build and patch each node up to state that
   is consistent with other nodes, so locking repos into "snapshots" is
   important.

-  We may need to be able to roll back to a previous set up patches for
   the sake of recovering from an issue, so retaining previous repo
   "snapshots" is important.

-  We need to be able to "branch" our repos so that dev and test
   machines can see newer "snapshots" and production machines need to
   see slightly older "snapshots", so having multiple active "snapshots"
   is important.

Solutions for Managing and Hosting Yum Repos
--------------------------------------------

Pakrat + createrepo + web server
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Overview
~~~~~~~~

`Pakrat <https://github.com/ryanuber/pakrat>` is a Python-based tool
for mirroring and versioning Yum repositories. In our
investigation/setup we are running it with a wrapper script via CRON
(originally this was weekly but we've moved to daily). Each time the
wrapper script runs Pakrat syncs several repos and then uses createrepo
to rebuild the Yum metadata for the repo. Apache is used to serve out
the repos.

Each repo synced by Pakrat consists of:

-  a top-level 'Packages' directory - stores RPMs

-  sub-folders for each versioned snapshot - looks like a Yum repo and
   contains metadata for a given point in time

-  a 'latest' symlink pointing to the most recent snapshot (we're not
   using this symlink)

Each repo snapshot consists of a symlink to the top-level 'Packages'
directory and a unique 'repodata' metadata sub-folder. The 'repodata' is
created immediately after syncing on a given date and it only refers to
RPMs that are available in 'Packages' at that the time. As long as
'repodata' is not recreated in a given snapshot folder machines using
that repo will not be able to access additional RPMs added to 'Packages'
in the intervening time.

Here's a diagram of the structure of a repo:

-  repobase/

   -  2017-07-10/

      -  Packages -> ../Packages

      -  repodata/

   -  2017-07-17/

      -  Packages -> ../Packages

      -  repodata/

   -  ...

   -  2017-07-31/

      -  Packages -> ../Packages

      -  repodata/

   -  latest -> 2017-07-31

   -  Packages/

In our investigation/setup we point Yum clients to a specific snapshot
so that they are in a consistent, repeatable state. We have the ability
to point test clients to a newer snapshot. We have Pakrat set to sync
monthly.

Storage baselining (~7/3/2017 - 9/20/2017; includes CentOS 7.3 & 7.4)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

+--------+--------+--------+--------+--------+--------+--------+--------+
| repo   | raw    | synced | comp-  | synced | comp-  | synced | comp   |
|        | size,  |        | arativ |        | arativ | ed     | arativ |
|        |        | daily  | e      | weekly | e      | monthl | e      |
|        |        |        | raw    |        | raw    | y      | raw    |
|        | single |        | size   |        | size   |        | size   |
|        | day    | (~11   |        | (17    |        | (4     |        |
|        |        | 7      | (raw   | weeks) | (raw   | months | (raw   |
|        |        | days)  | X 117  |        | X 17   | )      | X 4    |
|        |        |        | days)  |        | weeks) |        | months |
|        |        |        |        |        |        |        | )      |
+========+========+========+========+========+========+========+========+
| CentOS | 7.4G   | 14G    | ~865.8 | 12G    | ~125.8 | 7.4G   | ~29.6G |
| :      |        |        | G      |        | G      |        |        |
| base   |        |        |        |        |        |        |        |
+--------+--------+--------+--------+--------+--------+--------+--------+
| CentOS | 979M   | 2.2G   | ~111.9 | 1.6G   | ~16.3G | 1.1G   | ~3.8G  |
| :      |        |        | G      |        |        |        |        |
| centos |        |        |        |        |        |        |        |
| plus   |        |        |        |        |        |        |        |
+--------+--------+--------+--------+--------+--------+--------+--------+
| CentOS | 1.4G   | 1.7G   | ~163.8 | 1.6G   | ~23.8G | 1.4G   | ~5.6G  |
| :      |        |        | G      |        |        |        |        |
| extras |        |        |        |        |        |        |        |
+--------+--------+--------+--------+--------+--------+--------+--------+
| CentOS | 6.5G   | 11G    | ~760.5 | 9.4G   | ~110.5 | 7.2G   | ~26G   |
| :      |        |        | G      |        | G      |        |        |
| update |        |        |        |        |        |        |        |
| s      |        |        |        |        |        |        |        |
+--------+--------+--------+--------+--------+--------+--------+--------+
| EPEL:  | 13G    | 19G    | ~1.49T | 17G    | ~221G  | 16G    | ~52G   |
| epel   |        |        |        |        |        |        |        |
+--------+--------+--------+--------+--------+--------+--------+--------+
| Puppet | 2.3G   | 2.7G   | ~269.1 | 2.5G   | ~39.1G | 2.4G   | ~9.2G  |
| Labs:  |        |        | G      |        |        |        |        |
| puppet |        |        |        |        |        |        |        |
| labs-p |        |        |        |        |        |        |        |
| c1     |        |        |        |        |        |        |        |
+--------+--------+--------+--------+--------+--------+--------+--------+
| TOTAL  | 31G    | 50G    | 3.54 - | 43G    | 527 -  | 36G    | 124 -  |
|        |        |        | 3.61T  |        | 536.5G |        | 126.2G |
+--------+--------+--------+--------+--------+--------+--------+--------+

Storage baselining (~7/3/2017 - 7/8/2017)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

+--------+--------+--------+--------+--------+--------+--------+--------+
| repo   | raw    | synced | comp-  | synced | comp-  | sync   | comp-  |
|        | size,  |        | arativ | weekly | arativ | ed     | arativ |
|        |        | daily  | e      |        | e      | monthl | e      |
|        |        |        | raw    |        | raw    | y      | raw    |
|        | single |        | size   |        | size   |        | size   |
|        | day    | (~35   |        | (6     |        | (2     |        |
|        |        | days)  | (raw   | weeks) | (raw   | months | (raw   |
|        |        |        | X 35   |        | X 6    | )      | X 2    |
|        |        |        | days)  |        | weeks) |        | months |
|        |        |        |        |        |        |        | )      |
+========+========+========+========+========+========+========+========+
| CentOS | 7.4G   | 8.3G   | ~269G  | 7.5G   | ~44.4G | 7.4G   | ~14.8G |
| :      |        |        |        |        |        |        |        |
| base   |        |        |        |        |        |        |        |
+--------+--------+--------+--------+--------+--------+--------+--------+
| CentOS | 979M   | 1.4G   | ~33.5G | 1.1G   | ~5.7G  | 1.1G   | ~1.9G  |
| :      |        |        |        |        |        |        |        |
| centos |        |        |        |        |        |        |        |
| plus   |        |        |        |        |        |        |        |
+--------+--------+--------+--------+--------+--------+--------+--------+
| CentOS | 1.4G   | 1.5G   | ~49G   | 1.4G   | ~8.4G  | 1.4G   | ~2.8G  |
| :      |        |        |        |        |        |        |        |
| extras |        |        |        |        |        |        |        |
+--------+--------+--------+--------+--------+--------+--------+--------+
| CentOS | 6.5G   | 7.9G   | ~227.5 | 7.3G   | ~39G   | 7.2G   | ~13G   |
| :      |        |        | G      |        |        |        |        |
| update |        |        |        |        |        |        |        |
| s      |        |        |        |        |        |        |        |
+--------+--------+--------+--------+--------+--------+--------+--------+
| EPEL:  | 13G    | 15G    | ~455G  | 14G    | ~78G   | 14G    | ~26G   |
| epel   |        |        |        |        |        |        |        |
+--------+--------+--------+--------+--------+--------+--------+--------+
| Puppet | 2.3G   | 2.4G   | ~80.5G | 2.3G   | ~13.8G | 2.3G   | ~4.6G  |
| Labs:  |        |        |        |        |        |        |        |
| puppet |        |        |        |        |        |        |        |
| labs-p |        |        |        |        |        |        |        |
| c1     |        |        |        |        |        |        |        |
+--------+--------+--------+--------+--------+--------+--------+--------+
| TOTAL  | 31G    | 36G    | 1,085  | 34G    | 186 -  | 33G    | 62 -   |
|        |        |        | -      |        | 189.3G |        | 63.1G  |
|        |        |        | 1,114. |        |        |        |        |
|        |        |        | 5G     |        |        |        |        |
+--------+--------+--------+--------+--------+--------+--------+--------+

Puppet Implementation
~~~~~~~~~~~~~~~~~~~~~

-  modules:

   -  'apache', from Puppet Forge

   -  'apache_config', includes default config, firewall, and vhost

   -  'pakrat', includes base installation, wrapper, cron, and storage
      config

-  profiles

   -  'pakrat', includes pakrat module

   -  'yum_server', includes elements of apache_config

-  roles

   -  'pakrat_yum_server', uses profile::pakrat and profile::yum_server

Daily Ops
~~~~~~~~~

-  Note: This should be fleshed out a little more in the near-term, as
   necessary. If we elect to stick with Pakrat long-term then we can
   expand it even more.

-  When/how to run the Pakrat repo sync?

   -  The Pakrat repo sync wrapper script is installed
      at /root/cron/pakrat.sh.

      -  It depends on a pakrat.config file in the same directory.

   -  The wrapper script is run daily by cron at 4:25pm.

   -  The wrapper script can also be run manually.

   -  Resiliency/details:

      -  Repos will be given a pathname that ends with the Unix epoch
         timestamp so there should be no problem with running the script
         more than once per day.

      -  The wrapper script will exit if it detects that it is already
         being run (just in case there are issues with Pakrat/Yum under
         the hood that would make simultaneous runs problematic).

-  How to add additional repos for Pakrat to sync?

   -  Recommended procedures:

      -  Establish the client configuration for the repository on the
             Pakrat-Yum server.

      -  XXXXXXXXX

   -  NOTE: If/when we start dealing more with GPG keys we will need to
      update this procedure slightly. See
      also \ `LSST-1031 <https://jira.ncsa.illinois.edu/browse/LSST-1031>`.

Improvements - High Priority
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

-  GPFS

   -  overall:

      -  size: Dan suggests ~50TB but look at baselining data from
         object-data06

         -  synced daily for ~117 days leads to 50G of storage

      -  location: Andy says just inside GPFS root for now; mkdir -p
         pakrat/production (just in case)

      -  refactor Puppet code (apache_config) and Pakrat scripts to look
         for this location

      -  implement GPFS code in Puppet to make sure it is mounted

   -  add error checking into Pakrat script to handle case where GPFS is
      not available

   -  after further consideration, probably best to back up to GPFS but
      still store on disk (what happens if GPFS is broken and our goal
      is to push out a patch...?)

-  create more verbose timestamp via wrapper so that we can run Pakrat
   multiple times a day if necessary

   -  ran it twice in one day once (into the same snapshot) and
      encountered the errors described below for the elasticsearch-1.7
      and influxdb repos

      -  initially thought they were related to running Pakrat twice
         into the same output repo path but they are persisting on the
         regularly weekly runs and after adding the Unix epoch timestamp
         to the repo paths

-  fix the following issue: packages with unexpected filenames do not
   appear in local Pakrat-generated metadata:

   -  the particularly metadata issue we are concerned about is as
      follows and (so far) only affects the elasticsearch-1.7 and
      influxdb repos:

      -  results in errors in Pakrat output such as this:

         -  Cannot read file:
            /repos/centos/7/x86_64/influxdb/2017-08-14/Packages/chronograf-1.3.0-1.x86_64.rpm

   -  

      -  these errors correspond to the following scenario:

         -  as listed in the \*primary.xml metadata from the SOURCE
            repository

         -  version/release info in 'href' parameter of 'location' key
            does not match various versions shown in 'rpm-sourcerpm'
            key:

            -  `rpm:sourcerpm <http://rpmsourcerpm>` (hard to imagine
               this is relevant)

            -  `rpm:provides <http://rpmprovides>` -
               `rpm:entry <http://rpmentry>` (e.g., rel=)

         -  more specifically, the rpm name does NOT have a release
            segment in it

         -  e.g., 'elasticsearch-1.7.0.noarch.rpm' is the RPM and it
            does not have a release in it's name (e.g.,
            \*1.7.0\ -1.noarch.rpm) but SOURCE metadata indicates it
            is release -1:

            -  | <`rpm:sourcerpm <http://rpmsourcerpm>`>elasticsearch-1.7.0\ -1.src.rpm</\ `rpm:sourcerpm <http://rpmsourcerpm>`>
               | <`rpm:header-range <http://rpmheader-range>`
                 start="880" end="19168"/>
               | <`rpm:provides <http://rpmprovides>`>
               | <`rpm:entry <http://rpmentry>` name="elasticsearch"
                 flags="EQ" epoch="0" ver="1.7.0"\ rel="1"/>
               | <`rpm:entry <http://rpmentry>`
                 name="config(elasticsearch)" flags="EQ" epoch="0"
                 ver="1.7.0" \ rel="1"/>
               | </`rpm:provides <http://rpmprovides>`>

         -  Pakrat downloads the RPMs but does not include them in its
            local metadata (e.g., the only elasticsearch RPM that
            appears in Pakrat's metadata is 1.7.4-1, because that is the
            only RPM that has a properly-formatted name, including the
            release)

            -  thus they would be unknown to Yum clients going through
               Pakrat

   -  possible fixes:

      -  work with the vendor to release properly named RPMs

      -  improve Pakrat to address this scenario (i.e., use the source
         metadata to fix its local metadata)

         -  or is this an issue for the makerepo command

      -  see if Katello has the same issue or not

      -  mv or cp (or make symlinks for) the badly named RPMs after
         Pakrat downloads them; this may ensure that Pakrat includes
         them in its metadata

         -  could probably script this fix, i.e., when Pakrat sync
            uncovers one of these errors, look for RPM without release
            in its name and copy it to the version that it is looking
            for so that the next run can include it in its metadata
            (perhaps even schedule another run of the repo at the end)

         -  if we start cleaning out old "snapshots" and RPMs that are
            no longer used, then we may also have to build a workaround
            into that process

            -  although it's possible that the worst that would happen
               is that after a clean out, several badly named RPMs are
               redownloaded during the next Pakrat sync

            -  using symlinks may help us here:

               -  register the targets of all symlinks ahead of the
                  cleanup

               -  only remove a target if you are also going to remove
                  the symlink

-  find and implement additional repos

   -  search /etc/yum.repos.d using xdsh

   -  search for the following terms in Puppet:

      -  yum

      -  

         -  adm::puppetdb

         -  base::puppet

      -  rpm

      -  package

   -  

      -  tar

      -  wget

      -  curl

      -  .com

      -  .edu

      -  git

   -  sync all repos in Pakrat

   -  redo Puppet implementation for Yum clients

Improvements - Low Priority (e.g., only if we adopt Pakrat as a permanent solution)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

-  Apache:

   -  move vhost stuff into Hiera

   -  move firewall networks into Hiera

   -  should I eliminate apache_config module? move all Hiera references
      and 'apache' module references into profile?

-  Pakrat:

   -  move config (.config file, cron stuff) into Hiera

   -  is my approach for installing OK?

   -  

      -  how to handle the dependency that fails to install initially?

   -  improve verification/notification/fix when Pakrat sync is broken

      -  fix postfix for cron (this is a larger issue)

      -  are we sure that cron scheduling via crontab (as opposed to
         file-based /etc/cron.d scheduling) will result in emails for
         any output? yes

   -  how to know which RPM versions are included in each snapshot?

      -  look at \*-primary.xml.gz / \*-other.xml.gz; zcat piped to some
         xml parser?

   -  document troubleshooting/monitoring for Pakrat

Katello
^^^^^^^^^^^

.. _overview-1:

Overview
~~~~~~~~

`Katello <https://theforeman.org/plugins/katello/>` is a plug-in for
Foreman that is used to manage content, specifically local Yum and
Puppet repositories. Katello is an integrated control interface and UI
for Pulp and also Candlepin (RH subscription management). These products
are all components of the RedHat Satellite platform.

Decision to Not Use Katello (October 2017)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Areas where it possibly offers benefits or at least different features
as compared to the alternative (Puppet w/ Git and Pakrat, then Foreman
or xCAT):

1. Integrated change control for Yum and Puppet.

2. Ability to schedule releases of content.

3. GUI for managing Yum repo syncing and management.

4. Flexibility in managing which RPMs are offered in Yum repos.

5. Ability to discard old Yum RPMs.

6. Manages RHEL subscriptions.

7. Handles syncing from Foreman/Katello 'master' to Katello 'capsule' (a
   Foreman Smart Proxy with Katello content services):

-  

   -  https://theforeman.org/plugins/katello/2.4/user_guide/capsules/index.html

Reasons we have elected not to investigate Katello further at this time:

-  Install and design seems overly complicated.

   -  You must install Katello before installing Foreman, then run the
      foreman-installer with a special flag in order to install Foreman
      for use with Katello
      (`link <https://theforeman.org/plugins/katello/nightly/installation/index.html>`).

   -  Creates the need to consult both Katello's documentation and
      Foreman's documentation for some considerations.

-  The above features don't seem to offer anything critical that we need
   and which we haven't already solved with Pakrat and our current
   Puppet/Git change control process.

   -  

      1. We already have integrated change control, via Git, for Yum and
         Puppet. In fact, it's not clear whether or not Katello's state
         can be captured by Git.

      2. We don't really need to schedule the release of content. Our
         focus is more likely to be on scheduling patching or allowing a
         NHC process to do rolling patching.

      3. A GUI is probably not necessary. Our Git/Puppet work is done in
         the CL already. We will likely investigate the Hammer CLI for
         Foreman as well.

      4. This is a little tricky with Pakrat, although presumably we
         could set certain RPMs to the side and recreate/edit metadata.

      5. We can generate a manual process for discarding old Yum RPMs
         from Pakrat, although it might not be worth it. Space is cheap.

      6. We do not currently use RHEL.

      7. We could set up a Yum-Pakrat 'master' and have each Smart
         Proxy/Yum-Pakrat slave sync from it.

In summary, it doesn't appear that the benefits of Katello outweigh the
extra complications it seems to present.

Other Considerations
^^^^^^^^^^^^^^^^^^^^^^^^

If we ever decide that Pakrat seems lacking in some area we should
consider \ `Pulp <http://docs.pulpproject.org/>` (which is used by
Katello) and also survey the landscape to see if anything else is
available besides Katello.

Yum Client Config and Puppet Best Practices
-------------------------------------------

.. _overview-2:

Overview
^^^^^^^^

-  All of our nodes must be configured to look at our managed Yum repos:

   -  during or immediately after deployment (by xCAT, Foreman, etc.)

   -  before any attempts by Puppet or other actors to go out and get an
      RPM by running Yum

-  We need to implement other things in Puppet in such a way that they
   only use Yum to get RPMs.

   -  Anything that is not an RPM should either be built into an RPM and
      hosted locally, stashed in Git, or hosted and versioned in some
      other way.

-  All needed Yum repos should be managed (ideally Puppet would disable
   or uninstall unmanaged repos).

Current Practice
^^^^^^^^^^^^^^^^

-  EPEL hostname is configured by a resource from the 'epel' module from
   Puppet Forge using Hiera

   -  but where the the 'epel' module declared for each node? only in
      other modules that happen to be covering all nodes?

-  extra::yum was created to manage other repos (CentOS and Puppet Labs)
   using the 'file' resource

   -  also turns off delta RPMs

-  profile::yum_client was created to utilize the extra::yum manifest

   -  all roles reference this profile

-  various other modules install repos using the 'yumrepo' resource type
   or by installing RPMs that install repos

Improvements - High Priority (these are needed whether we use Pakrat or Katello)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

-  Yum:

   -  stop managing yumrepo files and use one or both the of the
      following:

      -  'yum' module (3rd-party Yum module)

         -  this might only be needed to manage other aspects of Yum
            configuration (e.g., turn off delta RPMs, throw out old
            kernels, etc.), beyond which repos are present, enabled,
            etc.

      -  'yumrepo' resource type

   -  put all repo URLs and other data in Hiera

   -  manage all repos that are needed, pulling updates from
      Pakrat/Katello

   -  will we need to install/manage GPG keys? which repos use them
      (EPEL does but this is handled)? how about Puppet Labs, etc.? how
      do we manage them?

      -  GPG keys are often installed by the RPMs that also install the
         .repo files, no
         (e.g., `ZFS <https://github.com/zfsonlinux/zfs/wiki/RHEL-%26-CentOS>`)?

      -  files are placed in /etc/pki/rpm-gpg (could be hosted
         in/installed by Puppet) and then installed using a command like
         "rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-zfsonlinux"

      -  can the 'yumrepo' Puppet resource help with this? does the
         'yum' Puppet module handle it better?

   -  disable any unmanaged repos (or even uninstall files for unmanaged
      repos? which is better / easier)

      -  can remove the xCAT provisioning repos after deployment:

         -  xCAT-centos7-path0

         -  xcat-otherpkgs0

      -  the following repos can be removed from adm01:

         -  centosplus-source/7

         -  dell-system-update_independent

         -  gitlab_gitlab-ce-source

   -  document daily procedures for pointing Yum clients at specific
      snaphots (this is \*probably\* needed for Katello as well, but
      possibly not)

   -  consider explicitly including the epel module in
      profile::yum_client

-  Other Puppet refactoring/updates:

   -  anything that requires a pkg MUST also require the appropriate Yum
      resources / EPEL module, etc. so that any managed repo is
      configured first; update and document

-  xCAT (or Foreman)

   -  install basic Yum config (CentOS, Puppet Labs, EPEL at a minimum);
      kind of a belt and suspenders thing, just in case some Puppet
      thing would otherwise sneak in an external RPM

Foreman
=======

Purpose and Background

ITS is already using this (for non-LSST resources) for Puppet ENC and
reporting.

Security is using this for their machine (largely VMs).

Investigation on LSST Test Cluster

Foreman is being installed on lsst-test-adm01. More info:

-  `Foreman Feature Matrix and
   Evaluation <file:////display/LSST/Foreman+Feature+Matrix+and+Evaluation>`

-  `Foreman on test
   cluster <file:////display/LSST/Foreman+on+test+cluster>`

Resources

project website: `theforeman.org <https://theforeman.org/>`

slideshare: `Host Orchestration with Foreman, Puppet and
Gitlab <https://www.slideshare.net/tullis/linux-host-orchestration-with-foreman-with-puppet-and-gitlab>`

Foreman Feature Matrix and Evaluation
-------------------------------------

-  `Overview <#ForemanFeatureMatrixandEvaluation-Overv>`

-  `Feature Matrix <#ForemanFeatureMatrixandEvaluation-Featu>`

   -  `Deployment <#ForemanFeatureMatrixandEvaluation-Deplo>`

   -  `BMC/firmware
      management <#ForemanFeatureMatrixandEvaluation-BMC/f>`

   -  `Integration w/
      Puppet <#ForemanFeatureMatrixandEvaluation-Integ>`

   -  `Yum repo
      hosting/management <#ForemanFeatureMatrixandEvaluation-Yumre>`

   -  `Distributed architecture and
      scalability <#ForemanFeatureMatrixandEvaluation-Distr>`

   -  `Reliability <#ForemanFeatureMatrixandEvaluation-Relia>`

   -  `Interface / workflow / ease of
      use <#ForemanFeatureMatrixandEvaluation-Inter>`

   -  `Documentation and
      support <#ForemanFeatureMatrixandEvaluation-Docum>`

-  `Summary Evaluation <#ForemanFeatureMatrixandEvaluation-Summa>`

-  `Addendum 1: Possible end
   states <#ForemanFeatureMatrixandEvaluation-Adden>`

-  `Addendum 2: Other considerations for making a
   decision <#ForemanFeatureMatrixandEvaluation-Adden>`

Overview
--------

The purpose of this page is to help us enumerate the features of a
Foreman-based solution vs. an xCAT-based solution to deployment and
management of nodes. It may pay to consider a hybrid solution, namely a
Foreman-based solution that also uses pieces of xCAT (or Confluent).

NOTE: We also need to indicate which of the listed features are
requirements. Some may not be.

Feature Matrix
--------------

 Priority key:

3) requirement - must have this or we cannot deliver for the project
and/or common/critical admin tasks would be hopelessly inefficient

2) very helpful to have - not a requirement but would increase admin
efficiency considerably around a common task, decrease risk, or harden
security further

1) somewhat helpful to have - not a requirement but would increase admin
efficiency in a minor fashion

0) not needed - not necessary and of little usefulness, to the point
that it is not worth the time

?) unknown

+-----------------+-----------------+-----------------+-----------------+
|   Feature       |   Priority      |  xCAT-oriented  | Foreman-        |
|                 |                 |                 | oriented        |
+=================+=================+=================+=================+
|   Deployment    | --              |                 |                 |
|                 |                 |                 |                 |
|                 |                 |                 |                 |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| DHCP for mgmt   | 3               | Yes - tested    | Yes - tested    |
| networks        |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| PXE & TFTP      | 3               | Yes - tested    | Preliminary yes |
|                 |                 | both Dell and   |                 |
|                 |                 | Lenovo          | -  tested       |
|                 |                 |                 |    Lenovo       |
|                 |                 |                 |    (believe we  |
|                 |                 |                 |    had to       |
|                 |                 |                 |    change one   |
|                 |                 |                 |    or more BIOS |
|                 |                 |                 |    settings to  |
|                 |                 |                 |    get machine  |
|                 |                 |                 |    to boot      |
|                 |                 |                 |    after        |
|                 |                 |                 |    install      |
|                 |                 |                 |    and/or to    |
|                 |                 |                 |    PXE boot)    |
|                 |                 |                 |                 |
|                 |                 |                 | -  test Dell    |
+-----------------+-----------------+-----------------+-----------------+
| Anaconda        | 3               | Yes - meeting   | Preliminary yes |
| installs for    |                 | our needs so    |                 |
| CentOS:         |                 | far             | - may need to   |
| kickstart,      |                 |                 |   test more     |
| partition, etc. |                 |                 |   customization |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| Support for     | **???**         | Other NCSA      | Should support  |
| other distros   |                 | clusters are    | others,         |
| or OSes         |                 | using RHEL w/   | including       |
|                 |                 | xCAT.           | (apparently)    |
| -  we may need  |                 |                 | Windows (via    |
|    to support a |                 | Should support  | vSphere         |
|    handful of   |                 | others,         | templates)      |
|    Windows      |                 | including       |                 |
|    machines     |                 | (apparently)    | -  anything to  |
|    (e.g., AD),  |                 | Windows.        |    investigate/ |
|    likely VMs   |                 |                 |    test?        |
|                 |                 | - anything to   |    not now      |
|                 |                 |   investigate / |                 |
|                 |                 |   test? not now |                 |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| Deploys ESXi on | 1               | Yes, appears to | Yes, appears to |
| bare metal      |                 | install ESXi on | install ESXi on |
|                 |                 | bare metal      | bare metal      |
| -  should be    |                 | (xCAT wiki)     | (Foreman wiki)  |
|    infrequent   |                 |                 |                 |
|    and only     |                 |                 |                 |
|    involve a    |                 |                 |                 |
|    relatively   |                 |                 |                 |
|    small number |                 |                 |                 |
|    of machines  |                 |                 |                 |
|                 |                 |                 |                 |
|                 |                 | -  investigate  | -  investigate  |
|                 |                 |    further/test |    further/test |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| Local DNS for   | **???**         | Yes, although   | Yes - tested    |
| location-specif |                 | we haven't been |                 |
| ic              |                 | using           |                 |
| mgmt and svc    |                 |                 |                 |
| networks        |                 | -  investigate/ |                 |
|                 |                 |     test?       |                 |
| -  do we need   |                 |                 |                 |
|    this? or     |                 |                 |                 |
|    could we /   |                 |                 |                 |
|    must we rely |                 |                 |                 |
|    on external  |                 |                 |                 |
|    DNS +        |                 |                 |                 |
|    /etc/hosts?  |                 |                 |                 |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| Manage DNS      | 1               | Probably not.   | Possibly...but  |
| hosted on       |                 |                 | needs           |
| external system |                 | -  investigate? | investigation   |
| (e.g., make     |                 |                 |                 |
| local DNS       |                 | -  test?        | -  investigate/ |
| authoritative   |                 |                 |    test?        |
| or have mgmt    |                 |                 |                 |
| system interact |                 |                 |                 |
| with external   |                 |                 |                 |
| DNS via an API) |                 |                 |                 |
|                 |                 |                 |                 |
| -  do we need   |                 |                 |                 |
|    this?        |                 |                 |                 |
|    probably not |                 |                 |                 |
|    but might be |                 |                 |                 |
|    nice for     |                 |                 |                 |
|    internal     |                 |                 |                 |
|    networks     |                 |                 |                 |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| Bare-metal      | 3               | Yes - tested    | Yes - tested    |
| deployment      |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| OS deployment   | 2               | Yes, but not    | Yes, but not    |
| to VMs          |                 | yet tested      | yet tested      |
|                 |                 |                 |                 |
| -  i.e., we     |                 | https://sourcef | https://thefore |
|    have a VM    |                 | orge.net/p/xcat | man.org/manuals |
|    that is      |                 | /wiki/XCAT_Virt | /1.15/#5.2.9VMw |
|    manually     |                 | ualization_with | areNotes        |
|    provisioned  |                 | _VMWare/        |                 |
|    or was       |                 |                 | -  investigate  |
|    provisioned  |                 | -  investigate  |    PXE booting  |
|    using xCAT   |                 |    PXE booting  |    pre-provisio |
|    or Foreman,  |                 |    pre-provisio |    provisioned  |
|    now we need  |                 |    ned VMs      |    VMs          |
|    to install   |                 |                 |                 |
|    an OS on it  |                 |                 | -  investigate  |
|    (e.g., via   |                 | -  investigate  |    other        |
|    PXE +        |                 |    other        |    options?     |
|    kickstart as |                 |    options?     |                 |
|    w/ bare      |                 |                 |                 |
|    metal)       |                 |                 |                 |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| Provisioning of | 1               | Yes, but not    | Yes, but not    |
| VMs within      |                 | yet tested      | yet tested      |
| VMware          |                 |                 |                 |
|                 |                 | https://sourcef | https://thefore |
|                 |                 | orge.net/p/xcat | man.org/manuals |
|                 |                 | /wiki/XCAT_Virt | /1.15/#5.2.9VMw |
|                 |                 | ualization_with | areNotes        |
|                 |                 | _VMWare/        |                 |
|                 |                 |                 | - investigate   |
|                 |                 | -  investigate  |   integration   |
|                 |                 |    integration  |   with VMware   |
|                 |                 |    with VMware  |   to provisio n |
|                 |                 |    to provision |   VMs           |
|                 |                 |    VMs          |                 |
|                 |                 |                 |   - has         |
|                 |                 |    -  has       |     access to   |
|                 |                 |       access to |     what it     |
|                 |                 |       what it   |     needs and   |
|                 |                 |       needs and |     only what   |
|                 |                 |       only what |     it needs?   |
|                 |                 |       it needs? |                 |
|                 |                 |                 |   - other       |
|                 |                 |    -  other     |     security    |
|                 |                 |       security  |     concerns?   |
|                 |                 |       concerns? |                 |
|                 |                 |                 |   - remote      |
|                 |                 |                 |     (other      |
|                 |                 |                 |     sites/      |
|                 |                 |                 |     datacenters)|
|                 |                 |                 |     provision   |
|                 |                 |                 |     via         |
|                 |                 |                 |     VMware?     |
|                 |                 |                 |     i.e., how   |
|                 |                 |                 |     does the    |
|                 |                 |                 |     Foreman     |
|                 |                 |                 |     master      |
|                 |                 |                 |     provision   |
|                 |                 |                 |     resources   |
|                 |                 |                 |     in a        |
|                 |                 |                 |     remote      |
|                 |                 |                 |     location?   |
|                 |                 |                 |     does it     |
|                 |                 |                 |     talk to a   |
|                 |                 |                 |     local       |
|                 |                 |                 |     vSphere     |
|                 |                 |                 |     which       |
|                 |                 |                 |     then        |
|                 |                 |                 |     handles     |
|                 |                 |                 |     the         |
|                 |                 |                 |     provision   |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| Provisioning of | **???**         | Not really; the | Some support    |
| cloud resources |                 | xCAT            | (manual         |
| (e.g., AWS EC2, |                 | documentation   | provisioning    |
| GCE, etc.)      |                 | recommends      | with            |
|                 |                 | using Chef to   | image-based     |
|                 |                 | interact with   | deployment of   |
|                 |                 | these           | the OS).        |
|                 |                 | resources.      |                 |
|                 |                 |                 |                 |
|                 |                 | luster/         |                 |
+-----------------+-----------------+-----------------+-----------------+
| Diskless        | **???**         | Yes, using in   | Unsure...it     |
| install /       |                 | various NCSA    | seems possible  |
| stateless nodes |                 | clusters        | (just PXE-boot  |
|                 |                 |                 | from your       |
| -  do we need   |                 |                 | desired boot    |
|    this?        |                 |                 | image rather    |
|                 |                 |                 | than an         |
|    - 2017-12-18 |                 |                 | Anaconda-based  |
|      Meeting    |                 |                 | install image)  |
|      Notes:     |                 |                 | but there       |
|      Batch      |                 |                 | doesn't seem to |
|      Production |                 |                 | be any specific |
|      Services   |                 |                 | how-tos or      |
|                 |                 |                 | tutorials on    |
|                 |                 |                 | this and no     |
|                 |                 |                 | sign that       |
|                 |                 |                 | anyone asking   |
|                 |                 |                 | has ever gotten |
|                 |                 |                 | detailed help   |
|                 |                 |                 | with it         |
|                 |                 |                 |                 |
|                 |                 |                 | -  investigate/ |
|    - LDM-144:   |                 |                 |    test?        |
|                 |                 |                 |                 |
|      need input |                 |                 | -  we could     |
|      into what  |                 |                 |    build an     |
|      stateless  |                 |                 |    image with   |
|      nodes, etc |                 |                 |    xCAT and     |
|      will look  |                 |                 |    boot nodes   |
|      like       |                 |                 |    from it with |
|                 |                 |                 |    Foreman      |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| Node discovery  | 2               | Yes, but        | Offers this     |
| (w/o            |                 | haven't pursued | feature         |
| interacting     |                 | enough to get   | (`Discovery     |
| with switches)  |                 | it to work      | Plugin <https:/ |
|                 |                 |                 | /theforeman.org |
| -  we don't     |                 | -  investigate/ | /plugins/forema |
|    install      |                 |    test         | n_discovery/9.1 |
|    nodes all    |                 |    further?     | /index.html>`   |
|    that often;  |                 |                 | ),              |
|    it is        |                 |                 | but not tested  |
|    possible to  |                 |                 |                 |
|    discover     |                 |                 | - investigate/  |
|    mgmt MACs    |                 |                 |   test?         |
|    via PXE log  |                 |                 |                 |
|    entries then |                 |                 |                 |
|    configure    |                 |                 |                 |
|    BMCs from OS |                 |                 |                 |
|    (on Dell via |                 |                 |                 |
|    dtk,         |                 |                 |                 |
|    possibly     |                 |                 |                 |
|    also Lenovo) |                 |                 |                 |
|                 |                 |                 |                 |
| -  on the other |                 |                 |                 |
|    hand it's    |                 |                 |                 |
|    not clear    |                 |                 |                 |
|    how          |                 |                 |                 |
|    efficient    |                 |                 |                 |
|    collaboratin |                 |                 |                 |
|    w/ local     |                 |                 |                 |
|    boots on the |                 |                 |                 |
|    ground will  |                 |                 |                 |
|    be for       |                 |                 |                 |
|    deployments  |                 |                 |                 |
|    in Chile     |                 |                 |                 |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| Switch-based    | 1               | Yes             | No?             |
| discovery       |                 |                 |                 |
| (i.e., SNMP     |                 | - investigate/  | - investigate/  |
| query of        |                 |   test          |   test          |
| switches)       |                 |   further?      |   further?      |
|                 |                 |                 |                 |
| -  we don't     |                 |                 |                 |
|    install      |                 |                 |                 |
|    nodes all    |                 |                 |                 |
|    that often;  |                 |                 |                 |
|    it is        |                 |                 |                 |
|    possible to  |                 |                 |                 |
|    discover     |                 |                 |                 |
|    mgmt MACs    |                 |                 |                 |
|    via PXE log  |                 |                 |                 |
|    entries then |                 |                 |                 |
|    configure    |                 |                 |                 |
|    BMCs from OS |                 |                 |                 |
|    (on Dell via |                 |                 |                 |
|    dtk,         |                 |                 |                 |
|    possibly     |                 |                 |                 |
|    also Lenovo) |                 |                 |                 |
|                 |                 |                 |                 |
| -  on the other |                 |                 |                 |
|    hand it's    |                 |                 |                 |
|    not clear    |                 |                 |                 |
|    how          |                 |                 |                 |
|    efficient    |                 |                 |                 |
|    collaboratin |                 |                 |                 |
|    w/ local     |                 |                 |                 |
|    boots on the |                 |                 |                 |
|    ground will  |                 |                 |                 |
|    be for       |                 |                 |                 |
|    deployments  |                 |                 |                 |
|    in Chile     |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| Configure       | 0.5             | Yes?            | No?             |
| Ethernet switch |                 |                 |                 |
| ports           |                 | - xCAT docs:    | - investigate/  |
|                 |                 |   switch        |   confirm?      |
| -  not even     |                 |   management    |                 |
|    sure NetEng  |                 |                 |                 |
|    would allow  |                 |                 |                 |
|    us to do     |                 |                 |                 |
|    this         |                 |                 |                 |
|                 |                 | - investigate/  |                 |
|                 |                 |   confirm?      |                 |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| BMC/firmware    |                 | Need to         |                 |
| management      |                 | strong focus of | investigate     |
|                 |                 | xCAT.           | what the BMC    |
|                 |                 |                 | Smart Proxy     |
|                 |                 |                 | offers us.      |
|                 |                 |                 |                 |
|                 |                 |                 | Also            |
|                 |                 |                 | investigate how |
|                 |                 |                 | we can use      |
|                 |                 |                 | IBM/Lenovo      |
|                 |                 |                 | Confluent       |
|                 |                 |                 | (next-generatio |
|                 |                 |                 | n               |
|                 |                 |                 | of xCAT) with   |
|                 |                 |                 | Foreman.        |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| Remote power    | 3               | Yes - rpower    | -  investigate  |
|                 |                 |                 |    SmartProxy   |
|                 |                 |                 |    BMC feature  |
|                 |                 |                 |                 |
|                 |                 |                 | -  investigate  |
|                 |                 |                 |    Confluent    |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| Remote console  | 3               | Yes - xCAT's    | -  investigate  |
| and console     |                 | rcons and       |    SmartProxy   |
| capture         |                 | conserver       |    BMC feature  |
|                 |                 |                 |    or other     |
|                 |                 |                 |    Foreman      |
|                 |                 |                 |    options      |
|                 |                 |                 |                 |
|                 |                 |                 | -  investigate  |
|                 |                 |                 |    Confluent    |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| Manage BIOS     | 3               | Yes - Lenovo:   | -  investigate  |
| settings        |                 | xCAT's pasu,    |    SmartProxy   |
| out-of-band     |                 | but sometimes   |    BMC feature  |
| (ideally w/o    |                 | requires a      |    or other     |
| reboot) and     |                 | reboot          |    Foreman      |
| programmaticall |                 |                 |    options      |
| y               |                 | Yes - Dell:     |                 |
|                 |                 | must use        | -  investigate  |
|                 |                 | racadm,         |    Confluent    |
|                 |                 | probably with a |                 |
|                 |                 | wrapper         |                 |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| Install         | 3               | Lenovo:         | -  investigate  |
| firmware        |                 | supported via   |    SmartProxy   |
| outside of OS   |                 | xCAT Genesis    |    BMC feature  |
|                 |                 | boot + Lenovo   |    or other     |
| -  on Lenovo we |                 | onecli          |    Foreman      |
|    have not yet |                 |                 |    options      |
|    found a way  |                 | -  Dell?        |                 |
|    to do this   |                 |                 | -  investigate  |
|    outside of   |                 |                 |    Confluent    |
|    the OS, we   |                 |                 |                 |
|    have to PXE  |                 |                 | -  adapt        |
|    boot the     |                 |                 |    approach of  |
|    node         |                 |                 |    xCAT's       |
|                 |                 |                 |    Genesis boot |
|    - then       |                 |                 |    approach     |
|      again,     |                 |                 |    and/or       |
|      what is    |                 |                 |    Industry's   |
|      the        |                 |                 |    firmware     |
|      difference |                 |                 |    approach     |
|      between    |                 |                 |                 |
|      installin  |                 |                 |                 |
|      from the   |                 |                 |                 |
|      Genesis    |                 |                 |                 |
|      kernel     |                 |                 |                 |
|      and        |                 |                 |                 |
|      installing |                 |                 |                 |
|      from the   |                 |                 |                 |
|      booted OS  |                 |                 |                 |
|                 |                 |                 |                 |
| -  could be     |                 |                 |                 |
|    useful in    |                 |                 |                 |
|    general      |                 |                 |                 |
|    since it     |                 |                 |                 |
|    allows us to |                 |                 |                 |
|    install      |                 |                 |                 |
|    firmware     |                 |                 |                 |
|    even if      |                 |                 |                 |
|    there are    |                 |                 |                 |
|    local disk   |                 |                 |                 |
|    problems or  |                 |                 |                 |
|    w/o          |                 |                 |                 |
|    modifying an |                 |                 |                 |
|    install on   |                 |                 |                 |
|    local disks  |                 |                 |                 |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| Integration     | 2               | Not             | High level of   |
| w/Puppet        |                 | integrated...   | integration     |
|                 |                 |                 | with Puppet;    |
|                 |                 | -  xCAT         | provides:       |
|                 |                 |    installs     |                 |
|                 |                 |    Puppet       | -  Foreman is   |
|                 |                 |                 |    installed    |
|                 |                 | -  BYO ENC      |    via/alongsid |
|                 |                 | - Puppet        |    Puppet ENC   |
|                 |                 |   module for    |                 |
|                 |                 |   xCAT          | -  ENC          |
|                 |                 |   (out-of-date) |    - tested     |
|                 |                 |                 | -  Puppet       |
|                 |                 | ...However, the |    logging      |
|                 |                 | main thing      |                 |
|                 |                 | missing right   |                 |
|                 |                 | now is better   |                 |
|                 |                 | Puppet          |    - look       |
|                 |                 | reporting,      |      closer at  |
|                 |                 | although in     |      thisat     |
|                 |                 | theory this is  |                 |
|                 |                 | already         |                 |
|                 |                 | available in    | -  further      |
|                 |                 | NPCF via        |    investigate  |
|                 |                 | centralized     |    management   |
|                 |                 | logging and is  |    of           |
|                 |                 | being looked at |    distributed  |
|                 |                 | via our         |    Puppet       |
|                 |                 | monitoring      |    infrastructur|
|                 |                 | stack.          |                 |
|                 |                 |                 |                 |
|                 |                 |                 |    - Puppet     |
|                 |                 |                 |      Master     |
|                 |                 |                 |                 |
|                 |                 |                 |    - Puppet CA  |
|                 |                 |                 |                 |
|                 |                 |                 |       - cert    |
|                 |                 |                 |         signing |
|                 |                 |                 |         and     |
|                 |                 |                 |         revoke  |
|                 |                 |                 |                 |
|                 |                 |                 |    - other high |
|                 |                 |                 |      avail.     |
|                 |                 |                 |      considera- |
|                 |                 |                 |      tions?     |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| Yum repo        | 3               | Pakrat:         | Pakrat (or      |
| hosting/        |                 |                 | perhaps         |
| management      |                 | -               | Pulp/Katello)   |
|                 |                 |                 |                 |
|                 |                 |    - we have a  | -               |
|                 |                 |      number of  |                 |
|                 |                 |      minor      |   - we have a   |
|                 |                 |      issues to  |     number of   |
|                 |                 |      investigate|     minor       |
|                 |                 |                 |     issues to   |
|                 |                 |                 |     investigate |
|                 |                 |                 |                 |
|                 |                 |    - implement  |    - implement  |
|                 |                 |      syncing    |      syncing    |
|                 |                 |      from       |      from       |
|                 |                 |      master to  |      master to  |
|                 |                 |      remote     |      remote     |
|                 |                 |      servers    |      servers    |
|                 |                 |                 |                 |
|                 |                 | -  investigate  | -  investigate  |
|                 |                 |    Pulp?        |    Pulp?        |
|                 |                 |                 |                 |
|                 |                 |                 | -  investigate  |
|                 |                 |                 |    Katello?     |
|                 |                 |                 |                 |
|                 |                 |                 |    - integrated |
|                 |                 |                 |      with       |
|                 |                 |                 |      Foreman    |
|                 |                 |                 |      and        |
|                 |                 |                 |      likely     |
|                 |                 |                 |      handles    |
|                 |                 |                 |      syncing    |
|                 |                 |                 |                 |
|                 |                 |                 |    - Jake       |
|                 |                 |                 |      feels      |
|                 |                 |                 |      it's not   |
|                 |                 |                 |      worth      |
|                 |                 |                 |      looking    |
|                 |                 |                 |      at right   |
|                 |                 |                 |      now        |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| Distributed     | --              | Allows for      | Allows for      |
| architecture    |                 | distributed     | distributed     |
| and             |                 | management via  | management via  |
| scalability     |                 | Service Nodes:  | Foreman Smart   |
|                 |                 |                 | Proxies:        |
|                 |                 | https://xcat-do |                 |
|                 |                 | cs.readthedocs. | https://thefore |
|                 |                 | io/en/2.13.8/ad | man.org/manuals |
|                 |                 | vanced/hierarch | /1.15/#1.Forema |
|                 |                 | y/index.html    | n1.15Manual     |
|                 |                 |                 |                 |
| -  Will our     |                 | - this seems    | -  this is      |
|    nodes all    |                 |   like a        |    front and    |
|    have         |                 |   somewhat      |    center with  |
|    "public"     |                 |   nonstandard   |    Foreman (it  |
|    interfaces   |                 |   configuration |    is described |
|    or at least  |                 |                 |    in the very  |
|    be able to   |                 |   (we don't     |    first part   |
|    "NAT out" to |                 |   seem to be    |    of the       |
|    reach remote |                 |   using at      |    Foreman      |
|    management   |                 |   NCSA anyway)  |    manual)      |
|    resources?   |                 |                 |                 |
|                 |                 | - handles       | Foreman Master  |
|                 |                 |   subnetting    | controls        |
|                 |                 |   for           | deployments     |
|                 |                 |   management    | (DHCP, local    |
|                 |                 |   networks via  | DNS, TFTP)      |
|                 |                 |   "setupforward |                 |
|                 |                 |                 | -  verify       |
|                 |                 |   setting"      |    usability    |
|                 |                 |                 |    with remote  |
|                 |                 |                 |    nodes that   |
|                 |                 |                 |    have no      |
|                 |                 |                 |    public       |
|                 |                 |                 |    address and  |
|                 |                 |                 |    no NAT       |
|                 |                 |                 |    capability   |
|                 |                 | - but           |    (may be an   |
|                 |                 |   definitely    |    artificial   |
|                 |                 |   does NOT      |    constraint;  |
|                 |                 |   seem set up   |    nodes should |
|                 |                 |   for           |    probably be  |
|                 |                 |   distribution  |    able to      |
|                 |                 |   across WAN    |    connect      |
|                 |                 |                 |    outside for  |
|                 |                 | - in other      |    SSL CRL,     |
|                 |                 |   words, we'd   |    etc.)        |
|                 |                 |   need to have  |                 |
|                 |                 |   a full xCAT   |                 |
|                 |                 |   master for    |                 |
|                 |                 |   each          |                 |
|                 |                 |   datacenter    |                 |
|                 |                 |   (specifically |                 |
|                 |                 |   for each      |                 |
|                 |                 |   management    |                 |
|                 |                 |   network)      |                 |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| Central         | 1               | No, does not    | Yes, definitely |
| execution of    |                 | seem to support | handles         |
| remote          |                 | this out of the | updating node   |
| deployments /   |                 | box (doesn't    | settings        |
| central         |                 | support remote  | (stored in      |
| updating of     |                 | infrastructure  | Foreman Master) |
| node settings   |                 | at all)         |                 |
| on remote       |                 |                 | -  verify       |
| deployment      |                 | - investigate   |    remote node  |
| infrastructure  |                 |   custom        |    deployment   |
| (i.e.,          |                 |   syncing /     |    across       |
| configure       |                 |   updating of   |    sites: DNS,  |
| deployment      |                 |   xCAT          |    DHCP, PXE,   |
| settings on a   |                 |   configuration |    kickstart w/ |
| master          |                 |                 |    remote       |
| deployment      |                 |    across WAN?  |    Foreman      |
| server at NCSA  |                 |                 |    Smart Proxy  |
| to affect how a |                 | - do be clear,  |                 |
| node deploys in |                 |   w/ xCAT we'd  | -  investigate  |
| Chile, handle   |                 |   need to log   |    client       |
| things like     |                 |   into a        |    enrollment   |
| DHCP, PXE,      |                 |   different     |    to local     |
| kickstart,      |                 |   xCAT master   |    Puppet       |
| etc.)           |                 |   for each      |    Master /     |
|                 |                 |   datacenter    |    Puppet CA    |
|                 |                 |   unless we do  |    during       |
|                 |                 |   something     |    initial      |
|                 |                 |   custom        |    deployment   |
|                 |                 |                 |    (without     |
|                 |                 |                 |    node         |
|                 |                 |                 |    connectivity |
|                 |                 |                 |    to Foreman   |
|                 |                 |                 |    Master)      |
|                 |                 |                 |                 |
|                 |                 |                 |    -  local     |
|                 |                 |                 |       Puppet    |
|                 |                 |                 |       Master    |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| Central         | 2               | No, does not    | A little        |
| management of   |                 | seem to support | bit...?         |
| remote          |                 | this directly   |                 |
| deployment      |                 |                 | -  there are    |
| infrastructure  |                 | -  investigate  |    nice Puppet  |
| (across WAN)    |                 |    method of    |    modules for  |
| (i.e., how do   |                 |    syncing      |    managing     |
| we keep remote  |                 |    content/     |    Foreman      |
| deployment      |                 |    settings     |    Master /     |
| servers         |                 |    for          |    Foreman      |
| up-to-date)     |                 |    deployment   |    Smart Proxy  |
|                 |                 |    servers      |    that could   |
| -  we are       |                 |    across WAN   |    help for     |
|    likely to at |                 |                 |    updating     |
|    least have   |                 | -  investigate  |    server       |
|    local        |                 |    remote       |    settings at  |
|    DHCP/        |                 |    syncing of   |    least        |
|    kickstart    |                 |    Puppet repos |                 |
|    servers      |                 |                 | - but it does   |
|                 |                 |                 |   seem like at  |
|                 |                 |                 |   a fundamental |
|                 |                 |                 |   level each    |
|                 |                 |                 |   Smart Proxy   |
|                 |                 |                 |   is installed  |
|                 |                 |                 |   and           |
|                 |                 |                 |   configured    |
|                 |                 |                 |   independently |
|                 |                 |                 |                 |
|                 |                 |                 | - investigate   |
|                 |                 |                 |   method of     |
|                 |                 |                 |   syncing       |
|                 |                 |                 |   content       |
|                 |                 |                 |   (e.g.,        |
|                 |                 |                 |   images/source |
|                 |                 |                 |   repos) for    |
|                 |                 |                 |   deployment    |
|                 |                 |                 |   servers       |
|                 |                 |                 |   across WAN    |
|                 |                 |                 |                 |
|                 |                 |                 | - investigate   |
|                 |                 |                 |   remote        |
|                 |                 |                 |   syncing of    |
|                 |                 |                 |   Puppet repos  |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| Initiate        | 2               | No, does not    | Maybe...        |
| IPMI/firmware/h |                 | support this    |                 |
| ardware         |                 | out of the box  | -  investigate  |
| management      |                 |                 |    IPMI to      |
| commands on     |                 | -  investigate  |    initiate PXE |
| remote machines |                 |    custom setup |    (BMC Smart   |
| from a central  |                 |    for          |    Proxy, etc.) |
| location (e.g., |                 |    executing    |    across sites |
| set to PXE,     |                 |    remote       |                 |
| reboot, install |                 |    IPMI/BMC     | -  investigate  |
| firmware,       |                 |    commands     |    other remote |
| configure BMC,  |                 |                 |    IMPI/BMC     |
| etc.)           |                 |                 |    commands     |
|                 |                 |                 |    (BMC Smart   |
| -  alternative  |                 |                 |    Proxy, etc.) |
|    is to log    |                 |                 |                 |
|    into a       |                 |                 |                 |
|    remote       |                 |                 |                 |
|    management   |                 |                 |                 |
|    server and   |                 |                 |                 |
|    execute      |                 |                 |                 |
|    there        |                 |                 |                 |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| Distributed     | **3 or 1**      | xCAT-based      | A Foreman-based |
| Puppet          |                 | solution offers | solution may    |
| architecture    |                 | no assistance   | make some of    |
|                 |                 | here but it     | this easier:    |
| - We only       |                 | should all be   |                 |
|   strictly      |                 | possible.       | - If            |
|   *need* this   |                 |                 |   Foreman-based |
|   if it's       |                 | -  Local ENC or |                 |
|   determined    |                 |    sync ENC     |   Puppet ENC    |
|   to be         |                 |    between      |   works even    |
|   necessary     |                 |    Puppet       |   when Foreman  |
|   from a        |                 |    Masters.     |   Master is     |
|   security      |                 |                 |   unavailable,  |
|   perspective   |                 |                 |   then that is  |
|   or if nodes   |                 |                 |   a plus.       |
|   have no       |                 |                 |                 |
|   "public"      |                 |                 | - Foreman       |
|   interface     |                 |                 |   installer     |
|   and cannot    |                 |                 |   might make    |
|   NAT out.      |                 |                 |   setup of      |
|                 |                 |                 |   Puppet CA vs  |
| - Puppet repos  |                 |                 |   Puppet        |
|   need to be    |                 |                 |   Master        |
|   pulled from   |                 |                 |   somewhat      |
|   same Git or   |                 |                 |   easier (or    |
|   synced from   |                 |                 |   at least      |
|   authoritative |                 |                 |   offer a       |
|   repo.         |                 |                 |   template).    |
|                 |                 |                 |                 |
|                 |                 |                 | - We could      |
| - Can we have a |                 |                 |   further       |
|                 |                 |                 |   investigate   |
|   centralized   |                 |                 |   what (if      |
|   Puppet CA or  |                 |                 |   anything)     |
|   do we need    |                 |                 |   Katello has   |
|   it to be      |                 |                 |   to offer in   |
|   local?        |                 |                 |   this area,    |
|                 |                 |                 |   e.g., w/      |
|                 |                 |                 |   Puppet        |
|                 |                 |                 |   repository/   |
|                 |                 |                 |   module        |
|                 |                 |                 |   management.   |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| Distributed     | 3               | Yes, but        | Yes, but        |
| environments    |                 | investigate     | investigate     |
| can operating   |                 | Puppet (esp.    | Puppet (esp.    |
| during WAN cut  |                 | ENC and CA).    | ENC and CA).    |
|                 |                 |                 |                 |
| - Previously    |                 | - With our      | -  Does         |
|   deployed      |                 |   current ENC   |    Foreman's    |
|   machines can  |                 |   it probably   |    Puppet ENC   |
|   continue to   |                 |   makes sense   |    continue to  |
|   operate.      |                 |   to have it    |    operate      |
|                 |                 |   located on    |    during WAN   |
|   - This has    |                 |   each local    |    cut?         |
|     little to   |                 |   Puppet        |                 |
|     do with     |                 |   Master (but   |    -  If not,   |
|     which       |                 |   w/ syncing    |       verify    |
|     deployment  |                 |   from a        |       that we   |
|                 |                 |   central       |       can       |
|     solution    |                 |   source) so    |       instead   |
|     we pick.    |                 |   that the      |       use our   |
|     The main    |                 |   local Puppet  |       own ENC   |
|     considera-  |                 |   Master can    |       rather    |
|     tion        |                 |   continue      |       than      |
|     is, can     |                 |   functioning.  |       Foreman's |
|     machines    |                 |                 |                 |
|     continue    |                 | - What about    |                 |
|     to work     |                 |   Puppet CA?    | - What about    |
|     even if     |                 |   If we have a  |   Puppet CA?    |
|     Puppet      |                 |   single        |   If we have a  |
|     cannot      |                 |   Puppet CA     |   single        |
|     contact     |                 |   does a local  |   Puppet CA     |
|     its         |                 |   Puppet        |   does a local  |
|     master?     |                 |   Client-Puppet |   Puppet        |
|     As such     |                 |                 |   Client-Puppet |
|     it has      |                 |   Master        |                 |
|     more to     |                 |   session work  |   Master        |
|     do with     |                 |   w/o being     |   session work  |
|     whether     |                 |   able to       |   w/o being     |
|     or not we   |                 |   contact the   |   able to       |
|     need a      |                 |   remote        |   contact the   |
|     distributed |                 |   Puppet CA?    |   remote        |
|                 |                 |                 |   Puppet CA?    |
|     Puppet      |                 |                 |                 |
|     architecture|                 |                 |                 |
|                 |                 |                 |                 |
|   - Other       |                 |                 |                 |
|     considera-  |                 |                 |                 |
|     tions       |                 |                 |                 |
|     such as,    |                 |                 |                 |
|     do we       |                 |                 |                 |
|     need        |                 |                 |                 |
|     local       |                 |                 |                 |
|     DNS, NTP,   |                 |                 |                 |
|     SSL CRL,    |                 |                 |                 |
|     LDAP,       |                 |                 |                 |
|     etc. are    |                 |                 |                 |
|     directly    |                 |                 |                 |
|     not about   |                 |                 |                 |
|     deployment  |                 |                 |                 |
|     strictly    |                 |                 |                 |
|     speaking    |                 |                 |                 |
|     and more    |                 |                 |                 |
|     or less     |                 |                 |                 |
|     independent |                 |                 |                 |
|     of which    |                 |                 |                 |
|     deployment  |                 |                 |                 |
|     system we   |                 |                 |                 |
|     choose.     |                 |                 |                 |
|                 |                 |                 |                 |
| - Does not      |                 |                 |                 |
|   mean that we  |                 |                 |                 |
|   can initiate  |                 |                 |                 |
|   new           |                 |                 |                 |
|   deployments   |                 |                 |                 |
|   (how could    |                 |                 |                 |
|   we            |                 |                 |                 |
|   conceivably)  |                 |                 |                 |
|   although      |                 |                 |                 |
|   it'd be nice  |                 |                 |                 |
|   if one that   |                 |                 |                 |
|   was in        |                 |                 |                 |
|   progress      |                 |                 |                 |
|   would         |                 |                 |                 |
|   continue      |                 |                 |                 |
|   (hence 3 or   |                 |                 |                 |
|   2).           |                 |                 |                 |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| PXE over WAN    | 1               | No, xCAT does   | -  Investigate  |
|                 |                 | not seem to     |    further?     |
| - Not super     |                 | support PXE     |                 |
|   useful as it  |                 | over WAN.       |                 |
|   still         |                 |                 |                 |
|   requires      |                 |                 |                 |
|   local DHCP.   |                 |                 |                 |
|   It would      |                 |                 |                 |
|   just save us  |                 |                 |                 |
|   needing to    |                 |                 |                 |
|   have local    |                 |                 |                 |
|   installation  |                 |                 |                 |
|   repo/image.   |                 |                 |                 |
|                 |                 |                 |                 |
| - Does NOT      |                 |                 |                 |
|   include       |                 |                 |                 |
|   kickstart     |                 |                 |                 |
|   communication |                 |                 |                 |
|   itself (next  |                 |                 |                 |
|   topic).       |                 |                 |                 |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| Local kickstart | 3               | Yes, each xCAT  | Yes, Foreman    |
| server or       |                 | master would be | has a           |
| encryption of   |                 | local.          | "Templates"     |
| kickstart       |                 |                 | Smart Proxy     |
| communication   |                 |                 | feature that    |
|                 |                 |                 | supports        |
| - Kickstart     |                 |                 | distributed     |
|   files often   |                 |                 | sources         |
|   contain       |                 |                 | kickstart.      |
|   sensitive     |                 |                 |                 |
|   information   |                 |                 |                 |
|   so kickstart  |                 |                 |                 |
|   communication |                 |                 |                 |
|   should be     |                 |                 |                 |
|   encrypted or  |                 |                 |                 |
|   remain        |                 |                 |                 |
|   local.        |                 |                 |                 |
|                 |                 |                 |                 |
| - Encryption    |                 |                 |                 |
|   of kickstart  |                 |                 |                 |
|   communication |                 |                 |                 |
|   may be        |                 |                 |                 |
|   possible (w/  |                 |                 |                 |
|   RHEL, maybe   |                 |                 |                 |
|   CentOS) but   |                 |                 |                 |
|   it would be   |                 |                 |                 |
|   nonstandard   |                 |                 |                 |
|   w/ respect    |                 |                 |                 |
|   to both xCAT  |                 |                 |                 |
|   and Foreman.  |                 |                 |                 |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| Other security  | 3               | -  Security of  | -  Security of  |
| considerations  |                 |    any custom   |    remote IPMI  |
| (encryption of  |                 |    remote IPMI  |    solution /   |
| other other     |                 |    solution we  |    BMC Smart    |
| command data    |                 |    create, if   |    Proxy.       |
| across WAN;     |                 |    applicable.  |                 |
| authentication/ |                 |                 | -  Security of  |
| authorization;  |                 | -  Security of  |    any custom   |
| etc.)           |                 |    any custom   |    content      |
|                 |                 |    content      |    (Puppet,     |
|                 |                 |    (Puppet,     |    Yum, images, |
|                 |                 |    Yum, images, |    etc.)        |
|                 |                 |    etc.)        |    syncing      |
|                 |                 |    syncing      |    system we    |
|                 |                 |    system we    |    create, if   |
|                 |                 |    create, if   |    applicable.  |
|                 |                 |    applicable.  |                 |
|                 |                 |                 | -  Overall      |
|                 |                 | -  Overall      |    security     |
|                 |                 |    Security     |    vetting of   |
|                 |                 |    vetting of   |    Foreman.     |
|                 |                 |    whatever     |                 |
|                 |                 |    distributed  |                 |
|                 |                 |    setup we     |                 |
|                 |                 |    create.      |                 |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| Scalability     | 3               | Yes, an         | Yes, a          |
|                 |                 | xCAT-based      | Foreman-based   |
|                 |                 | solution should | solution should |
|                 |                 | be able to      | be able to      |
|                 |                 | scale to meet   | scale to meet   |
|                 |                 | our needs.      | our needs.      |
|                 |                 |                 |                 |
|                 |                 | -  To           | -  NOTE: Sounds |
|                 |                 |    reiterate,   |    like         |
|                 |                 |    we'd be      |    Foreman,     |
|                 |                 |    using        |    Puppet CA +  |
|                 |                 |    separate     |    Smart Proxy, |
|                 |                 |    xCAT masters |    Puppet ENC,  |
|                 |                 |    for each     |    & Reports on |
|                 |                 |    datacenter   |    one machine  |
|                 |                 |    (or more)    |    one w/ 1,000 |
|                 |                 |    and then     |    nodes could  |
|                 |                 |    setting up   |    be pushing   |
|                 |                 |    distributed  |    it a bit.    |
|                 |                 |    Puppet apart |    Move to high |
|                 |                 |    from/in      |    availability |
|                 |                 |    addition to. |    at or before |
|                 |                 |                 |    then is      |
|                 |                 |    - xCAT is    |    advised.     |
|                 |                 |      further    |    ("HA case    |
|                 |                 |      scalable   |    study")      |
|                 |                 |      within a   |                 |
|                 |                 |      datacenter |                 |
|                 |                 |      via the    |                 |
|                 |                 |      use of     |                 |
|                 |                 |      service    |                 |
|                 |                 |      nodes.     | -  How does     |
|                 |                 |                 |    Foreman's    |
|                 |                 | -  We've heard  |    BMC Smart    |
|                 |                 |    that the     |    Proxy        |
|                 |                 |    console      |    feature      |
|                 |                 |    server may   |    scale?       |
|                 |                 |    not scale    |                 |
|                 |                 |    (e.g.,       | -  How large is |
|                 |                 |    didn't seem  |    Security's   |
|                 |                 |    to work for  |    fleet and    |
|                 |                 |    iForge).     |    what kind of |
|                 |                 |    Multiple     |    load do they |
|                 |                 |    xCAT masters |    put on       |
|                 |                 |    could take   |    Foreman in   |
|                 |                 |    care of      |    terms of     |
|                 |                 |    that,        |    deploying    |
|                 |                 |    however.     |    machines?    |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| Reliability     |                 | Yes, seems      | Probably...     |
|                 |                 | solid overall   |                 |
|                 |                 | as evidenced by | -  Ask          |
|                 |                 | previous use at |    Security.    |
|                 |                 | NCSA, including |                 |
|                 |                 | LSST.           | -  Ask ITS.     |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| Ability to      | 3               | - See XCAT      | - See Foreman   |
| backup and      |                 |   Installation  |   Manual Backup,|
| restore         |                 |   Guide Backup  |   REcovery, and |
|                 |                 |   and Restore   |   Migration     |
|                 |                 |   section       |   section       |
|                 |                 |                 |                 | 
|                 |                 |    - Somewhat   | -  Test.        |
|                 |                 |      unclear    |                 |
|                 |                 |      if this    |                 |
|                 |                 |      encompass  |                 |
|                 |                 |      everything |                 |
|                 |                 |      needed     |                 |
|                 |                 |      for DHCP,  |                 |
|                 |                 |      DNS, etc.  |                 |
|                 |                 |      (maybe     |                 |
|                 |                 |      just run   |                 |
|                 |                 |      makedhcp,  |                 |
|                 |                 |      makedns,   |                 |
|                 |                 |      makehosts, |                 |
|                 |                 |      etc.       |                 |
|                 |                 |      after      |                 |
|                 |                 |      recovery). |                 |
|                 |                 |      Definitely |                 |
|                 |                 |      does not   |                 |
|                 |                 |      include    |                 |
|                 |                 |      /install   |                 |
|                 |                 |      directory. |                 |
|                 |                 |                 |                 |
|                 |                 |    - See also   |                 |
|                 |                 |      "informa-  |                 |
|                 |                 |      ion        |                 |
|                 |                 |      on xCAT    |                 |
|                 |                 |      high       |                 |
|                 |                 |      availabil- |                 |
|                 |                 |      ity"       |                 |
|                 |                 |      for other  |                 |
|                 |                 |      backup     |                 |
|                 |                 |      and        |                 |
|                 |                 |      storage    |                 |
|                 |                 |      considera- |                 |
|                 |                 |      tions.     |                 |
|                 |                 |                 |                 |
|                 |                 | -  Test.        |                 |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| High            | **3 or 1**      | Possible        | Possible        |
| availability -  |                 | roadmap:        | roadmap: `HA    |
| is this         |                 | `information on | case            |
| necessary?      |                 | xCAT high       | study <https:// |
|                 |                 | availability <h | theforeman.org/ |
| -  Production   |                 | ttp://xcat-docs | 2015/12/journey |
|    nodes should |                 | .readthedocs.io | _to_high_availa |
|    not depend   |                 | /en/stable/adva | bility.html>`   |
|    on           |                 | nced/hamn/index |                 |
|    deployment   |                 | .html>`         | -  A bit        |
|    and          |                 |                 |    complicated  |
|    management   |                 |                 |    though       |
|    infrastructu |                 |                 |    (involves    |
|                 |                 |                 |    memcache     |
|    (see         |                 |                 |    servers)?    |
|    discussion   |                 |                 |                 |
|    about        |                 |                 |                 |
|    Puppet,      |                 |                 |                 |
|    above).      |                 |                 |                 |
|                 |                 |                 |                 |
| -  But if we    |                 |                 |                 |
|    are running  |                 |                 |                 |
|    local DNS    |                 |                 |                 |
|    servers that |                 |                 |                 |
|    are tied to  |                 |                 |                 |
|    our          |                 |                 |                 |
|    management   |                 |                 |                 |
|    nodes then   |                 |                 |                 |
|    it matters.  |                 |                 |                 |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| Interface /     |                 |                 |                 |
| workflow /      |                 |                 |                 |
| ease of use     |                 |                 |                 |
|                 |                 |                 |                 |
|                 |                 |                 |                 |
|                 |                 |                 |                 |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| Reporting/centr | 1               | Yes. Adequate   | Yes. Also       |
| al              |                 | logging         | includes        |
| logging         |                 | including       | centralized     |
|                 |                 | console logs.   | reporting       |
| -  Note our     |                 |                 | console for     |
|    monitoring/  |                 |                 | Puppet.         |
|    logging      |                 |                 |                 |
|    stack should |                 |                 |                 |
|    take care of |                 |                 |                 |
|    this to a    |                 |                 |                 |
|    certain      |                 |                 |                 |
|    degree, but  |                 |                 |                 |
|    having it    |                 |                 |                 |
|    more         |                 |                 |                 |
|    integrated   |                 |                 |                 |
|    with the     |                 |                 |                 |
|    management/  |                 |                 |                 |
|    deployment   |                 |                 |                 |
|    system could |                 |                 |                 |
|    be nice.     |                 |                 |                 |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| Support for     | **3 or 2**      | No              | No              |
| change control: |                 | Git-integration | Git-integration |
| Git             |                 | by default, but | by default.     |
| integration,    |                 | we could easily | Custom          |
| rollback, and   |                 | customize.      | functionality   |
| auditing        |                 |                 | may be harder   |
| procedures      |                 | -  Use tabdump  | to implement    |
|                 |                 |    to import/   | and enforce.    |
| -  Not sure to  |                 |    export       |                 |
|    what degree  |                 |                 | -  Export       |
|    the project  |                 |    tables to    |    configs from |
|    will require |                 |    text and     |    DB to text   |
|    this.        |                 |    integrate w/ |    periodically |
|                 |                 |    Git          |    and import   |
|                 |                 |    workflow.    |    into Git?    |
|                 |                 |    Possibly     |                 |
|                 |                 |    build        | No built-in     |
|                 |                 |    wrapper      | undo.           |
|                 |                 |    commands to  |                 |
|                 |                 |    execute      | Has decent      |
|                 |                 |    changes to   | auditing of     |
|                 |                 |    tables.      | actions         |
|                 |                 |                 | performed via   |
|                 |                 | No built-in     | the Foreman     |
|                 |                 | undo.           | master (likely  |
|                 |                 |                 | includes CLI),  |
|                 |                 | Auditing may be | and may display |
|                 |                 | less than       | executing user  |
|                 |                 | desired since   | effectively     |
|                 |                 | we tend to do   | (esp. in web    |
|                 |                 | everything as   | UI; not sure    |
|                 |                 | root in xCAT.   | about CLI,      |
|                 |                 |                 | etc.)           |
|                 |                 |                 |                 |
|                 |                 |                 | -  Further      |
|                 |                 |                 |    evaluate     |
|                 |                 |                 |    auditing via |
|                 |                 |                 |    various      |
|                 |                 |                 |    interfaces?  |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| Overall ease of | 2               | -  CLI is very  | -  Evaluate     |
| use /           |                 |    responsive.  |    CLI, APIs,   |
| efficiency      |                 |                 |    etc.         |
|                 |                 | -  Table layout |                 |
|                 |                 |    takes some   | -  GUI has      |
|                 |                 |    time to      |    seemed       |
|                 |                 |    understand.  |    rather slow  |
|                 |                 |                 |    so far but   |
|                 |                 |                 |    perhaps it   |
|                 |                 |                 |    can be made  |
|                 |                 |                 |    more         |
|                 |                 |                 |    responsive.  |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| Specifically:   | 2               | -  We have this | -  Seems that   |
| ease of         |                 |    worked out   |    it would     |
| (re)deploying   |                 |    pretty well  |    require more |
| the OS on a     |                 |    for our      |    direct       |
| node            |                 |    nodes at     |    manipulation |
|                 |                 |    NPCF.        |    of kickstart |
| (incl. Puppet   |                 |                 |    files,       |
| ENC, NICs, disk |                 | -  xCAT-gen'ed  |    especially   |
| partitioning)   |                 |                 |    up front. It |
|                 |                 |    kickstart    |    doesn't      |
|                 |                 |    files suit   |    appear that  |
|                 |                 |    as fairly    |    Foreman      |
|                 |                 |    well for the |    gives you as |
|                 |                 |    most part.   |    much "for    |
|                 |                 |    Disk         |    free."       |
|                 |                 |    provisioning |                 |
|                 |                 |    is fairly    |                 |
|                 |                 |    smart.       |                 |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| Specifically:   | 1               | -  Evaluate     | -  Evaluate     |
| ease of         |                 |    node         |    node         |
| configuring new |                 |    discovery.   |    discovery.   |
| hardware (i.e., |                 |                 |                 |
| modifying BIOS  |                 | -  Can we       | -  Can we       |
| settings, other |                 |    install Dell |    install      |
| firmware,       |                 |    firmware via |    firmware in  |
| possibly        |                 |    Genesis boot |    the          |
| "discovery"     |                 |    as well?     |    discovery    |
| process)        |                 |                 |    environment  |
|                 |                 |                 |    or via some  |
| -  We don't     |                 |                 |    PXE image?   |
|    install new  |                 |                 |                 |
|    hardware all |                 |                 |                 |
|    that often.  |                 |                 |                 |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| Command-line    | 3               | Extensive and   | -  Investigate  |
| interface (and  |                 | fairly well     |    and evaluate |
| other           |                 | developed CLI.  |    CLI.         |
| scriptable      |                 |                 |                 |
| APIs)           |                 |                 | -  Evaluate     |
|                 |                 |                 |    other        |
| -  Automation   |                 |                 |    API(s).      |
|    and          |                 |                 |                 |
|    integration  |                 |                 |                 |
|    depends on a |                 |                 |                 |
|    CLI or some  |                 |                 |                 |
|    kind of API. |                 |                 |                 |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| GUI admin       | 1               | No...           | Yes.            |
| console         |                 |                 |                 |
|                 |                 | -  Well, maybe  | -  Has LDAP     |
|                 |                 |    Confluent.   |    integration. |
|                 |                 |                 |                 |
|                 |                 |                 |    - Can we     |
|                 |                 |                 |      secure     |
|                 |                 |                 |      with       |
|                 |                 |                 |      two-factor |
|                 |                 |                 |      or just    |
|                 |                 |                 |      require    |
|                 |                 |                 |      SSH with   |
|                 |                 |                 |      X11        |
|                 |                 |                 |      forwarding |
|                 |                 |                 |      from a     |
|                 |                 |                 |      bastion    |
|                 |                 |                 |      node?      |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| Granular        | **3 or 1**      | Not built in.   | Yes, but need   |
| permissions     |                 |                 | to evaluate     |
| (levels of      |                 | -  Create       | further if this |
| access, buckets |                 |    custom       | is important.   |
| of resources)   |                 |    script to    |                 |
|                 |                 |    view Puppet  | -  Evaluate     |
| - Not sure      |                 |    ENC? Or      |    further?     |
|   about         |                 |    build view   |                 |
|   project       |                 |    of           |                 |
|   requirements  |                 |    high-level   |                 |
|   around this.  |                 |    config into  |                 |
|                 |                 |    Puppet       |                 |
|                 |                 |    monitoring   |                 |
| - Regardless,   |                 |    stack?       |                 |
|   it might be   |                 |                 |                 |
|   nice to       |                 |                 |                 |
|   allow         |                 |                 |                 |
|   certain       |                 |                 |                 |
|   non-admins    |                 |                 |                 |
|   the ability   |                 |                 |                 |
|   to view the   |                 |                 |                 |
|   high-level    |                 |                 |                 |
|   configuratio  |                 |                 |                 |
|   (e.g.,        |                 |                 |                 |
|   Puppet        |                 |                 |                 |
|   role/site/    |                 |                 |                 |
|   datacenter/   |                 |                 |                 |
|   custer)       |                 |                 |                 |
|   of some/all   |                 |                 |                 |
|   nodes.        |                 |                 |                 |
|                 |                 |                 |                 |
| - Our           |                 |                 |                 |
|   monitoring    |                 |                 |                 |
|   stack might   |                 |                 |                 |
|   be able to    |                 |                 |                 |
|   provide some  |                 |                 |                 |
|   visibility    |                 |                 |                 |
|   into          |                 |                 |                 |
|   high-level    |                 |                 |                 |
|   configuration |                 |                 |                 |
|   as well.      |                 |                 |                 |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| Specifically:   | **3 or 1**      | Not built in.   | Seems to be     |
| Allow           |                 |                 | built in.       |
| developers to   |                 | -  Create       |                 |
| reprovision     |                 |    limited but  | -  Evaluate     |
| specific groups |                 |    privileged   |    further?     |
| of machines     |                 |    rebuild      |                 |
|                 |                 |    scripts for  |                 |
| -  Jim Parsons  |                 |    specific     |                 |
|    is           |                 |    groups       |                 |
|    interested   |                 |    and/or       |                 |
|    in this; not |                 |    targeted     |                 |
|    sure it's an |                 |    sudo config? |                 |
|    actual       |                 |                 |                 |
|    requirement  |                 |                 |                 |
|    nor how      |                 |                 |                 |
|    often it     |                 |                 |                 |
|    would be     |                 |                 |                 |
|    used.        |                 |                 |                 |
|                 |                 |                 |                 |
| -  Do we simply |                 |                 |                 |
|    provide a    |                 |                 |                 |
|    pool of      |                 |                 |                 |
|    development  |                 |                 |                 |
|    machines w/  |                 |                 |                 |
|    separate     |                 |                 |                 |
|    deployment   |                 |                 |                 |
|    management   |                 |                 |                 |
|    infrastructu |                 |                 |                 |
|    to which     |                 |                 |                 |
|    (certain)    |                 |                 |                 |
|    developers   |                 |                 |                 |
|    have         |                 |                 |                 |
|    more/full    |                 |                 |                 |
|    access?      |                 |                 |                 |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| Notifications   | 1               | No, does not    | Yes, seem to be |
|                 |                 | seem to be      | built in.       |
| -  Monitoring   |                 | built in.       |                 |
|    stack should |                 |                 | -  Evaluate     |
|    take care of |                 |                 |    further?     |
|    this.        |                 |                 |                 |
|                 |                 |                 | -  Bill uses    |
| -  What kind(s) |                 |                 |    Foreman for  |
|    would we     |                 |                 |    ITS and      |
|    want?        |                 |                 |    remarks that |
|                 |                 |                 |    Puppet       |
|                 |                 |                 |    report       |
|                 |                 |                 |    notification |
|                 |                 |                 |    emails are   |
|                 |                 |                 |    only sent to |
|                 |                 |                 |    the "owner"  |
|                 |                 |                 |    of the       |
|                 |                 |                 |    machine.     |
|                 |                 |                 |                 |
+-----------------+-----------------+-----------------+-----------------+
| Documentation   | 2               | xCAT            | Foreman         |
| and support     |                 | documentation   | documentation   |
|                 |                 | is decent (both | is decent (it   |
|                 |                 | comprehensive   | is a really big |
|                 |                 | and specific,   | product and the |
|                 |                 | although there  | documentation   |
|                 |                 | seem to be      | sometimes lacks |
|                 |                 | quite a few new | specificity     |
|                 |                 | features that   | and/or concrete |
|                 |                 | are not yet     | examples).      |
|                 |                 | documented).    |                 |
|                 |                 |                 | foreman-users   |
|                 |                 | xcat-user list  | Google group    |
|                 |                 | on SourceForge  | had about 2.5   |
|                 |                 | has been        | times more      |
|                 |                 | reasonably      | messages than   |
|                 |                 | useful.         | xcat-user list  |
|                 |                 |                 | in a            |
|                 |                 | Current vendor  | representative  |
|                 |                 | relationship    | time frame (the |
|                 |                 | with            | Google group is |
|                 |                 | NETSource/Lenov | now defunct;    |
|                 |                 | o               | use .           |
|                 |                 | allows us       |                 |
|                 |                 | somewhat        | Using RedHat    |
|                 |                 | privileged      | Satellite       |
|                 |                 | access to xCAT  | (Foreman +      |
|                 |                 | team.           | Katello & more) |
|                 |                 |                 | might get us    |
|                 |                 | NCSA is already | support but     |
|                 |                 | using xCAT for  | would almost    |
|                 |                 | Systems         | certainly       |
|                 |                 | (Industry and   | require using   |
|                 |                 | ICC in addition | RHEL and would  |
|                 |                 | to LSST) and    | almost          |
|                 |                 | has a few team  | certainly       |
|                 |                 | members with    | require         |
|                 |                 | extensive       | additional      |
|                 |                 | experience with | cost.           |
|                 |                 | xCAT.           |                 |
|                 |                 |                 | NCSA is already |
|                 |                 |                 | using Foreman   |
|                 |                 |                 | for ITS (basic  |
|                 |                 |                 | UI/Puppet       |
|                 |                 |                 | reports & ENC   |
|                 |                 |                 | only, so far)   |
|                 |                 |                 | and Security    |
|                 |                 |                 | (more extensive |
|                 |                 |                 | use, including  |
|                 |                 |                 | Katello).       |
|                 |                 |                 | Security's      |
|                 |                 |                 | person with     |
|                 |                 |                 | most experience |
|                 |                 |                 | recently left.  |
+-----------------+-----------------+-----------------+-----------------+

Summary Evaluation
------------------

Both products—xCAT and Foreman—or a combination of these products would
seem to meet our needs at a fundamental level. In any case we'd be using
the product(s) for IPMI functions, (possibly) bare-metal discovery /
VMware provisioning, and PXE-boot OS installs with as minimal a
configuration as possible with Puppet handling as much of the
configuration as possible.

Foreman is a newer tool but seems to have broader functionality and
appears to have a larger user community. It also appears to be a more
complex tool, which could lead to greater management overhead.

Foreman also appears to offer better out-of-the-box support for a
distributed architecture with centralized control and secure
communication between the deployment servers. On the other hand,
pursuing a more centralized point of control would likely push us more
strongly towards high availability of the central/master resources,
which could introduce even more complexity/management overhead.

The actual design and implementation of our solution, or future shifts
in our design/implementation, may be influenced by a few outstanding
questions about project requirements and architecture (e.g., will we
need to support stateless nodes? will we need to manage DNS with our
solution? will we need to offer role-based access to admins or the
capability for non-admins to view/update configuration? will we need to
support cloud resources?)

Addendum 1: Possible end states
-------------------------------

(1) Use current NPCF model (xCAT for deployment and IPMI functions,
Puppet for configuration management, Pakrat for Yum repo management, new
monitoring stack, possibly Confluent for IPMI functions)

(2) Same + use Foreman for Puppet integration (ENC, reporting,
certificates) alongside xCAT, etc.

-  It may not be possible for xCAT and Foreman Master to live on the
   same server. By default a Foreman Master includes TFTP server by
   default as does xCAT and their settings according to
   /etc/xinitd.d/tftp seem to conflict. We could ask online to see if it
   is possible to install a Foreman Master without TFTP. Also see
   Foreman Manual for \ `customization of
   TFTP <https://www.theforeman.org/manuals/1.16/index.html#4.3.9TFTP>`.

-  If pursuing (2) it might make sense to have general
   admin/xCAT/IPMI/bastion functions on one node and Foreman/Puppet (CA,
   Master, ENC, reporting)/GitLab on another node.

-  Our GitLab on lsst-adm01 uses PostgreSQL as does Foreman (by
   default). Handle Foreman + GitLab with care.

(3) Same + use Foreman for node deployment (DHCP/PXE, kickstart,
possibly DNS) instead of xCAT (still use xCAT/Confluent for IPMI
functions, Pakrat for Yum repo management).

(4) Same + use Foreman BMC Smart Proxy for IPMI functions (still use
Pakrat for Yum repo management)

NOTE: (2), (3), and (4) also offer the possibility of using Katello for
Git/Puppet branch management and/or Yum repo management.

-  We could also look at using Katello components (esp. Pulp) directly
   w/ (1), (2), (3), or (4).

Addendum 2: Other considerations for making a decision
------------------------------------------------------

-  We would save some time up front by going with (1) because we're
   basically already there with NPCF.

   -  There are quite a few improvements we should make, however.

   -  And we should rebuild our current xCAT/Puppet master/management
      node (lsst-adm01) at some point. Do we want to rebuild
      more-or-less as-is or rebuild with Foreman, whether (2), (3), or
      (4)?

-  By sticking with (1), merging NCSA 3003 into a shared environment can
   be a stronger focus more immediately (and there are many benefits to
   getting this done sooner rather than later).

   -  Standing up another xCAT master for NCSA 3003 would take very
      little time and would offer a good opportunity for refining our
      backup/rebuild procedures for our xCAT master at NPCF.

-  (2), (3), and (4) could be pursued later on (with more awareness of
   both project requirements and of Foreman) and also pursued
   incrementally, e.g., (1)->(2)->(3)->(4)->....

