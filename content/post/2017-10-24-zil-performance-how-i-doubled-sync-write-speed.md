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

- _An SLOG is not necessary_

   - By default (no SLOG), ZIL will write to main pool VDEVs

- _An SLOG can be used to improve latency of ZIL writes_

   - When attached, ZIL writes to SLOG instead of main pool

- _Conceptually, SLOG is different than the ZIL_

   - Difference between SLOG and ZIL, similar difference of DMU and VDEV

   - ZIL is mechanism for writing, SLOG is device written to

---

# Why does the ZIL exist?

 - Writes in ZFS are "write-back"

 - Without the ZIL, sync operations inherit latency of `spa_sync()`<sup>\*</sup>

    - `spa_sync()` can take tens of seconds (or more) to complete

 - Further, with the ZIL, write amplification can be mitigated

 - ZIL is essentially a performance optimization

.footnote[<sup>\*</sup>All operations inherit this latency, but only sync operations wait for completion]
???

- _Writes in ZFS are "write-back"_

   - Data is first written and stored in-memory, in DMU layer...

   - At some later point, data for whole pool written to disk via `spa_sync()`.

- _Without ZIL, all sync operations inherit latency of `spa_sync()`_

   - We don't want each sync operation to take this log

- _Further, with the ZIL, write amplification can be mitigated_

   - A single ZPL operation, can cause many writes to occur

     - e.g. 1 block write to a file causes additional writes to update
       indirect blocks

   - ZIL allows operation to "complete" with minimal data written

- _ZIL is essentially a performance optimization_

  - Strictly speaking, correctness could be achieved without it...

       - e.g. all sync ops could just wait for `spa_sync()` to complete

    - But practically speaking, it's necessary...

       - Performance would be unreasonably bad without it

---

# ZIL On-Disk Format

 - Each dataset has it's own unique ZIL on-disk

 - ZIL stored on-disk as a singly linked list of ZIL blocks (`lwb`'s)

<hr style="visibility:hidden;" />

.center[![](zil-on-disk-format.svg)]

???

- _ZIL stored on-disk as a singly linked list of ZIL blocks (`lwb`'s)_

   - each ZIL block contains a pointer to the "next" ZIL block

---

class: middle, center

# 2 &ndash; How is the ZIL used?

---

# How is the ZIL used?

 - ZPL will generally interact with the ZIL in two phases:

    1. Log the operation(s) &mdash; `zil_itx_assign`

    2. Commit the operation(s) &mdash; `zil_commit`

???

1. _Log the operation(s)_

   - Tells the ZIL the operation occurred

   - Only called when _new_ ZPL operations occur

2. _Commit the operation(s)_

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

- These `zil_itx_*` functions register the write operation with the ZIL

- Later, `zfs_write` will call `zil_commit` (if it's a sync write)

   - This causes ZIL to write previously generated write `itx` to disk

- All `zfs_log_*` functions similarly call `zil_itx_{create,assign}`

---

# Example: `zfs_fsync`

 - `zfs_fsync` &rarr; `zil_commit`

    - no _new_ operations to log... no `zfs_log_fysnc` function

???

- `zfs_fsync` doesn't have a "log" function

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

     1. find all relavant `itx`'s, move them to the "commit list"

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

- So, why do we move sync list `itx`'s to the commit list?

   - So the thread that writes these `itx`'s out to disk...

      - can work with a list that won't change due to concurrent activity

   - If there's concurrent ZPL operations occurring...

      - those operations may insert new `itx`'s onto the sync list.

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

![](zil-commit-2-01.svg)

???

- Starting with the commit list from before...

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-2-02.svg)

???

- Select the first `itx` in the list...

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-2-03.svg)

???

- allocate our first ZIL block...

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

- _ZIL blocks must be "pre-allocated", due to on-disk format_

   - Each ZIL block contains pointer to next ZIL block on disk...

      - Thus, "next" block allocated when the "current" is issued to disk

- _Allocated block size can dramatically impact performance:_

   - Because blocks "pre-allocated"...

      - the size of "next" block is decided before number of `itx`'s is known

- _"too big"_

   - Block only partially filled with `itx`'s

   - Can result in small, but fast, SLOG filling to capacity

      - full SLOG can cause poor performance

- _"too small"_

   - Can cause disk saturation and poor performance

- _"just right"_

   - ZIL blocks fully utilized with `itx`'s (no wasted SLOG space)

   - Fewer IOPs for same number of `itx`'s

      - e.g. 15 8K writes using 1 128K lwb vs. 4 32K lwb's

---

class: middle, center

# 3 &ndash; Problem

---

# Problem(s)

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

- Same batch from prior slides...

---

# Example "itx S1"

<br />

![](problem-02.svg)

???

- If a thread is waiting for "itx S1" to be persisted...

---

# Example "itx S1"

<br />

![](problem-03.svg)

???

- it must actually wait for **all** `lwb`'s to be persisted.

---

# Implications

 1. `zil_commit` latency proportional to system workload, _not_ disk latency

 2. Disk "anomalies" &rarr; larger batches &rarr; increased `zil_commit` latency

 3. New calls to `zil_commit` wait for "current" batch, _and_ "next" batch

???

1. _`zil_commit` latency_

   - Fast SLOG may not compensate for large workload

2. _Disk "anomalies"_

   - e.g. Temporary network delays when using SAN storage

3. _New calls to `zil_commit`_

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

1. _Allow `zil_commit` to issue new ZIL block writes immediately_

   - In contrast to waiting for "current" batch to complete

2. _Notify threads immediately when dependent `itx`'s on disk_

   - In contrast to waiting for **all** `itx`'s on disk (dependent or not)

---

# Example "Batch"

<br />

![](solution-01.svg)

???

- Same example as before

---

# Example "itx S1"

<br />

![](solution-02.svg)

???

- Same thread waiting for "itx S1" to be persisted...

---

# Example "itx S1"

<br />

![](solution-03.svg)

???

- But, now it must wait for _only_ "lwb 1" to complete

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

---

# Example: Before

<br />

.center[![](flush-sequence-before.svg)]

???

- 2 `lwb`'s are written to VDEV 1...

   - but only a single flush is issued to it.

- Additionally, all flushes are issued at the very end.

---

# Example: After

<br />

.center[![](flush-sequence-after.svg)]

???

- The same 2 `lwb`'s are written to VDEV 1...

   - but 2 flushes are issued to it.

- Additionally, the flushes are issued immediately after corresponding
  "lwb done".

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

- _ZIL blocks issued to disk using ZIOs_

   - ZIOs represented as directed acyclic graph (DAG)

   - Parent ZIOs cannot complete until all children complete

---

# Example: Before

<br />

.center[![](zio-tree-before-01.svg)]

???

- First, the root ZIO for the batch is generated

---

# Example: Before

<br />

.center[![](zio-tree-before-02.svg)]

???

- Then the `lwb` ZIOs are created as children

---

# Example: Before

<br />

.center[![](zio-tree-before-03.svg)]

---

# Example: Before

<br />

.center[![](zio-tree-before-04.svg)]

---

# Example: Before

<br />

.center[![](zio-tree-before-05.svg)]

???

- After all `lwb` ZIOs complete...

- the root ZIO for the VDEV flush commands is generated

---

# Example: Before

<br />

.center[![](zio-tree-before-06.svg)]

???

- And we'd issue a flush to each VDEV written to

---

# Example: Before

<br />

.center[![](zio-tree-before-07.svg)]

???

- Once all VDEV flushes complete...

- we're done, and any waiting threads would be notified

---

# Example: After

![](zio-tree-after-01.svg)

???

- The ZIO tree is dramatically different after my changes...

- Now, we create a root ZIO similar to before...

- But this root ZIO is specific to an `lwb`, as will become clear

---

# Example: After

![](zio-tree-after-02.svg)

???

- Next, we create another ZIO for the "write" of the `lwb`

- This ZIO contains the actual data and `itx`'s to be persisted

- Also, it's a child of the `lwb`'s root ZIO

---

# Example: After

![](zio-tree-after-03.svg)

???

- After that "write" ZIO **completes**, the flush ZIO is issued

- The flush ZIO is also a child of the `lwb` root ZIO

---

# Example: After

![](zio-tree-after-04.svg)

???

# Example: After

- If another `lwb` is to be written...

- it'll also have a root ZIO of its own

- Further, it'll have the prior `lwb`'s root ZIO as a child

   - we do this so that the root ZIO for "lwb 2" can't complete...

   - until "lwb 1" completes.

   - The importance of the order of these ZIO completions...

   - will hopefully become more clear later.

---

# Example: After

![](zio-tree-after-05.svg)

???

- Next, we create the `lwb` write ZIO just like before

---

# Example: After

![](zio-tree-after-06.svg)

???

- To make things more interesting...

   - let's say the next `lwb` is issued before this `lwb`'s "write" completes.

- In this case, the "write" for "lwb 2" will still be outstanding...

   - so the flush for that lwb will not have been issued...

   - but a new `lwb` can still be created and issued.

---

# Example: After

![](zio-tree-after-07.svg)

???

 - And this `lwb` will also have a child "write" ZIO

---

# Example: After

![](zio-tree-after-08.svg)

???

- It's even possible for the "write" of "lwb 3" to complete before...

   - the "write" of "lwb 2"

- If that occurs, the flush for "lwb 3" will get issued before...

   - the flush for "lwb 2"

- It's important to note...

   - even if the flush of "lwb 3" were to complete...

   - the root ZIO "lwb 3" can't complete...

   - becuase the root for "lwb 2" is still in progress.

   - This is enforced by the ZIO layer...

      - a "parent" ZIO can't complete until all children complete

---

# Example: After

![](zio-tree-after-09.svg)

???

- Now, lets say the "write" for "lwb 2" completes...

   - so the flush for "lwb 2" is issued.

- Since "lwb 3" may have already completed it's write and flush...

   - it's possible for "lwb 2" and "lwb 3" to complete "simultaneously"

---

# Example: After

![:scale 100%](zio-tree-after-10.svg)

???

 - This pattern will continue as long as there's `lwb`'s to issue to disk

---

# Example: After

![:scale 100%](zio-tree-after-11.svg)

---

# Example: After

![:scale 100%](zio-tree-after-12.svg)

???

- As `lwb` root ZIOs complete...

   - they'll simply be removed from the "tail" of the tree

- Here, "lwb 1" has been removed

---

# Example: After

![](zio-tree-after-13.svg)

???

- Now "lwb 2"

---

# Example: After

![](zio-tree-after-14.svg)

???

- And "lwb 3"... etc.

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

???

- Here's a possible sequence of events...

- But don't try and understand it all right now

---

# Example: Before

.center[![](waiter-sequence-before-02.svg)]

???

- Instead, I want to hightlight how/when processes become "done"

- Specifically, writer threads must stop writing...

   - in order to notifiy and wake up any waiting threads

- Further, this notification only occurs after an entire batch completes

---

# Details: After

 - Each `lwb` has a list of CVs

 - Each time a process calls `zil_commit`:

    - A new CV is allocated for this specific process to wait on

    - This CV is attached to the `lwb` that will persist the process's
      data<sup>\*</sup>

 - When an `lwb`'s ZIO completes, each CV in the `lwb`'s list is signalled

 - This allows processes to be signalled while `lwb`'s are being written

.footnote[<sup>\*</sup>How we determine which `lwb` does this, isn't covered.]

---

# Example: After

.center[![](waiter-sequence-after-01.svg)]

???

- Here's another sequence of events...

- Like before, there's a lot to take in...

- so don't try and understand it all

---

# Example: After

.center[![](waiter-sequence-after-02.svg)]

???

- Instead, I want to highlight how/when processes become "done"

- In this "new world", processes are signalled as done from the "ZIO
  done" callback

- Further, we can see that they're signalled while `lwb`'s are being
  written

- Thus, there's no "start"/"stop" sort of behavior like previously...

   - Where we'd have to stop writing, in order to signal waiters

- Now the waiter's are notified "async" from the context of the writer...

   - immediately from the ZIO done callback for that process's `lwb`

---

class: middle, center

# 5 &ndash; Performance testing and results.

---

# Details

 - Two `fio` workloads used to drive a sync write workload

    1. `fio` was trying to perform sync writes as fast as it could

    2. `fio` was trying to perform 64 sync writes per second

 - IOPs and latency measured with and without my changes<sup>\*</sup>

 - 1, 2, 4, and 8 disk zpools; both SSD and HDD

 - fio threads ranging from 1 to 1024; increasing in powers of 2

 - Full details can be found [here][perf-results]

.footnote[<sup>\*</sup>Other metrics also observed, but not covered here.]

---

class: middle, center

# 5 &ndash; Max Rate Workload &ndash; HDDs

---

background-image: url(max-rate-hdd-iops-pctchange.svg)
background-size: 100%

# % Change IOPs &ndash; Max Rate &ndash; HDDs

---

background-image: url(max-rate-hdd-lat-pctchange.svg)
background-size: 100%

# % Change Latency &ndash; Max Rate &ndash; HDDs

---

class: middle, center

# 5 &ndash; Max Rate Workload &ndash; SSDs

---

background-image: url(max-rate-ssd-iops-pctchange.svg)
background-size: 100%

# % Change IOPs &ndash; Max Rate &ndash; SSDs

---

background-image: url(max-rate-ssd-lat-pctchange.svg)
background-size: 100%

# % Change Latency &ndash; Max Rate &ndash; SSDs

---

class: middle, center

# 5 &ndash; Fixed Rate Workload &ndash; HDDs

---

background-image: url(fixed-rate-hdd-iops-pctchange.svg)
background-size: 100%

# % Change IOPs &ndash; Fixed Rate &ndash; HDDs

---

background-image: url(fixed-rate-hdd-lat-pctchange.svg)
background-size: 100%

# % Change Latency &ndash; Fixed Rate &ndash; HDDs

---

class: middle, center

# 5 &ndash; Fixed Rate Workload &ndash; SSDs

---

background-image: url(fixed-rate-ssd-iops-pctchange.svg)
background-size: 100%

# % Change IOPs &ndash; Fixed Rate &ndash; SSDs

---

background-image: url(fixed-rate-ssd-lat-pctchange.svg)
background-size: 100%

# % Change Latency &ndash; Fixed Rate &ndash; SSDs

[perf-results]: https://www.prakashsurya.com/post/2017-09-08-performance-testing-results-for-openzfs-447/
