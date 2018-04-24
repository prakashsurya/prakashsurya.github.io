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

 - Converts source code, into virtual machine (VM) artifacts

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

![:scale 100%](overview-of-current-architecture.svg)

---

class: middle, center

# 2 &ndash; Problems with current build architecture

---

# Latency

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

# Reliability

 - If any one step fails, the whole build fails

    - Checkout step can fail due to broken infrastructure

    - Build step can fail due to broken code checked in

    - Upgrade image generation can fail due to broken IPS repo

       - No change control over IPS repo

    - ISO install can fail due to broken hypervisors

    - Snapshots can fail due to broken Ansible roles
---

# Product Variants

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

 - Generate bootable VM artifacts; OVA, VHD, etc.

 - Don't need "on-prem" hypervisors to produce VM artifacts

## 3. Support different product, development, and QA variations

 - Generate different VM artifacts for our different product types

 - Generate different VM artifacts for QA and development purposes

---

class: middle, center

# 4 &ndash; Overview of new build architecture

---

# Overview of new build architecture

 - Converts *packages* into VM artifacts

    - OVA, VHD, qcow2, etc.

 - Steps involved:

    1. Install "base" packages into a "chroot" directory

    2. Copy "base" directory into variant-specific directories

    3. Apply unique customizations to variant-specific directories

    4. Convert variant "chroot" directories into VM artifacts

---

class: middle, center

![:scale 100%](overview-of-new-architecture.svg)

---

class: middle, center

# 5 &ndash; Details of new build architecture

---

# Details of new build architecture

 - Build logic contained in new "appliance-build" repository

    - Currently on GitLab, eventually on GitHub

 - Repository contents:

     ```
     $ ls -lF
     total 8
     drwxr-xr-x 2 psurya psurya    3 Apr 18 21:35 docker/
     drwxr-xr-x 5 psurya psurya   10 Apr 16 20:16 jenkins/
     drwxr-xr-x 6 psurya psurya    6 Apr 18 20:38 live-build/
     -rw-r--r-- 1 psurya psurya  941 Apr 21 18:11 Makefile
     -rw-r--r-- 1 psurya psurya 4196 Apr 12 08:37 README.md
     drwxr-xr-x 2 psurya psurya    8 Apr 22 17:03 scripts/
     ```

---

# Docker Expected

 - Build is expected to be run inside Docker container

 - All build dependencies specified in `docker/Dockerfile`:

     ```
     RUN \
       apt-get update && \
       apt-get install -y software-properties-common && \
       apt-add-repository -y ppa:ansible/ansible && \
       apt-get update && \
       apt-get install -y \
           ansible=2.5.* \
           livecd-rootfs=2.478.1 \
           zfsutils-linux=0.6.5.11-* \
           awscli \
           coreutils \
           gdisk \
           git \
           jq \
           kpartx \
           make \
           man \
           shellcheck \
           vim && \
       rm -rf /var/lib/apt/lists/*
     ```

 - Plus, ZFS kernel modules from Docker host

---

# Useful Scripts

 - Various scripts used to aid development:

     ```
     $ ls -lF scripts
     total 9
     -rwxr-xr-x 1 psurya psurya  288 Apr 12 08:37 docker-build.sh*
     -rwxr-xr-x 1 psurya psurya  828 Apr 16 21:36 docker-install-ubuntu.sh*
     -rwxr-xr-x 1 psurya psurya 1026 Apr 19 23:18 docker-run.sh*
     -rwxr-xr-x 1 psurya psurya  199 Apr 12 08:37 kernel-modules-load.sh*
     -rwxr-xr-x 1 psurya psurya  202 Apr 12 08:37 kernel-modules-unload.sh*
     -rwxr-xr-x 1 psurya psurya 2441 Apr 18 23:48 run-live-build.sh*
     ```

 - Example:

     ```
     $ sudo ./scripts/docker-install-ubuntu.sh
     $ sudo ./scripts/kernel-modules-load.sh
     $ sudo ./scripts/docker-build.sh
     $ sudo ./scripts/docker-run.sh make
     $ sudo ./scripts/kernel-modules-unload.sh
     ```

 - `Makefile` leverages `./scripts/run-live-build.sh` internally

---

# Base Configuration

 - All "variants" derive from "base" configuration:

     ```
     $ tree live-build/base/config/
     live-build/base/config/
     ├── archives
     │   ├── llvm.key
     │   ├── llvm.list
     │   ├── postgres.key
     │   └── postgres.list
     ├── hooks
     │   └── 00-oracle-java8.chroot
     └── package-lists
         ├── minimal.list.chroot
         ├── oracle-java8.list.chroot
         └── zfs-rootfs.list.chroot
     ```
     ```
     $ cat live-build/base/config/package-lists/minimal.list.chroot
     cloud-init
     debsums
     net-tools
     open-vm-tools
     openssh-server
     ubuntu-minimal
     ```

---

# Variants Configuration

 - Product variants fully customized via Ansible:

     ```
     ± tree live-build/variants/
     live-build/variants/
     ├── external-standard
     │   ├── ansible
     │   │  ├── playbook.yml
     │   │  └── roles -> ../../../misc/ansible-roles
     │   └── config
     │       └── hooks -> ../../../misc/live-build-hooks
     ├── internal-dev
     │   ├── ansible
     │   │  ├── playbook.yml
     │   │  └── roles -> ../../../misc/ansible-roles
     │   └── config
     │       └── hooks -> ../../../misc/live-build-hooks
     └── internal-qa
         ├── ansible
         │  ├── playbook.yml
         │  └── roles -> ../../../misc/ansible-roles
         └── config
             └── hooks -> ../../../misc/live-build-hooks
     ```

---

# Variant Configuration Examples

 - Variant "external-standard" configuration:

     ```
     $ cat live-build/variants/external-standard/ansible/playbook.yml
     - hosts: all
       connection: chroot
       gather_facts: no
       vars:
         ansible_python_interpreter: /usr/bin/python3
       roles:
         - appliance-build.minimal-common
         - appliance-build.minimal-external
         - appliance-build.masking-common
         - appliance-build.virtualization-common
         - appliance-build.virtualization-external
         - appliance-build.virtualization-standard
     ```

---

# Variant Configuration Examples

 - Variant "internal-dev" configuration:

     ```
     $ cat live-build/variants/internal-dev/ansible/playbook.yml
     - hosts: all
       connection: chroot
       gather_facts: no
       vars:
         ansible_python_interpreter: /usr/bin/python3
       roles:
         - appliance-build.minimal-common
         - appliance-build.minimal-internal
         - appliance-build.masking-common
         - appliance-build.masking-development
         - appliance-build.virtualization-common
         - appliance-build.virtualization-internal
         - appliance-build.virtualization-standard
         - appliance-build.virtualization-development
         - appliance-build.zfsonlinux-development
     ```

---

# Variants Built via live-build Hooks

 - `live-build` binary hooks used to:

     1. Run Ansible

     2. Build VM artifacts

 - All variants share the following hooks:

     ```
     $ ls -lF live-build/misc/live-build-hooks/
     total 16
     -rwxr-xr-x 1 psurya psurya  684 Apr 19 22:45 80-ansible-playbook.binary*
     -rwxr-xr-x 1 psurya psurya  794 Apr 19 22:45 81-etc-resolv-conf.binary*
     -rwxr-xr-x 1 psurya psurya 3770 Apr 16 21:36 90-raw-disk-image.binary*
     -rwxr-xr-x 1 psurya psurya  389 Apr 16 16:42 91-qcow2-disk-image.binary*
     -rwxr-xr-x 1 psurya psurya  328 Apr 20 23:44 91-vhd-disk-image.binary*
     -rwxr-xr-x 1 psurya psurya  333 Apr 20 23:44 91-vhdx-disk-image.binary*
     -rwxr-xr-x 1 psurya psurya  803 Apr 20 22:04 91-vmdk-disk-image.binary*
     -rwxr-xr-x 1 psurya psurya 1861 Apr 20 22:04 92-ova-machine-image.binary*
     ```

 - Hooks executed in order, resulting in variant VM artifacts

---

# Summary

 1. Docker container contains build environment/dependencies

 2. live-build used to build "base" configuration

     - output: "binary" directory of root filesystem contents

 3. Ansible used to customized "base" into variants

     - output: customized "binary" directory specific to variant

 4. VM artifacts generated from customized "binary" directory

     - output: `.ova`, `.qcow2`, `.vhd`, etc.
