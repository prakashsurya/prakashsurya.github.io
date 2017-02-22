+++
date = "2017-02-22T00:00:00-08:00"
lastmod = "2017-02-22T00:00:00-08:00"
title = "OpenZFS: Refresher on `zpool reguid` Using Examples"
draft = false
+++

## Introduction

The `zpool reguid` command can be used to regenerate the [GUID][guid]
for an [OpenZFS][openzfs] pool, which is useful when using device level
copies to generate multiple pools all with the same contents.

## Example using File VDEVs

As a contrived example, lets create a zpool backed by a single file
vdev:
```
# mkdir /tmp/tank1
# mkfile -n 256m /tmp/tank1/vdev
# zpool create tank1 /tmp/tank1/vdev
# zpool list tank1
NAME    SIZE  ALLOC   FREE  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
tank1   240M    78K   240M         -     1%     0%  1.00x  ONLINE  -
```

Now we can export this zpool, copy the file, and attempt to import two
identical zpools; one backed by the original file we created above, and
another zpool backed by the copy of the original file.
```
# zpool export tank1
# mkdir /tmp/tank2
# cp /tmp/tank1/vdev /tmp/tank2/vdev
# zpool import -d /tmp/tank1 tank1
# zpool import -d /tmp/tank2 tank1 tank2
cannot import 'tank1': a pool with that name is already created/imported,
and no additional pools with that name were found
```

While the intention was to have two pools imported, `tank1` and `tank2`,
as one can see, the second `zpool import` failed. The failure was due
to the two pools sharing the same GUID; we can verify the two pools
share the same GUID using `zdb`:
```
# zpool export tank1
# zdb -e -p /tmp/tank1 -C tank1 | grep pool_guid
        pool_guid: 750003400893681264
# zdb -e -p /tmp/tank2 -C tank1 | grep pool_guid
        pool_guid: 750003400893681264
```

This is where `zpool reguid` becomes useful; after importing `tank1` we
can use `zpool reguid` to change that zpool's GUID such that we can then
import `tank2`:
```
# zpool import -d /tmp/tank1 tank1
# zpool reguid tank1
# zpool import -d /tmp/tank2 tank1 tank2
# zpool list tank1 tank2
NAME    SIZE  ALLOC   FREE  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
tank1   240M   110K   240M         -     3%     0%  1.00x  ONLINE  -
tank2   240M   114K   240M         -     2%     0%  1.00x  ONLINE  -
```

Additionally, we can use `zdb` again to verify the GUID for the two
pools is different:
```
# zdb -C tank1 | grep pool_guid
        pool_guid: 12195257967456241841
# zdb -C tank2 | grep pool_guid
        pool_guid: 750003400893681264
```
note: we don't need the `-e` and `-p` options like before, because these
two pools are imported.

## Examples Using Damaged ZPOOLs

The manpage for the `zpool reguid` command states this can only be used
on zpools that are "online and healthy":
```
     zpool reguid pool
             Generates a new unique identifier for the pool. You must ensure
             that all devices in this pool are online and healthy before
             performing this action.
```
So, what happens when a device is damaged and this command is attempted
to be used? Let's try some examples and find out.

### Example Using a Damaged VDEV in an Empty ZPOOL

For this example, we'll create another zpool using file based VDEVs, but
this time the zpool will be striped across 3 files:
```
# mkdir /tmp/tank
# mkfile -n 256m /tmp/tank/vdev1
# mkfile -n 256m /tmp/tank/vdev2
# mkfile -n 256m /tmp/tank/vdev3
# zpool create tank /tmp/tank/vdev{1,2,3}
# zpool list -v tank
NAME                SIZE  ALLOC   FREE  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
tank                720M    78K   720M         -     0%     0%  1.00x  ONLINE  -
  /tmp/tank/vdev1   240M    30K   240M         -     0%     0%
  /tmp/tank/vdev2   240M    18K   240M         -     0%     0%
  /tmp/tank/vdev3   240M    30K   240M         -     0%     0%
```

Now lets damage the pool by writing random data over the most of one of
the VDEVs. We explicitly avoid writing to the first and last 1MB of the
file to leave the VDEV's labels intact.
```
root@oi-hipster:~# dd if=/dev/urandom of=/tmp/tank/vdev3 bs=1M seek=1 count=254
254+0 records in
254+0 records out
266338304 bytes transferred in 4.534832 secs (58731677 bytes/sec)
```

To verify the damage, we use `zpool scrub` and `zpool status`:
```
# zpool scrub
# zpool status -v tank
  pool: tank
 state: DEGRADED
status: One or more devices has experienced an unrecoverable error.  An
        attempt was made to correct the error.  Applications are unaffected.
action: Determine if the device needs to be replaced, and clear the errors
        using 'zpool clear' or replace the device with 'zpool replace'.
   see: http://illumos.org/msg/ZFS-8000-9P
  scan: scrub repaired 28.5K in 0h0m with 0 errors on Wed Feb 22 22:34:18 2017
config:

        NAME               STATE     READ WRITE CKSUM
        tank               DEGRADED     0     0     0
          /tmp/tank/vdev1  ONLINE       0     0     0
          /tmp/tank/vdev2  ONLINE       0     0     0
          /tmp/tank/vdev3  DEGRADED     0     0    24  too many errors

errors: No known data errors
```

Now, when we attempt to run `zpool reguid`, the command fails due to the
zpool being in an unhealthy state:
```
# zpool reguid tank
cannot reguid 'tank': one or more devices is currently unavailable
```
but if we use `zpool clear` to bring the pool back into a healthy state,
we can issue the `zpool reguid` command without issues:
```
# zpool clear tank
# zpool reguid tank
```
note: remember that `zpool scrub` will have corrected the damage
previously done by `dd`.

### Example Using a Damaged VDEV in a Non-Empty ZPOOL

Exanding on the previous example, we'll perform an almost identical
test, except we'll now fill the zpool with a 256MB file instead of using
an empty zpool. So, again, we start by creating the zpool and our 256MB
file:
```
# zpool export tank
# rm /tmp/tank/*
# mkfile -n 256m /tmp/tank/vdev1
# mkfile -n 256m /tmp/tank/vdev2
# mkfile -n 256m /tmp/tank/vdev3
# zpool create tank /tmp/tank/vdev{1,2,3}
# dd if=/dev/urandom of=/tank/file bs=1M count=256
256+0 records in
256+0 records out
268435456 bytes transferred in 4.641438 secs (57834542 bytes/sec)
# zpool list -v tank
NAME                SIZE  ALLOC   FREE  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
tank                720M   256M   464M         -    10%    35%  1.00x  ONLINE  -
  /tmp/tank/vdev1   240M  85.3M   155M         -    11%    35%
  /tmp/tank/vdev2   240M  85.3M   155M         -    10%    35%
  /tmp/tank/vdev3   240M  85.8M   154M         -    10%    35%
```

And now damaging the VDEV, and running `zpool scrub`:
```
# dd if=/dev/urandom of=/tmp/tank/vdev3 bs=1M seek=1 count=254
254+0 records in
254+0 records out
266338304 bytes transferred in 4.225401 secs (63032676 bytes/sec)
# zpool scrub tank
# zpool status -v tank
  pool: tank
 state: DEGRADED
status: One or more devices has experienced an error resulting in data
        corruption.  Applications may be affected.
action: Restore the file in question if possible.  Otherwise restore the
        entire pool from backup.
   see: http://illumos.org/msg/ZFS-8000-8A
  scan: scrub repaired 87.5K in 0h0m with 686 errors on Wed Feb 22 22:58:40 2017
config:

        NAME               STATE     READ WRITE CKSUM
        tank               DEGRADED     0     0   686
          /tmp/tank/vdev1  ONLINE       0     0     0
          /tmp/tank/vdev2  ONLINE       0     0     0
          /tmp/tank/vdev3  DEGRADED     0     0 1.37K  too many errors

errors: Permanent errors have been detected in the following files:

        /tank/file
```

As expected, there's "Permanent errors" detected to the file that we
created, but the pool itself should still be intact and reparied by the
`zpool scrub`. Thus, we'll use `zpool clear` to clear the errors like we
did in the previous example:
```
# zpool clear tank
# zpool status -v tank
  pool: tank
 state: ONLINE
status: One or more devices has experienced an error resulting in data
        corruption.  Applications may be affected.
action: Restore the file in question if possible.  Otherwise restore the
        entire pool from backup.
   see: http://illumos.org/msg/ZFS-8000-8A
  scan: scrub repaired 87.5K in 0h0m with 686 errors on Wed Feb 22 22:58:40 2017
config:

        NAME               STATE     READ WRITE CKSUM
        tank               ONLINE       0     0     0
          /tmp/tank/vdev1  ONLINE       0     0     0
          /tmp/tank/vdev2  ONLINE       0     0     0
          /tmp/tank/vdev3  ONLINE       0     0     0

errors: Permanent errors have been detected in the following files:

        /tank/file
```

At this point, I would expect that we could now use `zpool reguid`
on this zpool; even though there's "Permanent errors" in the user data
file, the pool is online an healthy.
```
# zdb -C tank | grep pool_guid
        pool_guid: 6847294342266961459
# zpool reguid tank
# zdb -C tank | grep pool_guid
        pool_guid: 13248255241205296188
```
and this is exactly the behavior seen.

[guid]: https://utcc.utoronto.ca/~cks/space/blog/solaris/ZFSGuids
[openzfs]: http://www.open-zfs.org
