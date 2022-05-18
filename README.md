# artix
personal setup guide for private computer

# Setup Artix
## prerequisites

### Download the Artix iso

Go to the offician artix site: https://artixlinux.org and download the iso.
```
https://artixlinux.org/download.php
```

Create a bootable usb-drive with with the freshly downloaded iso in this case the openrc version.

Boot from the USB-drive and start the ssh service, now you can login from another machine into the installation device (this is not nececcary)this makes it more convinient since we can copy and paste.
``` rc-service sshd start ```

Now ssh into the machine with user artix password artix
``` 
ssh [IP-ADDRESS-OF-THE-PC] -l artix
ssh 192.168.0.192 -l artix
```

```
su root
sudo rc-update add sshd default
sudo rc-service sshd start
```

### Partitioning - https://wiki.artixlinux.org/Main/Installation#Partition_your_disk_.28BIOS.29

> NOTE: The BIOS boot partition is necessary on UEFI systems with a GPT-partitioned disk. EFI system partition has to be created and mounted at /mnt/boot and the suggested size is around 512 MiB.

Since I all of my installed harddrives have nothing imoportant on them I will just wipe them out.

```
wipefs -a /dev/DEVICENAME
wipefs -a /dev/sda
wipefs -a /dev/nvme0n1
wipefs -a /dev/nvme1n1
```


Now you have to create partitions, i choose the following partitioning scheme:

- partition 1 boot 128M
- parititon 2 root 15G
- parittion 3 home 25G
- partition 4 data rest of space

### Format Partition and create filesystem - https://wiki.artixlinux.org/Main/Installation#Format_partitions

> The -L switch assigns labels to the partitions, which helps referring to them later through /dev/disk/by-label without having to remember their numbers

If you are doing a UEFI installation, the boot partition is not optional and needs to be formatted as fat.

```
mkfs.fat /dev/nvme0n1p1
mkfs.ext4 -L ROOT /dev/nvme0n1p2
mkfs.ext4 -L HOME /dev/nvme0n1p3
mkfs.ext4 -L DATA /dev/nvme0n1p4

fatlabel /dev/nvme0n1p1 BOOT

mkfs.ext4 -L GameLibrary /dev/sda1
```
### Mounting - https://wiki.artixlinux.org/Main/Installation#Mount_Partitions


Now the corresponding folders need to be created and mounted to the specific partition: 

```
 mount /dev/nvme0n1p2 /mnt
 mkdir /mnt/{boot,home,data,GameLibrary}
 mount /dev/nvme0n1p1 /mnt/boot
 mount /dev/nvme0n1p3 /mnt/home
 mount /dev/nvme0n1p4 /mnt/data
 mount /dev/sda1 /mnt/GameLibrary
```
 
 ## Installing OS - https://wiki.artixlinux.org/Main/Installation#Install_base_system

Start to install the base system and handy applications
```
basestrap /mnt base base-devel openrc elogind-openrc linux linux-firmware intel-ucode git zsh vim
```

### Generate a new fstab file with the integrated fstabgen script 
> Use fstabgen to generate /etc/fstab, use -U for UUIDs as source identifiers and -L for partition labels:

```
fstabgen -U /mnt >> /mnt/etc/fstab 
```

### Log into the system and start to configure it - https://wiki.artixlinux.org/Main/Installation#Configure_the_base_system 

> Note that this will default to UTC. If you use Windows and you want the time to be synchronized in both Artix and Windows, follow https://wiki.archlinux.org/title/System_time#UTC_in_Windows for instructions to enable UTC in there also.



```
 artix-chroot /mnt
```

```
 ln -sf /usr/share/zoneinfo/Europe/Vienna /etc/localtime
 hwclock --systohc
 date # to check time
```

```
vim /etc/locale.gen # write sed command to uncomment line en_US.UTF-8
UTF-8
locale-gen
```


### Installing a Bootloader - grub
```
 pacman -S grub os-prober efibootmgr
 grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=grub   
 grub-mkconfig -o /boot/grub/grub.cfg
```

### creating a new user and adding it to sudoers
```
passwd
useradd -m stywen
usermod -G wheel stywen
passwd stywen
vim /etc/sudoers # search for wheel
```

### Change the hostname
```
echo "suecyde-machine" > /etc/hostname
```

### install essential packages like dhcpcd and ssh

```
pacman -S dhcpcd-openrc
pacman -S openssh-openrc
pacman -S connman-openrc

rc-update add connmand
rc-update add dhcpcd
rc-update add sshd
```

### exit and reboot
```
exit                         
umount -R /mnt
reboot
```

### After installation configuration

ssh into user and su to root

``` ssh 192.168.0.192 -l stywen ```

### enable arch reposetories
```
sudo pacman -S artix-archlinux-support
sudo vim /etc/pacman.conf 
```

Uncomment the artix lib32 repositories and add the following lines
```

[extra]
Include = /etc/pacman.d/mirrorlist-arch

[community]
Include = /etc/pacman.d/mirrorlist-arch

[multilib]
Include = /etc/pacman.d/mirrorlist-arch
```

```
sudo pacman-key --populate archlinux
sudo pacman -Syy
sudo pacman -Syu
```

### install amd gpu drivers
```
sudo pacman -S lib32-mesa vulkan-radeon lib32-vulkan-radeon vulkan-icd-loader lib32-vulkan-icd-loader xf86-video-amdgpu  libva-mesa-driver mesa-vdpau -y

sudo echo "RADV_PERFTEST=aco" >> /etc/environment

```

### install yay

```
cd /opt && sudo git clone https://aur.archlinux.org/yay-git.git && sudo chown -R $USER:$USER ./yay-git && cd yay-git && makepkg -si
```


### install windowmanager and more essential tools

### Keyring - https://wiki.archlinux.org/index.php/GNOME/Keyring
> GNOME Keyring is "a collection of components in GNOME that store secrets, passwords, keys, certificates and make them available to applications."
```
sudo pacman -S gnome-keyring seahorse
```




# Xorg
```
sudo pacman -S xorg xorg-xinit

vim ~/.xinitrc

# add the following lines to .xinitrc
#!/bin/sh
#
# ~/.xinitrc
#
# Executed by startx (run your window manager from here)

if [ -d /etc/X11/xinit/xinitrc.d ]; then
  for f in /etc/X11/xinit/xinitrc.d/*; do
    [ -x "$f" ] && . "$f"
  done
  unset f
fi
exec i3
```

### Fix Screentearing

copy the following lines into /etc/X11/xorg.conf.d/20-amd-gpu.conf

```
sudo vim /etc/X11/xorg.conf.d/20-amd-gpu.conf

Section "Device"
    Identifier  "AMD Graphics"
    Driver      "amdgpu"
    Option      "TearFree"  "true"
EndSection

```

### configure Resolution
#### make sure there is no x-session running

```
X -configure
cp xorg.conf.new /etc/X11/xorg.conf.d/xorg.conf
vim /etc/X11/xorg.conf.d/xorg.conf

# add the following line in every subsection where "Depth" has the value 24
	SubSection "Display"
		Viewport   0 0
		Depth     24
>>>		Modes     "5120x1440"
	EndSubSection
# change will take effect after restarting xorg
```


### install misc

```
sudo pacman -S i3 dmenu alacritty pulseaudio pulseaudio-alsa alsa-utils pavucontrol
```


# Gaming (Optional)
## Make the GameLibrary accessable for the user and writable
```
sudo chown stywen /GameLibrary
rm -rf /GameLibrary/lost*
```

#### install winde and lutris


```
sudo pacman -S wine-staging giflib lib32-giflib libpng lib32-libpng libldap lib32-libldap gnutls lib32-gnutls mpg123 lib32-mpg123 openal lib32-openal v4l-utils lib32-v4l-utils libpulse lib32-libpulse libgpg-error lib32-libgpg-error alsa-plugins lib32-alsa-plugins alsa-lib lib32-alsa-lib libjpeg-turbo lib32-libjpeg-turbo sqlite lib32-sqlite libxcomposite lib32-libxcomposite libxinerama lib32-libgcrypt libgcrypt lib32-libxinerama ncurses lib32-ncurses opencl-icd-loader lib32-opencl-icd-loader libxslt lib32-libxslt libva lib32-libva gtk3 lib32-gtk3 gst-plugins-base-libs lib32-gst-plugins-base-libs vulkan-icd-loader lib32-vulkan-icd-loader lutris steam -y
```

## increase the Esync-overhead
```
sudo vim /etc/security/limits.conf

# add the following line
stywen hard nofile 524288
```

#### Install a browser
```
yay -S brave
```

### i3 
```
echo "gaps inner 10" >> ~/.config/i3/config
echo "gaps outer 5" >> ~/.config/i3/config
echo "default_border pixel 3">> ~/.config/i3/config
```


# Wayland
## install and setup wayland

https://www.fosskers.ca/en/blog/wayland
> Wayland is the next generation Display Protocol for Linux. You've probably heard of "X" (or "X11" or "XOrg"), but you may not have known of its issues: age, performance, security, and dev-friendliness. Even Adam Jackson, the long-time release manager of X calls for the adoption of Wayland. That said, X is well-established and the transition won't happen over night. Many core apps on a Linux system are bound tightly to its ecosystem:

```
sudo pacman -S sway waybar alacritty  wofi xorg-xwayland xorg-xlsclients qt5-wayland glfw-wayland pulseaudio pulseaudio-alsa alsa-utils pavucontrol
```

#### setup the default sway config

```
mkdir -p ~/.config/sway && cp /etc/sway/config ~/.config/sway
```


#### A 10-pixel border around every window.
````
echo "gaps inner 10" >> ~/.config/sway/config
echo "gaps outer 5" >> ~/.config/sway/config
````

#### Removes the title bar of each window.
```
echo "default_border pixel 3">> ~/.config/sway/config
```
echo "output * resolution --custom 5120x1440" >> ~/.config/sway/config

neofetch
