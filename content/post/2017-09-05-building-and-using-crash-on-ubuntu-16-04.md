+++
date = "2017-09-05T00:00:00-08:00"
lastmod = "2017-09-05T00:00:00-08:00"
title = "Building and Using \"crash\" on Ubuntu 16.04"
draft = false
+++

## Introduction

I've been working on the ZFS on Linux project recently, and had a need
to use `crash` on the Ubuntu 16.04 based VM I was using. The following
is some notes regarding the steps I had to take, in order to build,
install, and ultimately run the utility against the "live" system.

## Build and Install "crash"

First, I had to install the build dependencies:

    $ sudo apt-get install -y \
        git build-essential libncurses5-dev zlib1g-dev bison

Then I could checkout the source code, build, and install:

    $ git clone https://github.com/crash-utility/crash
    $ cd crash
    $ make
    $ sudo make install

## Install Kernel Debug Symbols

In order to run the `crash` utility against the live system, we need
access to the uncompressed kernel image. We can obtain this by
installing the corresponding `-dbgsym` package for the running kernel.

First, we have to enable the APT repositories that contain these
packages:

    $ sudo tee /etc/apt/sources.list.d/ddebs.list <<EOF
    deb http://ddebs.ubuntu.com/ $(lsb_release -c -s)          main restricted universe multiverse
    deb http://ddebs.ubuntu.com/ $(lsb_release -c -s)-updates  main restricted universe multiverse
    deb http://ddebs.ubuntu.com/ $(lsb_release -c -s)-proposed main restricted universe multiverse
    EOF


Then add the correct key for these repositories:

    $ sudo apt-key adv \
        --keyserver keyserver.ubuntu.com \
        --recv-keys 428D7C01 C8CAB6595FDFF622

And now we can refresh the APT cache and install the kernel's `-dbgsym`
package:

    $ sudo apt-get update -y
    $ sudo apt-get install -y linux-image-$(uname -r)-dbgsym

## Run "crash" Against Live System

Finally, with all of the above done, I could run the `crash` utility
against the running system like so:

    $ sudo crash /usr/lib/debug/boot/vmlinux-$(uname -r)

Here's a quick example for dumping the kernel backtrace for a specific
process (the process I needed to debug):

    crash> bt 28365
    PID: 28365  TASK: ffff887b7b978e00  CPU: 5   COMMAND: "fio"
     #0 [ffff887cd2167870] __schedule at ffffffff8183e9ee
     #1 [ffff887cd21678c0] schedule at ffffffff8183f0d5
     #2 [ffff887cd21678d8] spl_panic at ffffffffc02d70ca [spl]
     #3 [ffff887cd2167a60] zil_commit at ffffffffc1271daa [zfs]
     #4 [ffff887cd2167b50] zfs_write at ffffffffc1260d6d [zfs]
     #5 [ffff887cd2167d48] zpl_write_common_iovec at ffffffffc128b391 [zfs]
     #6 [ffff887cd2167dc0] zpl_iter_write at ffffffffc128b4e1 [zfs]
     #7 [ffff887cd2167e30] new_sync_write at ffffffff8120ec0b
     #8 [ffff887cd2167eb8] __vfs_write at ffffffff8120ec76
     #9 [ffff887cd2167ec8] vfs_write at ffffffff8120f5f9
    #10 [ffff887cd2167f08] sys_pwrite64 at ffffffff81210465
    #11 [ffff887cd2167f50] entry_SYSCALL_64_fastpath at ffffffff818431f2
        RIP: 00007f08274fbda3  RSP: 00007ffc1eae8a70  RFLAGS: 00000293
        RAX: ffffffffffffffda  RBX: 00007f080f6d8240  RCX: 00007f08274fbda3
        RDX: 0000000000002000  RSI: 0000000000f9b660  RDI: 0000000000000004
        RBP: 00007f080f6d8240   R8: 0000000000000000   R9: 0000000000000001
        R10: 0000000000374000  R11: 0000000000000293  R12: 000000000009bea3
        R13: 0000000000000001  R14: 0000000000002000  R15: 0000000000372000
        ORIG_RAX: 0000000000000012  CS: 0033  SS: 002b

And another example showing information about the running system:

    crash> sys
          KERNEL: /usr/lib/debug/boot/vmlinux-4.4.0-93-generic
        DUMPFILE: /proc/kcore
            CPUS: 32
            DATE: Tue Sep  5 22:55:59 2017
          UPTIME: 06:58:42
    LOAD AVERAGE: 512.32, 512.25, 512.26
           TASKS: 1035
        NODENAME: ubuntu-16-04
         RELEASE: 4.4.0-93-generic
         VERSION: #116-Ubuntu SMP Fri Aug 11 21:17:51 UTC 2017
         MACHINE: x86_64  (2199 Mhz)
          MEMORY: 512 GB
