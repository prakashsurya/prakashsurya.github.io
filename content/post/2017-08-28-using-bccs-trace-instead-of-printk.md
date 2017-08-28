+++
date = "2017-08-28T00:00:00-08:00"
lastmod = "2017-08-28T00:00:00-08:00"
title = "Using BCC's \"trace\" Instead of \"printk\""
draft = false
+++

## Introduction

Recently I've been working on porting some changes that I made to the
OpenZFS ZIL over to the ZFS on Linux codebase; see [here][openzfs-pr]
for the OpenZFS pull request, and [here][zfsonlinux-pr] for the ZFS on
Linux pull request.

In my initial port, I was running into a problem where the automated
tests would trigger a "hang" as a result of the `readmmap` program
calling `msync`:

    $ pstree -p 2337
    test-runner.py(2337)-+-sudo(3183)---mmap_read_001_p(3185)---readmmap(3198)
                         `-{test-runner.py}(3184)

    $ sudo cat /proc/3198/stack
    [<ffffffff9bdafb68>] wait_on_page_bit_common+0x118/0x1d0
    [<ffffffff9bdafd34>] __filemap_fdatawait_range+0x114/0x190
    [<ffffffff9bdafdc4>] filemap_fdatawait_range+0x14/0x30
    [<ffffffff9bdb2477>] filemap_write_and_wait_range+0x57/0x90
    [<ffffffffc08f049d>] zpl_fsync+0x3d/0x110 [zfs]
    [<ffffffff9be7b93b>] vfs_fsync_range+0x4b/0xb0
    [<ffffffff9bdf6af2>] SyS_msync+0x182/0x200
    [<ffffffff9c4d453b>] entry_SYSCALL_64_fastpath+0x1e/0xad
    [<ffffffffffffffff>] 0xffffffffffffffff

Once this state was reached, the `msync` call would never return.

Without diving too far into the technical details, my hunch was that
the `zfs_putpage_commit_cb` function was not being called properly. At
this point, I wanted to verify this, so I could then revisit the code
with concrete data to support my suspicision.

If I hit this issue on illumos, I would have quickly jumped to using
either `dtrace` or `mdb` to help verify and debug the situation. Since
I was on Linux, I had neither of these tools at my disposal. Thankfully
though, I did have a test case that would reliably reproduce
the issue in a matter of minutes.

I thought about adding some `printk` or `zfs_dbgmsg` statments to the
code and re-compiling to gather some data, but after having used
`dtrace` on illumos, I resented that idea. I had previously read about
[Linux BPF][linux-bpf] and [Linux BCC][linux-bcc], so this felt like a
good opportunity try and experiment with those to see if I could use it
to gain better insight into my problem, without making any code changes.

## Building and Installing BCC from Source

First off, I wanted to build and install BCC from source, rather than
use a pre-built package. This decision was strictly for educational
purposes; I wanted to learn how easy or difficult this would be, in case
I ever wanted to make modifications to it in the future.

The project contains easy to follow instructions for doing this in its
[Install.md][bcc-install] document. Since I was running on Ubuntu 17.04,
the process was documented and works as described:

    # Trusty and older
    VER=trusty
    $ echo "deb http://llvm.org/apt/$VER/ llvm-toolchain-$VER-3.7 main
    $ deb-src http://llvm.org/apt/$VER/ llvm-toolchain-$VER-3.7 main" | \
      sudo tee /etc/apt/sources.list.d/llvm.list
    $ wget -O - http://llvm.org/apt/llvm-snapshot.gpg.key | sudo apt-key add -
    $ sudo apt-get update

    # All versions
    $ sudo apt-get -y install bison build-essential cmake flex git libedit-dev \
      libllvm3.7 llvm-3.7-dev libclang-3.7-dev python zlib1g-dev libelf-dev

    # For Lua support
    $ sudo apt-get -y install luajit luajit-5.1-dev

    $ git clone https://github.com/iovisor/bcc.git
    $ mkdir bcc/build; cd bcc/build
    $ cmake .. -DCMAKE_INSTALL_PREFIX=/usr
    $ make
    $ sudo make install

After running those commands, I found the various BCC tools, examples,
and manpages installed in `/usr/share/bcc`.

## Using BCC's "trace"

Now that I had the BCC tools installed, I could use them to gather some
data about my issue. While there's quite a few tools in the BCC
repository to help trace the various Linux subsystems, what I needed was
targeted specifically at the `zfs_putpage_commit_cb` function. I wanted
to see if that function was getting called at all; and for that, the `trace`
command was just what I needed.

First I used this tool while running the ZFS modules without my changes,
to provide a baseline:

    # /usr/share/bcc/tools/trace -K zfs_putpage_commit_cb 2>/dev/null
    PID    TID    COMM         FUNC
    27605  27605  readmmap     zfs_putpage_commit_cb
            zfs_putpage_commit_cb+0x1 [kernel]
            zil_commit+0x17 [kernel]
            zpl_writepages+0xd6 [kernel]
            do_writepages+0x1e [kernel]
            __filemap_fdatawrite_range+0xc6 [kernel]
            filemap_write_and_wait_range+0x41 [kernel]
            zpl_fsync+0x3d [kernel]
            vfs_fsync_range+0x4b [kernel]
            sys_msync+0x182 [kernel]
            entry_SYSCALL_64_fastpath+0x1e [kernel]
    ^C

With the reproducer running in another shell, this told me that this
function definitely was being called when my changes were not applied,
and even provided a stack trace that lead to the function call.

Now that I verified the behavior without my changes, it was time to run
the same test, but with my modified version of ZFS:

    # /usr/share/bcc/tools/trace -K zfs_putpage_commit_cb 2>/dev/null
    PID    TID    COMM         FUNC
    ^C

Just like before, I had the reproducer running in another shell, but
this time the `trace` command didn't produce any output, which means the
function wasn't called.

With these observations in mind, I was able to re-visit the code and
ultimately track down my problem.

[openzfs-pr]: https://github.com/openzfs/openzfs/pull/447
[zfsonlinux-pr]: https://github.com/zfsonlinux/zfs/pull/6566
[linux-bpf]: http://www.brendangregg.com/blog/2016-03-05/linux-bpf-superpowers.html
[linux-bcc]: https://github.com/iovisor/bcc
[bcc-install]: https://github.com/iovisor/bcc/blob/master/INSTALL.md
