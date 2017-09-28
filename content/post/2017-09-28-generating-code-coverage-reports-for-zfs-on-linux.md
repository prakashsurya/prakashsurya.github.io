+++
date = "2017-09-28T00:00:00-08:00"
lastmod = "2017-09-28T00:00:00-08:00"
title = "Generating Code Coverage Reports for ZFS on Linux"
draft = false
+++

## Introduction

This is another post about collecting code coverage data for the ZFS on
Linux project. We've recently added a new make target to the project, so
I wanted to highlight how easy it is to use this to generate static HTML
based code coverage reports, and/or to generate a report that can be
used with other tools and services (e.g. [codecov.io][codecov-zfs]).

## Examples

Before I get into the specifics for how to run the tests and generate
the coverage report, I want to show off the results. After the tests
have been run, one can use `make code-coverage-capture` to generate the
code coverage information.

This generates a directory full of static HTML pages, and a `.info`
file; both of which describe the lines executed for the set of tests
that were run. The HTML pages are to be consumed by humans, and the
`.info` file consumed by tools.

 - See [here](zfs-0.7.0-coverage/index.html) to view an example of the
   static HTML pages.
 - See [here](zfs-0.7.0-coverage.info) to view/download an example of
   the `.info` file.

I'll go into the details below, but that coverage was generated after
running `zloop` and ZTS, and only contains data for user execution.
Generating this data for kernel execution isn't difficult, it just
requires a little more work, and is out of scope for this post (I have
some notes on this at the end of this post).

## Details

With the end result in mind, lets now dive into the details of how I
generated those reports.

As one might expect, we first need to compile the ZFS project, and
ensure it's configured to correctly generate the needed code coverage
data. We do this via the new `--enable-code-coverage` flag to
`configure`:

    $ ./autogen.sh && ./configure --enable-code-coverage && make

Now, before we jump into running the tests, we need to run the following
commands to setup our environment:

    $ ./scripts/zfs-tests.sh -c         # create and populate constrained PATH
    $ sudo ./scripts/zfs-helpers.sh -vi # create various symlinks in "/"
    $ sudo ./scripts/zfs.sh -u          # unload kernel modules
    $ sudo ./scripts/zfs.sh             # load kernel modules

We need to run the tests completely in-tree, so that the `.gcda` files
will be created within the ZFS repository tree. This enables the make
target to find these files, which it needs to generate the coverage
reports. Thus, the above commands are needed to configure the system to
enable running the tests in-tree vs. installing the various libraries,
commands, and kernel modules into the root filesystem.

With that done, we can go ahead and run our first test. This test will
run `ztest` in a loop for an hour:

    $ sudo ./scripts/zloop.sh -t $((60*60))

After that completes, we can run ZTS (ZFS Test Suite):

    $ ./scripts/zfs-tests.sh -vx

And finally, now we can generate the code coverage data for the tests
that we ran above:

    $ export CODE_COVERAGE_BRANCH_COVERAGE=1
    $ make code-coverage-capture
      LCOV   --capture zfs-0.7.0-coverage.info
      LCOV   --remove /tmp/*
    lcov: WARNING: negative counts found in tracefile zfs-0.7.0-coverage.info.tmp
      GEN    zfs-0.7.0-coverage
    file:///export/home/delphix/zfs/zfs-0.7.0-coverage/index.html

That's it!

After running that `make` command it should emit a `.info` file, and a
directory full of static HTML pages; much like the ones that I linked to
in the "Examples" section earlier in this post.

## More Details

If you've read this far, you might be interested in the following
details as well.

### Branch Coverage

We set the `CODE_COVERAGE_BRANCH_COVERAGE` environment variable prior to
running `make code-coverage-capture`, so that we'll get coverage data
for which branches of a conditional are executed, and which branches are
not. This allows us to see when a conditional is executed, but only a
subset of all possibilities of the branch are covered.

By default, `lcov` (which is used by the make target) has this feature
disabled, so branch coverage will not be collected without setting this
variable.

### Kernel Coverage

I mentioned this before, but it's worth pointing out again. The
instructions in this post will only produce code coverage data for user
execution. Generating data for kernel execution isn't much more
difficult than what's described above, but it requires a couple of
additional things.

 1. The kernel must be compiled with `gcov` support. If your kernel
    isn't already compiled with this support, than you'll have to build
    a new kernel that's configured to expose the required code coverage
    data. I wrote some notes on this [here][kernel-notes]. A more
    authoritative source can be found [here][kernel-build] and
    [here][kernel-gcov].

 2. Once running a kernel that supports exporting gcov data, the
    coverage data for the kernel modules need to be copied into the zfs
    tree prior to generating the reports (i.e. prior to running `make
    code-coverage-capture`). The following small script is one way this
    can be done:

        # Allow access to gcov files as a non-root user
        $ sudo chmod -R a+rx /sys/kernel/debug/gcov
        $ sudo chmod a+rx /sys/kernel/debug

        $ GCOV_KERNEL="/sys/kernel/debug/gcov"
        $ ZFS_BUILD="$(readlink -f ~/zfs)"
        $ pushd "$GCOV_KERNEL$ZFS_BUILD" >/dev/null
        $ find . -name "*.gcda" -exec sh -c 'cp -v $0 '$ZFS_BUILD'/$0' {} \;
        $ find . -name "*.gcno" -exec sh -c 'cp -vdn $0 '$ZFS_BUILD'/$0' {} \;
        $ popd >/dev/null

    The idea here, is to copy the kernel module's `.gcda` and `.gcno`
    files out of `/sys/kernel/debug/gcov` and place them along side the
    `.c` file they correspond to. This way, these files can be found by
    `make code-coverage-capture` just like the user execution files. See
    [here][buildbot-script] for an example of how we're doing this in
    the ZFS on Linux project's automation.

### Commit Adding New Make Target

The new `code-coverage-capture` make target was added in this commit to
the ZFS on Linux project:

    commit acf044420b134b022da5c866b19df69934ad3778
    Author: Prakash Surya <prakash.surya@delphix.com>
    Date:   Fri Sep 22 18:49:57 2017 -0700

        Add support for "--enable-code-coverage" option

        This change adds support for a new option that can be passed to the
        configure script: "--enable-code-coverage". Further, the "--enable-gcov"
        option has been removed, as this new option provides the same
        functionality (plus more).

        When using this new option the following make targets are available:

         * check-code-coverage
         * code-coverage-capture
         * code-coverage-clean

        Note: these make targets can only be run from the root of the project.

        Reviewed-by: Brian Behlendorf <behlendorf1@llnl.gov>
        Signed-off-by: Prakash Surya <prakash.surya@delphix.com>
        Closes #6670

Thus, in order to use this new make target, one will need to use a
version of the project that contains this commit.

### ZFS Commit Tested

For the testing and coverage data I showed in the "Examples" section, I
used the following commit from the ZFS on Linux's "master" branch:

    commit 269db7a4b3ef2bc14f3c2cf95f050479cbd69e72
    Author: Simon Guest <simon.guest@tesujimath.org>
    Date:   Thu Sep 28 06:39:47 2017 +1300

        vdev_id: extension for new scsi topology

        On systems with SCSI rather than SAS disk topology, this change enables
        the vdev_id script to match against the block device path, and therefore
        create a vdev alias in /dev/disk/by-vdev.

        Reviewed-by: Tony Hutter <hutter2@llnl.gov>
        Reviewed-by: Brian Behlendorf <behlendorf1@llnl.gov>
        Signed-off-by: Simon Guest <simon.guest@tesujimath.org>
        Closes #6592

[codecov-zfs]: https://codecov.io/gh/zfsonlinux/zfs
[kernel-notes]: http://localhost:1313/post/2017-09-11-using-gcov-with-zfs-on-linux-kernel-modules/
[kernel-build]: https://kernelnewbies.org/KernelBuild
[kernel-gcov]: https://www.kernel.org/doc/html/latest/dev-tools/gcov.html
[buildbot-script]: https://github.com/prakashsurya/zfs-buildbot/blob/master/scripts/bb-test-cleanup.sh
