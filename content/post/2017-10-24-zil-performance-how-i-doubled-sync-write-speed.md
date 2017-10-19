+++
date = "2017-10-24T00:00:00-08:00"
lastmod = "2017-10-24T00:00:00-08:00"
title = "ZIL Performance: How I Doubled Sync Write Speed"
presenter = "Prakash Surya"
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

???

- Here's a brief overview of what I plan to discuss.

- It's broken up into roughly 3 parts:

   1. First I'll give some background, and discuss:

      - What the ZIL is...

      - how it's used...

      - and how it works.

   2. Then I'll get into:

      - The problem I set out to fix...

      - how I fixed it...

      - and provide some details on how I did that.

   3. And then, lastly, I'll show off some graphs...

      - and the results of my work.

- So, with that out of the way, let's get started.

---

class: middle, center

# 1 &ndash; What is the ZIL?

???

- First off... what is the ZIL?

---

# What is the ZIL?

 - ZIL: Acronym for (Z)FS (I)ntent (L)og

    - Logs synchronous operations to disk, before `spa_sync()`

    - What operations get logged?

       - `zfs_create`, `zfs_remove`, `zfs_write`, etc.

       - Doesn't include non-modifying ZPL operations:

          - `zfs_read`, `zfs_seek`, etc.

    - What gets logged?

       - The fact that a logical operation is occurring is logged

          - `zfs_remove` &rarr; directory object ID + name only

       - Not logging which blocks will change due to logical operation

???

- ZIL stands for ZFS Intent Log.

- It's the mechanism that's responsible for logging synchronous
  operations to disk.

- The operations that get logged are "logical" operations, such as:

   - `zfs_create`, `zfs_remove`, etc.

- This does not include non-modifying operations, such as:

   - `zfs_read`, `zfs_seek`, etc.

- The data that get's logged, is simply the fact that the logical
  operation occured.

- For example, for `zfs_remove`, it's only...

   - the name of the object to remove...

   - and the object ID of the directory from which that name will be
     removed.

- It's _not_ which blocks on-disk would change due to that removal.

---

# When is the ZIL used?

 - Always<sup>\*</sup>

    - ZPL operations (`itx`'s) logged via in-memory lists

    - lists of in-memory `itx`'s written to disk via `zil_commit()`

    - `zil_commit()` called for:

       - _any_ sync write<sup>\**</sup>

.footnote[<sup>\*</sup>Except when dataset configured with: `sync=disabled`.
          <sup>\**</sup>Except when dataset configured with: `sync=always`.]

???

- The ZIL is almost always used.

- Whenever any of these logged operations occur...

   - they're inserted into the ZIL's in-memory list of operations.

- These operations are often called as `itx`'s, or "intent log
  transactions".

- When `zil_commit` is called...

   - the `itx`'s tracked by the in-memory ZIL are then written to disk.

- The caveat to all of this, being...

   - none of this occurs if the dataset is configured with
     `sync=disabled`.

- If `sync=disabled`, `itx`'s aren't tracked in-memory...

   - nor are they written to disk.

---

# What is the SLOG?

 - SLOG: Acronym for (S)eperate (LOG) Device

 - Conceptually, SLOG is different than the ZIL

   - ZIL is mechanism for writing, SLOG is device written to

 - An SLOG is not necessary

    - By default (no SLOG), ZIL will write to main pool VDEVs

 - An SLOG can be used to improve latency of ZIL writes

    - When attached, ZIL writes to SLOG instead of main pool<sup>\*</sup>

.footnote[<sup>\*</sup>For some operations; see code for details.]

???

- Since I often see the ZIL and SLOG terms used incorrectly...

   - I wanted to briefly address this.

- SLOG stands for Seperate Log Device.

- The ZIL and SLOG are different, in that:

   - the ZIL is the mechanism for issuing writes to disk

   - and the SLOG _may_ be the disk that the writes are issued to.

- With that said, an SLOG is **not** necessary.

- By default, ZIL writes will go to the main pool's disks.

- _But_, an SLOG can be used to try and improve the latency of ZIL
  writes...

   - if the main pool's VDEVs are deemed "too slow".

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

???

- So, why exactly does the ZIL exist in the first place?

- Well, writes in ZFS are "write back".

- Data is first modified and stored in-memory, in the DMU layer...

   - and then at some later point, this data is written to disk via
     `spa_sync()`.

- The problem is, `spa_sync()` can take tens of seconds or more...

   - to write out this data.

- It's unacceptable for all sync writes to take tens of seconds to
  complete.

- Further, writes in ZFS often cause more writes to occur.

- For example. a singe file write...

   - modifying a single block of "user data"...

   - will then cause indirect blocks to also be modified and written.

- The ZIL allows this "write amplification" effect to be mitigated.

- Essentially, the ZIL exists as a performance optimization.

- To provide synchronous sematics to applications...

   - faster than what could be achieved with `spa_sync()` alone.

- While correctness _could_ be acheived without the ZIL...

   - performance would be unreasonably bad...

   - which makes it a necessity.

---

# ZIL On-Disk Format

 - Each dataset has it's own unique ZIL on-disk

 - ZIL stored on-disk as a singly linked list of ZIL blocks (`lwb`'s)

<hr style="visibility:hidden;" />

.center[![](zil-on-disk-format.svg)]

???

- Before I jump into the next section...

   - I wanted to quickly go over the on-disk format of the ZIL.

- Each ZFS dataset maintains it own unique ZIL on disk.

- Each of these ZIL's is a singly liked list of ZIL blocks....

   - or, as they're also called, `lwb`'s...

   - which stands for "log write block"

- As one can see here:

   - the uberblock has a pointer to the MOS...

   - which then has pointers to each dataset in the pool...

   - and each of these datasets has a pointer it's own ZIL header.

- Each ZIL header then points to an `lwb`...

   - and that block then points to the "next" block in the list.

---

class: middle, center

# 2 &ndash; How is the ZIL used?

???

- Now let's get into how the ZIL is used...

---

# How is the ZIL used?

 - ZPL will generally interact with the ZIL in two phases:

    1. Log the operation(s) &mdash; `zil_itx_assign`

        - Tells the ZIL an operation is occurring

    2. Commit the operation(s) &mdash; `zil_commit`

        - Causes the ZIL to write log record of operation to disk

???

- The ZIL is used by the ZFS Posix Layer, or ZPL for short.

- The ZPL generally interacts with the ZIL in two phases:

   1. First it uses `zil_itx_assign`.

      - This cause the ZIL to log the fact that an operation
        is occurring.

   2. Then it uses `zil_commit`.

      - This tells the ZIL to write out these log records to disk.

---

# Example: `zfs_write`

 - `zfs_write` &rarr; `zfs_log_write`

 - `zfs_log_write`

    &rarr; `zil_itx_create`

    &rarr; `zil_itx_assign`

 - `zfs_write` &rarr; `zil_commit`

???

- Let's look at `zfs_write` as an example of this.

- `zfs_write` will call `zfs_log_write`.

- `zfs_log_write` will then call:

   - `zil_itx_create`...

      - which will create the `itx` structure in RAM...

   - and then it'll call `zil_itx_assign`...

      - which will insert that `itx` into the ZIL's in-memory state.

- Finally, if this is a sync write, `zfs_write` will then call
  `zil_commit`...

   - which will cause the `itx` to get written out to disk.

---

# Example: `zfs_fsync`

 - `fsync` &rarr; `zil_commit`

   - `fsync` doesn't create any new modifications

   - only writes previous `itx`'s to disk

      - thus, no `zfs_log_fsync` function

???

- Now let's look at `zfs_fsync` as another example.

- `fsync` doesn't create any new modifications.

- Instead, it simply ensures any previous operations are written to disk
  before it returns.

- Thus, `zfs_fsync` doesn't call `zil_itx_create`...

   - nor does it call `zil_itx_assign`.

- Instead it _only_ calls `zil_commit`.

- Calling `zil_commit` will ensure all previous operations are written
  to disk, before `fsync` returns.

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

- The parameters of `zil_commit` are such that...

   - the caller will pass in enough information to uniquely identify an
     object whose data is to be committed.

- The contract the ZIL maintains with this caller is...

   - All operations _relevant_ to the object specified...

   - will be _persistent_ on disk by the time `zil_commit` returns.

- By _relevant_, it means...

   - all operations that would modify that object.

- And by _persistent_, it means...

   - the operations are written to disk...

   - and the disks used for those writes are flushed.

- Further, we must issue the disk flush _after_ the writes complete.

- Lastly, the interface for `zil_commit` doesn't allow the caller to
  specify _which_ operations they care about.

- Thus, `zil_commit` must write _all_ operations for a given object...

   - even if the caller only cares about a subset of those operations.

- For example, if there's multiple threads writing to the same file,
  but at different offsets...

   - all offsets must be written to disk before `zil_commit` returns...

   - _even_ if the calling thread only cares about one of those offsets.

---

class: middle, center

# 2 &ndash; How does the ZIL work?

???

- So, how does the ZIL accomplish this?

---

# How does the ZIL work?

 - In memory ZIL contains an `itxg_t` structure<sup>\*</sup>

 - Each `itxg_t` contains:

    - A single list of sync operations (for all objects)

    - Object specific lists of async operations

.footnote[<sup>\*</sup>Actually multiple `itxg_t` structures, one per-txg.]

???

- Well, as I alluded to previously, the ZIL maintains an in-memory list
  of `itx`'s that have occurred...

   - but haven't yet been written to disk.

- This list is maintained via the `itxg` structure in each ZIL.

- Each `itxg` structure contains the following:

   - a single list of all sync operations that have occurred...

      - for all objects in the dataset.

   - plus, per-object lists of async operations for each object
     modified.

---

# Example: itx lists

<br />

![](itx-lists.svg)

???

- Here's what this might look like...

- In this example, the `itxg`'s sync list has two `itx`'s in it...

   - each of which could map to an operation for any object in the dataset.

- And then a list of async operations that occurred for "object A"

- And _another_ list of async operations that occurred for "object B"

---

# How are itx's written to disk?

 - `zil_commit` handles the process of writing `itx_t`'s to disk:

???

- When `zil_commit` is called...

   - How do these `itx`'s get written out to disk?

---

# How are itx's written to disk?

 - `zil_commit` handles the process of writing `itx_t`'s to disk:

     1. find all relevant `itx`'s, move them to the "commit list"

???

- Well...

- First we must determine which of these `itx`'s are _relevant_...

- And then we move these _relevant_ `itx`'s to a new list...

   - called the "commit list".

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-1-01.svg)

???

- Here, we have the same `itx` lists as before...

- Except now, we also show an empty "commit list" at the bottom.

- Also, let's presume `zil_commit` is being called for "object B".

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-1-02.svg)

???

- The first step is to move all of object B's async `itx`'s to the sync
  list.

- So, we select the `itx`'s...

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-1-03.svg)

???

- And move them.

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-1-04.svg)

???

- Next, we move the entire contents of the sync list...

   - to the commit list.

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-1-05.svg)

???

- Additionally, it's worth pointing out...

- The point of the "commit list" is so we have a list of `itx`'s to
  write out...

   - that will _not_ be modified by any concurrent ZPL activity.

- As new ZPL operations occur, the sync list may change...

   - for example, operations may be added to it...

   - but the commit list will remain the same.

---

# How are itx's written to disk?

 - `zil_commit` handles the process of writing `itx_t`'s to disk:

     1. ~~Move async itx's for object being commited, to the sync list~~

     2. Write all commit list `itx`'s to disk

???

- Now that we have a list of `itx`'s to be written out, it's time to
  actually issue them to disk.

- We do this by iterating over all the `itx`'s in the commit list.

- Then, for each `itx`:

   - We attempt to copy it into the currently "open" ZIL block.

   - If there's insufficient space in the block...

      - then we allocate a new block, and issue old one to disk.

- Lastly, after all `itx`'s are copied into `lwb`'s...

   - we issue the last "open" block to disk...

   - allocating the _next_ "open" block in the process.

- Here's what I mean...

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-2-03.svg)

???

- Given the commit list from before...

   - and "lwb 1" as the currently "open" ZIL block...

- We select the first `itx` in the list...

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-2-04.svg)

???

- and copy this into the block's buffer.

- This block will remain "open"...

   - as denoted by the dotted line...

   - since it may also be used for the next `itx` in the list.

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-2-05.svg)

???

- So, now we move on to the next `itx`...

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-2-06.svg)

???

- This one doesn't quite fit in the currently "open" block...

- So...

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-2-07.svg)

???

- We must allocate a new block, and issue the current one to disk.

- Here, "lwb 1" now has a solid line, to indicate it's been issued
  to disk.

- And "lwb 2" has a dashed line, to indicate it's the new "open"
  ZIL block.

- Further, "lwb 1" maintains a pointer to "lwb 2" on disk.

- Now we can go back to processing the commit list...

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-2-08.svg)

???

- and we copy the `itx` into the new `lwb`.

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-2-09.svg)

???

- Finally, we reach the last `itx` in the list...

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-2-10.svg)

???

- Since this fits into "lwb 2"...

   - it's copied directly into place.

- At this point, the commit list is empty...

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-2-11.svg)

???

- so we issue "lwb 2" to disk...

   - allocating the next "open" block in the process.

- As one can see, "lwb 1 and 2" have a solid line...

   - to indicate they've both been issued to disk.

- While, "lwb 3" has a dashed line, to indicate it's now the new
  "open" block...

    - and it will be used when writing out the next "batch" of
      `itx`'s.

---

# How are itx's written to disk?

 - `zil_commit` handles the process of writing `itx_t`'s to disk:

     1. ~~Move async itx's for object being commited, to the sync list~~

     2. ~~Write all commit list `itx`'s to disk~~

     3. Wait for all ZIL block writes to complete

???

- Now...

- After we've issued all ZIL blocks to disk...

- We must wait for them to complete.

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-3-01.svg)

???

- The blocks can complete in any order...

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-3-02.svg)

???

- Here, "lwb 2" completes first...

---

# Example: `zil_commit` Object B

<br />

![](zil-commit-3-03.svg)

???

- and then "lwb 1".

- At this point, all blocks have completed...

---

# How are itx's written to disk?

 - `zil_commit` handles the process of writing `itx_t`'s to disk:

     1. ~~Move async itx's for object being commited, to the sync list~~

     2. ~~Write all commit list `itx`'s to disk~~

     3. ~~Wait for all ZIL block writes to complete~~

     4. Flush VDEVs

???

- So it's time to issue a disk flush to all VDEVs involved...

---

# How are itx's written to disk?

 - `zil_commit` handles the process of writing `itx_t`'s to disk:

     1. ~~Move async itx's for object being commited, to the sync list~~

     2. ~~Write all commit list `itx`'s to disk~~

     3. ~~Wait for all ZIL block writes to complete~~

     4. ~~Flush VDEVs~~

     5. Notify waiting threads

???

- Once those flushes complete...

- we can notify any waiting threads...

- letting them know their data is safe on disk.

---

class: middle, center

# 3 &ndash; Problem

???

- Now, let's dive into the problem with all of this...

---

# Problem

 1. `itx`'s grouped and written in "batches"

    - The commit list constitutes a batch

    - Batch size proportional to sync workload on system

 2. Waiting threads only notified when _all_ ZIL blocks in batch complete

 3. Only a single batch processed at a time

???

- The main issues can be summarized into the following 3 points:

- First, `itx`'s are grouped and written "batches".

- Where the commit list constitues a batch...

   - and the batch size is proportional to the sync workload on the
     system.

- Next, threads waiting for `zil_commit` to complete, are only notified
  when _all_ ZIL blocks in a given batch complete.

   - So, if a given batch is large, that means `zil_commit` must wait
     for all of those blocks to complete...

   - _even_ if the caller only cares about a small percentage of the
     data in that batch.

- And lastly, only a single batch can be processed and written out at
  any given time.

---

# Problem

![](problem.svg)

 - Time spent servicing `lwb`'s for each disk

 - Color indicates order waiting threads notified

???

- Here's an example of what this ends up looking like...

- This is a timeline of the disk activity of an example system.

- What can be seen here, is...

   - block A through E are all written in the first batch...

   - but the disk activity is slightly uneven...

      - while disk 2, 3, and 4 only receive a single ZIL block to write

      - disk 1 receives two blocks.

- Thus, disks 2, 3, and 4 complete their writes and then remain idle...

   - while disk 1 finishes its work.

- This idle time is due to the fact that only a single batch can be
  processed at a time...

   - which leads to inefficient usage of the storage.

- For example...

   - blocks F, G, and H _could_ have been issued to disks 2, 3, and 4...

   - filling this idle time...

   - but the batching mechanism prevents this.

- Another issue...

   - is the fact that waiting threads will only be notified once _all_
     blocks in a batch complete.

- For example...

   - if a thread was waiting on data to be written by "block A"...

   - it would have to wait for "block E" to be written as well.

- This unnecessarily increases the latency of `zil_commit`...

   - and, in this case, potentially doubling it.

---

class: middle, center

# 3 &ndash; Solution

???

- The solution is somewhat obvious...

---

# Solution

 - Remove concept of "batches":

    1. Allow `zil_commit` to issue new ZIL block writes immediately

       - In contrast to waiting for the current batch to complete

    2. Notify threads immediately when _dependent_ `lwb`'s on disk

       - In contrast to waiting for _all_ `lwb`'s on disk

???

- Let's just remove the concept of "batches".

- Rather than waiting for the current batch to complete...

   - we should issue new ZIL blocks to disk immediately...

   - as soon as they can be written out.

- Further, rather than waiting for a batch to complete before notifying
  threads...

   - these threads should be notified immediately when their data is
     safe on disk.

---

# Problem

![](problem.svg)

 - Time spent servicing `lwb`'s for each disk

 - Color indicates order waiting threads notified

???

- If we did that...

- Then we could go from this diagram, which I showed earlier...

---

# Solution

![](solution.svg)

 - Time spent servicing `lwb`'s for each disk

 - Color indicates order waiting threads notified

???

- To this one.

- Where all disks in the pool are saturated...

   - _and_ threads are notified as soon as each individual block
     completes.

- As this diagram illustrates...

   - we'd be able to service the same number of ZIL blocks...

   - in nearly half the time...

   - potentially doubling our IOPs.

- And, this is without changing a single thing about the workload...

   - nor the underlying storage characteristics.

---

class: middle, center

# 4 &ndash; Details on the Changes I Made

???

- So how was this accomplished?

---

background-image: url(changes-before-01.svg)
background-size: 65%

## Before

???

- The bulk of changes revolve around 3 things:

   1. changing how ZIL blocks are issued to disk...

   2. changing when the flush commands are sent...

   3. and changing how we notify waiting threads.

- Previously, this was a sequential 3 step process...

---

background-image: url(changes-before-02.svg)
background-size: 65%

## Before

???

- Step 1 would consist of creating the ZIL blocks...

   - issuing these to disk...

   - and then waiting for the IO for _all_ the blocks to complete.

---

background-image: url(changes-before-03.svg)
background-size: 65%

## Before

???

- Next, after the blocks completed...

- Step 2 would consist of issuing the flush to each VDEV...

   - and then waiting for those flushes to complete.

---

background-image: url(changes-before-04.svg)
background-size: 65%

## Before

???

- Finally, after all flushes completed...

   - the ZIL's CV would be signalled to notify any waiting threads.

- All threads that called `zil_commit` would be waiting on this CV...

   - so this was the mechanism for letting them know their data was
     safe on disk.

- All 3 of these steps would consist of a single batch.

---

background-image: url(changes-before-05.svg)
background-size: 65%

## Before

???

- After one batch completes...

   - another would start up again.

---

background-image: url(changes-before-06.svg)
background-size: 65%

## Before

???

- And another.

- This 3 step process would repeat as long as the workload would allow.

---

background-image: url(changes-after-01.svg)
background-size: 85%

## After

???

- Now the process is entirely different...

   - and heavily leaverages the ZIO infrastructure.

- Instead of a single root ZIO for an entire batch of blocks...

   - each block now has it's own, unique, root ZIO.

- Each root will eventually have two children:

   1. A "write" ZIO, containing the `itx` data to be written...

   2. And a "flush" ZIO, that is issued after the "write" completes.

- Since these are each child ZIOs, the "root" cannot complete until
  both the "write" and the "flush" complete.

   - This is enforced by the pre-existing ZIO parent-child semantics.

- Further, the "root" ZIO of the "previous" block...

   - will also be a child of the "next" block.

   - for example, here, "lwb 1" is a child of "lwb 2"

- This ensures `lwb` root ZIOs completes in the correct order...

   - again, leveraging ZIO dependencies to acheive this.

- Finally, each `lwb` maintains a list of CVs...

   - where each CV on this list maps to a single thread that called
     `zil_commit`.

- Then, when the "root" ZIO for any ZIL block completes...

   - each CV in the block's list is signalled...

   - notifying the waiting threads that their data is safe on disk.

- Let's walk through an example of what this looks like...

---

background-image: url(changes-after-02.svg)
background-size: 85%

## After

???

- First, the `lwb` structure and "root" ZIO is created.

- Initially, the list of CVs will be empty...

   - but as `itx`'s are copied into this block's buffer...

   - it will begin to accumulate a list of threads waiting on it.

---

background-image: url(changes-after-03.svg)
background-size: 85%

## After

???

- Next, when the block's buffer is full, its write will be issued
  to disk...

---

background-image: url(changes-after-04.svg)
background-size: 85%

## After

???

- Then, when that write completes, the flush will be issued...

---

background-image: url(changes-after-05.svg)
background-size: 85%

## After

???

- When the "flush" completes...

   - _and_ since this ZIL block doesn't point to any previous block...

- this will allow the "root" IO for this specific block to complete...

- at which point, each CV in the block's list will be signalled.

---

background-image: url(changes-after-06.svg)
background-size: 85%

## After

???

- The same process will occur for the next block...

---

background-image: url(changes-after-07.svg)
background-size: 85%

## After

???

- And the next.

- At this point, though, it's important to note...

   - the sequence of events doesn't have to occur in this specific order...

---

background-image: url(changes-after-08.svg)
background-size: 85%

## After

???

- For example, it's possible for "block 2" to issue it's write...

   - before the write for "block 1" completes.

- In this case...

   - the write for "block 1" would have been issued...

   - then the write for "block 2"...

   - even though block 1's write was still being serviced by the disk.

- Previously this would have been prevented due to the "batching"
  mechanism...

   - the write for "block 2" could **not** have been issued, until the
     write for "block 1" was complete.

---

background-image: url(changes-after-09.svg)
background-size: 85%

## After

???

- The same goes for the write for "block 3"...

---

background-image: url(changes-after-10.svg)
background-size: 85%

## After

???

- It's even possible for the writes of blocks 2 and 3...

   - to complete before the write of block 1.

- If this happens, though...

   - The root of block 2 and 3 will still be blocked...

   - waiting for block 1 to complete.

- Thus, even if the flushes were issued and completed for
  blocks 2 and 3...

   - their CVs would **not** get signalled yet.

---

background-image: url(changes-after-11.svg)
background-size: 85%

## After

???

- As soon as the write for block 1 completes...

- The flush would be issued...

- And once block 1's flush completes...

---

background-image: url(changes-after-12.svg)
background-size: 85%

## After

???

- Then, all 3 blocks would "complete" simultaneously...

   - and the CVs for all of these blocks would be notified.

- So...

   - while before...

   - this process was very sequential.

- Now, it's completely driven by the incoming sync workload...

   - and disk completion events.

- ZIL blocks are created and issued whenever there's data to write...

   - and waiting threads are notified whenever those writes complete.

---

# New Tunable: `lwb` Timeout

<br />

.pull-left[![:scale 100%](zil-commit-2-10.svg)]
.pull-right[![:scale 100%](zil-commit-2-11.svg)]

???

- Before jumping into the performance results...

   - I wanted to quickly talk about how we determine when to issue
     a ZIL block to disk.

- If you remember from earlier in the talk...

   - we build up the these blocks by iterating over the commit list...

   - and copying each `itx` into one of the `lwb` buffers.

- Previously, once we reached end of the commit list...

   - In essence, the end of a "batch"...

   - we would issue the last `lwb` to disk.

- Now that we're "batch-less" we don't necessarily want to do that.

- If we reach the end of the commit list...

   - but there's still buffer space available in the current block...

      - for example, we could have a 128K ZIL block...

      - with only a single 8K write in it...

   - then we _actually_ want to delay issuing that block...

   - in case new `itx`'s are generated that would fit into that block.

- This way, we can write out more `itx`'s using fewer IOPs.

- The problem is, we have no way to predict the future...

   - We don't know, _for sure_, if more `itx`'s will be generated.

- Thus, if we wait for future `itx`'s but none are generated...

   - then we're adding additional latency to the current `lwb`
     for no benefit.

- But...

   - if we don't wait at all, and additional `itx`'s _are_ generated...

   - we could end up using more IOPs than we need to...

   - and potentially degrade performance by saturating the disk.

- The solution we implemented is...

   - a ZIL block may be delayed up to 5% of the latency of the last
     completed ZIL block.

- For example, if the last block took 5ms to be serviced by the storage...

   - than the next block will wait a maximum of 250us.

- If it's filled within that 250us, it'll be issued to disk immediately.

- If it's not filled, it'll timeout after 250us...

   - and be issued to disk partially filled.

- For those wondering, 5% is the default...

   - and not that I recommend doing this...

   - it _can_ that can be changed using the new tunable
     (`zfs_commit_timeout_pct`)

---

class: middle, center

# 5 &ndash; Performance testing and results

???

- Finally, let's go over the results of the performance tests that were
  used to verify these changes.

---

background-image: url(max-rate-hdd-iops-pctchange.svg)
background-size: 115%

## ~83% Increase in IOPs on Average &ndash; Max Rate &ndash; 8 HDDs

???

- I used two different `fio` workloads to verify.

- For this workload...

   - each `fio` thread was submitting sync writes as fast as could...

   - and I measured the total number of IOPs that were achieved...

   - while varying the number of `fio` threads, from 2 to 1024.

- This graphs shows the percentage difference in IOPs...

   - between illumos _with_ my changes, and illumos _without_ my changes.

- The dashed line at the bottom is a visual aid...

   - to highlight where in the graph a 0% difference is.

- Anything above that line is improvement.

- On average, I measured an 83% increase in IOPs with my changes.

- The dotted line is another visual aid...

   - showing where exactly 83% improvment is...

   - in relation to the actual measurements taken.

- Additionally, the zpool used for this graph consisted of 8 traditional
  spinning drives.

---

background-image: url(max-rate-ssd-iops-pctchange.svg)
background-size: 115%

## ~48% Increase in IOPs on Average &ndash; Max Rate &ndash; 8 SSDs

???

- I also ran that same workload on a zpool consisting of 8 SSDs.

- When running on SSDs, the improvment isn't as dramatic...

   - but I was _still_ able to measure about a 48% improvement,
     on average.

- The same visual aids are here...

   - anything above the dashed line at the bottom is improvement...

   - and the dotted line corresponds to the average.

---

background-image: url(fixed-rate-hdd-lat-pctchange.svg)
background-size: 115%

## ~27% Decrease in Latency on Average &ndash; Fixed Rate &ndash; 8 HDDs

.footnote[<sup>\*</sup>IOPs increased with new code, and >64 threads; those data points omitted.]

???

- The second workload tested, was again using `fio`...

   - but this time...

   - each `fio` thread would attempt to issue a _maximum_ of 64 sync
     writes per-second.

- Thus, the number of IOPs was constant with and without my changes...

   - but...

   - the latency of each sync write was still improved.

- Since we're now measuring latency, rather than IOPs...

   - any value _below_ the dashed line is improvement.

- When running this test on my 8 HDD pool...

   - I measured the latency of each sync write to decrease by an average
     of 27%

- Also worth noting...

   - the IOPs began to diverage at thread counts >64...

   - where the new code started doing more IOPs than the old code...

   - so, I removed those data points to keep the comparison "fair".

---

background-image: url(fixed-rate-ssd-lat-pctchange.svg)
background-size: 115%

## ~16% Decrease in Latency on Average &ndash; Fixed Rate &ndash; 8 SSDs

???

- And lastly, I also ran this workload on my 8 SSD system.

- The IOPs were the same for all thread counts on this pool...

   - so I didn't have to remove any data points.

- Here, I measured the latency to decrease by an average of 16% on this
  system.

---

# More Details

 - Two `fio` workloads were used:

    1. each thread submitting sync writes as fast as it could

    2. each thread submitting 64 sync writes per second

 - 1, 2, 4, and 8 disk zpools; both SSD and HDD

 - `fio` threads ranging from 1 to 1024; increasing in powers of 2

 - Full details can be found [here][perf-results]

???

- Here's some more details on the performance tests that I ran...

   - I just wanted to include this in the slides for posterity's sake.

- Additionally, there's a link to even more information about the
  testing I did, and my results.

[perf-results]: https://www.prakashsurya.com/post/2017-09-08-performance-testing-results-for-openzfs-447/
