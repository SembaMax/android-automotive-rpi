# Android Automotive RPI

## Hardware

* Raspberry Pi 4B - with minimum 4GB RAM
* HDMI IPS Touchscreen
* Minimum 32 GB SD card


## Build machine
You need a reasonably powerful machine because AOSP build process will take several hours.

* Minimum 16GB RAM - recommended 32 GB
* Recommended at least 8 cores CPU
* Minimum 512 GB disk space
* Debian 11 OS

## Install prerequisites
Installing the packages needed for the process

```
$ sudo apt install gcc-aarch64-linux-gnu git-core python-is-python3 fdisk curl libssl-dev flex build-essential bison rsync meson
```

And you will need also to install Repo tool manually

```
$ mkdir -p ~/.bin
$ PATH="${HOME}/.bin:${PATH}"
$ curl https://storage.googleapis.com/git-repo-downloads/repo > ~/.bin/repo
$ chmod a+rx ~/.bin/repo
```

## Download Android source

Refer to http://source.android.com/source/downloading.html

First you need to init the Android source, here we are using Android 12L according to [AOSP AVD for Automotive](https://source.android.com/docs/devices/automotive/start/avd/android_virtual_device)
```
$ repo init -u https://android.googlesource.com/platform/manifest -b android-12L
```

Then checkout the local manifest with the required changes for Automotive OS
```
$ git clone https://github.com/SembaMax/local_manifests .repo/local_manifests -b arpi-12L
```

At last you should run `repo sync` to kick off the download process.


## Patch
Once `repo sync` is done, there's set of manual modifications you will need to do before proceeding to the build phase.

Please follow the instructions [here](https://github.com/android-rpi/device_arpi_rpi4/wiki/arpi-12-:-framework-patch)


## Build AOSP
Refer to http://source.android.com/source/building.html

Now after you had a successful Android source download, you are good to go for the build step.

The build step consists two main build processes

### Build Android source
Here you come to generate the Android automotive OS image

```
  $ source build/envsetup.sh
  $ lunch automotive_rpi4-eng
  $ make ramdisk systemimage vendorimage
```

Use -j[n] option with make in order to accelerate the build process, if your build machine has a good number of CPU cores.

If you are using build machine with RAM less than 20GB, you may encounter to `OUT_OF_MEMORY` error.

As a workaround for this problem without upgrading your hardware you can use `swap`

### Build linux kernel
For Android 11, the kernel directory is under the Android source repo.

you can run the following commands to build the kernel:

```
$ cd kernel/arpi
$ ARCH=arm64 scripts/kconfig/merge_config.sh arch/arm64/configs/bcm2711_defconfig kernel/configs/android-base.config kernel/configs/android-recommended.config
$ ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make Image.gz
$ ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- DTC_FLAGS=”-@” make broadcom/bcm2711-rpi-4-b.dtb
$ ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- DTC_FLAGS=”-@” make overlays/vc4-kms-v3d-pi4.dtbo
```

Otherwise, please follow the the next instructions.

Kernel build process is a little bit different starting from Android 12, you will need to download and build the linux kernel separately from the Android source.

#### Download kernel
Make separate kernel directory apart from Android source.

```
  $ cd <kernel directory>
  $ repo init -u https://github.com/android-rpi/kernel_manifest -b arpi-5.10
  $ repo sync
```

#### Build kernel
```
  $ build/build.sh
```

Output files will be under $OUT/arpi-5.10/dist/

## Prepare sd card
Now as the last step you need to prepare your SD card for deploying the AOSP image.

Partitions of the SD card should be set exactly like followings

| #  | Size      | Partition | Options                            |
| -- | --------- | --------- | ---------------------------------- |
| p1 | 128MB     | boot      | set W95 FAT32(LBA) & bootable type |
| p2 | 2048MB    | /system   |                                    |
| p3 | 128MB     | /vendor   |                                    |
| p4 | remaining | /data     |                                    |

Create file system on the following two partitions (p1, p4)
```
$ mkfs.vfat /dev/sdX1
$ mkfs.ext4 /dev/sdX4
```

images will be written on the other two partitions (p2, p3), so no need to create file systems on them.

## SD card deployment
Most likely the SD card will appear as /dev/sdX on your build machine.

For the following steps you just need to replace X with what appears on your machine.


### Write system & vendor partition
```
# dd if=$OUT/target/product/rpi/system.img of=/dev/sdX2 bs=1M status=progress
# dd if=$OUT/target/product/rpi/vendor.img of=/dev/sdX3 bs=1M status=progress
```

### Copy firmware & ramdisk to boot partition
```
$ mount /dev/sdX1 /mnt
$ cp device/arpi/rpi4/boot/* /mnt
$ cp $OUT/target/product/rpi4/ramdisk.img /mnt
```

### Copy kernel binaries to boot partition
```
$ mkdir /mnt/overlays
$ cp <kernel directory>/out/arpi-5.10/dist/Image.gz /mnt
$ cp <kernel directory>/out/arpi-5.10/dist/bcm2711-rpi-*.dtb /mnt
$ cp <kernel directory>/out/arpi-5.10/dist/vc4-kms-v3d-pi4.dtbo /mnt/overlays
```

And finally remove the SD card
```
umount /mnt
eject /dev/sdX
```

## Booting the Raspberry PI
Congratulations! you have made it that far.

Now all you need to insert the SD card to the raspberry pi and boot it up.
