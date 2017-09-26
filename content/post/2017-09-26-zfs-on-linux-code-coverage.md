+++
date = "2017-09-26T00:00:00-08:00"
lastmod = "2017-09-26T00:00:00-08:00"
title = "ZFS on Linux Code Coverage"
type = "slides"
draft = false
+++

# Branches + Pull Requests

 - Code coverage data is collected for:

    - All commits merged to a branch (e.g. master)

    - All pull requests for the "zfs" project

 - Code coverage collected after running **all** tests

    - ztest, zfstest, zfsstress, etc.

 - Data generated using `make code-coverage-capture` ...

    - Emits `.info` file and static HTML pages

 - `.info` file uploaded to `codecov.io`

 - [ZFS on Linux + Codecov](https://codecov.io/gh/zfsonlinux/zfs)

---

# User + Kernel

 - Code coverage data is collected for:

    - User mode execution (e.g. libzpool, zdb, etc.)

    - Kernel mode execution (e.g. "zfs" kernel module)

 - Same file _may_ be executed in user and kernel mode

    - e.g. libzpool references files in "modules" directories

 - User coverage enabled via `--enable-code-coverage` option

 - Kernel coverage enabled via custom kernel

    - More details [here](/post/2017-09-11-using-gcov-with-zfs-on-linux-kernel-modules/)

---

# Codecov: Header

.center[![:scale 100%](header.png)]

---

# Codecov: Diff

.center[![:scale 100%](diff.png)]

---

# Codecov: Files

.center[![:scale 100%](files.png)]

---

# Codecov: Files (continued)

.center[![:scale 100%](files-module-zfs.png)]

---

# Codecov: Flags

.center[![:scale 100%](flags.png)]

---

# Codecov: Build

.center[![:scale 100%](build.png)]

---

# Codecov: Graphs

.center[![:scale 100%](graphs.png)]

---

# Codecov: PR Comment

.center[![:scale 75%](pr-comment.png)]

---

# Coverage Integrated w/ Commits Page

.center[![:scale 100%](commits-page.png)]
