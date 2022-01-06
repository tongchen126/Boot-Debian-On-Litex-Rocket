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
1) Don't specify 'litex_rocket_initramfs' config when building kernel, as it will boot from Debian directly.
2) Before building BBL(boot.bin), modify the bootargs of the dts file to specify the boot device. For example,
```
bootargs = "earlycon=sbi console=liteuart,115200 swiotlb=noforce root=/dev/mmcblk0p2 rootwait";

```

# Step3: Build Debian rootfs
Currently, there are some issues using Systemd as init function. (Like RCU Stall, Kernel oops).
So we use sysvinit instead of systemd.
```
sudo apt-get install debootstrap qemu-user-static binfmt-support debian-ports-archive-keyring -y
# This is where you store the Debian rootfs.
mkdir riscv64-chroot       
sudo debootstrap --arch=riscv64 --keyring /usr/share/keyrings/debian-ports-archive-keyring.gpg --include=debian-ports-archive-keyring unstable riscv64-chroot http://deb.debian.org/debian-ports
sudo chroot riscv64-chroot
sudo apt-get update
# Set up basic networking
cat >>/etc/network/interfaces <<EOF
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet dhcp
EOF

# Change from systemd to sysvinit
apt-get install sysvinit-core libpam-elogind

# Set root password
passwd
# Disable the getty on hvc0 as hvc0 and ttyS0 share the same console device in qemu.
ln -sf /dev/null /etc/systemd/system/serial-getty@hvc0.service  

# Fix locales issues
sudo apt-get install locales
dpkg-reconfigure locales

# Add Litex Serial login console
echo "T0:2345:respawn:/sbin/getty -L ttyLXU0 115200 vt100" >> /etc/inittab

# You can also change the hostname of your Debian rootfs by editing /etc/hostname
# Save & Exit
sync
exit
```
# Step4: Copy the 'boot.bin' and rootfs to your sdcard
'boot.json' is here.
```
cp boot.json 'where the first part of your sdcard mounts'
cp boot.bin 'where the first part of your sdcard mounts'
cp -r riscv64-chroot/* 'where the second part of your sdcard mounts'
```

