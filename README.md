Creating ZFS disks under Linux
==============================

This guide explains how to create a new, single disk ZFS pool under Linux. It does not cover more advanced ZFS configurations such as creating a mirror or a RAIDZ pool. 

Note that you cannot take advantage of some of ZFS' best features when using a single disk ZFS pool, such as self-healing when performing zpool scrubs. You can however still use features such as snapshots so it is still worthwhile using ZFS even with single disk configurations as is often the case with removable and external drives.

Install ZFS and gdisk
---------------------

This depends upon your Linux distro. Under Ubuntu you would run:

```
$ sudo apt install zfs-dkms gdisk
```

To install the required software.

Identify the disks device name
------------------------------

We need to know the device name of the disk we want to create our ZFS pool on so to do that we'll use the `lsblk` command. Attach the drive you wish to format then run `lsblk` in the terminal. The device name will usually be something like `sdb' or 'sdc'. You should be able to identify the correct device by its size. If you have multiple drives of the same size attached, compare the output of `lsblk` with and without your drive attached.

Create a solaris (ZFS) partition
--------------------------------

Lets assume `lsblk` showed that **sdb** is the device I want to create my pool on. We will first remove all existing partitions on **sdb** so be VERY careful when running this next command because if you get the device name wrong you could end up formatting your system drive or any other attached drive and potentially lose ALL data on the drive so make **ABSOLUTELY** sure the device name is correct before running theses commmands! It is recommend you detach all of the drives that you do not need to have attached to your system before running this:

```
$ sudo sgdisk --zap-all /dev/sdb
```

You must replace `/dev/sdb` with `/dev/sdc` or whichever device `lsblk` tells you the disk is. 

We will now create a Solaris ZFS partition on the same drive:

```
$ sudo sgdisk --new=1:0:0 --typecode=1:BF00 /dev/sdb
```

Create a new ZFS pool
---------------------

There are two ways you can create your pool when creating it under Linux. If you use the `zpool create` defaults, it will create a pool that can only be imported with read and write permissions on another sufficiently recent ZFS on Linux machine. If you want you pool to be readable and writable under FreeBSD, NetBSD, Solaris and Windows as well as Linux then a few extra ZFS options must be disabled when creating the pool.

If you are certain you will only ever want to import your pool (mount your ZFS disk) on Linux machines then you can run:

$ sudo zpool create -f -d -m none -o ashift=12 -o feature@lz4_compress=enabled backup01 /dev/sdb

Replace **backup01** with the name of your ZFS pool and **/dev/sdb** with the correct device. The pool name is the name you will use with `zpool import` to import (mount) the ZFS pool/disk.

If you don't mind losing some ZFS-on-Linux specific features and would prefer to create a multi-platform friendly ZFS pool instead, run:

```
$ sudo zpool create -f -d -m none -o ashift=12 -o feature@lz4_compress=enabled \
-o feature@multi_vdev_crash_dump=disabled -o feature@large_dnode=disabled \
-o feature@sha512=disabled -o feature@skein=disabled -o feature@edonr=disabled \
-o feature@userobj_accounting=disabled backup01 /dev/sdb
```

Again, replace **backup01** with the name of your ZFS pool and **/dev/sdb** with the device with the empty Solaris partition to use for the pool.

Create a dataset
----------------

`zfs snapshot` operates on datasets so if you want to use ZFS snapshots you must create at least one dataset and set a couple of options. Here we call this dataset **main**:

```
$ sudo zfs create backup01/main
$ sudo zfs set compression=lz4 backup01/main
$ sudo zfs set atime=off backup01/main
$ sudo zfs set mountpoint=/backup01 backup01
```

Replace **backup01** with the name of your pool and **main** with the name of the dataset you want to create, which you can name almost anything but its best to pick something that reflects its intended content. You can add as many datasets as you want now or at a later date. Enabling lz4 compression and disabling updating of file access times (atime) both help improve performance. I have opted to mount this pool under /backup01 to match its pool name but you can set the pool to be mounted where you wish.

You may now check your datasets settings:

```
$ zfs get all backup01/main
```

Exporting the pool
------------------

The ZFS pool has been created and configured along with a dataset. We can now finalise our changes to the pool by exporting it:

```
$ sudo zpool export backup01
```

Fixing boot delays caused by unimportable pools
-----------------------------------------------

If you are using multiple ZFS pools with some on removable disks that aren't always connected to your machine then you may experience a delay during boot if you have updated your kernel when an external pool is imported that isn't always attached because it gets added to the initramfs zpool cache when a kernel update/install updates it. The fix is to clear your zpool cache and update your initramfs as [described for Arch here.](https://wiki.archlinux.org/index.php/ZFS#Fix_slow_boot_caused_by_failed_import_of_unavailable_pools_in_the_initramfs_zpool.cache).

