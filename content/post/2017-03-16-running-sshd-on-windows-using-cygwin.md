+++
date = "2017-03-16T00:00:00-08:00"
lastmod = "2017-03-27T00:00:00-08:00"
title = "Running `sshd` on Windows using Cygwin"
draft = false
+++

## Introduction

As part of our effort to support [Delphix][1] in the [Azure][2] cloud
environment, we're writing some automation to convert our `.iso` install
media into a [VHD][3] image, leveraging [Jenkins][4] and [Packer][5] in
the process.

Essentially, we want to use a [Windows][6] server as a Jenkins "slave",
and run Packer from within a Jenkins job that will run on that Windows
system.

In order to do that, the Jenkins "master" needs to connect with the
Windows system, such that it can configure the system to act as a
Jenkins slave. Additionally, due to some of our existing Jenkins
infrastructure, this requires the Jenkins "master" using `ssh` to
connect to the Windows box. Thus, `sshd` is needed on the Windows
system.

## Running `sshd` on Windows using Cygwin

The following is a quick run-down of the steps I needed to perform, to
install and configure `sshd` on our Windows server, enabling `ssh`
clients (and our Jenkins master) to connect to it:

  1. Install Cygwin on the Windows server:
    - I used RDP to connect to the server as the `Administrator` user
    - Used a web browser to access the [installation instructions][7]
    - Downloaded the `setup-x86_64.exe` file
    - Executed `setup-x86_64.exe`, and followed the setup instructions
    - When reaching the package selection screen of the setup wizard, I
      explicitly selected the following packages:
      - `cygrunsrv` from the `Admin` group
      - `openssh` from the `Net` group
  2. After the Cygwin setup finished, I used the instructions found
     [here][8] to configure and start the `sshd` service. This involved
     running the following commands in a Cygwin shell:
    - `ssh-host-config -y`
    - `cygrunsrv -S sshd`
  3. At this point, the `sshd` service was running, but the Windows
     server's firewall was blocking incoming connections to port 22. So,
     I had to edit the server's firewall rules to allow incomming
     connections on port 22 (the port `sshd` was running on). For this,
     I followed the instructions that I found [here][9].

## Installing Additional Software

In addition to installing Cygwin and running `sshd`, we also needed to
perform a little more configuration to the Windows system, as our
Jenkins job had these additional dependencies:

  - To allow ssh access to other systems in our environment from the
    Jenkins job, the necessary `ssh` keys had to be installed in the
    `Administrator` user's home directory.
  - `git` was installed using the Cygwin installer.
  - `packer` was downloaded from the project's [download page][10] and
    installed simply by placing the binary at the correct location.
  - `pv` was manually compiled and installed following the directions on
    the [project's website][11]; i.e. using these commands:
    - `wget http://www.ivarch.com/programs/sources/pv-1.3.4.tar.bz2`
    - `tar -xf pv-1.3.4.tar.bz2`
    - `cd pv-1.3.4`
    - `./configure --prefix=`
    - `make`
    - `make install`

[1]: https://www.delphix.com/
[2]: https://en.wikipedia.org/wiki/Microsoft_Azure
[3]: https://en.wikipedia.org/wiki/VHD_(file_format)
[4]: https://en.wikipedia.org/wiki/Jenkins_(software)
[5]: https://en.wikipedia.org/wiki/Packer_(software)
[6]: https://en.wikipedia.org/wiki/Microsoft_Windows
[7]: https://cygwin.com/install.html
[8]: http://www.noah.org/ssh/cygwin-sshd.html
[9]: https://techtorials.me/cygwin/configure-windows-firewall/
[10]: https://www.packer.io/downloads.html
[11]: https://sourceforge.net/projects/pvforcygwin/
