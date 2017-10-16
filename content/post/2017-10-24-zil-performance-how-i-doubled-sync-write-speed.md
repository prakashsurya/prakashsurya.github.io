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

    - What operations get logged?

       - `zfs_create`, `zfs_remove`, `zfs_write`, etc.

       - Doesn't include non-modifying ZPL operations:

          - `zfs_read`, `zfs_seek`, etc.

    - What gets logged?

       - The fact that a logical operation is occuring is logged

          - `zfs_remove` &rarr; directory object ID + name only

       - Not logging which blocks will change due to logical operation

---

# When is the ZIL used?

 - Always<sup>\*</sup>

    - ZPL operations (`itx`'s) logged via in-memory lists

    - lists of in-memory `itx`'s written to disk via `zil_commit()`

    - `zil_commit()` called for:

       - _any_ sync write<sup>\**</sup>

.footnote[<sup>\*</sup>Except when dataset configured with: `sync=disabled`.
          <sup>\**</sup>Except when dataset configured with: `sync=always`.]

---

# What is the SLOG?

 - SLOG: Acronym for (S)eperate (LOG) Device

    - An SLOG is not necessary

       - By default (no SLOG), ZIL will write to main pool VDEVs

    - An SLOG can be used to improve latency of ZIL writes

       - When attached, ZIL writes to SLOG instead of main pool<sup>\*</sup>

 - Conceptually, SLOG is different than the ZIL

   - ZIL is mechanism for writing, SLOG is device written to

 - ZIL is used, even if no SLOG attached

.footnote[<sup>\*</sup>For some operations; see code for details.]

---

# Why does the ZIL exist?

 - Writes in ZFS are "write-back"

    - Data is first written and stored in-memory, in DMU layer

    - Later, data for whole pool written to disk via `spa_sync()`

 - Without the ZIL, sync operations could wait for `spa_sync()`

    - `spa_sync()` can take tens of seconds (or more) to complete

 - Further, with the ZIL, write amplification can be mitigated

    - A single ZPL operation can cause many writes to occur

    - ZIL allows operation to "complete" with minimal data written

 - ZIL needed to provide "fast" synchronous semantics to applications

    - Correctness could be acheived without it, but would be "too slow"

---

# ZIL On-Disk Format

 - Each dataset has it's own unique ZIL on-disk

 - ZIL stored on-disk as a singly linked list of ZIL blocks (`lwb`'s)

<hr style="visibility:hidden;" />

.center[![](zil-on-disk-format.svg)]

---

class: middle, center

# 2 &ndash; How is the ZIL used?

---

# How is the ZIL used?

 - ZPL will generally interact with the ZIL in two phases:

    1. Log the operation(s) &mdash; `zil_itx_assign`

        - Tells the ZIL an operation is occuring

    2. Commit the operation(s) &mdash; `zil_commit`

        - Causes the ZIL to write log record of operation to disk

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

- These `zil_itx_*` functions register the write operation with the ZIL

- Later, `zfs_write` will call `zil_commit` (if it's a sync write)

   - This causes ZIL to write previously generated write `itx` to disk

- All `zfs_log_*` functions similarly call `zil_itx_{create,assign}`

---

# Example: `zfs_fsync`

 - `zfs_fsync` &rarr; `zil_commit`

   - `fsync` doesn't create any new modifications

   - only writes previous `itx`'s to disk

      - thus, no `zfs_log_fsync` function

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

- _relevant &ndash; all operations that would modify that object_

   - "sync" and "async"...

   - e.g. "async" writes written along with "sync" writes

- _persistent &ndash; Log block(s) written (completed) &rarr; disk flushed_

   - All "prior/dependent" log blocks written and completed

   - We can't issue the flush before the write completes...

      - or else, disk flush may not contain log block's contents.

         - concurrent write + flush can be reordered

      - if flush doesn't contain log block(s), flush was meaningless

- _Interface of `zil_commit` doesn't specify _which_ operation(s) to commit_

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

- _In memory ZIL contains per-txg `itxg_t` structures_

   - An `itxg_t` exists for each DMU TXG not yet synced to disk

   - Each `itx` is specific to a TXG...

      - assigned to corresponding `itxg_t` for that TXG

---

# Example: itx lists

<br />

![](itx-lists.svg)

---

# How are itx's written to disk?

 - `zil_commit` handles the process of writing `itx_t`'s to disk:

???

---

# How are itx's written to disk?

 - `zil_commit` handles the process of writing `itx_t`'s to disk:

     1. find all relevant `itx`'s, move them to the "commit list"

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-1-01.svg)

???

 - Same itx lists as before... plus the commit list

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-1-02.svg)

???

- Calling `zil_commit` for object "B"

- First step is to move object B's async `itx`'s to the sync list

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-1-03.svg)

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-1-04.svg)

???

- Now we move the entire sync list to the commit list

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-1-05.svg)

???

- We create the "commit list" so the list of `itx`s to write out...

   - is isolated from this concurrent ZPL activity

---

# How are itx's written to disk?

 - `zil_commit` handles the process of writing `itx_t`'s to disk:

     1. ~~Move async itx's for object being commited, to the sync list~~

     2. Write all commit list `itx`'s to disk

???

- We do this by iterating over the `itx`'s in the commit list

- For each `itx`:

   1. If insufficient space in the currently "open" ZIL block:

      - Allocate a new (empty) ZIL block, issue write of old block

   2. Copy the `itx` into the "open" ZIL block

- Last, we issue last "open" ZIL block to disk

   - this also means, allocated another block...

   - to be the next "open" ZIL block

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-2-03.svg)

???

- Starting with a ZIL header and `lwb` block already allocated...

- Select the first `itx` in the list...

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-2-04.svg)

???

- and copy the `itx` into the `lwb`'s buffer.

- The `lwb` still hasn't been issued to disk yet...

   - it may be used for the next `itx`.

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-2-05.svg)

???

- Now we move on to the next `itx` in the list...

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-2-06.svg)

???

- This `itx` doesn't fit in the currently "open" lwb

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-2-07.svg)

???

- so we must allocate a new `lwb`, and issue the "open" one to disk

- The newly allocated `lwb` becomes the new "open" lwb

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-2-08.svg)

???

- Now we can copy the `itx` into the new `lwb`

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-2-09.svg)

???

- Next, we move on to the last `itx` in the list

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-2-10.svg)

???

- This fits in the currently "open" `lwb`, so it's copied into place

- At this point, the commit list is empty...

   - so we must issue the current `lwb` to disk.

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-2-11.svg)

???

- To do that, we must allocate a new `lwb`

- This new `lwb` will then be used for the next "batch" of `itx`'s

---

# How are itx's written to disk?

 - `zil_commit` handles the process of writing `itx_t`'s to disk:

     1. ~~Move async itx's for object being commited, to the sync list~~

     2. ~~Write all commit list `itx`'s to disk~~

     3. Wait for all ZIL block writes to complete

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-3-01.svg)

???

- The `lwb`'s can complete in any order...

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-3-02.svg)

???

- here, "lwb 2" completes first...

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-3-03.svg)

???

- and then "lwb 1".

- At this point, all `lwb`'s have been written and writes have completed.

---

# How are itx's written to disk?

 - `zil_commit` handles the process of writing `itx_t`'s to disk:

     1. ~~Move async itx's for object being commited, to the sync list~~

     2. ~~Write all commit list `itx`'s to disk~~

     3. ~~Wait for all ZIL block writes to complete~~

     4. Flush VDEVs

---

# How are itx's written to disk?

 - `zil_commit` handles the process of writing `itx_t`'s to disk:

     1. ~~Move async itx's for object being commited, to the sync list~~

     2. ~~Write all commit list `itx`'s to disk~~

     3. ~~Wait for all ZIL block writes to complete~~

     4. ~~Flush VDEVs~~

     5. Notify waiting threads

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

# Problem

![](problem.svg)

 - Time spent servicing `lwb`'s for each disk

 - Color indicates order waiting threads notified

---

# Implications

 1. `zil_commit` latency proportional to system workload, _not_ disk latency

   - Fast SLOG may not compensate for large workload

 2. Disk "anomalies" &rarr; larger batches &rarr; increased `zil_commit` latency

   - e.g. temporary network delays when using network storage

 3. New calls to `zil_commit` wait for "current" batch, _and_ "next" batch

   - Average `zil_commit` latency equal to latency of 1.5 batches

---

class: middle, center

# 3 &ndash; Solution

---

# Solution

 - Remove concept of "batches":

    1. Allow `zil_commit` to issue new ZIL block writes immediately

       - In contrast to waiting for the current batch to complete

    2. Notify threads immediately when _dependent_ `lwb`'s on disk

       - In contrast to waiting for _all_ `lwb`'s on disk

???

1. _Allow `zil_commit` to issue new ZIL block writes immediately_


2. _Notify threads immediately when dependent `itx`'s on disk_

   - In contrast to waiting for **all** `itx`'s on disk (dependent or not)

---

# Problem

![](problem.svg)

 - Time spent servicing `lwb`'s for each disk

 - Color indicates order waiting threads notified

---

# Solution

![](solution.svg)

 - Time spent servicing `lwb`'s for each disk

 - Color indicates order waiting threads notified

---

class: middle, center

# 4 &ndash; Details on the Changes I Made

---

background-image: url(changes-before-01.svg)
background-size: 65%

## Before

???

- Bulk of changes I made revolves around the following 3 things:

   1. how `lwb`'s are issued to disk

   2. how flush commands are sent to the disks

   3. how waiting threads are notified

- Previously, this was a mostly a sequential 3 step process...

---

background-image: url(changes-before-02.svg)
background-size: 65%

## Before

???

- Step 1 would consist of creating the `lwb`'s...

   - issuing these to disk...

   - and then waiting for the IO for all the `lwb`'s to complete.

---

background-image: url(changes-before-03.svg)
background-size: 65%

## Before

???

- Once that Step 1 completes...

   - Step 2 would consist of issuing the flush commands to each VDEV...

   - and then waiting for all of them to complete.

---

background-image: url(changes-before-04.svg)
background-size: 65%

## Before

???

- And then, _finally_, after all flushes have completed...

   - the ZIL's CV is signalled to notify any waiting threads...

   - letting them know their data is safe on disk.

- All 3 of these steps would consist of a single batch.

---

background-image: url(changes-before-05.svg)
background-size: 65%

## Before

???

- And, after one batch completes...

   - another would start up again.

---

background-image: url(changes-before-06.svg)
background-size: 65%

## Before

???

- And another.

- This would repeat as long as...

   - there was a continuous stream of incoming sync operations to process.

---

background-image: url(changes-after-01.svg)
background-size: 85%

## After

???

- Now the process is entirely different...

   - and heavily leaverages the ZIO infrastructure

- Each `lwb` will have a root ZIO, which will eventually have 2 children:

   1. A "write" ZIO containing the `itx` data to be written

   2. A "flush" ZIO that will be issued after the "write" ZIO completes

- Since these are children, the `lwb`'s "root" ZIO...

   - cannot complete until both the "write" and the "flush" complete

   - This is enforced by the ZIO parent-child dependency semantics

- Further, the "root" ZIO of each "previous" `lwb`...

   - will be a "child" of the "root" ZIO for the "next" `lwb`

   - This ensures the `lwb`'s completes in correct order...

      - again, leveraging ZIO dependencies to acheive this ordering

- Additionally, each `lwb` maintains a list of CVs...

   - Each CV on this list maps to a waiting thread

   - When the "root" ZIO for the `lwb` completes...

      - each CV in this list is signalled...

      - which notifies the waiting thread that it's data is safe on disk

- Let's walk through an example of what this looks like...

---

background-image: url(changes-after-02.svg)
background-size: 85%

## After

???

- First, the "root" ZIO for the `lwb` is created...

---

background-image: url(changes-after-03.svg)
background-size: 85%

## After

???

- And then the "write" ZIO is issued...

---

background-image: url(changes-after-04.svg)
background-size: 85%

## After

???

- When that "write" ZIO completes, the flush will immediately be issued

---

background-image: url(changes-after-05.svg)
background-size: 85%

## After

???

- Then, when the "flush" ZIO completes...

- that'll allow the "root" ZIO for this specific `lwb` to complete...

- at which point, each CV in the `lwb`'s list of CVs will be signalled

---

background-image: url(changes-after-06.svg)
background-size: 85%

## After

???

- The same process would occur of the next `lwb`...

---

background-image: url(changes-after-07.svg)
background-size: 85%

## After

???

- And the next.

- It's important to note...

   - the sequence of events doesn't have to occur in this order

- For example, it's possible for "lwb 2" to have issued it's...

   - "write" ZIO before the "write" of "lwb 1" completes.

- It can even complete before the "write" of "lwb 1"...

   - Same goes for the "flush" for "lwb 2"

- **But**, due to the ZIO dependency tree, the "root" ZIO for "lwb 2"...

   - cannot complete until the "root" for "lwb 1" completes.

- Thus, the CVs for "lwb 2" will never be signalled...

   - prior to "lwb 1" completing

- **So**, while this example might imply...

   - this process is still very sequential, just like before...

   - it doesn't have to be...

- Now, it's all driven by the incoming workload...

   - and disk completion events.

   - `lwb`'s are created and issued whenever there's data to write...

   - and waiting threads are notified whenever those writes complete.

---

# New Tunable: `lwb` Timeout

.center[![:scale 90%](lwb-timeout.svg)]

.footnote[<sup>\*</sup>New tunable named: `zfs_commit_timeout_pct`]

???

- Prior to issuing an `lwb` to disk, it remains "open"...

   - such that we can continue to use this `lwb`...

   - for any future `itx`'s that occur, and would fit in the `lwb`

- This is done to help fill `lwb` blocks with useful `itx`'s

   - in an attempt to use as few `lwb`'s as possible...

   - for as many `itx`'s as possible.

- This is critical to maintain good performance...

   - with a continuous stream of incoming sync operations.

- But, if incoming sync operations stop...

   - we need to bound the amount of time this "open" lwb waits...

   - before it is issued to disk, partially filled.

- We determine the amount of time to wait...

   - by waiting a percentage of the last `lwb` write's latency.

- By default, wait 5% of the last `lwb` write's latency...

- If the "open" `lwb` isn't filled in that amount of time...

   - it will "timeout", and be issued to disk partially filled.

---

class: middle, center

# 5 &ndash; Performance testing and results

---

background-image: url(max-rate-hdd-iops-pctchange.svg)
background-size: 115%

## ~83% Increase in IOPs on Average &ndash; Max Rate &ndash; 8 HDDs

---

background-image: url(max-rate-ssd-iops-pctchange.svg)
background-size: 115%

## ~48% Increase in IOPs on Average &ndash; Max Rate &ndash; 8 SSDs

---

background-image: url(fixed-rate-hdd-lat-pctchange.svg)
background-size: 115%

## ~27% Decrease in Latency on Average &ndash; Fixed Rate &ndash; 8 HDDs

.footnote[<sup>\*</sup>IOPs increased with new code, and >64 threads; those data points omitted.]

---

background-image: url(fixed-rate-ssd-lat-pctchange.svg)
background-size: 115%

## ~16% Decrease in Latency on Average &ndash; Fixed Rate &ndash; 8 SSDs

---

# More Details

 - Two `fio` workloads were used:

    1. each thread submitting sync writes as fast as it could

    2. each thread submitting 64 sync writes per second

 - 1, 2, 4, and 8 disk zpools; both SSD and HDD

 - `fio` threads ranging from 1 to 1024; increasing in powers of 2

 - Full details can be found [here][perf-results]

[perf-results]: https://www.prakashsurya.com/post/2017-09-08-performance-testing-results-for-openzfs-447/
