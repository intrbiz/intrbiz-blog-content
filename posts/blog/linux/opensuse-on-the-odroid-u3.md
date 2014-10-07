---
Author: Chris Ellis
Date: 2014-10-07
Category: blog/linux
---
# openSUSE on the Odroid U3

A while back I got an [Odroid U3](http://hardkernel.com/main/products/prdt_info.php?g_code=G138745696275), 
an ARM based single board computer, much like the Raspberry PI, only a lot more 
powerful - the Odroid U3, has a quad core 1.7GHz ARM CPU and 2GB RAM.

However only images for xubuntu are provided officially, being an openSUSE user 
I wanted to run openSUSE on the Odroid U3.  I managed to hack together an image 
a few months back, but sadly didn't really document how I did it, rather silly 
of me.

The basic approach is to hack together an image, from the xubuntu Odroid U3 
images and the openSUSE JEOS root file system images.  This allows the use of 
the Odroid U3 customised kernels with an openSUSE based user space.  We can then 
use the Odroid U3 kernel updating script once our image boots to get the latest 
kernel images.

To build our image, we are going to need a machine, with a few gigs of free 
space to be able to assemble and then copy the image over.  You will also need 
a USB micro-SD card reader to be able to copy the image to the card.

All steps should be executed as root!

First we need to download some stuff that we need to build our franken-image:

    buildhost:~ # wget http://dn.odroid.com/4412/Linux/ubuntu-u2-u3/xubuntu-13.10-desktop-armhf_odroidu_20140211.img.xz
    buildhost:~ # xz -d xubuntu-13.10-desktop-armhf_odroidu_20140211.img.xz
    buildhost:~ # wget http://download.opensuse.org/ports/armv7hl/distribution/13.1/appliances/openSUSE-13.1-ARM-JeOS.armv7-rootfs.armv7l-1.12.1-Build33.1.tbz
    buildhost:~ # wget http://builder.mdrjr.net/tools/boot.scr_opensuse.tar
    buildhost:~ # wget http://builder.mdrjr.net/tools/kernel-update.sh
    buildhost:~ # wget http://builder.mdrjr.net/kernel-3.8/00-LATEST/odroidu2.tar.xz
    buildhost:~ # wget http://builder.mdrjr.net/tools/firmware.tar.xz
    buildhost:~ # xz -d odroidu2.tar.xz
    buildhost:~ # xz -d firmware.tar.xz

Now take a copy of the xubuntu image, this will form the basis for our custom 
image.  By starting with a working image, we have the partitioning and 
bootloader already installed.  As I understand it, the first 1MiB of the image 
is taken up by the phase one bootloader and the position of partitions is 
important.  The phase one bootloader will read boot.scr from the `vfat` 
partition which then loads the zImage (compressed kernel).

    buildhost:~ # cp xubuntu-13.10-desktop-armhf_odroidu_20140211.img custom_opensuse.img

Next we need to ensure that the loop device module is loaded.

    buildhost:~ # modprobe loop.

Now we can mount our image.

    buildhost:~ # losetup /dev/loop0 custom_opensuse.img
    buildhost:~ # kpartx -a /dev/loop0
    buildhost:~ # mkdir tmp_boot
    buildhost:~ # mkdir tmp_root
    buildhost:~ # mount /dev/mapper/loop0p1 tmp_boot
    buildhost:~ # mount /dev/mapper/loop0p2 tmp_root

Now empty out the entire root file system, we are going to replace that 
with the openSUSE root file system.  We will then patch in the latest kernels.

    buildhost:~ # cd tmp_root
    buildhost:~/tmp_root # rm -rf *

Now extract the openSUSE root file system into our image

    buildhost:~/tmp_root # tar -xjf ~/openSUSE-13.1-ARM-JeOS.armv7-rootfs.armv7l-1.12.1-Build33.1.tbz

Now we can patch in the odroid kernel

    buildhost:~/tmp_root # cd boot
    buildhost:~/tmp_root/boot # cp ../../boot.scr_opensuse.tar ./
    buildhost:~/tmp_root/boot # tar -xvf boot.scr_opensuse.tar
    buildhost:~/tmp_root/boot # rm boot.scr_opensuse.tar
    buildhost:~/tmp_root/boot # rm -rvf x
    buildhost:~/tmp_root/boot # mv x2u2/* ./
    buildhost:~/tmp_root/boot # rm -rvf x2u2
    buildhost:~/tmp_root/boot # cp boot-hdmi-720p60hz.scr boot.scr
    buildhost:~/tmp_root/boot # cd ..
    buildhost:~/tmp_root # cd lib/modules
    buildhost:~/tmp_root/lib/modules # rm -rf *
    buildhost:~/tmp_root/lib/modules # cd ../..
    buildhost:~/tmp_root # cp ../odroidu2.tar ./
    buildhost:~/tmp_root # tar -xvf odroidu2.tar
    buildhost:~/tmp_root # rm odroidu2.tar
    buildhost:~/tmp_root # cd lib/firmware
    buildhost:~/tmp_root/lib/firmware # rm -rf *
    buildhost:~/tmp_root/lib/firmware # cp ../../../firmware.tar.xz ./
    buildhost:~/tmp_root/lib/firmware # tar -xvf firmware.tar.xz
    buildhost:~/tmp_root/lib/firmware # rm firmware.tar.xz
    buildhost:~/tmp_root/lib/firmware # cd ../../..
    buildhost:~ # cd tmp_boot
    buildhost:~/tmp_boot # rm -rvf *
    buildhost:~/tmp_boot # cp ../tmp_root/boot/* ./

Now we need to setup fstab, edit `~/tmp_root/etc/fstab` to have the following contents:
    
    devpts  /dev/pts          devpts  mode=0620,gid=5 0 0
    proc    /proc             proc    defaults        0 0
    sysfs   /sys              sysfs   noauto          0 0
    debugfs /sys/kernel/debug debugfs noauto          0 0
    usbfs   /proc/bus/usb     usbfs   noauto          0 0
    tmpfs   /run              tmpfs   noauto          0 0
    UUID=e139ce78-9841-40fe-8823-96a304a09859 / ext4 defaults 0 0
    /dev/mmcblk0p1 /boot vfat defaults 0 0

Note: use `blkid` to check the UUID of the root fs (/dev/mapper/loop0p2) is `e139ce78-9841-40fe-8823-96a304a09859`

Now lets make some changes to the systemd journal config, edit `~/tmp_root/etc/systemd/journald.conf` 
change the following variables:

    Storage=volatile
    ForwardToConsole=yes
    TTYPath=/dev/tty12

This allows us to see the journal on TTY 12 (`CTRL-ALT-F12`) which is handy for 
debugging, it also doesn't store the journal on disk (as SD cards can be 
horrifically slow).

Now we need to make some changes to the systemd config:

    buildhost:~/tmp_root # cd etc/systemd/system
    buildhost:~/tmp_root/etc/systemd/system # cd default.target.wants
    buildhost:~/tmp_root/etc/systemd/system/default.target.wants # rm -rvf *
    buildhost:~/tmp_root/etc/systemd/system/default.target.wants # cd ..
    buildhost:~/tmp_root/etc/systemd/system # cd multi-user.target.wants
    buildhost:~/tmp_root/etc/systemd/system/multi-user.target.wants # rm -f wpa_supplicant.service
    buildhost:~/tmp_root/etc/systemd/system/multi-user.target.wants # rm -f remote-fs.target
    buildhost:~/tmp_root/etc/systemd/system/multi-user.target.wants # cd ../..
    buildhost:~/tmp_root/etc/systemd/ # ln -sf /usr/lib/systemd/system/multi-user.target default.target
    
It's handy to copy in the kernel update script:
    
    buildhost:~/tmp_root/root # cp ../../kernel-update.sh ./
    buildhost:~/tmp_root/root # chmod +x kernel-update.sh

Finally, we can unmount and flash the image

    buildhost:~/ # sync
    buildhost:~/ # umount tmp_root
    buildhost:~/ # umount tmp_boot
    buildhost:~/ # dd if=./custom_opensuse.img of=/dev/sdb bs=32M

Note: make sure `/dev/sdb` is the SD card you want to write to, check `dmesg` if need be.

Nore: your liable to get odd errors from the SD card if you fail to set `bs`, a 32MiB 
block seemed the fastest for me.

Now you should be able to boot your Odroid U3 into openSUSE 13.1 (console).  Once 
booted you can use the zypper to update any RPMs or to install a GUI.

It is worth noting, that the I/O performance of cheap SD cards is pretty terrible
(or at least this is true for SD card one I have), remember to be patient.

Any issues and I can probably give you a hand in the #suse IRC channel or tweet me @intrbiz

## TL; DR

I'll upload my image shortly.

