########################################################################################
#########################Encrypted installation of Arch Linux###########################
########################################################################################

### Set-up an Internet Connection
ip a
ping -c 5 google.com

### Increase Terminal Font-Size
pacman -Syy terminus-font && fc-cache -fv && setfont ter-v32b

### Overwrite Drive (optional)
shred -vfz -n 3 /dev/sd*

### Create Partitions
fdisk -l
fdisk /dev/sd*
g (gpt)
# EFI partition
n
ENTER
ENTER
+500M
t
1 (For EFI)
# boot partition
n
ENTER
ENTER
+500M
# LVM partition
n
ENTER
ENTER
ENTER (left space)
t
30 (Linux LVM)
p (control)
w (write)

### Set-up encryption on /dev/sd*3
cryptsetup luksFormat -c aes-xts-plain -y -s 512 -h sha512 /dev/sd*

### Open encrypted partition
cryptsetup luksOpen /dev/sd*3 lvm

### LVM Setup
# physical volume
pvcreate /dev/mapper/lvm
# volumegroup
vgcreate main /dev/mapper/lvm
# LVM
lvcreate -L 50GB -n lv_root main
lvcreate -l 100%FREE -n lv_home main

# Activate volumegroup
modprobe dm-crypt
vgscan
vgchange -ay

### Create filesystems
# lv_root
mkfs.ext4 /dev/main/lv_root
# lv_home
mkfs.ext4 /dev/main/lv_home
# boot
mkfs.ext4 /dev/sd*2
# EFI Partition
mkfs.fat -F32 /dev/sd*1

### Mount volumes
# lv_root
mount /dev/main/lv_root /mnt
# lv_home
mkdir /mnt/home
mount /dev/main/lv_home /mnt/home
# boot
mkdir /mnt/boot
mount /dev/sd*2 /mnt/boot
# control
mount

### Create fstab
mkdir /mnt/etc
genfstab -Up /mnt >> /mnt/etc/fstab
# control
cat /mnt/etc/fstab

### Configurate mirrorlist
# backup mirrorlist
sudo cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.backup
# install reflector
pacman -Syy reflector reflector rsync curl
# Update mirrorlist
reflector --verbose --country 'United States' -l 5 -p https --sort rate --save /etc/pacman.d/mirrorlist
# Refresh package list
pacman -Syy

### Install arch-base package
pacstrap -i /mnt base base-devel

### Sign into arch
arch-chroot /mnt

### Install basic packeges
pacman -S git nano vim bash-completion grub efibootmgr dosfstools os-prober mtools linux linux-headers linux-firmware lvm2 cryptsetup net-tools networkmanager network-manager-applet netctl wireless_tools wpa_supplicant dialog intel-ucode

### Enable networkmanager
systemctl enable NetworkManager     [NetworkManager | not networkmanager]

### Install and configure bootloader
nano /etc/default/grub

change
'GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet"'
to
'GRUB_CMDLINE_LINUX_DEFAULT="cryptdevice=/dev/sd*3:main:allow-discards loglevel=3 quiet"'

uncomment
'#GRUB_ENABLE_CRYPTODISK=y'

# mount EFI Partition
mkdir /boot/EFI
mount /dev/sd*1 /boot/EFI

# install GRUB
grub-install --target=x86_64-efi --efi-directory=/boot/EFI --bootloader-id=grub_uefi --recheck --debug
mkdir -p /boot/grub/locale
cp /usr/share/locale/en\@quot/LC_MESSAGES/grub.mo /boot/grub/locale/en.mo
grub-mkconfig -o /boot/grub/grub.cfg

### Disable GRUB delay
# Hide GRUB
/etc/default/grub 
'GRUB_TIMEOUT=0'

### Update mkinitcpio
nano /etc/mkinitcpio.conf

change
HOOKS=(base udev autodetect modconf block filesystems keyboard fsck)
to
HOOKS=(base udev autodetect modconf block encrypt lvm2 filesystems keyboard fsck)

change 
MODULES=(nvme)

# update initramfs image
mkinitcpio -p linux

### Configurations
# Set Language and Locales
nano /etc/locale.gen [en_US.UTF-8 in every case]
# generate locales
locale-gen
# Set-up language in locale.conf
echo LANG=en_US.UTF-8 > /etc/locale.conf

# Set timezone
tzselect
ln -sf /usr/share/zoneinfo/America/Denver /etc/localtime
hwclock --systohc --utc

# Set Hostname
echo Arch > /etc/hostname

### Set Root password
passwd

### Create user Account
useradd -m -g users -G wheel <username>
passwd <username>
# grant user sudo Powers
nano /etc/sudoers
uncomment
#%wheel ALL=(ALL) ALL

### Set-up swap
fallocate -l 8G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
# control
cat /etc/fstab

### Reboot ...
exit
umount -a
reboot
