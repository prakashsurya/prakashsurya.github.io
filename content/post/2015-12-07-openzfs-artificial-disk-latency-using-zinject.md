+++
date = "2015-12-07T00:00:00-08:00"
lastmod = "2015-12-07T00:00:00-08:00"
title = "OpenZFS: Artificial Disk Latency Using zinject"
draft = false
+++

About a year ago I had the opportunity to work on a small extension to
the [OpenZFS][openzfs] `zinject` command with colleagues Matt Ahrens and
Frank Salzmann, during one of our [Delphix][delphix] engineering wide
hackathon events. Now that it's in the [process of landing][pull] in the
upstream [OpenZFS repository][repo], I wanted to take a minute to show
it off.

To describe the new functionality, I'll defer to the help message:

    $ zinject -h
    ... <snip> ...

    zinject -d device -D latency:lanes pool

            Add an artificial delay to IO requests on a particular
            device, such that the requests take a minimum of 'latency'
            milliseconds to complete. Each delay has an associated
            number of 'lanes' which defines the number of concurrent
            IO requests that can be processed.

            For example, with a single lane delay of 10 ms (-D 10:1),
            the device will only be able to service a single IO request
            at a time with each request taking 10 ms to complete. So,
            if only a single request is submitted every 10 ms, the
            average latency will be 10 ms; but if more than one request
            is submitted every 10 ms, the average latency will be more
            than 10 ms.

            Similarly, if a delay of 10 ms is specified to have two
            lanes (-D 10:2), then the device will be able to service
            two requests at a time, each with a minimum latency of
            10 ms. So, if two requests are submitted every 10 ms, then
            the average latency will be 10 ms; but if more than two
            requests are submitted every 10 ms, the average latency
            will be more than 10 ms.

            Also note, these delays are additive. So two invocations
            of '-D 10:1', is roughly equivalent to a single invocation
            of '-D 10:2'. This also means, one can specify multiple
            lanes with differing target latencies. For example, an
            invocation of '-D 10:1' followed by '-D 25:2' will
            create 3 lanes on the device; one lane with a latency
            of 10 ms and two lanes with a 25 ms latency.

    ... <snip> ...

Given that explanation, lets dive in and give this a test drive with a
few examples.

## Setup and Baseline

First, lets start with a basic example using a pool with a single disk:

    # zpool create tank c1t1d0
    # zfs create tank/foo

Now let's do some IO to the pool and use `dtrace` to examine the latency
of the requests to get a baseline expectation for the IO latency of
writes to this pool.

Running in one terminal I have a simple, single threaded write workload
running in an infinite loop. This will continually copy some random data
to a file in the pool, remove that file, and repeat.

    # dd if=/dev/urandom of=/tmp/urandom-cache bs=1M count=1024 &>/dev/null
    # while true; do
    > cp /tmp/urandom-cache /tank/foo/urandom-cache
    > sync
    > rm /tank/foo/urandom-cache
    > sync
    done

Now, lets look at the latency of the write requests issued to disk by
the filesystem, using `dtrace` (see [here](#zio-latency) for the source
of this `dtrace` script):

    # dtrace -qs zio-rw-latency.d -c 'sleep 60' tank write

    histogram of latencies (us):

               value  ------------- Distribution ------------- count
                 128 |                                         0
                 256 |                                         248
                 512 |@@@@@                                    6204
                1024 |@@@@@@@@@@@@@@@@@@@@@@                   26218
                2048 |@@@@@@@@@@@                              13070
                4096 |@@                                       2474
                8192 |                                         416
               16384 |                                         56
               32768 |                                         7
               65536 |                                         0

    avg latency (us)   stddev (us)   iops   throughput (KB/s)
    2025               1609          811    99623

This is showing us a histogram of the latency of all writes that were
submitted and completed to our pool during that specific interval.
Additionally, it's showing us an average and standard deviation of the
latency over all the requests, number of request issued and completed
per second, and the throughput achieved by these writes.

This sets the baseline performance that we expect to get out of this
storage configuration; most requests take less than 8 ms to be serviced,
with the average being about 2ms with a standard deviation of roughly
1.5ms.

## First Experiment: Single Disk, Single Lane

Now that we have some baseline numbers to compare against, lets use the
new `zinject` functionality to slow down the requests. Let's say we want
to delay all requests such that they take a minimum of 10ms to complete.

Remembering that we created this pool in the previous section using a
single disk named `c1t1d0`, we can set the minimum latency of this disk
to be our desired 10ms:

    # zinject -d c1t1d0 -D 10:1 tank
    Added handler 1 with the following properties:
      pool: tank
      vdev: c138fa4fbf3d8f2

And to double check that this is the only delay registered on our pool,
we can run `zinject` without any parameters to dump a list of all
registered delays:

    # zinject
     ID  POOL             DELAY (ms)       LANES            GUID
    ---  ---------------  ---------------  ---------------  ----------------
      1  tank             10               1                c138fa4fbf3d8f2

The "lanes" column describes the number of requests that can be serviced
simultaneous by each registered delay; thus this delay specifies that
only a single request can be serviced at a time, and each request will
take a minimum of 10ms to complete.

Now, with that delay in place, and the same write workload from the
previous section still running, we can again use `dtrace` to inspect the
write request latencies, IOPs, and throughput values. Given this delay,
we should expect to see no request complete sooner than 10ms, and as a
result, the IOPs and throughput should take a significant hit:

    # dtrace -qs zio-rw-latency.d -c 'sleep 60' tank write

    histogram of latencies (us):

               value  ------------- Distribution ------------- count
                4096 |                                         0
                8192 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@             4177
               16384 |@@@@@@                                   894
               32768 |@@@@@                                    807
               65536 |                                         1
              131072 |                                         0

    avg latency (us)   stddev (us)   iops   throughput (KB/s)
    17080              12448         97     12518

Great! This matches our expectation.

Since we're using a logarithmic histogram, we can't actually
differentiate between requests in the 8-16ms range, but judging by the
fact that there aren't any request in the 0-8ms range, it's safe to
assume all the request that landed in the 8-16ms range, actually took
10-16ms to complete (as they should have).

## Wait, What About the Average Latency?

At this point, you might be looking at the average latency output from
the previous experiment, and be wondering why it's so much larger than
our target latency of 10ms. Without the delay, most requests would
complete in under 8ms, so why isn't the average latency with the delay
closer to the desired 10ms?

Remember that when we created the delay we specified it to have a single
"lane". What this means is only a single request can be serviced at a
time. If more than a single write is submitted every 10ms to the
underlying disk, then each new write request will get backed up by any
existing requests.

In other words, each lane represents a [FIFO][fifo] queue. When a new
IO request is submitted, the FIFO queue with the shortest wait time will
be selected (e.g. if there are multiple lanes available to choose from),
and then the request is placed in the queue and waits its turn until
it can be processed and eventually completed. Thus, in our example
above, if a request is submitted while another request is currently
being processed and still has 7ms to finish, this new request's latency
will be a minimum of 17ms; 7ms spent waiting for the prior request to be
processed, and 10ms spent being processed itself. Due to the way OpenZFS
issues writes, multiple requests will certainly be submitted
simultaneously, so it's common for this situation to occur and explains
the average latency in the previous experiment.

## Second Experiment: Single Disk, Multiple Lanes

If the previously described behavior isn't actually what is desired, we
can simulate an implementation where requests pay no attention to prior
requests by adding a delay with enough lanes such that there will always
be an open lane to accept new requests. As an example, let's create a
delay with the same 10ms minimum latency as before, but instead of 1
lane, we'll create 16 lanes:

    # zinject -c all
    removed all registered handlers

    # zinject -d c1t1d0 -D 10:16 tank
    Added handler 2 with the following properties:
      pool: tank
      vdev: c138fa4fbf3d8f2

    # zinject
     ID  POOL             DELAY (ms)       LANES            GUID
    ---  ---------------  ---------------  ---------------  ----------------
      2  tank             10               16               c138fa4fbf3d8f2

Assuming the magic number of 16 is sufficient to prevent new requests
from piling up behind older, in-progress requests, we should now see the
average latency trend much closer to the specified 10ms:

    # dtrace -qs zio-rw-latency.d -c 'sleep 60' tank write

    histogram of latencies (us):

               value  ------------- Distribution ------------- count
                4096 |                                         0
                8192 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@ 8836
               16384 |                                         39
               32768 |                                         11
               65536 |                                         1
              131072 |                                         1
              262144 |                                         0

    avg latency (us)   stddev (us)   iops   throughput (KB/s)
    10261              3022          148    18463

Perfect! As expected, the average latency is now much closer to our
desired 10ms. Unfortunately though, there were 52 requests that actually
took longer than 16ms to be completed by the underlying storage (2 of
which took longer than 64ms!), which pulls the average slightly above
the specified target of 10ms.

## Third Experiment: Setup and Baseline

Another interesting example to demonstrate is simulating a pool with
disks of different latency characteristics. This can happen for a
variety of reasons, none of which I'll touch on in this article. To
simulate this case, we'll need a pool with two disks:

    # zpool create tank c1t1d0 c1t2d0
    # zfs create tank/foo

We'll use a slightly modified version of the write workload used in
previous sections; instead of a single thread, we'll use 8 threads to
increase the amount of data being written and increase the amount of
simultaneous write requests occurring:

    # while true; do
    > PIDS=""
    > for i in $(seq 1 8); do
    >     cp /tmp/urandom-cache /tank/foo/urandom-cache-$i &>/dev/null &
    >     PIDS="$PIDS $!"
    > done
    > sync
    > for pid in $PIDS; do
    >     wait $pid
    > done
    >
    > PIDS=""
    > for i in $(seq 1 8); do
    >     rm /tank/foo/urandom-cache-$i &>/dev/null &
    >     PIDS="$PIDS $!"
    > done
    > sync
    > for pid in $PIDS; do
    >     wait $pid
    > done
    done

And with a slight variation to the previous `dtrace` script, we can show
the same statistics as the prior sections, but individually for each
vdev in the pool (see [here](#vdev-latency) for the source of this
`dtrace` script):

    # dtrace -qs vdev-rw-latency.d -c 'sleep 60' tank write

    histogram of latencies (us):

    f1d32d9b9514d7a3
               value  ------------- Distribution ------------- count
                 128 |                                         0
                 256 |                                         43
                 512 |                                         665
                1024 |@@                                       3231
                2048 |@                                        1848
                4096 |@@@@@@@@@@@@@                            17886
                8192 |@@@@@@@@@@@@@@@@@@@@                     26511
               16384 |@@@                                      3493
               32768 |                                         310
               65536 |                                         4
              131072 |                                         2
              262144 |                                         0

    f4eb4119d7170682
               value  ------------- Distribution ------------- count
                 128 |                                         0
                 256 |                                         41
                 512 |                                         627
                1024 |@@                                       3207
                2048 |@                                        1865
                4096 |@@@@@@@@@@@@@                            17809
                8192 |@@@@@@@@@@@@@@@@@@@@                     26416
               16384 |@@@                                      3570
               32768 |                                         312
               65536 |                                         9
              131072 |                                         1
              262144 |                                         0

    GUID               avg latency (us)   stddev (us)   iops   throughput (KB/s)
    f1d32d9b9514d7a3   9475               5292          899    111305
    f4eb4119d7170682   9496               5298          897    111116

As expected, without any delays injected with `zinject` each vdev
performs nearly identically.

## Third Experiment: Multiple Disks, Different Latencies

Now that we've verified the vdevs to be performing similarly in the
absence of any artificial delays, we can simulate a case where one disk
performs half as slow as the other by specifying two different `zinject`
delays for the different disks:

    # zinject -d c1t1d0 -D 16:16 tank
    Added handler 3 with the following properties:
      pool: tank
      vdev: f4eb4119d7170682

    # zinject -d c1t2d0 -D 32:16 tank
    Added handler 4 with the following properties:
      pool: tank
      vdev: f1d32d9b9514d7a3

    # zinject
     ID  POOL             DELAY (ms)       LANES            GUID
    ---  ---------------  ---------------  ---------------  ----------------
      3  tank             16               16               f4eb4119d7170682
      4  tank             32               16               f1d32d9b9514d7a3

In this scenario we'd expect the total throughput to take a significant
hit, resulting from the large delays, but we'd also expect the faster
disk to service twice the number of write requests in any given
interval. Thus, we should see the faster disk demonstrate roughly twice
the throughput when inspected with `dtrace`:

    # dtrace -qs vdev-rw-latency.d -c 'sleep 60' tank write

    histogram of latencies (us):

    f4eb4119d7170682
               value  ------------- Distribution ------------- count
                4096 |                                         0
                8192 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@  29964
               16384 |@                                        1012
               32768 |                                         117
               65536 |                                         4
              131072 |                                         0

    f1d32d9b9514d7a3
               value  ------------- Distribution ------------- count
                4096 |                                         0
                8192 |                                         26
               16384 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@ 16104
               32768 |                                         162
               65536 |                                         5
              131072 |                                         0

    GUID               avg latency (us)   stddev (us)   iops   throughput (KB/s)
    f4eb4119d7170682   16391              2128          518    64651
    f1d32d9b9514d7a3   32310              1680          271    33630

Which is exactly what I've measure above! Disk `f4eb4119d7170682` was
given the 16ms delay and was measured as having nearly double the IOPs
and nearly double the throughput during the measured interval, success!

## DTrace Scripts

In the spirit of full disclosure, here's the source for the `dtrace`
scripts that I used in the previous sections.

### ZIO Latency

Here's the script used in the first two experiments which does not break
down the statistics based on vdev:

    # cat zio-rw-latency.d
    BEGIN
    {
            start = timestamp;
    }

    fbt:zfs:vdev_disk_io_start:entry
    /this->zio = args[0],
     this->zio->io_type == ($$2 == "read" ? 1 : 2) &&
     stringof(this->zio->io_spa->spa_name) == $$1/
    {
            ts[this->zio] = timestamp;
    }

    fbt:zfs:zio_interrupt:entry
    /this->zio = args[0], ts[this->zio]/
    {
            delta = (timestamp - ts[this->zio])/1000;

            @i = count();
            @a = avg(delta);
            @s = stddev(delta);
            @l = quantize(delta);
            @t = sum(this->zio->io_size);

            ts[this->zio] = 0;
    }

    END
    {
            printf("\nhistogram of latencies (us):");
            printa(@l);

            normalize(@i, (timestamp - start) / 1000000000);
            normalize(@t, (timestamp - start) / 1000000000 * 1024);

            printf("%-16s   %-11s   %-4s   %-17s\n",
                "avg latency (us)", "stddev (us)", "iops", "throughput (KB/s)");
            printa("%@-16u   %@-11u   %@-4u   %@-17u\n", @a, @s, @i, @t);
    }

This script is intended to take two parameters: `$$1` is the pool name,
i.e. `tank` in our examples, and `$$2` is either `"read"` or `"write"`,
we only used `"write"` in the examples in this article.

### VDEV Latency

Here's the script used in the last experiment, which differentiates the
statistics based on the individual vdevs in the pool:

    # cat vdev-rw-latency.d
    BEGIN
    {
            start = timestamp;
    }

    fbt:zfs:vdev_disk_io_start:entry
    /this->zio = args[0],
     this->zio->io_type == ($$2 == "read" ? 1 : 2) &&
     stringof(this->zio->io_spa->spa_name) == $$1/
    {
            ts[this->zio] = timestamp;
    }

    fbt:zfs:zio_interrupt:entry
    /this->zio = args[0], ts[this->zio]/
    {
            delta = (timestamp - ts[this->zio])/1000;

            @i[this->zio->io_vd->vdev_guid] = count();
            @a[this->zio->io_vd->vdev_guid] = avg(delta);
            @s[this->zio->io_vd->vdev_guid] = stddev(delta);
            @l[this->zio->io_vd->vdev_guid] = quantize(delta);
            @t[this->zio->io_vd->vdev_guid] = sum(this->zio->io_size);

            ts[this->zio] = 0;
    }

    END
    {
            printf("\nhistogram of latencies (us):\n\n");
            printa("%x %@u\n", @l);

            normalize(@i, (timestamp - start) / 1000000000);
            normalize(@t, (timestamp - start) / 1000000000 * 1024);

            printf("%-16s   %-16s   %-11s   %-4s   %-17s\n", "GUID",
                "avg latency (us)", "stddev (us)", "iops", "throughput (KB/s)");
            printa("%-16x   %@-16u   %@-11u   %@-4u   %@-17u\n", @a, @s, @i, @t);
    }

This script takes the same `$$1` and `$$2` parameters as the previous
`zio-rw-latency.d` script described above.

### Why Not Use DTrace's "io" Provider?

Inevitably there will be at least one curious reader that looks at the
above `dtrace` scripts and wonders why I didn't use the [stable `io`
provider][dtrace-io] (i.e. `io:::start` and `io:::done`). This is intentional
because the delays injected using `zinject` occur above the level of
the `io` provider.

Essentially, what's happening is the underlying storage will complete
the request as fast as it can, and the `io` provider will report on this
latency. After the storage has completed the request, it will return the
result back up to OpenZFS, and it is at this point that the request may
be delayed further if using this new `zinject` feature. Once the result
is passed back to OpenZFS, the latency has already been recorded when
using the `io` provider, so any additional latency introduced by
`zinject` will go unnoticed.

I can easily demonstrate this with the following example:

    # zpool create tank c1t1d0
    # zfs create tank/foo

    # while true; do
    > cp /tmp/urandom-cache /tank/foo/urandom-cache
    > sync
    > rm /tank/foo/urandom-cache
    > sync
    > done &
    [1] 9388

    # zinject -d c1t1d0 -D 10:1 tank
    Added handler 5 with the following properties:
      pool: tank
      vdev: e2503467ad6a2e62

    # zinject
     ID  POOL             DELAY (ms)       LANES            GUID
    ---  ---------------  ---------------  ---------------  ----------------
      5  tank             10               1                e2503467ad6a2e62

So, we have a pool with a single disk and a delay of 10ms with a single
lane. Lets look at the write request latency on the system as reported
by the `io` provider:

    # dtrace -qs io-rw-latency.d -c 'sleep 60' write


               value  ------------- Distribution ------------- count
                 128 |                                         0
                 256 |                                         26
                 512 |@@                                       266
                1024 |@@@@@@@@@@@@@@@@@@@@                     2208
                2048 |@@@@@@@@@@                               1079
                4096 |@@@@                                     461
                8192 |@@                                       206
               16384 |@                                        54
               32768 |                                         10
               65536 |                                         1
              131072 |                                         0

It's clear that the latency reported here does not match the 10ms
minimum latency requested by the `zinject` command; and to verify that
the delay is actually working correctly, let's use the `dtrace` script
from the prior sections ([this one](#zio-latency)):

    # dtrace -qs zio-rw-latency.d -c 'sleep 60' tank write

    histogram of latencies (us):

               value  ------------- Distribution ------------- count
                4096 |                                         0
                8192 |@@@@@@@@@@@@@@@@@@                       2606
               16384 |@@@@@@                                   898
               32768 |@@@@@@@@@                                1308
               65536 |@@@@@@@                                  1095
              131072 |                                         0

    avg latency (us)   stddev (us)   iops   throughput (KB/s)
    33768              26629         98     12573

Clearly, the delay is in place, it's just that the `io` provider doesn't
detect the additional latency introduced by `zinject`.

And finally, here's the source for the `dtrace` script used above, which
uses the `io` provider:

    # cat io-rw-latency.d
    io:::start
    /args[0]->b_flags & ($$1 == "read" ? B_READ : B_WRITE)/
    {
            ts[args[0]->b_edev, args[0]->b_lblkno] = timestamp;
    }

    io:::done
    /ts[args[0]->b_edev, args[0]->b_lblkno]/
    {
            @ = quantize((timestamp -
                ts[args[0]->b_edev, args[0]->b_lblkno]) / 1000);
            ts[args[0]->b_edev, args[0]->b_lblkno] = 0;
    }

[openzfs]: http://www.open-zfs.org
[delphix]: http://www.delphix.com
[pull]: https://github.com/openzfs/openzfs/pull/39
[repo]: https://github.com/openzfs/openzfs
[fifo]: https://en.wikipedia.org/wiki/FIFO_(computing_and_electronics)
[dtrace-io]: http://dtrace.org/guide/chp-io.html
