+++
date = "2018-11-29T00:00:00-08:00"
lastmod = "2018-11-29T00:00:00-08:00"
title = "Robust Upgrade on Ubuntu, using ZFS and Containers"
draft = false
+++

## Introduction

Recently I stumbled across a twitter post, highlighting the benefits of
ZFS Boot Environments; see [here][1]. Next in that thread, it states:

> Unfortunately, #linux has little to offer with the same functionality
> as #ZFS, especially with Red Hat abandoning #BTRFS

See [here][2]. I took this as an implication that it's not possible to
implement a solution similar to ZFS Boot Environments on Linux, which I
don't completely agree with.

The following provides some details about how ZFS Boot Environments
could be implemented on Ubuntu, if somebody wanted to do it (we're doing
a variation of this at Delphix).

## Step 1: Create an Ubuntu system using ZFS as the root filesystem

Before we can begin, we first need to build an Ubuntu system such that
it uses ZFS for the root filesystem; this means using [OpenZFS on
Linux][3], which is readily available and supported on the most recent
Ubuntu releases.

There's many different ways to go about doing this, and there's also
excellent tutorials such as [this one][4], so I'll leave this as an
exercise for the reader (we currently use [this script][5] at Delphix).

As an example, I have the following Ubuntu 18.04 system, with its root
filesystem configured on ZFS:

    $ lsb_release -a
    No LSB modules are available.
    Distributor ID: Ubuntu
    Description:    Ubuntu 18.04.1 LTS
    Release:        18.04
    Codename:       bionic

    $ mount | awk '$3 == "/" {print $0}'
    rpool/ROOT/delphix.9aTvgfL on / type zfs (rw,relatime,xattr,posixacl)

    $ zfs list
    NAME                         USED  AVAIL  REFER  MOUNTPOINT
    rpool                       21.2G  46.1G    64K  none
    rpool/ROOT                  14.7G  46.1G    65K  none
    rpool/ROOT/delphix.9aTvgfL  14.7G  46.1G  14.7G  /
    rpool/crashdump             65.5K  17.4G  65.5K  /var/crash
    rpool/home                  6.50G  46.1G  6.50G  /export/home
    rpool/update                  68K  15.0G    68K  /var/dlpx-update

## Step 2: Create container from clone of root filesytem

Once this is in place, it's relatively straight forward to create a
container from a root filesystem clone, using [systemd-nspawn][6].

1. Create the clone, mounted in `/var/lib/machines`:

        # mktemp -d -p /var/lib/machines -t delphix.XXXXXXX
        /var/lib/machines/delphix.DEepQDm

        # zfs snapshot rpool/ROOT/delphix.9aTvgfL@delphix.DEepQDm

        # zfs clone -o mountpoint=/var/lib/machines/delphix.DEepQDm \
        > rpool/ROOT/delphix.9aTvgfL@delphix.DEepQDm rpool/ROOT/delphix.DEepQDm

        # zfs list -o mountpoint,mounted,origin rpool/ROOT/delphix.DEepQDm
        MOUNTPOINT                         MOUNTED  ORIGIN
        /var/lib/machines/delphix.DEepQDm      yes  rpool/ROOT/delphix.9aTvgfL@delphix.DEepQDm

2. Install the `systemd-container` package, which contains `systemd-nspawn`:

        # apt-get install -y systemd-container

3. Override default container configuration:

        # mkdir -p /etc/systemd/nspawn
        # cat >/etc/systemd/nspawn/delphix.DEepQDm.nspawn <<EOF
        > [Exec]
        > PrivateUsers=no
        > [Files]
        > PrivateUsersChown=no
        > EOF

4. Start the container:

        # systemd-nspawn --capability=all --boot --machine delphix.DEepQDm

At this point, a container should be running on the system, using the
cloned root filesystem as it's root filesystem. Let's verify this:

    # machinectl list
    MACHINE         CLASS     SERVICE        OS     VERSION ADDRESSES
    delphix.DEepQDm container systemd-nspawn ubuntu 18.04   -

    # machinectl status delphix.DEepQDm | head -n10
    delphix.DEepQDm(5c5c6f1ae47a4ff29024aa845091ae8c)
               Since: Thu 2018-11-29 23:48:37 UTC; 38s ago
              Leader: 6597 (systemd)
             Service: systemd-nspawn; class container
                Root: /var/lib/machines/delphix.DEepQDm
                  OS: Ubuntu 18.04.1 LTS
                Unit: machine-delphix.DEepQDm.scope
                      ├─init.scope
                      │ └─6597 /lib/systemd/systemd
                      └─system.slice

We can even run commands in the container like so:

    # systemd-run --quiet --machine delphix.DEepQDm --pipe /usr/bin/env systemd-detect-virt
    systemd-nspawn

    # systemd-run --quiet --machine delphix.DEepQDm --pty /bin/bash
    (container)# exit
    exit

## Step 3: Upgrade the container

Now that we have a container running based on a clone of the host's root
filesystem, we can test out upgrade, without ever modifying the host's
root filesystem.

In this example, the host is running Ubuntu's "bionic" release, so the
container is also running that release. Next, we're going to upgrade
the container to Ubuntu's latest release, which is "cosmic".

1. Check the container's Ubuntu version before the upgrade:

        # systemd-run --quiet --machine delphix.DEepQDm --pipe /usr/bin/lsb_release -a
        No LSB modules are available.
        Distributor ID: Ubuntu
        Description:    Ubuntu 18.04.1 LTS
        Release:        18.04
        Codename:       bionic

2. Perform the upgrade of the container:

        # systemd-run --quiet --machine delphix.DEepQDm --pty /bin/bash
        (container)# sed -i 's/bionic/cosmic/g' /etc/apt/sources.list
        (container)# apt-get update
        (container)# apt-get dist-upgrade -y
        (container)# exit

3. Check the container's Ubuntu version after the upgrade:

        # systemd-run --quiet --machine delphix.DEepQDm --pipe /usr/bin/lsb_release -a
        No LSB modules are available.
        Distributor ID: Ubuntu
        Description:    Ubuntu 18.10
        Release:        18.10
        Codename:       cosmic

4. Verify the host's Ubuntu version remains the same:

        # lsb_release -a
        No LSB modules are available.
        Distributor ID: Ubuntu
        Description:    Ubuntu 18.04.1 LTS
        Release:        18.04
        Codename:       bionic

## Step 4: Use the container's (upgraded) root filesystem to boot the host

So far, we've created a container using a clone of the original root
filesystem, and have upgrade that container (and the associated cloned
filesystem) to the new Ubuntu version. The last step is to update the
host's bootloader configuration, such that it'll use this upgraded root
filesystem as it's root filesystem, the next time the host boots.

1. Stop the container (if it's still running):

        # machinectl poweroff delphix.DEepQDm

2. Bind mount directories required to update bootloader:

        # for dir in /proc /sys /dev; do
        > mount --rbind "$dir" "/var/lib/machines/delphix.DEepQDm$dir"
        > mount --make-rslave "/var/lib/machines/delphix.DEepQDm$dir"
        done

3. Update bootloader:

        # chroot /var/lib/machines/delphix.DEepQDm update-grub
        # chroot /var/lib/machines/delphix.DEepQDm grub-install /dev/sda

4. Remove mounts created in (2):

        # for dir in /proc /sys /dev; do
        > umount -R "/var/lib/machines/delphix.DEepQDm$dir"
        done

5. Unmount new root dataset, configure it, reboot:

        # zfs umount rpool/ROOT/delphix.DEepQDm
        # zfs set canmount=noauto rpool/ROOT/delphix.DEepQDm
        # zfs set mountpoint=/ rpool/ROOT/delphix.DEepQDm
        # reboot

After the system reboots, it'll be running the root filesystem that was
previously upgraded by the container; i.e. the host will be running the
Ubuntu "cosmic" release.

[1]: https://twitter.com/rvstaveren/status/1063057191513587712
[2]: https://twitter.com/rvstaveren/status/1063059487920218112
[3]: https://github.com/zfsonlinux/zfs
[4]: https://github.com/zfsonlinux/zfs/wiki/Ubuntu-18.04-Root-on-ZFS
[5]: https://github.com/delphix/appliance-build/blob/a3f5aa1/live-build/misc/live-build-hooks/90-raw-disk-image.binary
[6]: https://www.freedesktop.org/software/systemd/man/systemd-nspawn.html
