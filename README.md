# Introduction
 How-To: Make Ubuntu Autoinstall ISO with Cloud-init.
 The autoinstall method uses a “user-data” file similar in usage to what is done with cloud-init. The Ubuntu installer, ubiquity, was modified for this and became subiquity (server ubiquity).
 The autoinstall “user-data” YAML file is a superset of the cloud-init user-data file and contains directives for the install tool curtin. 
 The cloud-init and curtin are good tools but the overall experience of trying to work with these for the special case of the Ubuntu autoinstall is not good. Hopefully this post will help those trying get something working.
 ## Step 0) Prerequisites
 I am building the autoinstall ISO on an Ubuntu 22.04 system. Here are a few packages you will need,

  * 7z sudo apt install p7zip for unpacking the source ISO (including mbr and efi partition images)
  * wget sudo apt install wget to download a fresh daily build of the 22.04 service ISO
  * xorriso sudo apt install xorriso for building the modified ISO

 ## Step 1) Set up the build environment
 Make a directory to work in and get a fresh copy of the server ISO.

```bash
mkdir u22.04-autoinstall-ISO
cd u22.04-autoinstall-ISO
mkdir source-files
wget https:https://releases.ubuntu.com/jammy/ubuntu-22.04.5-live-server-amd64.iso

```

## Step 2) Unpack files and partition images from the Ubuntu 22.04 live server ISO
The Ubuntu 22.04 server ISO layout differs from the 20.04 ISO. 20.04 used a single partition on the ISO but 22.04 has separate gpt partitions for mbr, efi, and the install root image.

7zip is very nice for unpacking the ISO since it will create image files for the mbr and efi partitions for you!

```bash
7z -y x ubuntu-22.04.5-live-server-amd64.iso -osource-files
```

Note, there is no space in -osource-files

## Step 3) Edit the ISO grub.cfg file

Edit source-files/boot/grub/grub.cfg and add the following stanza above the existing menu entries,

```bash
set timeout=0

loadfont unicode

set menu_color_normal=white/black
set menu_color_highlight=black/light-gray

menuentry "Autoinstall Ubuntu Server" {
    set gfxpayload=keep
    linux   /casper/vmlinuz quiet autoinstall ds=nocloud\;s=/cdrom/server/  ---
    initrd  /casper/initrd
}
menuentry "Try or Install Ubuntu Server" {
        set gfxpayload=keep
        linux   /casper/vmlinuz  ---
        initrd  /casper/initrd
}
menuentry "Ubuntu Server with the HWE kernel" {
        set gfxpayload=keep
        linux   /casper/hwe-vmlinuz  ---
        initrd  /casper/hwe-initrd
}
grub_platform
if [ "$grub_platform" = "efi" ]; then
menuentry 'Boot from next volume' {
        exit 1
}
menuentry 'UEFI Firmware Settings' {
        fwsetup
}
else
menuentry 'Test memory' {
        linux16 /boot/memtest86+.bin
}
fi
```
Note the backslash “\” in front of the semi-colon. That is needed to escape the “;”, otherwise grub would treat the rest of the line as a comment!

This menu entry adds the autoinstall kernel directive and the “data source” (ds) for cloud-init of type “nocloud”. s=/cdrom/server/ is a reference to the directory where we will add user-data and meta-data files that contain the installer configuration yaml. /cdrom is the top level directory of the ISO.

…add the directory for the user-data and meta-data files
```bash
mkdir source-files/server
```
Note; you can create other directories to contain alternative user-data file configurations and add extra grub menu entries pointing to those directories. That way you could have multiple install configurations on the same ISO and select the appropriate one from the boot menu during install.

## Step 4) Create and add your custom autoinstall user-data files
This is where you will need to read the documentation for the user-data syntax and format. I will provide a sample file to get you started.

Note; the meta-data file is just an empty file that cloud-init expects to be present (it would be populated with data needed when using cloud services)
```bash
touch source-files/server/meta-data
```

## user-data example file

```bash
#cloud-config
autoinstall:
  version: 1
  early-commands:
        # workaround to stop ssh for packer as it thinks it timed out
         - sudo systemctl stop ssh
  locale: en_US
  keyboard:
    layout: us
  storage:
    layout:
      name: lvm
  identity:
    hostname: ubuntu
    username: ubuntu
    password: $6$RRahlEeL23Ch0J.Q$CrfrPOrh/aQCU7AOfxtY77qWxPhyd8Eu.lBaNNFJQjd9Jxxj3rpgvrf8f.BCdLcSmUIwUZep9x5CP03dqpckC.
  ssh:
    install-server: yes
    allow-pw: true
    ssh_quiet_keygen: true
  packages:
    - tmux
    - whois
    - dnsutils
  user-data:
    disable_root: false
  late-commands:
    - echo 'admin ALL=(ALL) NOPASSWD:ALL' > /target/etc/sudoers.d/admin
    - curtin in-target --target=/target -- chmod 440 /etc/sudoers.d/admin
    - echo "user created "
```

## Step 5) Generate a new Ubuntu 22.04 server autoinstall ISO

The following command is helpful when trying to setup the arguments for building an ISO. It will give flags and data to closely reproduce the source base install ISO.


```bash
xorriso -indev jammy-live-server-amd64.iso -report_el_torito as_mkisofs
```

Editing the report from the above I was able to come up with the command below for creating the autoinstall ISOs.

```bash
cd source-files

xorriso -as mkisofs -r \
  -V 'Ubuntu 22.04 LTS AUTO (EFIBIOS)' \
  -o ../ubuntu-22.04-autoinstall.iso \
  --grub2-mbr ../BOOT/1-Boot-NoEmul.img \
  -partition_offset 16 \
  --mbr-force-bootable \
  -append_partition 2 28732ac11ff8d211ba4b00a0c93ec93b ../BOOT/2-Boot-NoEmul.img \
  -appended_part_as_gpt \
  -iso_mbr_part_type a2a0d0ebe5b9334487c068b6b72699c7 \
  -c '/boot.catalog' \
  -b '/boot/grub/i386-pc/eltorito.img' \
    -no-emul-boot -boot-load-size 4 -boot-info-table --grub2-boot-info \
  -eltorito-alt-boot \
  -e '--interval:appended_partition_2:::' \
  -no-emul-boot \
  .
```

Note; the “.” at the end. That is indicating the path to the current directory i.e. “use the current directory for the source files to build the ISO”.

The partition images that 7z extracted for us are being added back to the recreated ISO.
