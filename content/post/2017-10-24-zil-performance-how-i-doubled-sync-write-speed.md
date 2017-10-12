+++
date = "2017-10-24T00:00:00-08:00"
lastmod = "2017-10-24T00:00:00-08:00"
title = "ZIL Performance: How I Doubled Sync Write Speed"
type = "slides"
draft = false
+++

# Agenda

### 1. What is the ZIL?

### 2. How is it used? How does it work?

### 3. The problem to be fixed; the solution.

### 4. Details on the changes I made.

### 5. Performance testing and results.

.footnote[<sup>\*</sup>Press "p" for notes, and "c" for split view.]

---

class: middle, center

# 1 &ndash; What is the ZIL?

---

# What is the ZIL?

 - ZIL: Acronym for (Z)FS (I)ntent (L)og

    - Logs synchronous operations to disk, before `spa_sync()`

    - What constitutes a "synchronous operation"?

       - most _modifying_ ZPL operations:

          - e.g. `zfs_create`, `zfs_unlink`, `zfs_write` (some), etc.

       - doesn't include non-modifying ZPL operations:

          - e.g. `zfs_read`, `zfs_seek`, etc.

???

# What is the ZIL?

 - ZIL: Acronym for (Z)FS (I)ntent (L)og

    - Logs synchronous operations to disk, before `spa_sync()`

    - What constitutes a "synchronous operation"?

       - most _modifying_ ZPL operations:

          - e.g. `zfs_create`, `zfs_unlink`, `zfs_write` (sync), etc.

       - doesn't include non-modifying ZPL operations:

          - e.g. `zfs_read`, `zfs_seek`, etc.

---

# When is the ZIL used?

 - Always<sup>\*</sup>

    - ZPL operations (`itx`'s) logged via in-memory lists

    - lists of in-memory `itx`'s written to disk via `zil_commit()`

    - `zil_commit()` called for:

       - _any_ sync write

       - other sync operations (e.g. create, unlink), **and** `sync=always`

       - _some_ reads (`sync=always` or `FRSYNC` set)

.footnote[<sup>\*</sup>Except when dataset configured with: `sync=disabled`]

???

# When is the ZIL used?

 - Always<sup>\*</sup>

    - ZPL operations (`itx`'s) logged via in-memory lists

    - lists of in-memory `itx`'s written to disk via `zil_commit()`

    - `zil_commit()` called for:

       - _any_ sync write

       - other sync operations (e.g. create, unlink), **and** `sync=always`

       - _some_ reads (`sync=always` or `FRSYNC` set)

 - Caveat: None of this applies if dataset configured with: `sync=disabled`

    - in-memory `itx`'s aren't created

    - `zil_commit` doesn't do writes to disk

---

# What is the SLOG?

 - SLOG: Acronym for (S)eperate (LOG) Device

    - An SLOG is not necessary

    - An SLOG can be used to improve latency of ZIL writes

 - Conceptually, SLOG is different than the ZIL

 - ZIL is used, even if no SLOG attached

???

# What is the SLOG?

 - SLOG: Acronym for (S)eperate (LOG) Device

    - An SLOG is not necessary

       - By default (no SLOG), ZIL will write to main pool VDEVs

    - An SLOG can be used to improve latency of ZIL writes

       - When attached, ZIL writes to SLOG instead of main pool

 - Conceptually, SLOG is different than the ZIL

    - Difference between SLOG and ZIL, similar difference of DMU and VDEV

    - ZIL is mechanism for writing, SLOG is device written to

 - ZIL is used, even if no SLOG attached

---

# Why does the ZIL exist?

 - Writes in ZFS are "write-back"

 - Without the ZIL, sync operations inherit latency of `spa_sync()`<sup>\*</sup>

    - `spa_sync()` can take tens of seconds (or more) to complete

 - Further, with the ZIL, write amplification can be mitigated

 - ZIL is essentially a performance optimization

.footnote[<sup>\*</sup>All operations inherit this latency, but only sync operations wait for completion]
???

# Why does the ZIL exist?

 - Writes in ZFS are "write-back"

    - Data is first written and stored in-memory, in DMU layer...

    - At some later point, data for whole pool written to disk via `spa_sync()`.

 - Without ZIL, all sync operations inherit latency of `spa_sync()`

    - `spa_sync()` can take tens of seconds (or more) to complete

    - We don't want each sync operation to take this log

 - Further, with the ZIL, write amplification can be mitigated

    - A single ZPL operation, can cause many writes to occur

       - e.g. 1 block write to a file causes additional writes to update
         indirect blocks

    - ZIL allows operation to "complete" with minimal data written

 - ZIL is essentially a performance optimization

    - Strictly speaking, correctness could be achieved without it...

       - e.g. all sync ops could just wait for `spa_sync()` to complete

    - But practically speaking, it's necessary...

       - Performance would be unreasonably bad without it...

       - due to reasons just described

---

# ZIL On-Disk Format

 - Each dataset has it's own unique ZIL on-disk

 - ZIL stored on-disk as a singly linked list of ZIL blocks (`lwb`'s)

<hr style="visibility:hidden;" />

.center[![](zil-on-disk-format.svg)]

???

# ZIL On-Disk Format

 - Each dataset has it's own unique ZIL on-disk

 - ZIL stored on-disk as a singly linked list of ZIL blocks (`lwb`'s)

    - each ZIL block contains a pointer to the "next" ZIL block

    - no indirect blocks, so no write amplification

    - checksums calculated (and verified) differently

       - "next" checksum is "current" checksum, plus one
```
        error = zio_alloc_zil(...);
        if (error == 0) {
                ASSERT3U(bp->blk_birth, ==, txg);
                bp->blk_cksum = lwb->lwb_blk.blk_cksum;
                bp->blk_cksum.zc_word[ZIL_ZC_SEQ]++;

                /*
                 * Allocate a new log write block (lwb).
                 */
                nlwb = zil_alloc_lwb(zilog, bp, slog, txg);
        }
```

---

class: middle, center

# 2 &ndash; How is the ZIL used?

---

# How is the ZIL used?

 - ZPL will generally interact with the ZIL in two phases:

    1. Log the operation(s) &mdash; `zil_itx_assign`

    2. Commit the operation(s) &mdash; `zil_commit`

???

# How is the ZIL used?

 - ZPL will generally interact with the ZIL in two phases:

    1. Log the operation(s) &ndash; via `zil_itx_assign`

        - Tells the ZIL the operation occurred

        - Only called when _new_ ZPL operations occur

    2. Commit the operation(s)

        - Causes the ZIL to write log record of operation to disk

        - Only called if sync write, or `sync=always`

---

# Example: `zfs_write`

 - `zfs_write` &rarr; `zfs_log_write`

 - `zfs_log_write`

    &rarr; `zil_itx_create`

    &rarr; `zil_itx_assign`

 - `zfs_write` &rarr; `zil_commit`

 - Most ZPL operations have a corresponding `zfs_log_*` function

    - `zfs_log_create`
    - `zfs_log_remove`
    - `zfs_log_link`
    - `zfs_log_symlink`
    - `zfs_log_truncate`
    - `zfs_log_setattr`
    - ...

???

# Example: `zfs_write`

 - `zfs_write` first calls `zfs_log_write`

 - Within `zfs_log_write` we call:

    - `zil_itx_create` ...

    - and then, `zil_itx_assign`

 - These `zil_itx_*` functions register the write operation with the ZIL

 - Later, `zfs_write` will call `zil_commit` (if it's a sync write)

    - This causes ZIL to write previously generated write `itx` to disk

 - Most ZPL operations have a corresponding `zfs_log_*` function

    - All `zfs_log_*` functions similarly call `zil_itx_{create,assign}`

---

# Example: `zfs_fsync`

 - `zfs_fsync` &rarr; `zil_commit`

    - no _new_ operations to log... no `zfs_log_fysnc` function

???

# Example: `zfs_fsync`

 - As another example, `zfs_fsync` doesn't have a "log" function

    - `fsync` doesn't create any new modifications...

    - instead, it only ensures prior modifying operations complete.

    - as a result, there isn't a `zfs_log_fsync` function

---

# Contract between ZIL and ZPL.

 - Parameters to `zil_commit`: ZIL pointer, object number

    - These uniquely identify an object whose data is to be committed

 - When `zil_commit` returns:

    - Operations _relevant_ to the object specified, will be _persistent_
      on disk

    - relevant &ndash; all operations that would modify that object

    - persistent &ndash; Log block(s) written (completed) &rarr; disk flushed

 - Interface of `zil_commit` doesn't specify _which_ operation(s) to commit

???

# Contract between ZIL and ZPL.

 - Parameters to `zil_commit`: ZIL pointer, object number

    - These uniquely identify an object whose data is to be committed

 - When `zil_commit` returns:

    - Operations _relevant_ to the object specified, will be _persistent_
      on disk

    - relevant &ndash; all operations that would modify that object

       - "sync" and "async"...

       - e.g. "async" writes written along with "sync" writes

    - persistent &ndash; Log block(s) written (completed) &rarr; disk flushed

       - All "prior/dependent" log blocks written and completed

       - We can't issue the flush before the write completes...

          - or else, disk flush may not contain log block's contents.

             - concurrent write + flush can be reordered

          - if flush doesn't contain log block(s), flush was meaningless

 - Interface of `zil_commit` doesn't specify _which_ operation(s) to commit

    - `zil_commit` doesn't know which operation(s) the caller cares about...

       - thus, it must write all operations for the object

       - e.g. multiple threads writing to same object, but different offsets...

       - all offsets must be written before `zil_commit` returns

---

class: middle, center

# 2 &ndash; How does the ZIL work?

---

# How does the ZIL work?

 - In memory ZIL contains per-txg `itxg_t` structures

 - Each `itxg_t` contains:

    - A single list of sync operations (for all objects)

    - Object specific lists of async operations


???

 - In memory ZIL contains per-txg `itxg_t` structures

    - An `itxg_t` exists for each DMU TXG not yet synced to disk

    - Each `itx` is specific to a TXG...

       - assigned to corresponding `itxg_t` for that TXG

 - Each `itxg_t` contains:

    - A single list of sync operations (for all objects)

    - Object specific lists of async operations

---

# Example: itx lists

<br />

![](itx-lists.svg)

---

# How are itx's written to disk?

 - `zil_commit` handles the process of writing `itx_t`'s to disk:

???

# How are itx's written to disk?

 - `zil_commit` handles the process of writing `itx_t`'s to disk:

---

# How are itx's written to disk?

 - `zil_commit` handles the process of writing `itx_t`'s to disk:

     1. find all relavant `itx`'s, move them to the "commit list"

???

# How are itx's written to disk?

 - ~~`zil_commit` handles the process of writing `itx_t`'s to disk:~~

     1. find all relavant `itx`'s, move them to the "commit list"

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-1-01.svg)

???

# Example: `zil_commit` Object B

 - We'll start with the same itx lists from before...

 - Except now, we also have any empty commit list

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-1-02.svg)

???

# Example: `zil_commit` Object B

 - We're calling `zil_commit` for object "B"...

 - So the first step is to move object B's async `itx`'s to the sync list

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-1-03.svg)


???

# Example: `zil_commit` Object B

 - Nothing to say...

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-1-04.svg)

???

# Example: `zil_commit` Object B

 - Now we move the entire sync list to the commit list

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-1-05.svg)

???

# Example: `zil_commit` Object B

 - Finally, all of the `itx`'s that were linked off the sync list, are
   not linked off the commit list

 - So why do we do this?

    - Why do we move the `itx`'s to a new list, vs. simply using the
      sync list?

    - We do this so the thread that writes these `itx`'s out to disk...

       - can work with a list that won't change due to concurrent activity

    - If there's concurrent ZPL operations occurring...

       - those operations may insert new `itx`'s onto the sync list.

    - We create a new "commit list" so the list of `itx`s to write out...

       - is isolated from this concurrent ZPL activity

---

# How are itx's written to disk?

 - `zil_commit` handles the process of writing `itx_t`'s to disk:

     1. Move async itx's for object being commited, to the sync list

     2. Write all commit list `itx`'s to disk

???

# How are itx's written to disk?

 - ~~`zil_commit` handles the process of writing `itx_t`'s to disk:~~

     1. ~~Move async itx's for object being commited, to the sync list~~

     2. Write all commit list `itx`'s to disk

        - We do this by iterating over the `itx`'s in the commit list

        - For each `itx`:

           1. If insufficient space in the currently "open" ZIL block:

              - Allocate a new (empty) ZIL block...

              - issue write of old block

           2. Copy the `itx` into the "open" ZIL block

        - Finally, issue last "open" ZIL block to disk

           - this also means, allocated another block...

           - to be the next "open" ZIL block

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-2-01.svg)

???

# Example: `zil_commit` Object B

 - Starting with the commit list from before...

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-2-02.svg)

???

# Example: `zil_commit` Object B

 - Select the first `itx` in the list...

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-2-03.svg)

???

# Example: `zil_commit` Object B

 - allocate our first ZIL block...

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-2-04.svg)

???

# Example: `zil_commit` Object B

 - and copy the `itx` into the `lwb`'s buffer.

 - The `lwb` still hasn't been issued to disk yet...

    - it may be used for the next `itx`.

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-2-05.svg)

???

# Example: `zil_commit` Object B

 - Now we move on to the next `itx` in the list...

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-2-06.svg)

???

# Example: `zil_commit` Object B

 - This `itx` doesn't fit in the currently "open" lwb

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-2-07.svg)

???

# Example: `zil_commit` Object B

 - so we must allocate a new `lwb`...

    - and issue the "open" one to disk

 - The newly allocated `lwb` becomes the new "open" lwb

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-2-08.svg)

???

# Example: `zil_commit` Object B

 - Now we can copy the `itx` into the newly allocated, and "open", `lwb`

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-2-09.svg)

???

# Example: `zil_commit` Object B

 - And move on to the last `itx` in the list

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-2-10.svg)

???

# Example: `zil_commit` Object B

 - This fits in the currently "open" `lwb`...

 - so it can simply be copied into place

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-2-11.svg)

???

# Example: `zil_commit` Object B

 - And finally, we can issue the currently "open" lwb to disk

    - To do this, we must also allocate a new `lwb`...

       - to be the "next open" `lwb`

    - This "next open" lwb will used for the next "batch" of `itx`'s

---

# How are itx's written to disk?

 - `zil_commit` handles the process of writing `itx_t`'s to disk:

     1. Move async itx's for object being commited, to the sync list

     2. Write all commit list `itx`'s to disk

     3. Wait for all ZIL block writes to complete

???

# How are itx's written to disk?

 - ~~`zil_commit` handles the process of writing `itx_t`'s to disk:~~

     1. ~~Move async itx's for object being commited, to the sync list~~

     2. ~~Write all commit list `itx`'s to disk~~

     3. Wait for all ZIL block writes to complete

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-3-01.svg)

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-3-02.svg)

???

# Example: `zil_commit` Object B

 - The `lwb`'s can complete in any order

 - In this example, "lwb 2" completes first...

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-3-03.svg)

???

# Example: `zil_commit` Object B

 - Then "lwb 1"...

 - At this point, all `lwb`'s have completed.

---

# How are itx's written to disk?

 - `zil_commit` handles the process of writing `itx_t`'s to disk:

     1. Move async itx's for object being commited, to the sync list

     2. Write all commit list `itx`'s to disk

     3. Wait for all ZIL block writes to complete

     4. Flush VDEVs and notify waiting threads

???

# How are itx's written to disk?

 - ~~`zil_commit` handles the process of writing `itx_t`'s to disk:~~

     1. ~~Move async itx's for object being commited, to the sync list~~

     2. ~~Write all commit list `itx`'s to disk~~

     3. ~~Wait for all ZIL block writes to complete~~

     4. Flush VDEVs and notify waiting threads

---

class: middle, center

# 2 &ndash; ZIL Block Sizing + Performance

---

# ZIL Block Sizing + Performance

 - ZIL blocks must be "pre-allocated", due to on-disk format

    - Block size chosen at time of allocation

 - Allocated block size can dramatically impact performance:

    - "too big" &ndash; wasted space

    - "too small" &ndash; too many (small) IOPs issued to disk

    - "just right" &ndash; large IOPs filled with `itx`'s

???

# ZIL Block Sizing + Performance

 - ZIL blocks must be "pre-allocated", due to on-disk format

    - Each ZIL block contains pointer to next ZIL block on disk...

       - Thus, "next" block allocated when the "current" is issued to disk

    - Block size chosen at time of allocation

 - Allocated block size can dramatically impact performance:

    - Because blocks "pre-allocated"...

       - the size of "next" block is decided before number of `itx`'s is known

    - "too big" &ndash; wasted space

       - Block only partially filled with `itx`'s

       - Can result in small, but fast, SLOG filling to capacity

          - full SLOG can cause poor performance

    - "too small" &ndash; too many (small) IOPs issued to disk

       - Can cause disk saturation and poor performance

    - "just right" &ndash; large IOPs filled with `itx`'s

       - ZIL blocks fully utilized with `itx`'s (no wasted SLOG space)

       - Fewer IOPs for same number of `itx`'s

          - e.g. 15 8K writes using 1 128K lwb vs. 4 32K lwb's

---

class: middle, center

# 3 &ndash; Problem

---

# Problem

 1. `itx`'s grouped and written in "batches"

    - The commit list constitutes a batch

    - Batch size proportional to sync workload on system

 2. Waiting threads only notified when _all_ ZIL blocks in batch complete

 3. Only a single batch processed at a time

---

# Example Batch

<br />

![](problem-01.svg)

???

# Example Batch

 - Example batch taken from prior slides

---

# Example "itx S1"

<br />

![](problem-02.svg)

???

# Example "itx S1"

 - Thread waiting for "itx S1"

    - e.g. a single sync write

---

# Example "itx S1"

<br />

![](problem-03.svg)

???

# Example "itx S1"

 - Must wait for **all** lwb's to be written and completed

---

# Implications

 1. `zil_commit` latency proportional to system workload, _not_ disk latency

 2. Disk "anomalies" &rarr; larger batches &rarr; increased `zil_commit` latency

 3. New calls to `zil_commit` wait for "current" batch, _and_ "next" batch

???

# Implications

 1. `zil_commit` latency proportional to system workload, _not_ disk latency

    - Fast SLOG may not compensate for large workload

 2. Disk "anomalies" &rarr; larger batches &rarr; increased `zil_commit` latency

    - E.g. Temporary network delays when using SAN storage

 3. New calls to `zil_commit` wait for "current" batch, _and_ "next" batch

    - Average `zil_commit` latency equal to latency of 1.5 batches

---

class: middle, center

# 3 &ndash; Solution

---

# Solution

 - Remove concept of "batches":

    1. Allow `zil_commit` to issue new ZIL block writes immediately

    2. Notify threads immediately when _dependent_ `itx`'s on disk

???

# Solution

 - Remove concept of "batches":

    1. Allow `zil_commit` to issue new ZIL block writes immediately

       - Rather than waiting for "current" batch to complete

    2. Notify threads immediately when _dependent_ `itx`'s on disk

       - Rather than when **all** writes complete (dependent or not)

---

# Example "Batch"

<br />

![](solution-01.svg)

???

# Example "Batch"

 - Same example as before

---

# Example "itx S1"

<br />

![](solution-02.svg)

???

# Example "itx S1"

 - Thread waiting for "itx S1" (just like before)

---

# Example "itx S1"

<br />

![](solution-03.svg)

???

# Example "itx S1"

 - Must wait for _only_ "lwb 1" to complete

 - Previously, would have also waited for "lwb 2"

    - Might not seem significant for a small example with 2 `lwb`'s...

    - But it can be significant with 100's of `lwb`'s on a real system

---

class: middle, center

# 4 &ndash; Changes to VDEV Flush

---

# Details

 - A ZIL block is not "persistent" until the VDEV is flushed

 - Prior mechanics:

    - Single VDEV flush for each VDEV, after batch completes

    - 1 flush per many lwb's

 - New mechanics:

    - VDEV flush issued after each ZIL block written

    - 1 flush per 1 lwb

???

# Details

 - A ZIL block is not "persistent" until the VDEV is flushed

 - Prior mechanics:

    - Single VDEV flush for each VDEV, after batch completes

    - 1 flush per many lwb's

 - New mechanics:

    - VDEV flush issued after each ZIL block written

    - 1 flush per 1 lwb

---

# Example: Before

<br />

.center[![](flush-sequence-before.svg)]

???

# Example: Before

 - 2 `lwb`'s are written to VDEV 1...

    - but only a single flush is issued to it.

 - Additionally, all flushes are issued at the very end.

---

# Example: After

<br />

.center[![](flush-sequence-after.svg)]

???

# Example: After

 - The same 2 `lwb`'s are written to VDEV 1...

    - but 2 flushes are issued to it.

 - Additionally, the flushes are intertwined with other activity.

---

class: middle, center

# 4 &ndash; Changes to ZIL Block ZIO Tree

---

# Details

 - ZIL blocks issued to disk using ZIOs

 - Prior mechanics:

    - "root" ZIO created for each batch

       - "write" ZIOs, for all `lwb`'s in batch, are children of root ZIO

    - "flush" ZIOs issued separately after root ZIO completes

 - New mechanics:

    - "root" ZIO created for each lwb

       - "write" and "flush" ZIOs are child of root ZIO

    - "next" lwb root ZIO become parent of "current" lwb root ZIO

???

# Details

 - ZIL blocks issued to disk using ZIOs

    - ZIOs represented as directed acyclic graph (DAG)

    - Parent ZIOs cannot complete until all children complete

 - Prior mechanics:

    - "root" ZIO created for each batch

       - "write" ZIOs, for all `lwb`'s in batch, are children of root ZIO

    - "flush" ZIOs issued separately after root ZIO completes

 - New mechanics:

    - "root" ZIO created for each lwb

       - "write" and "flush" ZIOs are child of root ZIO

    - "next" lwb root ZIO become parent of "current" lwb root ZIO

---

# Example: Before

<br />

.center[![](zio-tree-before-01.svg)]

???

# Example: Before

 - First, create the batch root ZIO

---

# Example: Before

<br />

.center[![](zio-tree-before-02.svg)]

???

# Example: Before

 - Then the lwb "write" ZIOs are created as children

---

# Example: Before

<br />

.center[![](zio-tree-before-03.svg)]

???

# Example: Before

 - Nothing to say...

---

# Example: Before

<br />

.center[![](zio-tree-before-04.svg)]

???

# Example: Before

 - Nothing to say...

---

# Example: Before

<br />

.center[![](zio-tree-before-05.svg)]

???

# Example: Before

 - After all lwb "write" ZIOs complete...

 - we create the root ZIO for the VDEV flush commands

---

# Example: Before

<br />

.center[![](zio-tree-before-06.svg)]

???

# Example: Before

 - And issue the flush to the first VDEV

---

# Example: Before

<br />

.center[![](zio-tree-before-07.svg)]

???

# Example: Before

 - And now the second VDEV

 - Once both of these flushes complete, we're done...

 - and notify the waiting threads

---

# Example: After

![](zio-tree-after-01.svg)

???

# Example: After

 - Now, the ZIO tree is drastically different, as we'll soon see

 - In the new code, first we create a root ZIO...

 - that is specific to the lwb being written

---

# Example: After

![](zio-tree-after-02.svg)

???

# Example: After

 - Then we create a "write" ZIO for the same lwb...

 - It's this ZIO that contains the actual data that will be written..

 - Containing the `itx`'s that need to be persisted

---

# Example: After

![](zio-tree-after-03.svg)

???

# Example: After

 - Then, after the "write" ZIO completes...

 - we'll issue the flush ZIO

 - This flush must not be issued until the "write" completes

---

# Example: After

![](zio-tree-after-04.svg)

???

# Example: After

 - The next lwb "root" ZIO will become a parent of the prior lwb "root"

---

# Example: After

![](zio-tree-after-05.svg)

???

# Example: After

 - And like before, it'll have a "write" ZIO as a child

---

# Example: After

![](zio-tree-after-06.svg)

???

# Example: After

 - But, for the sake of example...

    - let's say the next `lwb` is issued before this "write" completes.

 - In this case, the "write" will still be outstanding...

    - but a new `lwb` can be created and issued.

 - This new `lwb` will, again, be a parent of the prior `lwb`

---

# Example: After

![](zio-tree-after-07.svg)

???

# Example: After

 - And this `lwb` will also have a child "write" ZIO

---

# Example: After

![](zio-tree-after-08.svg)

???

# Example: After

 - It's possible, that the "write" for "lwb 3" will complete before "lwb 2"

 - In which case, "lwb 3" will issue the flush...

    - even though "lwb 2" is still outstanding

 - It's important to note...

    - "lwb 3" can't "complete" until "lwb 2" is complete

    - this is enforced by the ZIO layer

       - a "parent" ZIO can't complete until all children complete

---

# Example: After

![](zio-tree-after-09.svg)

???

# Example: After

 - Then, later, when the "write" for "lwb 2" completes...

    - The flush will be issued

 - Since "lwb 3" may have already completed it's IO...

    - it's possible for "lwb 2" and "lwb 3" to complete "simultaneously"

---

# Example: After

![:scale 100%](zio-tree-after-10.svg)

???

# Example: After

 - This pattern will continue...

    - as long as there's `lwb`'s to issue to disk

---

# Example: After

![:scale 100%](zio-tree-after-11.svg)

???

# Example: After

 - As lwb ZIOs complete...

    - they'll simply "drop off" from the bottom of the tree

---

# Example: After

![:scale 100%](zio-tree-after-12.svg)

???

# Example: After

 - Nothing to say...

---

# Example: After

![](zio-tree-after-13.svg)

???

# Example: After

 - Here, both "lwb 1" and "lwb 2" have completed

    - so they've been removed from the tree

---

# Example: After

![](zio-tree-after-14.svg)

---

class: middle, center

# 4 &ndash; Changes to Waiter Notification

---

# Details: Before

 - 2 condition variables (CV), for "current" and "next" batch

 - Threads that called `zil_commit`:

    - Assigned to "next batch", wait on next batch's CV

 - When "current" batch completes

    1. All threads waiting on "current" signalled, they'd return

    2. One thread waiting on "next" signalled, becomes "writer"

    3. "next" and "current" CV swapped

 - Ultimately, these two CVs are the source of original problem

---

# Example: Before

.center[![](waiter-sequence-before-01.svg)]

---

# Example: Before

.center[![](waiter-sequence-before-02.svg)]

---

# Details: After

 - Each time a process calls `zil_commit`:

    - A new CV is allocated for this specific process to wait on

    - A new `TX_COMMIT` itx is inserted into the ZIL itx tree

       - The "commit itx" has a pointer to the process's CV

 - When a commit itx is copied to an lwb:

    - No data copied into the lwb's buffer

    - Instead, itx's CV added to lwb's list of CVs

 - When lwb's ZIO completes, list of CVs iterated and signalled

    - This is how we map which lwb a process is waiting for

---

# Example: After

.center[![](waiter-sequence-after-01.svg)]

---

# Example: After

.center[![](waiter-sequence-after-02.svg)]

---

class: middle, center

# 5 &ndash; Performance testing and results.

---

# Details

 - Two `fio` workloads used to drive a sync write workload

    1. `fio` was trying to perform sync writes as fast as it could

    2. `fio` was trying to perform 64 sync writes per second

 - IOPs and latency measured with and without my changes

    - Other metrics also observed (`iostat`, flamegraphs, lwb info, etc.)

 - 1, 2, 4, and 8 disk zpools; tested both SSD and HDD

 - Full details can be found [here][perf-results]

---

class: middle, center

# 5 &ndash; Max Rate Workload &ndash; HDDs

---

background-image: url(max-rate-hdd-iops-pctchange.png)
background-size: 95%

# % Change IOPs &ndash; Max Rate &ndash; HDDs

---

background-image: url(max-rate-hdd-lat-pctchange.png)
background-size: 95%

# % Change Latency &ndash; Max Rate &ndash; HDDs

---

class: middle, center

# 5 &ndash; Max Rate Workload &ndash; SSDs

---

background-image: url(max-rate-ssd-iops-pctchange.png)
background-size: 95%

# % Change IOPs &ndash; Max Rate &ndash; SSDs

---

background-image: url(max-rate-ssd-lat-pctchange.png)
background-size: 95%

# % Change Latency &ndash; Max Rate &ndash; SSDs

---

class: middle, center

# 5 &ndash; Fixed Rate Workload &ndash; HDDs

---

background-image: url(fixed-rate-hdd-iops-pctchange.png)
background-size: 95%

# % Change IOPs &ndash; Fixed Rate &ndash; HDDs

---

background-image: url(fixed-rate-hdd-lat-pctchange.png)
background-size: 95%

# % Change Latency &ndash; Fixed Rate &ndash; HDDs

---

class: middle, center

# 5 &ndash; Fixed Rate Workload &ndash; SSDs

---

background-image: url(fixed-rate-ssd-iops-pctchange.png)
background-size: 95%

# % Change IOPs &ndash; Fixed Rate &ndash; SSDs

---

background-image: url(fixed-rate-ssd-lat-pctchange.png)
background-size: 95%

# % Change Latency &ndash; Fixed Rate &ndash; SSDs

[perf-results]: https://www.prakashsurya.com/post/2017-09-08-performance-testing-results-for-openzfs-447/
