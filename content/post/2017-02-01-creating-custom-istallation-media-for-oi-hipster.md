+++
date = "2017-02-01T00:00:00-08:00"
title = "Creating Custom Installation Media for OI Hipster"
draft = false
+++

## Preface

This post is a write up of my notes for creating custom installation
media for [OpenIndiana Hipster][oi-hipster], using a custom/patched
version of [illumos][illumos]. It assumes that OI Hipster has already
been installed on a machine (e.g. installed on a VM using their
[provided][oi-download] installation media); and this server will be
used to build our custom version of illumos, as well as the custom OI
installation media. The goal of this exercise is to create a "Live DVD"
that can be used to install our custom version of illumos.

## Step 1: Fetch illumos

The illumos repository [is on GitHub][illumos-gh], so we'll use this to
obtain our copy. First, we must install `git` on the server:

    $ sudo pkg install git

and then we can use `git clone` to fetch the code:

    $ git clone https://github.com/illumos/illumos-gate.git

## Step 2: Prep for Building illumos

Now we can move on to compiling the illumos codebase; we'll use the
[documented process][illumos-how-to-build] as a guide. We need to
download and install all the required build tools (e.g. compilers, etc),
which can can be easily done using the `build-essentials` meta-package:

    $ sudo pkg install build-essential

### Step 2.1: Download and Unpack Closed Binaries

After that completes, we have to download and unpack the "closed
binaries"; these are closed source components from Sun/Oracle that are
required to build illumos (and have not yet been replaced):

    $ cd illumos-gate/
    $ wget -c \
        https://download.joyent.com/pub/build/illumos/on-closed-bins.i386.tar.bz2 \
        https://download.joyent.com/pub/build/illumos/on-closed-bins-nd.i386.tar.bz2
    $ tar xjvpf on-closed-bins.i386.tar.bz2
    $ tar xjvpf on-closed-bins-nd.i386.tar.bz2


### Step 2.2: Configure Environment File -- illumos.sh

The illumos build system is configured using an "environment" file. This
file contains various environment variables that the build uses to
determine what actions it will take; for example:

  - Should it run lint checks?
  - Should it perform a "debug" or "non-debug" build? Or both?
  - etc...

For our purposes, we'll leave all of the default values, and only make
the necessary modifications as specified in the previously referenced,
["How To Build" document][illumos-how-to-build]:

    $ cp usr/src/tools/env/illumos.sh .
    $ echo 'export PKGVERS_BRANCH=2099.0.0.0' >> illumos.sh
    $ echo 'export PERL_VERSION="5.22"' >> illumos.sh
    $ echo 'export PERL_PKGVERS="-522"' >> illumos.sh

## Step 3: Build illumos

Now we're ready to start the build. This is expected to take a couple
hours to complete, so I'd recommend using a program such as `screen` or
`tmux`. That way, if the connection with the server performing the build
were to fail, the build process will remain running. For this guide I'll
use `screen`, since it's already installed:

    $ screen -S illumos-build
    $ cp usr/src/tools/scripts/nightly.sh .
    $ chmod +x nightly.sh
    $ time ./nightly.sh illumos.sh

    real    123m31.031s
    user    178m14.443s
    sys     42m54.293s

## Step 4: Prep for Building the Installation Media

In order to build the installation media, we need to use the
`distro_const` command, which is provided by the
`distribution-constructor` package:

    $ sudo pkg install distribution-constructor

To use `distro_const`, we must pass it a "manifest" file, which is
nothing more than a specially formatted XML file. We'll start with a
manifest template provided by the package, and then tweak the template
to suite our needs.

### Step 4.1: Copy distro_const Manifest Template

First, we need to copy the template so we can make modifications to it:

    $ cd ~
    $ cp /usr/share/distro_const/text_install/text_mode_x86.xml .

### Step 4.1: Replace "dev" IPS Repository with "hipster"

The first modification we need to make, is to point to the "hipster"
[IPS][ips] repository as opposed to the default "dev" repository. To do
this, we need to locate the `pkg_repo_default_authority` and
`post_install_repo_default_authority` sections, and replace
`http://pkg.openindiana.org/dev` with
`http://pkg.openindiana.org/hipster`. Here's `diff` of this change:

    $ diff -u text_mode_x86.xml text_mode_x86.xml.orig
    --- text_mode_x86.xml.orig      Thu Feb  2 02:09:32 2017
    +++ text_mode_x86.xml           Thu Feb  2 02:10:33 2017
    @@ -192,7 +192,7 @@
                    -->
                    <pkg_repo_default_authority>
                            <main
    -                               url="http://pkg.openindiana.org/dev"
    +                               url="http://pkg.openindiana.org/hipster"
                                    authname="openindiana.org"/>
                            <!--
                                 If you want to use one or more  mirrors that are
    @@ -229,7 +229,7 @@
                    -->
                    <post_install_repo_default_authority>
                            <main
    -                               url="http://pkg.openindiana.org/dev"
    +                               url="http://pkg.openindiana.org/hipster"
                                    authname="openindiana.org"/>
                            <!--
                                 Uncomment before using.

### Step 4.2: Point to IPS Repository from Custom illumos Build

As part of illumos build in [step 3](#step-3-build-illumos), an IPS
repository was generated under the `packages/i386/nightly/repo.redist`
directory. Since my illumos codebase is stored in the directory,
`/export/home/ps/illumos-gate`, the full path to this IPS repository is
`/export/home/ps/illumos-gate/packages/i386/nightly/repo.redist`.

In order for the installation media to use my custom built version of
illumos, the manifest needs to be modified to point to this IPS
repository, **in addition to** the default OI "hipster" repository. It's
important to note, that we're not replacing the "hipster" repository
with the custom illumos build's repository; rather, we want to use the
build's repository as the "main" repository, and the "hipster"
repository as an "additional" repository.

To do this, we'll modify the `url` specified in the
`pkg_repo_default_authority` section to point to the illumos build's
repository, using a `file://` based URL, and we'll modify the `url` of
the `pkg_repo_addl_authority` section to point to the original "hipster"
IPS repository. Here's a diff of these changes:

    $ diff -u text_mode_x86.xml.orig text_mode_x86.xml
    --- text_mode_x86.xml.orig      Thu Feb  2 02:18:19 2017
    +++ text_mode_x86.xml           Thu Feb  2 02:19:45 2017
    @@ -192,8 +192,8 @@
                    -->
                    <pkg_repo_default_authority>
                            <main
    -                               url="http://pkg.openindiana.org/hipster"
    -                               authname="openindiana.org"/>
    +                               url="file:///export/home/ps/illumos-gate/packages/i386/nightly/repo.redist"
    +                               authname="on-nightly"/>
                            <!--
                                 If you want to use one or more  mirrors that are
                                 setup for the authority, specify the urls here.
    @@ -210,15 +210,12 @@
                         If you want to use one or more  mirrors that are
                         setup for the authority, specify the urls here.
                    -->
    -               <!--
    -                    Uncomment before using.
                    <pkg_repo_addl_authority>
                            <main
    -                               url=""
    -                               authname=""/>
    +                               url="http://pkg.openindiana.org/hipster"
    +                               authname="openindiana.org"/>
                            <mirror url="" />
                    </pkg_repo_addl_authority>
    -               -->
                    <!--
                         The default preferred authority to be used by the system
                         after it has been installed.

### Step 4.3: Use OI Repository for "ssh" and "ftp" Packages

I don't fully understand why this step is necessary, but we need to
modify the manifest to force the use of the OpenIndiana's IPS repository
when fetching the following packages:

  - `pkg:/network/ssh`
  - `pkg:/service/network/ftp`
  - `pkg:/service/network/ssh`

This is done by adding the package URL for these packages to the
`packages` section of the manifest; the following `diff` does this:

    $ diff -u text_mode_x86.xml.orig text_mode_x86.xml
    --- text_mode_x86.xml.orig      Thu Feb  2 03:38:37 2017
    +++ text_mode_x86.xml           Thu Feb  2 03:56:46 2017
    @@ -266,6 +266,9 @@
                            <pkg name="pkg:/server_install"/>
                            <pkg name="pkg:/system/install/text-install"/>
                            <pkg name="pkg:/system/install/media/internal"/>
    +                       <pkg name="pkg://openindiana.org/network/ssh"/>
    +                       <pkg name="pkg://openindiana.org/service/network/ftp"/>
    +                       <pkg name="pkg://openindiana.org/service/network/ssh"/>
                    </packages>
     <!--
          Items below this line are rarely configured

### Step 4.4: Remove Certain Packages from illumos Build Repository

I understand even less about why this step is necessary, but we need to
remove certain packages from the IPS repository generated by the
previous illumos build (the build from [step 3](#step-3-build-illumos)),
otherwise the [next step](#step-5-building-the-installation-media) will
fail to build the installation media. The packages we need to remove
are:

  - `pkg:/network/ssh`
  - `pkg:/service/network/ssh`
  - `pkg:/service/network/ftp`
  - `pkg:/service/network/ssh-common`
  - `pkg:/service/network/smtp/sendmail`

We can use `pkgrepo` to remove them like so:

    $ REPO="/export/home/ps/illumos-gate/packages/i386/nightly/repo.redist"
    $ for pkg in "pkg:/network/ssh" \
    >            "pkg:/service/network/ssh" \
    >            "pkg:/service/network/ftp" \
    >            "pkg:/service/network/ssh-common" \
    >            "pkg:/service/network/smtp/sendmail"; do
    >     pkgrepo -s "$REPO" remove "$pkg"
    > done

## Step 5: Building the Installation Media

Now, with all of the above steps completed, we're ready to use
`distro_const` to build the installation media files; this part is
simple:

    $ sudo distro_const build text_mode_x86.xml
    ... <snip> ...
    Build completed Thu Feb  2 05:26:06 2017
    Build is successful.

After that completes, the installation media should be found in
`/rpool/dc/media`:

    $ ls -lh /rpool/dc/media/
    total 2793956
    -rw-r--r--   1 root     root        657M Feb  2 05:25 OpenIndiana_Text_X86.iso
    -r--r--r--   1 root     root        793M Feb  2 05:25 OpenIndiana_Text_X86.usb

Success!

Those files can now be used to install our custom OpenIndiana Hipster
distribution onto any machine of our choosing (e.g. a VM, bare metal,
etc), and the installed operating system will be our custom version of
illumos, plus all additional packages packages that constitute OI
Hipster.

[oi-hipster]: https://www.openindiana.org/overview/the-hipster-branch/
[oi-download]: https://www.openindiana.org/download/
[illumos]: https://illumos.org
[illumos-gh]: https://github.com/illumos/illumos-gate
[illumos-how-to-build]: https://wiki.illumos.org/display/illumos/How+To+Build+illumos
[ips]: https://en.wikipedia.org/wiki/Image_Packaging_System
