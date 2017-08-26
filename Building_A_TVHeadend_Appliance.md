Building a custom tvheadend system

This document serves as a rough guide on how to build a custom tvheadend system image.

Custom parts are:
* 3rd-party DVB drivers and subsystem
* 3rd-party Wireless drivers
* Customised tvheadend package

Finally, it is possible to put this all back on an image, prior to it being written to SD card, so it can be used to install new systems.

It is recommended that a fast disk and compilation host system be used for the process. A good system is a Pi3 and a USB SATA disk.

1. Create a compilation host on a Pi 3
A Raspberry Pi 3 has 4 cores and can build a full kernel in 1hr 30mins if the configuration is cut down. It is ideal for cross-compiling for Pi Zeros and 1st-gens, as the toolchain is very similar.

Download the Raspbian Lite image.

Write it to an SD card:
```
# dd if=2017-08-16-raspbian-stretch-lite.img of=/dev/disk1 bs=64k
```

Load up the ‘boot’ directory and put a file called ‘ssh’ on it.

Boot up the image and enable wifi to your local network:
```
# vi /etc/wpa_supplicant/wpa_supplicant.conf
…
network={
    ssid=“my_network”
    psk=“MyPassword”
}
```

Restart the networking or reboot:
```
# service networking restart
```

Check the kernel compilation details:
```
# cat /proc/version
Linux version 4.9.41+ (dc4@dc4-XPS13-9333) (gcc version 4.9.3 (crosstool-NG crosstool-ng-1.22.0-88-g8460611) ) #1023 Tue Aug 8 15:47:12 BST 2017
```

Download the matching GCC and other build utils:
```
# apt-get install gcc-4.9 g++-4.9 bc libncurses5-dev screen
```

Set the compiler as the default:
```
# update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-6 100
# update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-4.9 50
# update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-6 100
# update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-4.9 50
# update-alternatives --install /usr/bin/cpp cpp-bin /usr/bin/cpp-6 100
# update-alternatives --install /usr/bin/cpp cpp-bin /usr/bin/cpp-4.9 50
# update-alternatives --set g++ /usr/bin/g++-4.9
# update-alternatives --set gcc /usr/bin/gcc-4.9
# update-alternatives --set cpp-bin /usr/bin/cpp-4.9
```

2. Compiling kernel modules with patches

Download rpi-source:
```
# sudo wget https://raw.githubusercontent.com/notro/rpi-source/master/rpi-source -O /usr/bin/rpi-source 
# sudo chmod +x /usr/bin/rpi-source
# /usr/bin/rpi-source -q --tag-update
```

Attach the fast external disk:
```
# mount /dev/sda1 /mnt
```

Put the correct source to the fast disk:
```
# cd /mnt
# rpi-source
# cd linux
```

Apply any patches that are required:
```
# patch -p1 < /home/pi/dvb-backport-cc69-20170819.diff
# patch -p1 < /home/pi/dvb-backport-fixes-20170819.diff
```

Before compilation starts, it might be worth running the process in screen:
```
# screen
```

Put in place a default kernel configuration for a Pi 1st Gen:
```
# KERNEL=kernel
# make bcmrpi_defconfig
# make prepare
```

A Pi 2nd Gen:
```
# KERNEL=kernel7
# make bcm2709_defconfig
# make prepare
```

Check it is correct - 6=1st Gen, 7=2nd Gen and above:
```
# grep "ARCH_MULTI_V.=" .config
```

Add or remove necessary components (enable DVB card, then maybe disabling sound, USB ethernet, etc):
```
# make menuconfig
# make prepare
```

It is possible to compile a new kernel, but this is how to just do the modules:
```
# make -j4 ARCH=arm modules
```

Exit screen if it was used:
```
# exit
```

Install the modules to the fast disk:
```
# make INSTALL_MOD_PATH=/mnt modules_install
```

Archive the ones needed (this is for a TBS5890 / QBOX CI 2):
```
# cd /mnt
# tar cfvz dvb-modules.tar.gz lib/modules/4.9.41+/kernel/drivers/media/dvb-core/dvb-core.ko \
 lib/modules/4.9.41+/kernel/drivers/media/rc/rc-core.ko \
 lib/modules/4.9.41+/kernel/drivers/media/dvb-frontends/stv090x.ko \
 lib/modules/4.9.41+/kernel/drivers/media/dvb-frontends/stb6100.ko \
 lib/modules/4.9.41+/kernel/drivers/media/usb/dvb-usb/dvb-usb-tbsqbox2ci.ko \
 lib/modules/4.9.41+/kernel/drivers/media/usb/dvb-usb/dvb-usb.ko
```

3. Compiling other modules (an example)

With the source, it is possible to build other kernel modules, such as those for a wireless 802.11ac adapter, which is modified here to work on a Raspberry Pi 1B:
```
# curl -L -o abperiasamy-rtl8812AU_8821AU_linux.zip \
    https://github.com/abperiasamy/rtl8812AU_8821AU_linux/archive/master.zip
# unzip abperiasamy-rtl8812AU_8821AU_linux.zip
# cd rtl8812AU_8821AU_linux-master
# perl -pi.orig -e 's/I386_PC = y/I386_PC = n/; s/ARM_RPI = n/ARM_RPI = y/;' Makefile
# perl -pi.orig -e ’s/\/lib\/modules/\/mnt\/lib\/modules/g;’ Makefile
# perl -pi.orig -e ’s/O1/O2 -pipe -mcpu=arm1176jzf-s -mfpu=vfp/;’ Makefile
# KVER=“4.9.41+" make clean
# KVER=“4.9.41+" make
# KVER=“4.9.41+" make install
# cd ..
# tar cfvz wifi-module.tar.gz lib/modules/4.9.41+/kernel/drivers/net/wireless/rtl8812au.ko
```

4. Building a custom tvheadend (an example)

This is an example of building a minimal tvheadend for routing DVB to a network with no transcoding or other features:
```
# wget -c https://github.com/tvheadend/tvheadend/archive/v4.2.3.tar.gz
# tar xfvz v4.2.3.tar.gz
# cd tvheadend-4.2.3/Autobuild
# cp raspbianjessie-armhf.sh raspbianstretch-armhf.sh
# perl -pi.orig -e ’s/jessie/stretch/g;’ raspbianstretch-armhf.sh
# perl -pi.orig -e ’s/jessie/stretch/g;’ debian.sh
# cd ..
# patch -p1 < /home/pi/tvheadend-4.2.2-remove-uri+avahi-deps.patch
# patch -p1 < /home/pi/tvheadend-4.2.2-tbs5980-cam-init-fix.patch
# AUTOBUILD_CONFIGURE_EXTRA="--cpu=arm1176jzf-s --arch=arm --disable-hdhomerun_static --disable-uriparser --disable-avahi --disable-dvbscan --disable-libav --disable-ffmpeg_static --disable-libx264  --disable-libvorbis --disable-libtheora --disable-libvpx --disable-libx265 --disable-bintray_cache" ./Autobuild.sh raspbianstretch-armhf
```

There should be now 3 things complete: dvb modules, a wifi module and the tvheadend software package.

5. Preparing a ready-made image

Mount the second partition, using fdisk as a guide (512 bytes x 94208 sectors = 48234496 byte offset):
```
# cp 2017-08-16-raspbian-stretch-lite.img 2017-08-20-raspbian-stretch-tvheadend.img
# fdisk -l 2017-08-20-raspbian-stretch-tvheadend.img
Sector size (logical/physical): 512 bytes / 512 bytes
Device   Boot 	Start		End		Sectors	Size		Id	Type
file.img1		8192	93813	85622	41.8M	c	W95 FAT
file.img2		94208	3621911	3527704	1.7G		83	Linux
# losetup --offset 48234496 /dev/loop0 2017-08-20-raspbian-stretch-tvheadend.img
# mount /dev/loop0 /media
```

Add in some critical mounts:
```
# mount -o bind /proc /media/proc
# mount -o bind /dev /media/dev
```

Copy the stuff over to /media from /mnt:
```
# cp /mnt/wifi-module.tar.gz /media
# cp /mnt/dvb-modules.tar.gz /media
# cp /mnt/tvheadend_4.2.3_armhf.deb /media
```

Chroot into the image:
```
# chroot /media
```

Enable ssh:
```
# service ssh enable
```

Configure the wifi:
```
# vi /etc/wpa_supplicant/wpa_supplicant.conf
…
network={
    ssid=“my_network”
    psk=“MyPassword”
}
```

Unpack the modules straight over /lib:
```
# pwd
/
# tar xfvz dvb-modules.tar.gz
lib/modules/4.9.41+/kernel/drivers/net/wireless/rtl8812au.ko
# tar xfvz wifi-module.tar.gz
lib/modules/4.9.41+/kernel/drivers/media/dvb-core/dvb-core.ko
lib/modules/4.9.41+/kernel/drivers/media/rc/rc-core.ko
lib/modules/4.9.41+/kernel/drivers/media/dvb-frontends/stv090x.ko
lib/modules/4.9.41+/kernel/drivers/media/dvb-frontends/stb6100.ko
lib/modules/4.9.41+/kernel/drivers/media/usb/dvb-usb/dvb-usb-tbsqbox2ci.ko
lib/modules/4.9.41+/kernel/drivers/media/usb/dvb-usb/dvb-usb.ko
```

Now update the module dependencies:
```
# depmod -a 4.9.41+
```

Get the firmware for the DVB card:
```
# wget https://github.com/LibreELEC/dvb-firmware/raw/master/firmware/dvb-usb-tbsqbox-id5980.fw -O /lib/firmware/dvb-usb-tbsqbox-id5980.fw
```

Install dvb-apps and its dependencies:
```
# sudo apt-get install dvb-apps libzvbi0 dtv-scan-tables
```

Install tvheadend:
```
# dpkg -i tvheadend_4.2.3_armhf.deb
```

Cleanup:
```
# apt-get clean
# rm dvb-modules.tar.gz wifi-module.tar.gz tvheadend_4.2.3_armhf.deb
```

Exit out of the chroot:
```
# exit
```

Zero the remaining space if it is a concern:
```
# cd
# dd if=/dev/zero of=/media/zerofile
# rm /media/zerofile
```

Unmount:
```
# umount /media
# losetup -d /dev/loop0
```

Write image to a disk:
```
# dd if=2017-08-20-raspbian-stretch-tvheadend.img of=/dev/disk1 bs=64k
```

Compress it for archiving:
```
# xz 2017-08-20-raspbian-stretch-tvheadend.img
```
