+++
date = "2018-04-23T00:00:00-08:00"
lastmod = "2018-04-23T00:00:00-08:00"
title = "Ubuntu-Based Appliance Build Architecture"
presenter = "Prakash Surya"
type = "slides"
draft = false
+++

# Agenda

### 1. Overview of current illumos-based build architecture.

### 2. Problems with current build architecture.

### 3. Goals of new Ubuntu-based build architecture.

### 4. Overview of new build architecture.

### 5. Details of new build architecture.

.footnote[<sup>\*</sup>Press "p" for notes, and "c" for split view.]

---

class: middle, center

# 1 &ndash; Overview of current build architecture

---

# Overview of current build architecture

 - Converts source code, into virtual machine artifacts

    - OVA, VHD, qcow2, etc.

 - Steps involved:

    1. Clone repositories

    2. Build repositories

    3. Generate "OS" cpio archive

    4. Generate "DX" cpio archive

    5. Generate upgrade image

    6. Generate ISO

    7. Install from ISO (for each hypervisor)

    8. Generate VM artifacts

---

class: middle, center

![](overview-of-current-architecture.svg)

---

class: middle, center

# 2 &ndash; Problems with current build architecture

---

# Problem: Latency

 - Time to checkout all of our repositories (5 mins)

 - Time to build all of our repositories (70 mins)

 - Time to generate upgrade image (25 mins)

 - Time to build ISO (25 mins)

 - Time to copy artifacts (ISO + upgrade image) to NAS (5 mins)

 - Time to install from ISO on all supported hypervisors

    - VMware &rarr; OVA (20 mins) + DCenter Snapshots (25 mins)

    - Xen &rarr; VMDK (20 mins) + AWS Snapshots (70 mins)

    - HyperV &rarr; VHD (90 mins) + Azure Snapshots (40 mins)

## Total Latency: 4 hours, 30 mins

---

# Problem: Reliability

 - If any one step fails, the whole build fails

    - Checkout step can fail due to broken infrastructure

    - Build step can fail due to broken code checked in

    - Upgrade image generation can fail due to broken IPS repo

       - No change control over IPS repo

    - ISO install can fail due to broken hypervisors

    - Snapshots can fail due to broken Ansible roles
---

# Problem: Product Variants

 - We actually need to build multiple variantions of the appliance:

    - Standard, Demo, SAP, AWS Marketplace, AWS GovCloud, etc.

 - Each variation &times; each hypervisor, requires a unique ISO install:

    - VMware &rarr; Standard OVA

    - VMware &rarr; Demo OVA

    - VMware &rarr; SAP OVA

    - KVM &rarr; Standard QCOW2

    - HyperV &rarr; Standard VHD

    - Xen &rarr; AWS Standard AMI

    - Xen &rarr; AWS Marketplace AMI

    - Xen &rarr; AWS GovCloud AMI

---

class: middle, center

# 3 &ndash; Goals of new build architecture

---

# Goals of new build architecture

## 1. Eliminate build of each repository

  - Consume packages, not source code

  - Each repository built externally &mdash; produces `.deb` package

## 2. Eliminate hypervisor specific ISO installs

 - Generate bootable virtual machine (VM) artifacts; OVA, VHD, etc.

 - Don't need "on-prem" hypervisors to produce VM artifacts

## 3. Support different product, development, and QA variations

 - Generate different VM artifacts for our different product types

 - Generate different VM artifacts for QA and development purposes

---

class: middle, center

# 4 &ndash; Overview of new build architecture

---

class: middle, center

# 5 &ndash; Details of new build architecture
