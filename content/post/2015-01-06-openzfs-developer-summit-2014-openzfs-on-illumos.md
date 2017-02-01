+++
date = "2015-01-06T00:00:00-08:00"
title = "OpenZFS Developer Summit 2014: OpenZFS on illumos"
draft = false
+++

The [OpenZFS][1] project is growing! The [second annual OpenZFS developer
summit][2] concluded just under two months ago, and overall I thought it
went very well. There was roughly 70 attendees, twice as many as the
previous year, and the talks given were very engaging and interesting.

I gave a short talk about ZFS on the [illumos][3] platform, and also
touched briefly on some of my subjective opinions coming from a
[ZFS on Linux][4] background. While a video recording of [my talk][5] is
available, I thought it'd be good to also present some of that in written
format, which might be a more approachable format for those who weren't
able to attend the conference. So here goes...

## OpenZFS History

As most readers will already know, ZFS was originally designed and
developed at Sun Microsystems starting around 2001. It was later
released as open source in 2005 via OpenSolaris. In 2010 illumos was
born as a fork of OpenSolaris, and in 2013 the OpenZFS was created.

As a result of this lineage, OpenZFS really feels "at home" within the
illumos project; and what I mean by that, is it has a very polished
integration with the illumos operating system and the code base as a
whole. Just to state a couple explicit examples of this:

 - It's very easy to boot and run ZFS as the root filesystem (even in
   the face of on-disk feature changes) as it and the boot loader both
   live and change within the same repository.
 - The debugging tools on illumos (e.g. mdb) have knowledge of
   ZFS-specific structures and have ZFS-specific features to allow
   efficient debugging and development.

But, as OpenZFS is growing, its integration on other operating systems
is continually getting better.

## OpenZFS on illumos Development Process

The development model for OpenZFS on illumos follows the same
conventions as all other illumos development. Access to commit to the
repository is granted to "advocates", and these advocates rely on the
community (possibly themselves as well) to review any proposed changes,
and ensure the changes are correct and are of high quality. In practice,
each proposed change must reviewed and approved by a subject matter
expert (e.g. for OpenZFS, this is typically Matt Ahrens or George
Wilson) prior to an advocate committing the change.

Since the illumos project doesn't have any specific releases, this
implies that each commit to the repository must be of high quality as
each commit is effectively a "release". There isn't the option of
merging code known to be defective, with the intention of fixing the
defect in a follow up commit prior to the "next release".

## OpenZFS on illumos Collaboration

Since OpenZFS on illumos is the "upstream" platform for OpenZFS, it's
important that there is open communication and collaboration between
the illumos developers and developers of the other platforms. As
changes from one platform are merged into illumos, all other platforms
will (ideally) pull these changes into their own version of OpenZFS
which is mutually beneficial for a number of reasons:

 1. Less time spent on merge conflicts when pulling changes in between
    OpenZFS platforms. If each platform is active in pushing to illumos
    and pulling from illumos, then we'll all be using as close to the
    same source code as possible.
 2. Less duplication of effort. There's been a number of times where
    I've personally spent effort investigating or solving a specific
    problem, only to find out that it's already been solved on another
    OpenZFS platform. If each platform was more actively kept in sync
    with each other, this situation would be far less likely.
 3. More varied testing. If all platforms are running the same code,
    then we all benefit from the variety of workloads each platform
    brings to the table; and as a result, we all test each others' code.
 4. Proliferation of new OpenZFS features. Currently, most new ZFS
    features are first developed on, and committed to illumos. Thus, if
    other platforms are active in pushing changes into illumos, these
    new features are inherently developed around and incorporate the
    most recent changes in each platform. This makes it much easier to
    pull the new features into the downstream platforms, allowing all
    OpenZFS consumers to benefit from the new features as soon as
    possible.

In order to achieve this goal, the process of building and testing
changes to the OpenZFS on illumos code base needs to be simplified and
streamlined. Work is on-going on this front, and my hope is that the
necessary improvements will happen over the next year, and we'll be
able to sing a different tune for next year's developer summit.

## Developer/Debugging Tools on illumos

One of the nice things about OpenZFS development on illumos is that I
get to make use of the outstanding developer tools available on the
platform. Having spent much of the last few years working within the
Linux kernel, I've been implicitly trained to rely almost entirely on
the source code and stack traces for debugging most production and
development problems. Now that I've jumped to illumos, I can see
first hand what I've been missing with tools like dtrace, kmdb,
and **especially** mdb.

In my short time working on the illumos platform, I've found mdb to be
easily one of the best parts about the jump from ZFS on Linux. Certain
tasks that would taken hours, or have been prohibitively difficult, on
Linux can be done with ease using mdb's dcmds and pipelines.

For example, while working on the ZFS on Linux port I would often be
puzzled by what I found when looking at the ARC kstats. To try and make
some sense of it, I wanted to know exactly what was contained in the ARC
and what was contained in the dbuf cache; this eventually led to the
implementation of the "dbufs" proc handler on the Linux port. On
illumos, there's already a mdb dcmd to do exactly that:

    # Lines ending with '...' have been truncated to fit in 80 columns.
    $ sudo mdb -k
    > ::dbufs | ::dbuf ! head
            addr object lvl blkid holds os
    ffffff160e924b10     114f 0        19  0 rpool/versions/2014.12.2014.1 ...
    ffffff160e924c30     1c50 0        35  0 rpool/versions/2014.12.2014.1 ...
    ffffff160e924d50     1c4d 0        33  0 rpool/versions/2014.12.2014.1 ...
    ffffff160e9092f8     164b 0        24  0 rpool/versions/2014.12.2014.1 ...
    ffffff160e8cc050       a0 0         3  0 mos
    ffffff160e8c04e0     177b 0        3a  0 rpool/versions/2014.12.2014.1 ...
    ffffff160e8bcde8     1c4d 0        1a  0 rpool/versions/2014.12.2014.1 ...
    ffffff160e8b2bb8       74 0         0  0 mos
    ffffff160e8b2cd8     1c4c 0         f  0 rpool/versions/2014.12.2014.1 ...

And better yet, I can print the full contents for each buffer if that
level of detail is needed:

    $ sudo mdb -k
    > ::dbufs | ::print dmu_buf_impl_t ! head
    {
        db = {
            db_object = 0x114f
            db_offset = 0x320000
            db_size = 0x20000
            db_data = 0xffffff012e283000
        }
        db_objset = 0xffffff096dd82500
        db_dnode_handle = 0xffffff160e858d20
        db_parent = 0xffffff160e63cb70

Or filter out the specific dbufs of interest:

    # Lines ending with '...' have been truncated to fit in 80 columns.
    $ sudo mdb -k
    > ::dbufs -o 0x90 | ::dbuf ! head
            addr object lvl blkid holds os
    ffffff1609deb210       90 0         1  0 mos
    ffffff0980966790       90 0         2  0 mos
    ffffff0982633030       90 0         0  0 mos
    ffffff096e53c9a8       90 1         0  3 mos
    ffffff15edcd7bf0       90 0     bonus  1 tank/fish
    ffffff097340dab8       90 0     bonus  1 rpool/versions/2014.12.2014.1 ...
    ffffff096e118450       90 0         0  0 rpool/versions/2014.12.2014.1 ...
    ffffff09179a79e8       90 0     bonus  1 mos
    ffffff09157b6cc0       90 0     bonus  0 rpool/versions/2014.12.2014.1 ...

But what if one wants to filter on something that isn't supported by the
`::dbufs` command? Well, here's a more general approach, filtering out
only the buffers with a reference count of 3, using `::grep`:

    # Lines ending with '...' have been truncated to fit in 80 columns.
    # Also, mdb doesn't actually support continuation of input lines
    # using the "\" character; but it helps me format the command within
    # 80 columns for this post, and resembles a bash line continuation.

    $ sudo mdb -k
    > ::dbufs | ::print dmu_buf_impl_t db_holds.rc_count | \
      ::grep '.==0x3' | ::eval &lt;dbuf=K | ::dbuf ! head
            addr object lvl blkid holds os
    ffffff1607afe080      5f2 1         0  3 rpool/versions/2014.12.2014.1 ...
    ffffff1607afe080      5f2 1         0  3 rpool/versions/2014.12.2014.1 ...
    ffffff1607afe080      5f2 1         0  3 rpool/versions/2014.12.2014.1 ...
    ffffff1607afe080      5f2 1         0  3 rpool/versions/2014.12.2014.1 ...
    ffffff1607afe080      5f2 1         0  3 rpool/versions/2014.12.2014.1 ...
    ffffff1607afe080      5f2 1         0  3 rpool/versions/2014.12.2014.1 ...
    ffffff1607afe080      5f2 1         0  3 rpool/versions/2014.12.2014.1 ...
    ffffff1607afe080      5f2 1         0  3 rpool/versions/2014.12.2014.1 ...
    ffffff1607afe080      5f2 1         0  3 rpool/versions/2014.12.2014.1 ...

Additionally, the ability to get stack traces detailing where objects
were allocated using `kmem_flags=0x1` is hugely beneficial. Using it,
things like the `::refcount` dcmd can not only provide information about
the current number of holds, but it can also provide the full kernel
stack trace of the thread that took the hold when the hold was taken.

Likewise, the `::whatis` dcmd also makes use of this information and
displays it to the user. Imagine a case where you have a "random"
pointer to an object (e.g. a pointer to an `arc_buf_t`), and wanted to
know where this object came from. If using `kmem_flags=0x1` on illumos,
this is as simple as the following:

    $ sudo mdb -k
    > ::walk dmu_buf_impl_t ! head -n 1
    0xffffff0ab4001048
    > 0xffffff0ab4001048::whatis
    ffffff0ab4001048 is allocated from dmu_buf_impl_t:
                ADDR          BUFADDR        TIMESTAMP           THREAD
                                CACHE          LASTLOG         CONTENTS
    ffffff0ab4059128 ffffff0ab4001048      157ca42dbaf ffffff09e088e440
                     ffffff0958a5a008 ffffff090b0ebbc0 ffffff0932c80430
                     kmem_cache_alloc_debug+0x2e0
                     kmem_cache_alloc+0xdd
                     dbuf_create+0x79
                     dbuf_hold_impl+0x192
                     dbuf_hold+0x27
                     dmu_buf_hold_array_by_dnode+0x129
                     dmu_read_uio_dnode+0x5a
                     dmu_read_uio_dbuf+0x51
                     zfs_read+0x1a3
                     fop_read+0x5b
                     pread64+0x25d

I could go on and on about the cool things mdb allows, but this post is
already long enough, so I'll cut myself off here. It would be great to
see an mdb port to Linux! (or even a more feature rich version of
[crash][6] with OpenZFS support)

[1]: http://open-zfs.org/
[2]: http://www.open-zfs.org/wiki/OpenZFS_Developer_Summit_2014
[3]: http://www.illumos.org/
[4]: http://zfsonlinux.org/
[5]: http://youtu.be/UtlGt3ag0o0
[6]: http://people.redhat.com/anderson/crash_whitepaper/
