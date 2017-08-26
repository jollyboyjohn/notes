### Raspbian Kernel Compile
Notes on how to build a Raspbian kernel for the purpose of:
* Applying patches to the kernel
* Getting Module.symver for building external modules

This is actually what I use to build on a Pi3 (ARMv7 - quad core) for a Pi Zero (ARMv6 - single core).

1. Upgrade to the latest software and kernel (raspberrypi-kernel) 
```
sudo apt-get update 
sudo apt-get upgrade
```

2. Reboot into the latest kernel
```
reboot
```

3. Download and install ‘rpi-source’
```
sudo wget https://raw.githubusercontent.com/notro/rpi-source/master/rpi-source -O /usr/bin/rpi-source 
sudo chmod +x /usr/bin/rpi-source
/usr/bin/rpi-source -q --tag-update
```

4. Get the new kernel source for your currently running kernel - I put it on an external disk under /mnt
```
cd /mnt
/usr/bin/rpi-source
```

5. There’ll be three files, the source directory, a symlink to it, and an archive for the kernel source
```
cd /mnt/linux
```

6. Apply patches to the kernel if you need to (e.g. DVB patch)
```
patch -p1 < dvb-usb-tbsqbox-id5980.patch
```

7A. Preconfigure the kernel if it is for a Pi 1 / Zero…
```
KERNEL=kernel
make bcmrpi_defconfig
```

7B. Preconfigure the kernel if it is for a Pi 2 / 3 …
```
KERNEL=kernel7
make bcm2709_defconfig
```

8. Double check the version to be used; V6 = Pi 0/1, V7 = Pi 2/3
```
grep "ARCH_MULTI_V.=" .config
```

9. Configure the kernel if you need to enable / modify / disable features
```
make menuconfig
```

10. Compile kernel using all available cores (-j4 on a Pi 3 ) - 2 hours  20 mins on a Pi 3
```
make -j4 ARCH=arm zImage modules dtbs
```

11. Create a place to put the kernel, modules, and other boot materials (e.g. target with boot/overlays subdirectories) 
```
sudo mkdir -p /mnt/target/boot/overlays
```

12. Install the modules to the target location 
```
sudo make INSTALL_MOD_PATH=/mnt/target modules_install
```

13. Copy the kernel and boot materials to the location
```
sudo cp arch/arm/boot/zImage /mnt/target/boot/$KERNEL.img
sudo cp arch/arm/boot/dts/*.dtb /mnt/target/boot/
sudo cp arch/arm/boot/dts/overlays/*.dtb* /mnt/target/boot/overlays/
sudo cp arch/arm/boot/dts/overlays/README /mnt/target/boot/overlays/
```

14. Archive source, keeping only the critical files (saves about 90% on disk utilisation):
```
cp Module.symvers /tmp
cp .config /tmp
make distclean
cp /tmp/.config .
make modules_prepare
cp /tmp/Module.symvers .
cd ..
tar cfvz rpi-linux-source-4.9.24.tar.gz \
	linux/include linux/scripts linux/Makefile linux/Module.symvers \
	linux/arch/arm/boot/dts/include \
	linux/arch/arm/include \
	linux/arch/arm/plat* \
	linux/arch/arm/mach*
rm -rf linux*
```

15. Archive the boot and lib
```
cd target
tar cfvz rpi-linux-4.9.24-armv6.tar.gz boot lib
```

16. On the remote machine, install the kernel and modules, etc:
```
cd /
sudo tar xfz /home/pi/rpi-linux-4.9.24-armv6.tar.gz
```

17. If the source is needed, extract and link it
```
sudo mv /home/pi/rpi-linux-source-4.9.24.tar.gz /usr/src
cd /usr/src
sudo tar xfz rpi-linux-source-4.9.24.tar.gz
sudo ln -sf /usr/src/linux /lib/modules/4.9.24+/build
sudo ln -sf /usr/src/linux /lib/modules/4.9.24+/source
```

18. Example: Building a module for my AC Wifi adapter:
```
curl -L -o abperiasamy-rtl8812AU_8821AU_linux.zip \
    https://github.com/abperiasamy/rtl8812AU_8821AU_linux/archive/master.zip
unzip abperiasamy-rtl8812AU_8821AU_linux.zip
cd rtl8812AU_8821AU_linux-master
perl -pi.orig -e 's/I386_PC = y/I386_PC = n/; s/ARM_RPI = n/ARM_RPI = y/;' Makefile
KVER="4.9.24+" make clean
KVER="4.9.24+" make
KVER="4.9.24+" make install
```
