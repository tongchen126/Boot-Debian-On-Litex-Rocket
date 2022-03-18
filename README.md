# Here is a few steps that might help you to boot debian on [Litex Rocket](https://github.com/litex-hub/linux-on-litex-rocket).  
# Step1: SDCard preparation
Partition the sdcard into two partitions.  
The first partition is used to store boot.bin while the second to store rootfs.  
```
parted /dev/YourSDCard -s mklabel msdos  
parted /dev/YourSDCard -s mkpart primary fat32 1 256M  
parted /dev/YourSDCard -s mkpart primary ext4 256M 100%
mkfs.fat /dev/YourSDCard_FirstPart
mkfs.ext4 /dev/YourSDCard_SecondPart
```
# Step2: Prepare 'boot.bin'
Detailed information can be found [here](https://github.com/litex-hub/linux-on-litex-rocket#building-the-software-bootbin-busybox-linux-and-bbl).
Few points to mention:
1) You can skip the '1. Building BusyBox' and '2. Creating the initramfs.cpio' steps in the above link. And you don't have to specify 'litex_rocket_initramfs' config when building kernel. Because we will boot from Debian directly and initramfs is not needed.  

3) Before building BBL(boot.bin), modify the bootargs of the dts file to specify the second partition of sdcard as boot device. For example,
```
bootargs = "earlycon=sbi console=liteuart,115200 swiotlb=noforce root=/dev/mmcblk0p2 rootwait";

```
3) Also, as the latest kernel uses a newer MMC module, you should make some change to the [original dts](https://github.com/litex-hub/linux-on-litex-rocket/tree/master/conf) according to [this](https://github.com/tongchen126/Boot-Debian-On-Litex-Rocket/commit/1bcbc8b602e83ea81bc24e25deed52ecf9d9fdb9).  
4) If you have trouble building, please use the toolchain [here](https://github.com/tongchen126/Boot-Debian-On-Litex-Rocket/releases/download/rootfs/riscv64-unknown-elf-gcc-8.3.0.tgz).  
See the modified [nexys_video.dts](https://github.com/tongchen126/Boot-Debian-On-Litex-Rocket/blob/main/nexys_video_prebuilt/nexys_video.dts) for more detail.

# Step3: Build Debian rootfs
Currently, there are some issues using Systemd as init function. (Like RCU Stall, Kernel oops).
So we use sysvinit instead of systemd.
```
sudo apt-get install debootstrap qemu-user-static binfmt-support debian-ports-archive-keyring -y
# This is where you store the Debian rootfs.
mkdir riscv64-chroot 
# You can change to other mirrors like ftp.kr.debian.org or ftp.de.debian.org if deb.debian.org was slow or failed to download some packages.
sudo debootstrap --arch=riscv64 --keyring /usr/share/keyrings/debian-ports-archive-keyring.gpg --include=debian-ports-archive-keyring unstable riscv64-chroot http://deb.debian.org/debian-ports

sudo chroot riscv64-chroot
chmod 1777 tmp
chmod a+w /dev/null
apt-get update

# Set up basic networking
cat >>/etc/network/interfaces <<EOF
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet dhcp
EOF

# Fix locales issues
apt-get install locales -y
dpkg-reconfigure locales

# Change from systemd to sysvinit
apt-get install sysvinit-core libpam-elogind -y

# Set root password
passwd

# Add Litex Serial login console
echo "T0:2345:respawn:/sbin/getty -L ttyLXU0 115200 vt100" >> /etc/inittab

# Some common stuff
apt-get install net-tools vim bc ntp openssh-server -y

# Link the right timezone based on your location
ln -sf /usr/share/zoneinfo/Etc/GMT+8 /etc/localtime

# You can also change the hostname of your Debian rootfs by editing /etc/hostname
# Save & Exit
sync
exit
```
Refer to the pages for more information: [build debian rootfs](https://wiki.debian.org/RISC-V), [switch to sysvinit from systemd on debian](https://wiki.debian.org/Init).
# Step4: Copy the 'boot.bin' and rootfs to your sdcard
'boot.json' is [here](https://github.com/tongchen126/Boot-Debian-On-Litex-Rocket/blob/main/boot.json).
```
cp boot.json 'where the first part of your sdcard mounts'
cp boot.bin 'where the first part of your sdcard mounts'
cp -r riscv64-chroot/* 'where the second part of your sdcard mounts'
chmod 1777 'where the second part of your sdcard mounts'/tmp    # Make sure the tmp folder has the right access mode
chmod a+w 'where the second part of your sdcard mounts'/dev/null    # Make sure the /dev/null has the right access mode
sync
```
# Rootfs tarball
A prebuilt bitstream for nexys video is uploaded. The build args is './digilent_nexys_video.py --build --cpu-type rocket --cpu-variant full4d --with-sdcard --with-ethernet', running at 100MHz.  
  
A rootfs tarball is on the release page. The default root password is 'password'. You can skip Step 3 after downloading the tarball.  
A compiled kernel 'vmlinux' is also uploaded.  


# Demo on Nexys Video  
[video](https://user-images.githubusercontent.com/31961076/148379809-498dda97-2297-4b3c-8180-8a1d49f2c028.mp4)

# TODO
FIX the RCU stall and kernel oops when use systemd as init function.
