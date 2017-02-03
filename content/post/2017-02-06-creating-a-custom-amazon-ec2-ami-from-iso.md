+++
date = "2017-02-06T00:00:00-08:00"
lastmod = "2017-02-06T00:00:00-08:00"
title = "Creating a Custom Amazon EC2 AMI from ISO (using OI Hipster)"
draft = false
+++

## Preface

In this post, I'll pick up from where I left off
[last time][install-media], and demonstrate one potential way to convert
the installation ISO media generated in that post, into an [AMI][ami]
that can be used to create new VMs in the [Amazon EC2][ec2] environment.
It's important to note a couple things before we start:

  1. While I'll be generating an AMI based on [OI Hipster][oi-hipster],
     this process should be applicable to any Linux or FreeBSD based
     operating system as well (and quite possibly Windows too, but I
     don't know much about that platform).

  2. I make no gaurantees about whether the process that'll be
     demonstrated is correct, efficient, or complete. I have little
     to no expertise in this area, and this is simply a write up of the
     notes that I took as I was teaching myself how to do this. Just
     because the process below worked for me, does not necessarily mean
     it will work for anybody else. Thus, if you're using this as a
     guide to create your own custom AMI(s), be aware, YMMV.

With that out of the way, lets get started.

## Step 0: Create Template for Root Volume of AMI

The first step that we need to perform, is to create a disk image
that'll be used as the template for the root volume of the AMI that
we'll be creating. If you already have a `raw` VM disk image that you
intend to use as the AMI root volume template, than this step can be
skipped. For the sake of completeness, I'm including one possible way to
generate this `raw` disk image, that'll then be used in later steps.

In order to generate this `raw` disk image, we'll create a VM that'll be
used to execute the ISO installer, which will install our operating
system (OI Hipster, in my case) onto a "disk", that we can then upload
to Amazon and use as our AMI's root volume template.

For this guide, I'll be using a [Debian][debian] server that's
configured as a [Xen host][debian-xen] to create the VM and disk image.
First, lets create a sparse file that'll be attached as the VM's only
virtual disk; after the ISO installation completes, this file will be
our `raw` disk image that'll be converted into the AMI's root volume.

    $ truncate -s 64G ami-template.img

Now, to create the VM, I need to specify the hardware configuration to
use. I do this using the following Xen VM configuration:

    $ cat <<EOF > ami-template.cfg
    > builder='hvm'
    > name='ami-template'
    > vcpus=4
    > memory=4096
    > vif=['bridge=xenbr0, type=ioemu']
    > disk=[  'file:/root/OpenIndiana_Text_X86.iso,hdb:cdrom,r',
    >         'file:/root/ami-template.img,xvda,w' ]
    > boot='d'
    > vnc=1
    > vnclisten='0.0.0.0'
    > vncconsole=1
    > on_crash='preserve'
    > xen_platform_pci=1
    > serial='pty'
    > on_reboot='destroy'
    > EOF

For those unfamilar with Xen's configuration files, this states that the
VM will be given:

  - 4 CPUs
  - 4G of RAM
  - The installation ISO will be attached as a CDROM
  - The sparse file generated above will be used as the VM's only disk
  - The console will be displayed over [VNC][vnc]
  - etc...

With that configuration file generated, we can go ahead and create/start
the VM with the following command:

    $ xm create ami-template.cfg

With the VM running and it's console being displayed over VNC, I can
then connect to the console and manually walk through the installer's
instructions. After the installer completes, we'll have a disk image
(the `ami-template.img` file) that'll be uploaded to Amazon and used to
generate the AMI in later steps.

## Step 1: Convert Raw Disk Image into Stream Optimized VMDK Format

Now that we've run the OI Hipster installer to completion, we should
have a `ami-template.img` file that contains a complete installation of
our operating system. While we *could* upload directly to Amazon as-is,
we won't do this as it'd be an inefficient use of space (and time).

Remember that when we generated the file that we attached to the VM in
the [previous step][step-0], we used a sparse file; we can use `ls` to
see how much space was saved by doing so (as opposed to a non-sparse
file):

    $ ls -lsh ami-template.img
    3.6G -rw-r--r-- 1 root root 64G Feb  5 22:10 ami-template.img

From this output, we can see that while the file is 64G in size, it's
only taking up 3.6G of space. Thus, it'd be great if we could upload
only 3.6G (or less) worth of data to Amazon, rather than the full 64G
(most of which would be zeros).

That's what this step is about; rather than uploading the
`ami-template.img` file as-is, which would result in 64G of data being
transferred over the network, we'll convert the file into a "stream
optimized" VMDK and upload this converted file. As a result, we'll only
have to send a small fraction (of the original file's size) of data over
the network, when compared to if we hadn't done this conversion; which
is a huge benefit in terms of the latency required to perform the
upload.

We'll use the `VMDKStream.py` utility to do the conversion, so first we
need to download this tool:

    $ wget -O VMDK-stream-converter-0.2.tar.gz \
        https://github.com/imcleod/VMDK-stream-converter/archive/0.2.tar.gz

    $ tar -xf VMDK-stream-converter-0.2.tar.gz

    $ ls VMDK-stream-converter-0.2
    COPYING  README  setup.py  VMDKstream.py
    root@kvmserver1:~/psurya# ls -l VMDK-stream-converter-0.2
    total 40
    -rw-rw-r-- 1 root root 18092 Jun 29  2011 COPYING
    -rw-rw-r-- 1 root root   576 Jun 29  2011 README
    -rw-rw-r-- 1 root root   120 Jun 29  2011 setup.py
    -rwxrwxr-x 1 root root 11454 Jun 29  2011 VMDKstream.py

And now we can use it to read the original `raw` disk image, and convert
it into the stream optimized `vmdk` that we want:

    $ ./VMDK-stream-converter-0.2/VMDKstream.py ami-template.img ami-template.vmdk

After this completes, we can use `ls` again to see just how much less
data will be needed to transfer the disk image using this new format:

    $ ls -lsh ami-template.img ami-template.vmdk
     3.6G -rw-r--r-- 1 root root   64G Feb  5 22:10 ami-template.img
    1022M -rw-r--r-- 1 root root 1022M Feb  5 22:35 ami-template.vmdk

As one can see, the converted file is **much** smaller than even the
sparse file that we started with; rather than sending 64G over the
network, we'll only have to send about 1G.

Now with our stream optimized VMDK file generated, we can move on to
actually performing the upload of this file to Amazon.

## Step 2: Download EC2 Tools and Initialize Environment

To perform the upload of the VMDK file that we had just generated, as
well as the various other steps involved in creating the AMI, we'll use
the EC2 API Tools provided by Amazon. Thus, the first thing we need to
do, is to download these tools (assuming they aren't already installed).

### Step 2.1: Download EC2 API Tools

This is simple, using `wget` and `unzip`:

    $ wget http://s3.amazonaws.com/ec2-downloads/ec2-api-tools.zip
    $ unzip ec2-api-tools.zip

The tools will be extracted into a directory named `ec2-api-tools`, and
postfixed with the version of the tools downloaded. In this example,
version `1.7.5.1` was downloaded, so the directory containing the tools
will be `ec2-api-tools-1.7.5.1`, and we can verify this using `ls`:

    $ ls -l ec2-api-tools-1.7.5.1/
    total 100
    drwxr-xr-x 2 root root 36864 Sep  7  2015 bin
    drwxr-xr-x 2 root root  4096 Sep  7  2015 lib
    -rw-r--r-- 1 root root  4852 Sep  7  2015 license.txt
    -rw-r--r-- 1 root root   539 Sep  7  2015 notice.txt
    -rw-r--r-- 1 root root 46468 Sep  7  2015 THIRDPARTYLICENSE.TXT

### Step 2.2: Initialize Necessary Environment Variables

In conjunction with the EC2 API Tools, we'll also make use of the
following environment variables to configure the invocation of the
tools. The following is a list of the environment variables that we'll
use in the following sections, as well as a brief description of each:

  - `AWS_ACCESS_KEY`: This is required so the tools can authenticate as
    "you" when making API request to Amazon. This needs to be set based
    on each individual's specific key. For more information, see
    [here][access-keys].

  - `AWS_SECRET_KEY`: In addition to the "access key" described above,
    authenticating with the Amazon APIs also requires a "secret key".
    Exactly like the "access key", this must be set based on the
    individual's specific key. See [here][access-keys] for more
    information.

  - `AWS_ZONE`: This will specify the AWS availability zone that will be
    used when generating the AMI. For more information about AWS
    availability zones, see [here][regions-zones].

  - `AWS_S3_BUCKET`: This will specify the AWS S3 bucket used when
    uploading the disk image. The disk image will be uploaded to this
    bucket, and then a snapshot of the generated volume will be taken,
    and this snapshot used to create the root volume of the AMI. For
    more information about Amazon S3, see [here][s3].

  - `AWS_AMI_NAME`: This will specify the name given to the AMI.

  - `AWS_AMI_DESC`: This will specify the description given to the AMI.

The following is intended to serve as an example of the values that
could be used for these various environment variables:

    $ export AWS_ACCESS_KEY=AKIAIOSFODNN7EXAMPLE
    $ export AWS_SECRET_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
    $ export AWS_ZONE=us-west-1a
    $ export AWS_S3_BUCKET=ivol-openindiana-hipster
    $ export AWS_AMI_NAME="OpenIndiana Hipster Testing 2017.02.06 (HVM)"
    $ export AWS_AMI_DESC="Hipster is a codename for rapidly moving \
    > development branch of OpenIndiana and the development branch from \
    > which major releases are made."

## Step 3: Import the Stream Optimized VMDK to Amazon S3

Now that we have the EC2 API Tools installed, and the necessary
environment variables initialized, we can start the process of uploading
our disk image to Amazon, and converting that into an AMI.

The first step of this proces is to upload our stream optimized VMDK
(generated in [step 1][step-1]) to our Amazon S3 bucket; to do this,
we'll use the `ec2ivol` utility:

    $ ./ec2-api-tools-1.7.5.1/bin/ec2ivol -o $AWS_ACCESS_KEY \
    > -w $AWS_SECRET_KEY -b $AWS_S3_BUCKET -z $AWS_ZONE \
    > -d "$AWS_AMI_DESCRIPTION" -f vmdk ami-template.vmdk
    Requesting volume size: 64 GB
    TaskType        IMPORTVOLUME    TaskId  import-vol-fghpxu3l     ExpirationTime  2017-02-13T20:39:29Z    Status  active  StatusMessage   Pending
    DISKIMAGE       DiskImageFormat VMDK    DiskImageSize   1070948352      VolumeSize      64      AvailabilityZone        us-west-1a      ApproximateBytesConverted       0
    Creating new manifest at ivol-openindiana-hipster/0af50aa3-ee80-492d-96a5-bab6ab63bda4/ami-template.vmdkmanifest.xml
    Uploading the manifest file
    Uploading 1070948352 bytes across 103 parts
    ----------------------------------------------------------------------------------------------------
       Upload progress              Estimated time      Estimated speed
     / 100% [====================>]                     56.056 MBps
    ********************* All 1070948352 Bytes uploaded in 19s  *********************
    Done uploading.
    Average speed was 56.053 MBps
    The disk image for import-vol-fghpxu3l has been uploaded to Amazon S3
    where it is being converted into an EBS volume.  You may monitor the
    progress of this task by running ec2-describe-conversion-tasks.  When
    the task is completed, you may use ec2-delete-disk-image to remove the
    image from S3.

We can then use the `TaskId` value provided by the output of `ec2ivol`
(i.e. `import-vol-fghpxu3l` in this instance), and feed that into
`ec2dct` to retreive the status for the upload's conversion.

    $ ./ec2-api-tools-1.7.5.1/bin/ec2dct import-vol-fghpxu3l
    TaskType        IMPORTVOLUME    TaskId  import-vol-fghpxu3l     ExpirationTime  2017-02-13T20:39:29Z    Status  completed
    DISKIMAGE       DiskImageFormat VMDK    DiskImageSize   1070948352      VolumeId        vol-041fd3aa8c9435e7d   VolumeSize      64      AvailabilityZone        us-west-1a      ApproximateBytesConverted       1070945104

As can be seen from the `Status` section, this conversion has `completed`,
so we can move on to the next step. If the conversion hadn't `completed`
yet, e.g. it was still `active`, then we would need to wait (and poll
using `ec2dct`) until it `completed`.

## Step 4: Create a Snapshot of Imported Volume

The next step that we need to perform, is to create a snapshot from the
volume that we just created. Looking at the prior output from `ec2dct`,
we can see from the `VolumeId` field, that a volume was generated and
it's ID is `vol-041fd3aa8c9435e7d`. We will use the `ec2addsnap` utility
to create a snapshot of that volume:

    $ ./ec2-api-tools-1.7.5.1/bin/ec2addsnap vol-041fd3aa8c9435e7d
    SNAPSHOT        snap-01b80a642ef5f1f51  vol-041fd3aa8c9435e7d   pending 2017-02-06T20:49:11+0000                239688353806    64              Not Encrypted

Just like when importing the volume, we can check the status of the
snapshot generation process using `ec2dsnap` (the parameter passed to it
comes from the the output of `ec2addsnap` above):

    $ while true; do
    > ./ec2-api-tools-1.7.5.1/bin/ec2dsnap snap-01b80a642ef5f1f51
    > sleep 60
    > done
    SNAPSHOT        snap-01b80a642ef5f1f51  vol-041fd3aa8c9435e7d   pending 2017-02-06T20:49:11+0000        0%      239688353806    64              Not Encrypted
    SNAPSHOT        snap-01b80a642ef5f1f51  vol-041fd3aa8c9435e7d   pending 2017-02-06T20:49:11+0000        0%      239688353806    64              Not Encrypted
    SNAPSHOT        snap-01b80a642ef5f1f51  vol-041fd3aa8c9435e7d   completed       2017-02-06T20:49:11+0000        100%    239688353806    64              Not Encrypted
    ^C

## Step 5: Register AMI Using Previously Created Volume Snapshot

Finally, now that we have the volume snapshot generated, we can use this
to register our new custom AMI using `ec2reg`:

    $ ./ec2-api-tools-1.7.5.1/bin/ec2reg -s snap-01b80a642ef5f1f51 \
    > -d "$AWS_AMI_DESC" -n "$AWS_AMI_NAME" -a x86_64 \
    > --root-device-name /dev/xvda --virtualization-type hvm
    IMAGE   ami-5b065a3b

Success! The new custom built AMI is `ami-5b065a3b`.

It should now be possible to use this AMI to create new Amazon EC2
instances.

[install-media]: {{< relref "2017-02-01-creating-custom-istallation-media-for-oi-hipster.md" >}}
[ami]: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html
[ec2]: https://aws.amazon.com/documentation/ec2/
[oi-hipster]: https://wiki.openindiana.org/oi/Hipster
[debian]: https://www.debian.org/
[debian-xen]: https://wiki.debian.org/Xen
[vnc]: https://en.wikipedia.org/wiki/Virtual_Network_Computing
[access-keys]: https://aws.amazon.com/developers/access-keys/
[regions-zones]: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html
[s3]: https://aws.amazon.com/documentation/s3/
[step-0]: #step-0-create-template-for-root-volume-of-ami
[step-1]: #step-1-convert-raw-disk-image-into-stream-optimized-vmdk-format
