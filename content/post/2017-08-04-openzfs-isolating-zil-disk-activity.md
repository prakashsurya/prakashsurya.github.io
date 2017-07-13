+++
date = "2017-08-04T00:00:00-08:00"
lastmod = "2017-08-04T00:00:00-08:00"
title = "OpenZFS: Isolating ZIL Disk Activity"
draft = false
+++

I recently completed a project to improve the performance of the OpenZFS
ZIL (see [here](https://github.com/openzfs/openzfs/pull/421) for more
details); i.e. improving the performance of synchronous activity on
OpenZFS, such as writes using the `O_SYNC` flag.  As part of that work,
I had to run some performance testing and benchmarking of my code
changes (and the system as a whole), to ensure the system was behaving
as I expected.

Early on in my benchmarking exercises, I became confused by the data
that I was gathering. I was expecting only a certain number of writes to
be active on the underlying storage devices at any given time, based on
the known workload that I was applying to the zpool, and based on my
understanding of the ZIL's mechanics. When running these known workloads
and inspecting the `actv` column from `iostat` though, I was
consistently seeing more write activity on the devices than I expected.

At this point, I was starting to question my understanding of the code
that I had written, and my understanding of the ZIL's mechanics as a
whole. Since I knew exactly the IO workload that was being applied to
the system, why wasn't it behaving as I had predicted?

After scratching my head and consulting the code numerous times, I asked
Matt Ahrens if he had any clues as to what might be going on. Matt was
quick to remind me that I was failing to incorporate the IO that would
occur as part of `spa_sync()` into my mental model. Additionally, he
suggested that since it would be difficult to know exactly how many IOs
to expect from `spa_sync()`, which then makes it difficult to verify the
`actv` column from `iostat` w.r.t. my code changes, I should configure
the system to effectively disable `spa_sync()` altogether. This way, all
of the IOs that would be active on the disk would be a result of a ZIL
write, which is exactly what I was previously expecting.

To acheive this configuration, Matt pointed me at the following kernel
perameters: `zfs_dirty_data_max`, `zfs_dirty_data_sync`, and
`zfs_txg_timeout`. Basically, I had to set all of these values such that
the dirty limit would never be reached, and thus, a TXG sync would never
trigger as a result of the amount of dirty data my workload generated.
Since my test system had 128 GB of RAM, I used the following
commands/values to achieve this configuration:

    $ sudo mdb -kwe 'zfs_dirty_data_max/z 0t68719476736'
    $ sudo mdb -kwe 'zfs_dirty_data_sync/z 0t34359738368'
    $ sudo mdb -kwe 'zfs_txg_timeout/z 0t3600'

It's also important to note that these values are dependent on the rate
at which my workload would dirty data (i.e. the workload's write
throughput), and the duration of the test. I ensured that the workload
would not be able to dirty enough data to cause TXG sync prior to the
test completing. With all of this configured correctly, the only way
writes would get issued to disk would be via the ZIL, which is exactly
what I wanted.

Additionally, I further tuned the system to disable the IO aggregation
that ZFS may do when there's sufficient write activity to warrant it:

    $ sudo mdb -kwe 'zfs_vdev_aggregation_limit/z 0t0'

While this setting didn't help my workload's throughput, it did help me
validate the correctness of my code changes.
