+++
date = "2017-09-18T00:00:00-08:00"
lastmod = "2017-09-18T00:00:00-08:00"
title = "Code Coverage for ZFS on Linux"
draft = false
+++

I've been working with Brian Behlendorf on getting code coverage
information for the ZFS on Linux. The goal was to get code coverage data
for pull requests, as well as branches; this way, we can get a sense of
how well tested any given PR is by the automated tests, prior to landing
it. There's still some wrinkles that need to be ironed out, but we've
mostly achieved that goal by leveraging [codecov.io][codecov], along
with a small [bash script][script]. Here's an [example][example].

While what we've currently implemented is better than nothing, there's
some potential improvements that'd I'd like to investigate when I have
time (and this post serves as a way to help me remember what they are):

  1. Currently in the automated tests that are run for the ZFS on Linux
     project, the kernel's code coverage data isn't reset prior to
     running any tests. This is because in my testing, a hang would
     occur whenever I would write to the kernel's "reset" file. For
     example, I would run this command:

         # echo '' > /sys/kernel/debug/gcov/reset

    and it would never return; it would have the following backtrace:

        # cat /proc/1624/stack
        [<ffffffffb736f501>] __synchronize_srcu+0xe1/0x100
        [<ffffffffb736f5eb>] synchronize_srcu+0xcb/0x1c0
        [<ffffffffb7749390>] debugfs_remove+0xa0/0xe0
        [<ffffffffb73e5f82>] release_node+0x62/0x140
        [<ffffffffb73e613b>] reset_write+0xdb/0x120
        [<ffffffffb774a3e4>] full_proxy_write+0x84/0xd0
        [<ffffffffb758a90f>] __vfs_write+0x3f/0x230
        [<ffffffffb758e38e>] vfs_write+0xfe/0x270
        [<ffffffffb758e889>] SyS_write+0x79/0x130
        [<ffffffffb7f8083b>] entry_SYSCALL_64_fastpath+0x1e/0xa9
        [<ffffffffffffffff>] 0xffffffffffffffff

    Unfortunately, I never got around to doing any root cause analysis
    of the issue. Instead, we simply aren't resetting the kernel's
    coverage data, which is fine because we only upload a single unified
    report to Codecov (which contains the data for all tests in the
    automated test suite).

    Also, it's worth noting that while writing this post, I attempted
    that same command to clear the kernel's coverage data, and I was
    unable to reproduce the hang. I'm running a different kernel now, so
    perhaps the previous behavior was a bug in the kernel that's been
    fixed in a later release? Without an RCA, anything is possible.

  2. We're not collecting any data for the userspace libraries, such as
     `libzpool`; which unfortunately means we don't get much coverage
     data from runs of `ztest` and `zloop`. The reason behind this is
     simply due to the additional implementation complexity required to
     support the libraries:

       - Some of the libraries are composed of the same source files as
         the kernel modules. Thus, we'd need to be careful when
         generating the gcov reports if we included these libaries, such
         that the kernel report for a given source file isn't replaced
         by the userspace report (or vice versa). If we happened to be
         careless, it'd be easy to generate the kernel report for a
         source file, and then accidentally overwrite that report with
         the userspace data when generating the userspace report.

       - The `.gcda` files for the libraries often are found in a
         `.libs` sub-directory, rather than directly in the same
         directory as the source code (like non-libary sources). For
         example, the `.gcda` files for `libzpool` are here:

             $ find lib/libzpool -name '*.gcda' | head -n 5
             lib/libzpool/.libs/zfs_rlock.gcda
             lib/libzpool/.libs/zpool_prop.gcda
             lib/libzpool/.libs/vdev_raidz.gcda
             lib/libzpool/.libs/dsl_pool.gcda

         This is in contrast to where these files are stored for the
         commands:

             $ find cmd -name '*.gcda' | head -n 5
             cmd/zfs/zfs_iter.gcda
             cmd/zfs/zfs_main.gcda
             cmd/ztest/ztest.gcda
             cmd/mount_zfs/mount_zfs.gcda
             cmd/zpool/zpool_iter.gcda

         As a result, we have to use the `-o` option when running `gcov`
         for the libraries, but not for the commands:

             $ cd ~/zfs/lib/libzpool
             $ gcov -o .libs arc.gcno
             File '../../module/zfs/arc.c'
             Lines executed:60.72% of 3055
             Creating 'arc.c.gcov'

       - The gcov data files reference their corresponding source files
         using relative paths to parts of the project's repository. This
         happens when a library references one of the source files from
         the `module` directory (i.e. when file is built both for kernel
         mode and user mode), and is a result of how the build system
         works for these libraries.

         Thus, if `gcov` is run from the "root" of the project
         directory, it will be unable to find the source files for that
         library. For example:

             $ gcov -b lib/libzpool/arc.gcno
             ...
             Cannot open source file ../../module/zfs/arc.c

         This can be relatively easily worked around by first `cd`-ing
         into the directory that contains the `.gcno` file, and then
         running `gcov`, but that idea falls apart for the `libicp`
         library.

         For this libary, one must execute `gcov` from the `lib/libicp`
         directory, but the `.gcno` files are stored in subdirectories
         from there. For example, this fails:

             $ cd ~/zfs/lib/libicp/algs/sha2
             $ gcov -o .libs sha2.gcno
             File '../../module/icp/algs/sha2/sha2.c'
             Lines executed:67.41% of 135
             Creating 'sha2.c.gcov'
             Cannot open source file ../../module/icp/algs/sha2/sha2.c

         But this works:

             $ cd ~/zfs/lib/libicp
             $ gcov -o algs/sha2/.libs algs/sha2/sha2.gcno
             File '../../module/icp/algs/sha2/sha2.c'
             Lines executed:67.41% of 135
             Creating 'sha2.c.gcov'

       - The last complication to consider is that, since we can have
         two different modes of execution for any given source file
         (since we compile certain files in both kernel and user mode),
         it'd be ideal to have seperate coverage reports for kernel and
         user mode execution. Codecov appears to support this via its
         concept of "flags" (i.e. using the `-F` option of their [Bash
         uploader][uploader]), but that functionality still needs to be
         evaluated to make sure it'll work for our needs.

All in all, coming up with solutions for any (or all) of these
complications is possible, it just requires a little more work to be
done. For the first attempt at collecting coverage data, I opted to keep
the implementation simple yet functional and useful. I hope to revisit
these issues, and extend the coverage data to include the libraries
soon.

[codecov]: https://codecov.io/
[script]: https://github.com/zfsonlinux/zfs-buildbot/blob/master/scripts/bb-test-cleanup.sh
[example]: https://codecov.io/gh/zfsonlinux/zfs/pull/6566/diff
[uploader]: https://docs.codecov.io/v4.3.6/docs/about-the-codecov-bash-uploader
