+++
date = "2017-09-11T00:00:00-08:00"
lastmod = "2017-09-11T00:00:00-08:00"
title = "Using \"gcov\" with ZFS on Linux Kernel Modules"
draft = false
+++

## Building a "gcov" Enabled Linux Kernel

In order to extract "gcov" data from the Linux kernel, and/or Linux
kernel modules, a "gcov" enabled Linux kernel is needed. Since my
current development environment is based on Ubuntu 17.04, and the fact
that Ubuntu doesn't provide a pre-built kernel with "gcov" enabled, I
had to build the kernel from source. This was actually pretty simple,
and most of that process is already documented [here][kernelbuild].

### Install the Kernel Build Dependencies

    $ sudo apt-get install libncurses5-dev \
          gcc make git exuberant-ctags bc libssl-dev

### Obtaining the Mainline Kernel Sources

    $ git clone https://github.com/torvalds/linux.git ~/linux
    $ cd ~/linux
    $ git checkout v4.13

### Default Kernel Build Configuration

    $ cp /boot/config-$(uname -r) .config
    $ yes '' | make oldconfig

### Adding "gcov" Specific Configurations

    $ echo 'CONFIG_DEBUG_FS=y' >> .config
    $ echo 'CONFIG_GCOV_KERNEL=y' >> .config
    $ echo 'CONFIG_GCOV_FORMAT_AUTODETECT=y' >> .config
    $ echo 'CONFIG_GCOV_PROFILE_ALL=y' >> .config
    $ yes '' | make oldconfig

### Building and Installing the Custom Kernel

    $ make -j$(nproc)
    $ sudo make modules_install install
    $ sudo reboot

### Verifying "gcov" Support is Enabled

    $ sudo mount -t debugfs none /sys/kernel/debug
    $ sudo ls -1d /sys/kernel/debug/gcov
    /sys/kernel/debug/gcov

## Building ZFS on Linux Modules

There's nothing special that we need to do, in order to enable "gcov"
support for the ZFS on Linux modules. Since the kernel was built with
"gcov" enabled, the modules will be automatically built with it enabled
as well.

### Building the SPL Repository

    $ git clone git://github.com/zfsonlinux/spl.git ~/spl
    $ cd ~/spl
    $ ./autogen.sh && ./configure && make
    $ sudo make install

### Building the ZFS Repository

    $ git clone git://github.com/zfsonlinux/zfs.git ~/zfs
    $ cd ~/zfs
    $ ./autogen.sh && ./configure && make
    $ sudo make install

## Running a Test and Collecting Data Files

At this point, we're running on a "gcov" enabled Linux kernel, and the
ZFS on Linux kernel modules should be installed such that we can load
them easily with `modprobe`. Thus, we can finally run a trivial test,
loading and unloading the "zfs" module, and then use `gcov` to
extract the code coverage information from that test.

### Loading and Unloading the "zfs" module

    $ sudo modprobe zfs
    $ sudo modprobe -r zfs
    $ dmesg | grep ZFS
    [ 1214.307936] ZFS: Loaded module v0.7.0-1, ZFS pool version 5000, ZFS filesystem version 5
    [ 1216.475316] ZFS: Unloaded module v0.7.0-1

### Collecting "gcov" Data Files

    $ sudo chmod a+rx /sys/kernel/debug
    $ TEMPDIR=$(mktemp -d)
    $ cd /sys/kernel/debug/gcov/$HOME
    $ find . -type d -exec mkdir -p $TEMPDIR/{} \;
    $ find . -name '*.gcda' -exec sh -c 'sudo cat $0 > '$TEMPDIR'/$0' {} \;
    $ find . -name '*.gcno' -exec sh -c 'cp -d $0 '$TEMPDIR'/$0' {} \;

## Use Data Files to Extract Code Coverage Information

Now that we have the "gcov" data files stored in `$TEMPDIR`, we can
finally use these to extract the code converage data for any particular
source code file that we're interested in. For this example, we're going
to look at the `zfs_ioctl.c` file, and more specifically, at the `_fini`
function which is run a single time when the "zfs" module is unloaded:

    $ cd /tmp
    $ gcov -o $TEMPDIR/zfs/module/zfs zfs_ioctl.c
    $ cat zfs_ioctl.c.gcov
    ...
            -: 6751:static void __exit
            1: 6752:_fini(void)
            -: 6753:{
            1: 6754:        zfs_detach();
            1: 6755:        zfs_fini();
            1: 6756:        spa_fini();
            1: 6757:        zvol_fini();
            -: 6758:
            1: 6759:        tsd_destroy(&zfs_fsyncer_key);
            1: 6760:        tsd_destroy(&rrw_tsd_key);
            1: 6761:        tsd_destroy(&zfs_allow_log_key);
            -: 6762:
            1: 6763:        printk(KERN_NOTICE "ZFS: Unloaded module v%s-%s%s\n",
            -: 6764:            ZFS_META_VERSION, ZFS_META_RELEASE, ZFS_DEBUG_STR);
            1: 6765:}
    ...

Here we can see that each line of the `_fini` function was executed
exactly a single time, which is expected based on our trival load/unload
test case.

[kernelbuild]: https://kernelnewbies.org/KernelBuild
