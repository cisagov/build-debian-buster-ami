# build-debian-buster-ami #

[![Build Status](https://travis-ci.com/cisagov/build-debian-buster-ami.svg?branch=develop)](https://travis-ci.com/cisagov/build-debian-buster-ami)

[Hashicorp Packer](https://www.packer.io/) and [Debian
Preseed](https://wiki.debian.org/DebianInstaller/Preseed)
configurations for building a Debian Buster AMI from scratch.

## Prerequisites ##

A Linux host machine with the following installed:

* [Hashicorp Packer](https://www.packer.io/)
* [QEMU](https://www.qemu.org/)
* [AWS CLI](https://aws.amazon.com/cli/)

## Build the AMI ##

I was able to ferret out the official Debian AMI build process from
[this mailing list
post](https://lists.debian.org/debian-cloud/2019/07/msg00060.html).
Unfortunately the instructions there are rather sparse, but I was able
to fill in the gaps from other sources as described below.

The mailing list post references the
[cloud-team/debian-cloud-images](https://salsa.debian.org/cloud-team/debian-cloud-images.git)
hosted on Debian's GitLab site.  Through painful trial and error, I
found that the code in that repo really only works under Debian
Buster.  It will not work with earlier Debian releases because it
requires a recent version of Python 3.  The code relies on
[FAI](https://fai-project.org/), and I did try using Ubuntu since it
has a recent version of Python 3 and supports FAI.  Unfortunately that
fails because the Debian version of `fcopy` (part of FAI) supports a
`-S` command line option while the Ubuntu version does not.

This complicates matters, since ideally we would:

1. Use a non-Debian or pre-Buster Debian AMI to spin up an instance in
   AWS.
2. Use the cloud-team/debian-cloud-images code to create an EBS volume
   containing the raw Debian Buster image.
3. Use the AWS CLI to register a new Debian Buster AMI from the EBS
   volume.

Since that isn't possible, and there is no Debian Buster AMI publicly
available in AWS, we have to instead:

1. Locally create a QEMU VM from the Debian Buster install ISO.
2. Spin up the QEMU VM.
3. Use the cloud-team/debian-cloud-images code to create a raw Debian
   Buster image from inside the QEMU VM.
4. Copy the Debian Buster image from the VM to our local host.
5. Spin up a Linux AMI in AWS and attach an EBS volume.
6. `scp` the Debian Buster image from the local host to the AWS EC2
   instance.
7. Use `dd` to copy the Debian Buster image to the EBS volume.
8. Use the AWS CLI to register a new Debian Buster AMI from the EBS
   volume.

### Create a Debian Buster QEMU VM ###

Note that kernel support for
[KVM](https://www.linux-kvm.org/page/Main_Page) and
[Virtio](https://www.linux-kvm.org/page/Virtio) is required.  You can
read [here](https://wiki.archlinux.org/index.php/KVM) about how to
check if your hardware and kernel support KVM and Virtio.  You must
`modprobe` the `kvm` and `virtio*` modules if they are not built into
your kernel or already loaded.

At that point, creating the QEMU VM is as easy as:

```console
cd build-debian-buster-ami
packer build debian_buster_qcow2_image.json
```

This command will create the file `output/debian_buster.qcow2`.

Note that the `debian-buster-ami-preseed.txt` was created by starting
from [this
file](https://www.debian.org/releases/stable/example-preseed.txt) and
changing a few values.

### Spin up the Debian Buster VM ###

```console
cd build-debian-buster-ami
qemu-system-x86_64 -enable-kvm -m 1024 output/debian_buster.qcow2
```

You can log into the VM using the user `debian` with the password
`insecure`.

### Build the raw Debian Buster image for the AMI ###

These steps are performed from inside the QEMU VM:

1. `git clone
   https://salsa.debian.org/cloud-team/debian-cloud-images.git`
2. `cd debian-cloud-images`
3. `sudo apt-get install --no-install-recommends ca-certificates
   debsums dosfstools fai-server fai-setup-storage make python3
   python3-libcloud python3-marshmallow qemu-utils udev logtail`
4. `make buster-image-ec2`

Note that the last two steps come from the file
`debian-cloud-images/README.md`.

### Copy the raw Debian Buster image to the local host ###

By default, QEMU makes the host machine available to the VM at the
address `10.0.2.2`.  Therefore we can copy the raw Debian buster image
to the local host like so:

```console
scp ec2-buster-image.tar username@10.0.2.2:~
```

### Copy the raw Debian Buster image to the AWS EC2 instance ###

At this point you must spin up an EC2 instance (use any flavor of
Linux you like) with a 16GB root disk and an extra 8GB EBS volume.
Save the PEM file associated with the keypair to the local host.  The
8GB EBS volume will become the volume associated with the AMI.

Once the instance is spun up, copy the raw Debian Buster
image from the local host and un-`tar` it:

1. `scp ec2-buster-image.tar admin@XX.XX.XX.XX:~` (on local host)
2. `tar xf ec2-buster-image.tar` (on the EC2 instance)

### Copy the raw Debian Buster image to the EBS drive ###

On the EC2 instance:

```console
sudo dd if=~/disk.raw of=/dev/xvdf bs=512k
```

You should change `/dev/xvdf` to whatever device corresponds to the
8GB EBS volume.

### Register a new AMI from the EBS volume ###

For this task we will need to utilize the
[noahm/ec2-image-builder](https://salsa.debian.org/noahm/ec2-image-builder.git)
repository hosted on Debian's GitLab.  These tasks should be performed
on the local host:

1. `git clone https://salsa.debian.org/noahm/ec2-image-builder.git`
2. `cd ec2-image-builder`
3. `./volume-to-ami.sh --release buster --region us-west-1 vol-04efd1e5de1ff6b7b`

You should replace the region and volume ID with the appropriate
values.  `volume-to-ami.sh` also supports a `--profile` command line
option, in cause you need to use an AWS CLI profile other than the
default.

## Contributing ##

We welcome contributions!  Please see [here](CONTRIBUTING.md) for
details.

## License ##

This project is in the worldwide [public domain](LICENSE).

This project is in the public domain within the United States, and
copyright and related rights in the work worldwide are waived through
the [CC0 1.0 Universal public domain
dedication](https://creativecommons.org/publicdomain/zero/1.0/).

All contributions to this project will be released under the CC0
dedication. By submitting a pull request, you are agreeing to comply
with this waiver of copyright interest.
