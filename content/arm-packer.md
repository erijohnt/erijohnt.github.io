---
title: "Arm Packer"
date: 2021-03-28T14:30:08-05:00
---

# Packer for ARM images

I have a raspberry pi that I want to use for various microservices on my local
network, but I want to use Fedora instead of the usual Debian-based
distributions more widely supported by the community. As it turns out, the base
Fedora Server image for ARM has a lot of old packages that need to be updated,
and I hate sitting around watching updates spin on my raspberry pi when I could
do `dnf upgrade -y` on a more powerful machine beforehand and bake it into the
image I load onto a microSD card.

While at it, I can install Docker and any other prerequisites that'll be common
across the multiple pis I have so that provisioning becomes a lot easier.
Hashicorp Packer seems like the tool best suited for this.

## Downloading and inspecting the image

Fedora has documentation for [running on raspberry
pis](https://docs.fedoraproject.org/en-US/quick-docs/raspberry-pi/), but it's
not all that different from your usual disk imaging with `dd`.

Grab the Fedora ARM Server image from https://arm.fedoraproject.org/ for
inspection. It's a raw image file compressed (but not tar'd!) with `xz`.

```bash
curl https://download.fedoraproject.org/pub/fedora/linux/releases/33/Server/armhfp/images/Fedora-Server-armhfp-33-1.2-sda.raw.xz
# verify the md5sum
md5sum Fedora-Server-armhfp-33-1.2-sda.raw.xz
# decompress
unxz Fedora-Server-armhfp-33-1.2-sda.raw.xz
```

We can inspect the partition scheme with the raw image, mount it, and see what
filesystems it uses. This will be important later on for using packer.

Mount the raw image using `losetup`:
```
# losetup -f -P Fedora-Server-armhfp-33-1.2-sda.raw

# losetup -a
/dev/loop0: []: (/home/ec2-user/packer-builder-arm/Fedora-Server-armhfp-33-1.2-sda.raw)
```

Inspect the mounted partitions with `ls`, `fdisk`, and `lsblk`:
```
# ls /dev/loop0*
/dev/loop0  /dev/loop0p1  /dev/loop0p2  /dev/loop0p3

# fdisk -l /dev/loop0
Disk /dev/loop0: 3 GiB, 3242196992 bytes, 6332416 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xd8e4594a

Device       Boot   Start     End Sectors  Size Id Type
/dev/loop0p1         8192  163839  155648   76M  c W95 FAT32 (LBA)
/dev/loop0p2 *     163840 1163263  999424  488M 83 Linux
/dev/loop0p3      1163264 6047743 4884480  2.3G 83 Linux

# mount /dev/loop0p1 /mnt
# lsblk -f
NAME      FSTYPE LABEL  UUID                                 MOUNTPOINT
loop0
├─loop0p1 vfat          1F83-A331                            /mnt
├─loop0p2 ext4   _/boot 176641cf-a6cb-4948-acbb-4893b077d97b
└─loop0p3 xfs    _/     e5743c2d-7315-48ab-8ffb-260a51dff891
xvda
└─xvda1   xfs    /      1392b4f8-f22a-4b73-a785-3b9fe56ce620 /

```

This gives us the information we'll need to supply in our packer template for
the partition scheme.

## Troubleshooting

### Bad networking

I was running into issues where `sudo dnf upgrade -y` failed because it
couldn't resolve DNS names. This is apparently a known issue with a workaround:
remove `/etc/resolv.conf` and replace it with a basic DNS server.

```bash
mv /etc/resolv.conf /etc/resolv.conf.bak
echo 'nameserver 8.8.8.8' > /etc/resolv.conf
```

## References

- https://www.reddit.com/r/linux4noobs/comments/91v0oh/cant_mount_raw_disk_image_but_can_read_files_with/
- https://www.b-ehlers.de/blog/posts/2017-10-26-inspect-modify-qemu-images/
