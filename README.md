This guide applies to
-------------
  - Raspberry Pi 4
  - Android 11 in Tablet mode

Establish build environment :
-------------
Ubuntu 18.04.5 LTS (Bionic Beaver) is the recommended build environment, more details read through the AOSP guide https://source.android.com/setup/build/initializing

Required build tools : 
  - `sudo apt-get install git-core gnupg flex bison build-essential zip curl zlib1g-dev gcc-multilib g++-multilib libc6-dev-i386 lib32ncurses5-dev x11proto-core-dev libx11-dev lib32z1-dev libgl1-mesa-dev libxml2-utils xsltproc unzip fontconfig kpartx python-mako`
  - `git config --global user.name "Your Name"`
  - `git config --global user.email "you@example.com"`

Installing Repo tool :
-------------
Repo tool is required to sync AOSP source code, more details https://source.android.com/setup/develop/repo
  - `cd HOME_DIR`
  - `mkdir ~/bin`
  - `PATH=~/bin:$PATH`
  - `curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo`
  - `chmod a+x ~/bin/repo`

Sync AOSP source code with Raspberry Pi 4 device tree :
-------------
Syncing Android 11 source code, more details https://github.com/android-rpi/local_manifests

  - `mkdir WORKING_DIRECTORY`
  - `cd WORKING_DIRECTORY`
  - `repo init -u https://android.googlesource.com/platform/manifest -b android-11.0.0_r5`
  - `git clone https://github.com/android-rpi/local_manifests .repo/local_manifests -b arpi-11`
  - `repo sync`
  
(Optional) if network error :
-------------
More rarely, Linux clients experience connectivity issues, getting stuck in the middle of downloads (typically during receiving objects). It's been reported that tweaking the settings of the TCP/IP stack and using non-parallel commands can improve the situation. You need root access to modify the TCP setting:
  - `sudo sysctl -w net.ipv4.tcp_window_scaling=0`
  - `repo sync -j1`

Patch framework source
-------------
Now that we have the source code synced, we need to modify a few things for it to properly work with a Raspberry Pi 4

  - Apply arpi-11 [patch](https://github.com/android-rpi/device_arpi_rpi4/wiki/arpi-11-:-framework-patch)
  - [Change](https://github.com/android-rpi/device_arpi_rpi4/blob/arpi-11/Generic.kl#L85) to modify keyboard layout to include power menu, edit ```key 63 F5``` to ```key 63 POWER``` at `/device/arpi/rpi4/Generic.kl`

Patch boot from USB instead of SdCard
-------------
Default boot mode is via SD card but for better performance if you want to boot from a USB drive(SSD/Pendrive 3.0 etc), change the boot mode to USB. Modify the file at /device/arpi/rpi4/fstab.rpi4
  - edit ```/dev/block/mmcblk0p2``` to ```/dev/block/sda2```
  - edit ```/dev/block/mmcblk0p3``` to ```/dev/block/sda3```
  - edit ```/dev/block/mmcblk0p4``` to ```/dev/block/sda4```

Build Raspberry Pi 4 Kernel
-------------
This build uses the kernel from [arpi-5.4.y](https://github.com/android-rpi/kernel_arpi/tree/arpi-5.4.y) branch

  - If not already installed, make sure to install kernel build tools ```sudo apt install gcc-aarch64-linux-gnu libssl-dev```
  - ```cd /kernel/arpi```
  - ```ARCH=arm64 scripts/kconfig/merge_config.sh arch/arm64/configs/bcm2711_defconfig kernel/configs/android-base.config kernel/configs/android-recommended.config```
  - ```ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make Image.gz```
  - ```ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- DTC_FLAGS="-@" make broadcom/bcm2711-rpi-4-b.dtb```
  - ```ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- DTC_FLAGS="-@" make overlays/vc4-kms-v3d-pi4.dtbo```
  
Build AOSP source
-------------
 Now that we have everything ready, let's build Android 10 image for Raspberry Pi 4. Usually it takes a couple of hours for the build to complete depending on the speed of your machine. For more details read AOSP build instructions http://source.android.com/source/building.html
 
  - `cd WORKING_DIRECTORY`
  - ```source build/envsetup.sh```
  - ```lunch rpi4-eng```
  - ```make ramdisk systemimage vendorimage```
  - (Or a single command : ```source build/envsetup.sh && lunch rpi4-eng && make ramdisk systemimage vendorimage```)

 If build host machine has a good number of CPU cores, use -j[n] option with make to increase the build speed, example use 4 cpu core:
  - ```make -j4 ramdisk systemimage vendorimage```
  
Build errors?
-------------
(Only if machine is 8GB RAM or less, allocate atleast 6GB to jvm else chances of running into build error 
  - If error is `Exception in thread "main" java.lang.OutOfMemoryError: Java heap space` use ```export _JAVA_OPTIONS="-Xmx8g"```
  - If error is `Picked up _JAVA_OPTIONS: -Xmx8g Killed` use `-j1`
 
Create Android 11 Image
-------------
  - ```cd device/arpi/rpi4```
  - ```sudo ./mkimg.sh```
  
Note: Depending on system.img size, change system partition size [mkimg.sh](https://github.com/lohriialo/device_arpi_rpi4/blob/arpi-11-tablet/mkimg.sh#L32)

Flash Image to SD Card or USB pendrive or SSD
-------------
  - balena Etcher https://www.balena.io/etcher/
  - Raspberry Pi Imager https://www.raspberrypi.org/downloads
