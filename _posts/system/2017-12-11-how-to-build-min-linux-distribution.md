---
layout: post
title: How to build minimum Linux distribution ?
category: 系统
tags: linux
keywords: linux
description:
---

### Not verified yet

1. download Linux kernel and busybox source code.
2. extract source code at linux and busybox
3. install ncurses and bc
4. cd to linux folder and `make menuconfig` to have a gui to customize kernel configuration; then `make tar-pkg` to build kernel and package in a tarball.
5. cd to busybox folder and `make menuconfig`; check build busybox as a static binary; then `make`; copy out the executable binary file of busybox.
6. `dd if=/dev/zero of=disk.img count=1 bs=512M` to create 512MB disk; format the disk with `mkfs.ext4`; use `losetup` to bind the disk.img to a loop device such as /dev/loop0
7. `mkdir` a new folder and mount as /dev/loop0; extract built linux kernal to the folder; in the folder also do `mkdir dev proc sys bin sbin sys var etc dev`
8. copy busybox binary to bin folder; in sbin folder, do `ln` to create a soft link of `init` (→ busybox); in bin folder, create soft link of `sh`, `bash` (→ busybox)
9. run `grub2-install --root-directory=<mount_folder> --no-floppy --force /dev/loop0` to set up bootable section
10. convert disk.img to disk.vmdk (for example with `qemu-img`)
11. creaet a virtual machine to run the min linux
12. in grub boot screen: input `linux /boot/vmlinuz-<kernel_version> root=/dev/sda rw`; then type `boot` to get min linux run
13. use `busybox ifconfig` to config network