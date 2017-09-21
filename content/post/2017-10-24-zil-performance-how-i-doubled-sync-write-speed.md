+++
date = "2017-10-24T00:00:00-08:00"
lastmod = "2017-10-24T00:00:00-08:00"
title = "ZIL Performance: How I Doubled Sync Write Speed"
type = "slides"
draft = false
+++

# Agenda

 1. What is the ZIL? Why does it exist?

 2. How is it used? How does it work?

 3. The problem to be fixed; the solution.

 4. Details on the changes I made.

 5. Performance testing and results.

---

class: middle, center

# 1 &ndash; What is the ZIL?

---

# What is the ZIL?

 - ZIL: Acronym for (Z)FS (I)ntent (L)og

    - Log of all operations ZFS intends to write via `spa_sync()`

    - Both "async" and "sync" operations logged in-memory

    - Only sync (or "promoted" async) operations logged on-disk

---

# What is the SLOG?

 - SLOG: Acronym for (S)eperate (LOG) Device

    - An SLOG is not necessary

       - ZIL writes to main pool by default

    - An SLOG can be used to improve latency of ZIL writes

       - When attached, ZIL writes to SLOG instead of main pool<sup>\*</sup>

 - SLOG can be used by the ZIL, but it is _not_ the ZIL

.footnote[<sup>\*</sup>Details of when/how the SLOG is used is complicated and not covered]

---

# When is the ZIL used?

 - Always.

    - in-memory ZIL always used; i.e. log entries always created

    - `sync=always` &rarr; `zil_commit()` always called

       - for both sync and async ops

    - `sync=disabled` &rarr; `zil_commit()` doesn't do anything

       - but it's still called for sync ops

    - `zil_commit()` is the mechanism for writing log entries to disk

 - ZIL used even without an SLOG

 - On-disk ZIL only read from during "replay" (i.e. after a crash)<sup>\*</sup>

.footnote[<sup>\*</sup>Details w.r.t. replay not covered]

---

# Why does the ZIL exist?

 - Performance optimization; not required for correctness

 - Writes in ZFS are "write-back"

    - Data is first stored in-memory, in DMU layer...

    - Then **all** "dirty" data written to disk via `spa_sync()`

    - `spa_sync()` can tens of seconds (or more) to complete

 - Without ZIL, sync operations inherit latency of `spa_sync()`

    - All ops inherit latency; only sync ops _wait_ for completion

 - ZIL allows sync operations to "bypass" latency of `spa_sync()`

---

class: middle, center

# 2 &ndash; How is the ZIL used?

---

# How is the ZIL used?

 - Consumers generally interact with the ZIL in two phases:

    1. Log the operation(s)

     - Tells ZIL operation occurred

    2. Commit the operation(s)

     - Causes ZIL to write log of operation to disk

---

# Example: `zfs_write`

 - `zfs_write` &rarr; `zfs_log_write`

 - `zfs_log_write`

    &rarr; `zil_itx_create`

    &rarr; `zil_itx_assign`

 - `zfs_write` &rarr; `zil_commit`

 - Most ZPL syscalls have a `zfs_log_*` function

    - `zfs_log_create`
    - `zfs_log_remove`
    - `zfs_log_link`
    - `zfs_log_symlink`
    - `zfs_log_truncate`
    - `zfs_log_setattr`
    - ...

---

# Example: `zfs_fsync`

 - `zfs_fsync` &rarr; `zil_commit`

    - There's no _new_ operations to log.. thus, no `zfs_log_fysnc`

---

# Contract between ZIL and consumers.

 - `zil_commit` won't return until all operations are safe

 - What does "all operations" mean?

    - All sync ops for object being committed; i.e. `itx_sync=B_TRUE`

    - All async ops which sync ops depend on

    - In practice:

       - All sync ops for all objects

       - All async ops for specific object being committed

---

# What does "safe" mean?

 - Operation(s) won't be lost, forgotten, or modified

 - How is this accomplished:

    - Data written to disk &rarr; disk flushed

 - Remember: `spa_sync` takes too long

    - Special "log blocks" written not using `spa_sync`

    - Log blocks written faster than `spa_sync` could handle ops

    - Log blocks track operations not yet applied via `spa_sync`

    - Dataset not actually modified (on-disk) until later<sup>\*</sup>

 - If a crash occurs before `spa_sync` applies the modifications...

    - log blocks are "replayed" prior to mounting the dataset

 - If no crash, log blocks freed via `spa_sync`

---

class: middle, center

# 2 &ndash; How does the ZIL work?

---

# How does the ZIL work?

 - In memory ZIL contains 4 (per-txg) `itxg_t` structures

 - Each `itxg_t` contains a single `itxs_t`, which contains:

    - List of sync operations

    - AVL tree of lists of async operations

       - Each node in tree contains per-object list of async ops

 - Remember: `zil_itx_create` &rarr; `zil_itx_assign`

    1. `zil_itx_create` &ndash; allocates a new `itx_t`

    2. `zil_itx_assign` &ndash; inserts the `itx_t` into one of these lists

 - Terminology: "itx" refers to a ZIL operation (intent log transaction)

---

# Example: itx lists

<hr style="visibility:hidden;" />

<div class="mermaid">
graph LR
  SLIST(sync list) --> SLIST-1((itx S1))
  SLIST-1          --> SLIST-2((itx S2))

  ATREE(async tree) --> OBJ-A(object A)
  OBJ-A             --> OBJ-B(object B)
  OBJ-B             --> OBJ-B-1((itx B1))
  OBJ-B-1           --> OBJ-B-2((itx B2))
  OBJ-A             --> OBJ-A-1((itx A1))
  OBJ-A-1           --> OBJ-A-2((itx A2))
  OBJ-A-2           --> OBJ-A-3((itx A3))
  OBJ-A             --> OBJ-C(object C)
  OBJ-C             --> OBJ-C-1((itx C1))
  OBJ-C-1           --> OBJ-C-2((itx C2))
</div>

---

# How are itx's written to disk?

 - `zil_commit` handles the process of writing `itx_t`'s to disk:

---

# How are itx's written to disk?

 - `zil_commit` handles the process of writing `itx_t`'s to disk:

     1. Move async itx's for object being commited, to the sync list

---

# Example: `zil_commit` Object C

<hr style="visibility:hidden;" />

<div class="mermaid">
graph LR
  SLIST(sync list) --> SLIST-1((itx S1))
  SLIST-1          --> SLIST-2((itx S2))

  ATREE(async tree) --> OBJ-A(object A)
  OBJ-A             --> OBJ-B(object B)
  OBJ-B             --> OBJ-B-1((itx B1))
  OBJ-B-1           --> OBJ-B-2((itx B2))
  OBJ-A             --> OBJ-A-1((itx A1))
  OBJ-A-1           --> OBJ-A-2((itx A2))
  OBJ-A-2           --> OBJ-A-3((itx A3))
  OBJ-A             --> OBJ-C(object C)
  OBJ-C             --> OBJ-C-1((itx C1))
  OBJ-C-1           --> OBJ-C-2((itx C2))
</div>

---

# Example: `zil_commit` Object C

<hr style="visibility:hidden;" />

<div class="mermaid">
graph LR
  SLIST(sync list) --> SLIST-1((itx S1))
  SLIST-1          --> SLIST-2((itx S2))
  SLIST-2          --> OBJ-C-1((itx C1))
  OBJ-C-1          --> OBJ-C-2((itx C2))

  ATREE(async tree) --> OBJ-A(object A)
  OBJ-A             --> OBJ-B(object B)
  OBJ-B             --> OBJ-B-1((itx B1))
  OBJ-B-1           --> OBJ-B-2((itx B2))
  OBJ-A             --> OBJ-A-1((itx A1))
  OBJ-A-1           --> OBJ-A-2((itx A2))
  OBJ-A-2           --> OBJ-A-3((itx A3))
  OBJ-A             --> OBJ-C(object C)
</div>

---

# How are itx's written to disk?

 - `zil_commit` handles the process of writing `itx_t`'s to disk:

     1. Move async itx's for object being commited, to the sync list

     2. Move sync list to "commit list"; sync list now empty

---

# Example: `zil_commit` Object C

<hr style="visibility:hidden;" />

<div class="mermaid">
graph LR
  SLIST(sync list)   --> SLIST-1((itx S1))
  SLIST-1            --> SLIST-2((itx S2))
  SLIST-2            --> OBJ-C-1((itx C1))
  OBJ-C-1            --> OBJ-C-2((itx C2))

  CLIST(commit list)
</div>

---

# Example: `zil_commit` Object C

<hr style="visibility:hidden;" />

<div class="mermaid">
graph LR
  SLIST(sync list)

  CLIST(commit list) --> SLIST-1((itx S1))
  SLIST-1            --> SLIST-2((itx S2))
  SLIST-2            --> OBJ-C-1((itx C1))
  OBJ-C-1            --> OBJ-C-2((itx C2))
</div>

---

# How are itx's written to disk?

 - `zil_commit` handles the process of writing `itx_t`'s to disk:

     1. Move async itx's for object being commited, to the sync list

     2. Move sync list to "commit list"; sync list now empty

     3. For each itx in commit list:

         1. Copy itx data into ZIL block

         2. If space in ZIL block lacking...

           - allocate new (empty) block, write out old block

---

# Example: `zil_commit` Object C

<hr style="visibility:hidden;" />

<div class="mermaid">
graph LR
  CLIST(commit list) --> SLIST-1((itx S1))
  SLIST-1            --> SLIST-2((itx S2))
  SLIST-2            --> OBJ-C-1((itx C1))
  OBJ-C-1            --> OBJ-C-2((itx C2))
</div>

---

# Example: `zil_commit` Object C

<hr style="visibility:hidden;" />

<div class="mermaid">
graph LR
  CLIST(commit list) --> SLIST-1((itx S1))
  SLIST-1            --> SLIST-2((itx S2))
  SLIST-2            --> OBJ-C-1((itx C1))
  OBJ-C-1            --> OBJ-C-2((itx C2))

  style SLIST-1 fill:#ffff00,stroke:#000000,stroke-width:3px
</div>


---

# Example: `zil_commit` Object C

<hr style="visibility:hidden;" />

<div class="mermaid">
graph LR
  CLIST(commit list) --> SLIST-1((itx S1))
  SLIST-1            --> SLIST-2((itx S2))
  SLIST-2            --> OBJ-C-1((itx C1))
  OBJ-C-1            --> OBJ-C-2((itx C2))

  HEADER(ZIL header) --> LWB-1(lwb 1)

  style SLIST-1 fill:#ffff00,stroke:#000000,stroke-width:3px
</div>

 - Terminology: "lwb" refers to a ZIL block (log write block)

---

# Example: `zil_commit` Object C

<hr style="visibility:hidden;" />

<div class="mermaid">
graph LR
  CLIST(commit list) --> SLIST-2((itx S2))
  SLIST-2            --> OBJ-C-1((itx C1))
  OBJ-C-1            --> OBJ-C-2((itx C2))

  HEADER(ZIL header) --> LWB-1(lwb 1)
  LWB-1              --- SLIST-1((itx S1))

  style SLIST-1 fill:#ffff00,stroke:#000000,stroke-width:3px
</div>

---

# Example: `zil_commit` Object C

<hr style="visibility:hidden;" />

<div class="mermaid">
graph LR
  CLIST(commit list) --> SLIST-2((itx S2))
  SLIST-2            --> OBJ-C-1((itx C1))
  OBJ-C-1            --> OBJ-C-2((itx C2))

  HEADER(ZIL header) --> LWB-1(lwb 1)
  LWB-1              --- SLIST-1((itx S1))

  style SLIST-1 fill:#ffff00
  style SLIST-2 fill:#00ff00,stroke:#000000,stroke-width:3px
</div>

---

# Example: `zil_commit` Object C

<hr style="visibility:hidden;" />

<div class="mermaid">
graph LR
  CLIST(commit list) --> SLIST-2((itx S2))
  SLIST-2            --> OBJ-C-1((itx C1))
  OBJ-C-1            --> OBJ-C-2((itx C2))

  HEADER(ZIL header) --> LWB-1(lwb 1)
  LWB-1              --- SLIST-1((itx S1))
  LWB-1              --> LWB-2(lwb 2)
  LWB-2

  style SLIST-1 fill:#ffff00
  style SLIST-2 fill:#00ff00,stroke:#000000,stroke-width:3px
  style LWB-1   fill:#ffff00
</div>

---

# Example: `zil_commit` Object C

<hr style="visibility:hidden;" />

<div class="mermaid">
graph LR
  CLIST(commit list) --> OBJ-C-1((itx C1))
  OBJ-C-1            --> OBJ-C-2((itx C2))

  HEADER(ZIL header) --> LWB-1(lwb 1)
  LWB-1              --- SLIST-1((itx S1))
  LWB-1              --> LWB-2(lwb 2)
  LWB-2              --- SLIST-2((itx S2))

  style SLIST-1 fill:#ffff00
  style SLIST-2 fill:#00ff00,stroke:#000000,stroke-width:3px
  style LWB-1   fill:#ffff00
</div>

---

# Example: `zil_commit` Object C

<hr style="visibility:hidden;" />

<div class="mermaid">
graph LR
  CLIST(commit list) --> OBJ-C-1((itx C1))
  OBJ-C-1            --> OBJ-C-2((itx C2))

  HEADER(ZIL header) --> LWB-1(lwb 1)
  LWB-1              --- SLIST-1((itx S1))
  LWB-1              --> LWB-2(lwb 2)
  LWB-2              --- SLIST-2((itx S2))

  style SLIST-1 fill:#ffff00
  style SLIST-2 fill:#00ff00
  style OBJ-C-1 fill:#00ffff,stroke:#000000,stroke-width:3px
  style LWB-1   fill:#ffff00
</div>

---

# Example: `zil_commit` Object C

<hr style="visibility:hidden;" />

<div class="mermaid">
graph LR
  CLIST(commit list) --> OBJ-C-2((itx C2))

  HEADER(ZIL header) --> LWB-1(lwb 1)
  LWB-1              --- SLIST-1((itx S1))
  LWB-1              --> LWB-2(lwb 2)
  LWB-2              --- SLIST-2((itx S2))
  SLIST-2            --- OBJ-C-1((itx C1))

  style SLIST-1 fill:#ffff00
  style SLIST-2 fill:#00ff00
  style OBJ-C-1 fill:#00ffff,stroke:#000000,stroke-width:3px
  style LWB-1   fill:#ffff00
</div>

---

# Example: `zil_commit` Object C

<hr style="visibility:hidden;" />

<div class="mermaid">
graph LR
  CLIST(commit list) --> OBJ-C-2((itx C2))

  HEADER(ZIL header) --> LWB-1(lwb 1)
  LWB-1              --- SLIST-1((itx S1))
  LWB-1              --> LWB-2(lwb 2)
  LWB-2              --- SLIST-2((itx S2))
  SLIST-2            --- OBJ-C-1((itx C1))

  style SLIST-1 fill:#ffff00
  style SLIST-2 fill:#00ff00
  style OBJ-C-1 fill:#00ffff
  style OBJ-C-2 fill:#ffa500,stroke:#000000,stroke-width:3px
  style LWB-1   fill:#ffff00
</div>

---

# Example: `zil_commit` Object C

<hr style="visibility:hidden;" />

<div class="mermaid">
graph LR
  CLIST(commit list) --> OBJ-C-2((itx C2))

  HEADER(ZIL header) --> LWB-1(lwb 1)
  LWB-1              --- SLIST-1((itx S1))
  LWB-1              --> LWB-2(lwb 2)
  LWB-2              --- SLIST-2((itx S2))
  SLIST-2            --- OBJ-C-1((itx C1))
  LWB-2              --> LWB-3(lwb 3)
  LWB-3

  style SLIST-1 fill:#ffff00
  style SLIST-2 fill:#00ff00
  style OBJ-C-1 fill:#00ffff
  style OBJ-C-2 fill:#ffa500,stroke:#000000,stroke-width:3px
  style LWB-1   fill:#ffff00
  style LWB-2   fill:#ffff00
</div>

---

# Example: `zil_commit` Object C

<hr style="visibility:hidden;" />

<div class="mermaid">
graph LR
  CLIST(commit list)

  HEADER(ZIL header) --> LWB-1(lwb 1)
  LWB-1              --- SLIST-1((itx S1))
  LWB-1              --> LWB-2(lwb 2)
  LWB-2              --- SLIST-2((itx S2))
  SLIST-2            --- OBJ-C-1((itx C1))
  LWB-2              --> LWB-3(lwb 3)
  LWB-3              --- OBJ-C-2((itx C2))


  style SLIST-1 fill:#ffff00
  style SLIST-2 fill:#00ff00
  style OBJ-C-1 fill:#00ffff
  style OBJ-C-2 fill:#ffa500
  style LWB-1   fill:#ffff00
  style LWB-2   fill:#ffff00
</div>

---

# Example: `zil_commit` Object C

<hr style="visibility:hidden;" />

<div class="mermaid">
graph LR
  CLIST(commit list)

  HEADER(ZIL header) --> LWB-1(lwb 1)
  LWB-1              --- SLIST-1((itx S1))
  LWB-1              --> LWB-2(lwb 2)
  LWB-2              --- SLIST-2((itx S2))
  SLIST-2            --- OBJ-C-1((itx C1))
  LWB-2              --> LWB-3(lwb 3)
  LWB-3              --- OBJ-C-2((itx C2))


  style SLIST-1 fill:#ffff00
  style SLIST-2 fill:#00ff00
  style OBJ-C-1 fill:#00ffff
  style OBJ-C-2 fill:#ffa500
  style LWB-1   fill:#ffff00
  style LWB-2   fill:#ffff00
  style LWB-3   fill:#ffff00
</div>

---

# How are itx's written to disk?

 - `zil_commit` handles the process of writing `itx_t`'s to disk:

     1. Move async itx's for object being commited, to the sync list

     2. Move sync list to "commit list"; sync list now empty

     3. For each itx in commit list:

         1. Copy itx data into ZIL block

         2. If space in ZIL block lacking...

           - allocate new (empty) block, write out old block

     4. Wait for all ZIL block writes to complete

---

# Example: `zil_commit` Object C

<hr style="visibility:hidden;" />

<div class="mermaid">
graph LR
  HEADER(ZIL header) --> LWB-1(lwb 1)
  LWB-1              --- SLIST-1((itx S1))
  LWB-1              --> LWB-2(lwb 2)
  LWB-2              --- SLIST-2((itx S2))
  SLIST-2            --- OBJ-C-1((itx C1))
  LWB-2              --> LWB-3(lwb 3)
  LWB-3              --- OBJ-C-2((itx C2))

  style LWB-1   fill:#ffff00
  style LWB-2   fill:#ffff00
  style LWB-3   fill:#ffff00
</div>

---

# Example: `zil_commit` Object C

<hr style="visibility:hidden;" />

<div class="mermaid">
graph LR
  HEADER(ZIL header) --> LWB-1(lwb 1)
  LWB-1              --- SLIST-1((itx S1))
  LWB-1              --> LWB-2(lwb 2)
  LWB-2              --- SLIST-2((itx S2))
  SLIST-2            --- OBJ-C-1((itx C1))
  LWB-2              --> LWB-3(lwb 3)
  LWB-3              --- OBJ-C-2((itx C2))

  style LWB-1   fill:#00ff00
  style LWB-2   fill:#ffff00
  style LWB-3   fill:#ffff00
</div>

---

# Example: `zil_commit` Object C

<hr style="visibility:hidden;" />

<div class="mermaid">
graph LR
  HEADER(ZIL header) --> LWB-1(lwb 1)
  LWB-1              --- SLIST-1((itx S1))
  LWB-1              --> LWB-2(lwb 2)
  LWB-2              --- SLIST-2((itx S2))
  SLIST-2            --- OBJ-C-1((itx C1))
  LWB-2              --> LWB-3(lwb 3)
  LWB-3              --- OBJ-C-2((itx C2))

  style LWB-1   fill:#00ff00
  style LWB-2   fill:#ffff00
  style LWB-3   fill:#00ff00
</div>

---

# Example: `zil_commit` Object C

<hr style="visibility:hidden;" />

<div class="mermaid">
graph LR
  HEADER(ZIL header) --> LWB-1(lwb 1)
  LWB-1              --- SLIST-1((itx S1))
  LWB-1              --> LWB-2(lwb 2)
  LWB-2              --- SLIST-2((itx S2))
  SLIST-2            --- OBJ-C-1((itx C1))
  LWB-2              --> LWB-3(lwb 3)
  LWB-3              --- OBJ-C-2((itx C2))

  style LWB-1   fill:#00ff00
  style LWB-2   fill:#00ff00
  style LWB-3   fill:#00ff00
</div>

---

# How are itx's written to disk?

 - `zil_commit` handles the process of writing `itx_t`'s to disk:

     1. Move async itx's for object being commited, to the sync list

     2. Move sync list to "commit list"; sync list now empty

     3. For each itx in commit list:

         1. Copy itx data into ZIL block

         2. If space in ZIL block lacking...

           - allocate new (empty) block, write out old block

     4. Wait for all ZIL block writes to complete

     5. Flush all VDEVs written to

---

# How are itx's written to disk?

 - `zil_commit` handles the process of writing `itx_t`'s to disk:

     1. Move async itx's for object being commited, to the sync list

     2. Move sync list to "commit list"; sync list now empty

     3. For each itx in commit list:

         1. Copy itx data into ZIL block

         2. If space in ZIL block lacking...

           - allocate new (empty) block, write out old block

     4. Wait for all ZIL block writes to complete

     5. Flush all VDEVs written to

     6. Notify waiting threads

---

class: middle, center

# 2 &ndash; ZIL Blocks

---

# Constraints

 - On disk component of ZIL is a linked list of ZIL blocks (lwb's)

    - Doesn't use indirection like "normal" blocks

       - no indirect blocks

    - Each ZIL block contains pointer to next ZIL block on disk

 - ZIL blocks, once written to disk, cannot be overwritten

    - Due to data integrity concerns

 - Thus, "next" ZIL block allocated when "current" block written

    - Required for "current" block to contain pointer to "next"

 - Checksum for "next" block, stored in "previous" block

    - When traversing list, checksum failure means "list end"

---

# Implications w.r.t. Performance

 - ZIL block size chosen at time of allocation

 - ZIL block size can dramatically impact performance:

    - "too big" &ndash; results in "wasted" space

       - Unused portion of block cannot be "filled in" later

       - Can result in small, but fast, SLOG filling to capacity

    - "too small" &ndash; results in "too many" IOPs issued to disk

       - Can needlessly cause disk saturation and poor performance

    - "just right" &ndash; Large (enough) ZIL blocks filled with itx's

       - ZIL blocks fully utilized with itx's (no wasted SLOG space)

       - Fewer IOPs for same number of itx's

          - e.g. 15 8K writes using 1 128K lwb vs. 4 32K lwb's

---

class: middle, center

# 3 &ndash; Problem

---

# Problem

 1. itx's grouped and written in "batches"

    - The commit list constitutes a batch

    - Batch size proportional to sync workload on system

 2. Waiting threads only notified when _all_ ZIL blocks in batch complete

 3. Only a single batch processed at a time

---

# Example Batch

 - Example batch taken from prior slides

<div class="mermaid">
graph LR
  HEADER(ZIL header) --> LWB-1(lwb 1)
  LWB-1              --- SLIST-1((itx S1))
  LWB-1              --> LWB-2(lwb 2)
  LWB-2              --- SLIST-2((itx S2))
  SLIST-2            --- OBJ-C-1((itx C1))
  LWB-2              --> LWB-3(lwb 3)
  LWB-3              --- OBJ-C-2((itx C2))
</div>

---


# Example "itx S1"

 - Thread waiting for "itx S1" (e.g. a single sync write)

<div class="mermaid">
graph LR
  HEADER(ZIL header) --> LWB-1(lwb 1)
  LWB-1              --- SLIST-1((itx S1))
  LWB-1              --> LWB-2(lwb 2)
  LWB-2              --- SLIST-2((itx S2))
  SLIST-2            --- OBJ-C-1((itx C1))
  LWB-2              --> LWB-3(lwb 3)
  LWB-3              --- OBJ-C-2((itx C2))

  style SLIST-1 fill:#ffff00,stroke:#000000,stroke-width:3px
</div>

---

# Example "itx S1"

 - Must wait for **all** lwb's to be written and completed

<div class="mermaid">
graph LR
  HEADER(ZIL header) --> LWB-1(lwb 1)
  LWB-1              --- SLIST-1((itx S1))
  LWB-1              --> LWB-2(lwb 2)
  LWB-2              --- SLIST-2((itx S2))
  SLIST-2            --- OBJ-C-1((itx C1))
  LWB-2              --> LWB-3(lwb 3)
  LWB-3              --- OBJ-C-2((itx C2))

  style SLIST-1 fill:#ffff00,stroke:#000000,stroke-width:3px
  style LWB-1   fill:#ffff00,stroke:#000000,stroke-width:3px
  style LWB-2   fill:#ffff00,stroke:#000000,stroke-width:3px
  style LWB-3   fill:#ffff00,stroke:#000000,stroke-width:3px
</div>

---

# Example "itx S2"

 - Thread waiting for "itx S2"

<div class="mermaid">
graph LR
  HEADER(ZIL header) --> LWB-1(lwb 1)
  LWB-1              --- SLIST-1((itx S1))
  LWB-1              --> LWB-2(lwb 2)
  LWB-2              --- SLIST-2((itx S2))
  SLIST-2            --- OBJ-C-1((itx C1))
  LWB-2              --> LWB-3(lwb 3)
  LWB-3              --- OBJ-C-2((itx C2))

  style SLIST-2 fill:#ffff00,stroke:#000000,stroke-width:3px
</div>

---

# Example "itx S2"

 - Again, must wait for **all** lwb's to be written and completed

<div class="mermaid">
graph LR
  HEADER(ZIL header) --> LWB-1(lwb 1)
  LWB-1              --- SLIST-1((itx S1))
  LWB-1              --> LWB-2(lwb 2)
  LWB-2              --- SLIST-2((itx S2))
  SLIST-2            --- OBJ-C-1((itx C1))
  LWB-2              --> LWB-3(lwb 3)
  LWB-3              --- OBJ-C-2((itx C2))

  style SLIST-2 fill:#ffff00,stroke:#000000,stroke-width:3px
  style LWB-1   fill:#ffff00,stroke:#000000,stroke-width:3px
  style LWB-2   fill:#ffff00,stroke:#000000,stroke-width:3px
  style LWB-3   fill:#ffff00,stroke:#000000,stroke-width:3px
</div>

---

# Implications

 1. Latency of `zil_commit` proportional to system workload, not disk latency

    - Fast SLOG may not compensate for large workload

 2. Disk "anomalies" cause larger batches, increased `zil_commit` latency

    - E.g. Temporary network delays when using SAN storage

 3. New calls to `zil_commit` wait for "current" batch, _and_ "next" batch

    - Average `zil_commit` latency equal to latency of 1.5 batches

---

class: middle, center

# 3 &ndash; Solution

---


# Solution

 - Remove concept of "batches":

    1. New calls to `zil_commit` can issue ZIL block writes immediately ...

       - Rather than waiting for "current" batch to complete

    2. Threads notified immediately when **dependent** blocks complete ...

       - Rather than when **all** writes complete (dependent or not)

---

# Example "Batch"

 - Same example as before

<div class="mermaid">
graph LR
  HEADER(ZIL header) --> LWB-1(lwb 1)
  LWB-1              --- SLIST-1((itx S1))
  LWB-1              --> LWB-2(lwb 2)
  LWB-2              --- SLIST-2((itx S2))
  SLIST-2            --- OBJ-C-1((itx C1))
  LWB-2              --> LWB-3(lwb 3)
  LWB-3              --- OBJ-C-2((itx C2))
</div>

---

# Example "itx S1"

 - Thread waiting for "itx S1"

<div class="mermaid">
graph LR
  HEADER(ZIL header) --> LWB-1(lwb 1)
  LWB-1              --- SLIST-1((itx S1))
  LWB-1              --> LWB-2(lwb 2)
  LWB-2              --- SLIST-2((itx S2))
  SLIST-2            --- OBJ-C-1((itx C1))
  LWB-2              --> LWB-3(lwb 3)
  LWB-3              --- OBJ-C-2((itx C2))

  style SLIST-1 fill:#ffff00,stroke:#000000,stroke-width:3px
</div>

---

# Example "itx S1"

 - Must wait for _only_ "lwb 1" to complete

<div class="mermaid">
graph LR
  HEADER(ZIL header) --> LWB-1(lwb 1)
  LWB-1              --- SLIST-1((itx S1))
  LWB-1              --> LWB-2(lwb 2)
  LWB-2              --- SLIST-2((itx S2))
  SLIST-2            --- OBJ-C-1((itx C1))
  LWB-2              --> LWB-3(lwb 3)
  LWB-3              --- OBJ-C-2((itx C2))

  style SLIST-1 fill:#ffff00,stroke:#000000,stroke-width:3px
  style LWB-1   fill:#ffff00,stroke:#000000,stroke-width:3px
</div>

---

# Example "itx S2"

 - Thread waiting for "itx S2"

<div class="mermaid">
graph LR
  HEADER(ZIL header) --> LWB-1(lwb 1)
  LWB-1              --- SLIST-1((itx S1))
  LWB-1              --> LWB-2(lwb 2)
  LWB-2              --- SLIST-2((itx S2))
  SLIST-2            --- OBJ-C-1((itx C1))
  LWB-2              --> LWB-3(lwb 3)
  LWB-3              --- OBJ-C-2((itx C2))

  style SLIST-2 fill:#ffff00,stroke:#000000,stroke-width:3px
</div>

---

# Example "itx S2"

 - Must wait for _only_ "lwb 1" and "lwb 2"

<div class="mermaid">
graph LR
  HEADER(ZIL header) --> LWB-1(lwb 1)
  LWB-1              --- SLIST-1((itx S1))
  LWB-1              --> LWB-2(lwb 2)
  LWB-2              --- SLIST-2((itx S2))
  SLIST-2            --- OBJ-C-1((itx C1))
  LWB-2              --> LWB-3(lwb 3)
  LWB-3              --- OBJ-C-2((itx C2))

  style SLIST-2 fill:#ffff00,stroke:#000000,stroke-width:3px
  style LWB-1   fill:#ffff00,stroke:#000000,stroke-width:3px
  style LWB-2   fill:#ffff00,stroke:#000000,stroke-width:3px
</div>

---

class: middle, center

# 4 &ndash; Changes to VDEV Flush

---

# Details

 - A ZIL block not considered "safe" on disk until VDEV is flushed

 - Prior mechanics:

    1. All ZIL blocks written for entire batch

    2. Single VDEV flush issued to each VDEV written to

    - 1 flush to many lwb's

 - New mechanics:

    1. Single ZIL block written to single VDEV

    2. Single VDEV flush issued to that specific VDEV

    - 1 flush to 1 lwb

 - `dmu_sync` (indirect writes) complicates this slightly

---

# Example: Before

<hr style="visibility:hidden;" />

<div class="mermaid">
sequenceDiagram
  zil_commit ->>      VDEV 1: lwb 1 write
  zil_commit ->>      VDEV 2: lwb 2 write
  zil_commit ->>      VDEV 1: lwb 3 write
  VDEV 1     -->> zil_commit: lwb 1 done
  VDEV 2     -->> zil_commit: lwb 2 done
  VDEV 1     -->> zil_commit: lwb 3 done
  zil_commit ->>      VDEV 1: flush
  zil_commit ->>      VDEV 2: flush
</div>

---

# Example: After

<hr style="visibility:hidden;" />

<div class="mermaid">
sequenceDiagram
  zil_commit ->>      VDEV 1: lwb 1 write
  zil_commit ->>      VDEV 2: lwb 2 write
  zil_commit ->>      VDEV 1: lwb 3 write
  VDEV 1     -->> zil_commit: lwb 1 done
  zil_commit ->>      VDEV 1: flush
  VDEV 2     -->> zil_commit: lwb 2 done
  zil_commit ->>      VDEV 2: flush
  VDEV 1     -->> zil_commit: lwb 3 done
  zil_commit ->>      VDEV 1: flush
</div>

---

class: middle, center

# 4 &ndash; Changes to ZIL Block ZIO Tree

---

# Details

 - ZIL Blocks issued to disk using ZIOs

 - ZIOs represented as directed acyclic graph (DAG)

    - Parent ZIOs cannot complete until all children complete

 - Prior mechanics:

    - "root" ZIO created for each batch, lwb ZIOs are children

    - Flush issued after root ZIO completes (seperate ZIO tree)

 - New mechanics:

    - "root" ZIO created for each lwb

    - "write" and "flush" ZIOs are child of root ZIO

    - "next" lwb root ZIO become parent of "current" lwb root ZIO

---

# Example: Before

<hr style="visibility:hidden;" />

.pull-left[<div class="mermaid">
graph TD
  ROOT(batch root)
</div>]

---

# Example: Before

<hr style="visibility:hidden;" />

.pull-left[<div class="mermaid">
graph TD
  ROOT(batch root) --> LWB-1(lwb 1)
</div>]

---

# Example: Before

<hr style="visibility:hidden;" />

.pull-left[<div class="mermaid">
graph TD
  ROOT(batch root) --> LWB-1(lwb 1)
  ROOT             --> LWB-2(lwb 2)
</div>]

---

# Example: Before

<hr style="visibility:hidden;" />

.pull-left[<div class="mermaid">
graph TD
  ROOT(batch root) --> LWB-1(lwb 1)
  ROOT             --> LWB-2(lwb 2)
  ROOT             --> LWB-3(lwb 3)
</div>]

---

# Example: Before

<hr style="visibility:hidden;" />

.pull-left[<div class="mermaid">
graph TD
  ROOT(batch root) --> LWB-1(lwb 1)
  ROOT             --> LWB-2(lwb 2)
  ROOT             --> LWB-3(lwb 3)
</div>]

.pull-right[<div class="mermaid">
graph TD
  ROOT(flush root)
</div>]

---

# Example: Before

<hr style="visibility:hidden;" />

.pull-left[<div class="mermaid">
graph TD
  ROOT(batch root) --> LWB-1(lwb 1)
  ROOT             --> LWB-2(lwb 2)
  ROOT             --> LWB-3(lwb 3)
</div>]

.pull-right[<div class="mermaid">
graph TD
  ROOT(flush root) --> VDEV-1(VDEV 1 flush)
</div>]

---

# Example: Before

<hr style="visibility:hidden;" />

.pull-left[<div class="mermaid">
graph TD
  ROOT(batch root) --> LWB-1(lwb 1)
  ROOT             --> LWB-2(lwb 2)
  ROOT             --> LWB-3(lwb 3)
</div>]

.pull-right[<div class="mermaid">
graph TD
  ROOT(flush root) --> VDEV-1(VDEV 1 flush)
  ROOT             --> VDEV-2(VDEV 2 flush)
</div>]

---

# Example: After

<hr style="visibility:hidden;" />

<div class="mermaid">
graph LR
  LWB-1-ROOT(lwb 1 root)
</div>

---

# Example: After

<hr style="visibility:hidden;" />

<div class="mermaid">
graph LR
  LWB-1-ROOT(lwb 1 root)
  LWB-1-ROOT             --> LWB-1-WRITE(lwb 1 write)
</div>

---

# Example: After

<hr style="visibility:hidden;" />

<div class="mermaid">
graph LR
  LWB-1-ROOT(lwb 1 root)
  LWB-1-ROOT             --> LWB-1-WRITE(lwb 1 write)
  LWB-1-ROOT             --> LWB-1-FLUSH(VDEV 1 flush)
</div>

---

# Example: After

<hr style="visibility:hidden;" />

<div class="mermaid">
graph LR
  LWB-2-ROOT(lwb 2 root) --> LWB-1-ROOT

  LWB-1-ROOT(lwb 1 root)
  LWB-1-ROOT             --> LWB-1-WRITE(lwb 1 write)
  LWB-1-ROOT             --> LWB-1-FLUSH(VDEV 1 flush)
</div>

---

# Example: After

<hr style="visibility:hidden;" />

<div class="mermaid">
graph LR
  LWB-2-ROOT(lwb 2 root) --> LWB-1-ROOT
  LWB-2-ROOT             --> LWB-2-WRITE(lwb 2 write)

  LWB-1-ROOT(lwb 1 root)
  LWB-1-ROOT             --> LWB-1-WRITE(lwb 1 write)
  LWB-1-ROOT             --> LWB-1-FLUSH(VDEV 1 flush)
</div>

---

# Example: After

<hr style="visibility:hidden;" />

<div class="mermaid">
graph LR
  LWB-2-ROOT(lwb 2 root) --> LWB-1-ROOT
  LWB-2-ROOT             --> LWB-2-WRITE(lwb 2 write)
  LWB-2-ROOT             --> LWB-2-FLUSH(VDEV 2 flush)

  LWB-1-ROOT(lwb 1 root)
  LWB-1-ROOT             --> LWB-1-WRITE(lwb 1 write)
  LWB-1-ROOT             --> LWB-1-FLUSH(VDEV 1 flush)
</div>

---

# Example: After

<hr style="visibility:hidden;" />

<div class="mermaid">
graph LR
  LWB-3-ROOT(lwb 3 root) --> LWB-2-ROOT

  LWB-2-ROOT(lwb 2 root) --> LWB-1-ROOT
  LWB-2-ROOT             --> LWB-2-WRITE(lwb 2 write)
  LWB-2-ROOT             --> LWB-2-FLUSH(VDEV 2 flush)

  LWB-1-ROOT(lwb 1 root)
  LWB-1-ROOT             --> LWB-1-WRITE(lwb 1 write)
  LWB-1-ROOT             --> LWB-1-FLUSH(VDEV 1 flush)
</div>

---

# Example: After

<hr style="visibility:hidden;" />

<div class="mermaid">
graph LR
  LWB-3-ROOT(lwb 3 root) --> LWB-2-ROOT
  LWB-3-ROOT             --> LWB-3-WRITE(lwb 3 write)

  LWB-2-ROOT(lwb 2 root) --> LWB-1-ROOT
  LWB-2-ROOT             --> LWB-2-WRITE(lwb 2 write)
  LWB-2-ROOT             --> LWB-2-FLUSH(VDEV 2 flush)

  LWB-1-ROOT(lwb 1 root)
  LWB-1-ROOT             --> LWB-1-WRITE(lwb 1 write)
  LWB-1-ROOT             --> LWB-1-FLUSH(VDEV 1 flush)
</div>

---

# Example: After

<hr style="visibility:hidden;" />

<div class="mermaid">
graph LR
  LWB-3-ROOT(lwb 3 root) --> LWB-2-ROOT
  LWB-3-ROOT             --> LWB-3-WRITE(lwb 3 write)
  LWB-3-ROOT             --> LWB-3-FLUSH(VDEV 1 flush)

  LWB-2-ROOT(lwb 2 root) --> LWB-1-ROOT
  LWB-2-ROOT             --> LWB-2-WRITE(lwb 2 write)
  LWB-2-ROOT             --> LWB-2-FLUSH(VDEV 2 flush)

  LWB-1-ROOT(lwb 1 root)
  LWB-1-ROOT             --> LWB-1-WRITE(lwb 1 write)
  LWB-1-ROOT             --> LWB-1-FLUSH(VDEV 1 flush)
</div>

---

# Example: After

<hr style="visibility:hidden;" />

<div class="mermaid">
graph LR
  LWB-4-ROOT(lwb 4 root) --> LWB-3-ROOT
  LWB-4-ROOT             --> LWB-4-WRITE(lwb 4 write)
  LWB-4-ROOT             --> LWB-4-FLUSH(VDEV flush)

  LWB-3-ROOT(lwb 3 root) --> LWB-2-ROOT
  LWB-3-ROOT             --> LWB-3-WRITE(lwb 3 write)
  LWB-3-ROOT             --> LWB-3-FLUSH(VDEV 1 flush)

  LWB-2-ROOT(lwb 2 root) --> LWB-1-ROOT
  LWB-2-ROOT             --> LWB-2-WRITE(lwb 2 write)
  LWB-2-ROOT             --> LWB-2-FLUSH(VDEV 2 flush)

  LWB-1-ROOT(lwb 1 root)
  LWB-1-ROOT             --> LWB-1-WRITE(lwb 1 write)
  LWB-1-ROOT             --> LWB-1-FLUSH(VDEV 1 flush)
</div>

---

# Example: After

<hr style="visibility:hidden;" />

<div class="mermaid">
graph LR
  LWB-5-ROOT(lwb 5 root) --> LWB-4-ROOT
  LWB-5-ROOT             --> LWB-5-WRITE(lwb 5 write)
  LWB-5-ROOT             --> LWB-5-FLUSH(VDEV flush)

  LWB-4-ROOT(lwb 4 root) --> LWB-3-ROOT
  LWB-4-ROOT             --> LWB-4-WRITE(lwb 4 write)
  LWB-4-ROOT             --> LWB-4-FLUSH(VDEV flush)

  LWB-3-ROOT(lwb 3 root) --> LWB-2-ROOT
  LWB-3-ROOT             --> LWB-3-WRITE(lwb 3 write)
  LWB-3-ROOT             --> LWB-3-FLUSH(VDEV 1 flush)

  LWB-2-ROOT(lwb 2 root) --> LWB-1-ROOT
  LWB-2-ROOT             --> LWB-2-WRITE(lwb 2 write)
  LWB-2-ROOT             --> LWB-2-FLUSH(VDEV 2 flush)

  LWB-1-ROOT(lwb 1 root)
  LWB-1-ROOT             --> LWB-1-WRITE(lwb 1 write)
  LWB-1-ROOT             --> LWB-1-FLUSH(VDEV 1 flush)
</div>

---

class: middle, center

# 4 &ndash; Changes to Waiter Notification

---

# Details: Before

 - 2 condition variables (CV), for "current" and "next" batch

 - Threads call `zil_commit`:

    - Assigned to "next" batch, wait on "next" batch's CV

 - When "current" batch completes

    - All threads waiting on "current" signalled, they return

    - One thread waiting on "next" signalled, becomes "writer"

    - "next" and "current" CV swapped (e.g. "next" &rarr; "current")

 - Ultimately, these two CVs are the source of original problem

---

# Example: Before

<hr style="visibility:hidden;" />

<div class="mermaid">
sequenceDiagram
  participant processes
  participant CV next
  participant CV current
  participant writer

  processes  ->>     writer: process 1 becomes writer
  processes  ->>    CV next: process 2 waits
  processes  ->>    CV next: process 3 waits
  writer     ->>  processes: process 1 done writing
  writer     ->> CV current: signalled
  writer     ->>    CV next: process 3 signalled
  processes  ->>     writer: process 3 becomes writer
  processes -->> CV current: process 2 still waiting
  processes  ->>    CV next: process 4 waits
  writer     ->>  processes: process 3 done writing
  writer     ->> CV current: process 2 signalled
  writer     ->>    CV next: process 4 signalled
</div>

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

<hr style="visibility:hidden;" />

<div class="mermaid">
sequenceDiagram
  participant processes
  participant CV
  participant writer
  participant ZIO done

  processes ->>      writer: process 1 becomes writer
  writer    ->>   processes: process 1 done writing
  processes ->>          CV: process 1 waits
  processes ->>      writer: process 2 becomes writer
  ZIO done  ->>          CV: lwb done &ndash; process 1 signalled
  writer    ->>   processes: process 2 done writing
  processes ->>          CV: process 2 waits
  processes ->>      writer: process 3 becomes writer
  writer    ->>   processes: process 3 done writing
  processes ->>          CV: process 3 waits
  ZIO done  ->>          CV: lwb done &ndash; process 2 signalled
  ZIO done  ->>          CV: lwb done &ndash; process 3 signalled
</div>

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
background-size: 100%

# % Change IOPs &ndash; Max Rate &ndash; HDDs

---

background-image: url(max-rate-hdd-lat-pctchange.png)
background-size: 100%

# % Change Latency &ndash; Max Rate &ndash; HDDs

---

class: middle, center

# 5 &ndash; Max Rate Workload &ndash; SSDs

---

background-image: url(max-rate-ssd-iops-pctchange.png)
background-size: 100%

# % Change IOPs &ndash; Max Rate &ndash; SSDs

---

background-image: url(max-rate-ssd-lat-pctchange.png)
background-size: 100%

# % Change Latency &ndash; Max Rate &ndash; SSDs

---

class: middle, center

# 5 &ndash; Fixed Rate Workload &ndash; HDDs

---

background-image: url(fixed-rate-hdd-iops-pctchange.png)
background-size: 100%

# % Change IOPs &ndash; Fixed Rate &ndash; HDDs

---

background-image: url(fixed-rate-hdd-lat-pctchange.png)
background-size: 100%

# % Change Latency &ndash; Fixed Rate &ndash; HDDs

---

class: middle, center

# 5 &ndash; Fixed Rate Workload &ndash; SSDs

---

background-image: url(fixed-rate-ssd-iops-pctchange.png)
background-size: 100%

# % Change IOPs &ndash; Fixed Rate &ndash; SSDs

---

background-image: url(fixed-rate-ssd-lat-pctchange.png)
background-size: 100%

# % Change Latency &ndash; Fixed Rate &ndash; SSDs

[perf-results]: https://www.prakashsurya.com/post/2017-09-08-performance-testing-results-for-openzfs-447/
