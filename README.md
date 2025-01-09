# build-custom-live-sid-iso
I will demonstrate how to build your own custom iso based on Debian Sid. 


Install dependencies 
```
root@debian:~# apt install debootstrap arch-install-scripts squashfs-tools xorriso libisoburn-dev libisoburn1t64
```
I will create some directories and variables for the directories for the build. 

```
mkdir -vp /sid-live/chroot

export CR=/sid-live
export chrt=/sid-live/chroot

mkdir -vp "$CR"/{staging/{EFI/BOOT,boot/grub/x86_64-efi,live},tmp}
```

I will build the base system using debootstrap.

```
debootstrap --components=main,contrib,non-free,non-free-firmware sid $chrt http://ftp.us.debian.org/debian/
```

I will create hostname and hosts files.

```
echo 'crunchy-sid' > $chrt/etc/hostname


cat > $chrt/etc/hosts << EOF
127.0.0.1	localhost
127.0.1.1	crunchy-sid
::1  		localhost ip6-localhost ip6-loopback
ff02::1  	ip6-allnodes
ff02::2  	ip6-allrouters
EOF
```

I will create directories needed while in chroot, and then chroot into the new build. 

```
mkdir -vp $chrt/live-temp

mkdir -vp $chrt/live-boot-grub

mount --rbind $CR/tmp $chrt/live-temp

mount --rbind $CR/staging/EFI/BOOT $chrt/live-boot-grub

mount --bind $chrt $chrt

arch-chroot $chrt /bin/bash --login 
```

### While in Chroot

I will set a new passowrd for root.

```
passwd
New password: 
Retype new password: 
passwd: password updated successfully
```

I will install locales and reconfigure tzdata. I selected 97 for en_US.UTF-8 UTF-8 and then 2 for C.UTF-8 for my locales. I selected 2 for Americas and 37 for Chicago for tzdata. You may need to make changes to this. 

```
~# apt update && apt upgrade

~# apt install locales -y

~# dpkg-reconfigure locales
Locales to be generated: 97
Default locale for the system environment: 2
Generating locales (this might take a while)...
  en_US.UTF-8... done
Generation complete.

~# dpkg-reconfigure tzdata
Geographic area: 2
Time zone: 37
```
I will install what I will need for live boot.
```
apt install linux-headers-amd64 linux-image-amd64 live-boot live-config-systemd live-config live-boot-initramfs-tools live-tools grub-efi-amd64-bin grub2
```
This can be changed based on your preferences.
```
apt install --fix-missing libalpm13t64 sudo git curl libarchive-tools git help2man python3 rsync texinfo texinfo-lib ttf-bitstream-vera build-essential dosfstools efibootmgr mtools os-prober dmeventd libdevmapper-dev libdevmapper-event1.02.1 libdevmapper1.02.1 libfont-freetype-perl python3-freetype libghc-gi-freetype2-dev libghc-gi-freetype2-prof fuse2fs libconfuse2 libfuse2t64 gettext xorriso libisoburn1t64 libisoburn-dev autogen gnulib libfreetype-dev pkg-config m4 libtool automake flex fuse3 libfuse3-dev gawk autoconf-archive rdfind wget fonts-dejavu lzma liblzma5 liblzma-dev liblz1 liblz-dev unifont acl libzfslinux-dev initramfs-tools cryptsetup lvm2 attr firmware-linux-free firmware-linux firmware-linux-nonfree task-xfce-desktop base-files -y
```

I will create a new user and set the password. 

```
~# useradd -mG cdrom,floppy,sudo,audio,dip,video,plugdev,netdev -s /usr/bin/bash -c 'Crunchy Taco' crunchy

~# passwd crunchy
New password: 
Retype new password: 
passwd: password updated successfully
```

I will create the 'grubx64.efi'

```
grub-mkstandalone -O x86_64-efi \
--modules="bli all_video boot btrfs cat chain configfile echo efifwsetup efinet ext2 fat font gettext gfxmenu gfxterm gfxterm_background gzio halt help hfsplus iso9660 jpeg keystatus loadenv loopback linux ls lsefi lsefimmap lsefisystab lssal memdisk minicmd normal ntfs part_apple part_msdos part_gpt password_pbkdf2 png probe reboot regexp search search_fs_uuid search_fs_file search_label serial sleep smbios squash4 test tpm true video xfs zfs zfscrypt zfsinfo cpuid play cryptodisk gcry_arcfour gcry_blowfish gcry_camellia gcry_cast5 gcry_crc gcry_des gcry_dsa gcry_idea gcry_md4 gcry_md5 gcry_rfc2268 gcry_rijndael gcry_rmd160 gcry_rsa gcry_seed gcry_serpent gcry_sha1 gcry_sha256 gcry_sha512 gcry_tiger gcry_twofish gcry_whirlpool luks luks2 lvm mdraid09 mdraid1x raid5rec raid6rec" \
--output="/live-boot-grub/grubx64.efi" \
"boot/grub/grub.cfg=/live-temp/grub-embed.cfg"
```
I will logout of chroot and then make a squashfs. 
```
logout 
```
### From the Host

I will create the squashfs needed for the live build. 


```
mksquashfs \
"$chrt" \
"$CR/staging/live/filesystem.squashfs" \
-no-xattrs \
-e boot
```
I will create the 'filesystem.manifest'
```
arch-chroot $chrt dpkg-query -W --showformat='${Package} ${Version}\n' > $CR/staging/live/filesystem.manifest
```
I will create a new grub config file. 


```
cat <<'EOF' > "$CR/staging/boot/grub/grub.cfg"
load_video
insmod gzio
insmod part_gpt
insmod cryptodisk
insmod luks2
insmod gcry_rijndael
insmod gcry_rijndael
insmod gcry_sha256
insmod lvm
insmod ext2
insmod part_gpt
insmod part_msdos
insmod fat
insmod iso9660
insmod all_video
insmod font
insmod part_gpt
insmod part_msdos
insmod fat
insmod iso9660
insmod all_video
insmod font

set default="0"
set timeout=30

# If X has issues finding screens, experiment with/without nomodeset.

menuentry "Crunchy Live" {
    search --no-floppy --set=root --label CRUNCHY
    linux ($root)/live/vmlinuz boot=live components quiet splash noeject
    initrd ($root)/live/initrd
}

menuentry "Debian Live [EFI/GRUB] (nomodeset)" {
    search --no-floppy --set=root --label CRUNCHY
    linux ($root)/live/vmlinuz boot=live nomodeset
    initrd ($root)/live/initrd
}
EOF
```

I will create file for wrong cmdpath

```
cat <<'EOF' >> "$CR/tmp/grub-embed.cfg"
if ! [ -d "$cmdpath" ]; then
    # On some firmware, GRUB has a wrong cmdpath when booted from an optical disc.
    # https://gitlab.archlinux.org/archlinux/archiso/-/issues/183
    if regexp --set=1:isodevice '^(\([^)]+\))\/?[Ee][Ff][Ii]\/[Bb][Oo][Oo][Tt]\/?$' "$cmdpath"; then
        cmdpath="${isodevice}/EFI/BOOT"
    fi
fi
configfile "${cmdpath}/grub.cfg"
EOF
```
I will copy over the grub config file and modules

```
cp "$CR/staging/boot/grub/grub.cfg" "$CR/staging/EFI/BOOT/"

cp -r $chrt/usr/lib/grub/x86_64-efi* "$CR/staging/boot/grub/x86_64-efi/"
```

Copy over vmlinuz and initrd.

```
cp "$chrt/boot"/vmlinuz-* \
    "$CR/staging/live/vmlinuz" && \
cp "$chrt/boot"/initrd.img-* \
    "$CR/staging/live/initrd"
```

Copy over the shim, mok manager. 


```
cp $chrt/usr/lib/shim/shimx64.efi.signed $CR/staging/EFI/BOOT/bootx64.efi

cp $chrt/usr/lib/shim/mmx64.efi.signed $CR/staging/EFI/BOOT/mmx64.efi
```

Create the efiboot.img that will be the EFI partition for the ISO.


```
(cd "$CR/staging/boot" && \
    dd if=/dev/zero of=efiboot.img bs=1M count=20 && \
    mkfs.fat efiboot.img && \
    mmd -i efiboot.img ::/EFI ::/EFI/BOOT && \
    mcopy -vi efiboot.img \
        "$chrt/live-boot-grub/grubx64.efi" \
        "$CR/staging/EFI/BOOT/bootx64.efi" \
        "$CR/staging/EFI/BOOT/mmx64.efi" \
        "$CR/staging/boot/grub/grub.cfg" \
        ::/EFI/BOOT/
)
```

Finally, it is time to create the ISO


```
xorriso \
-as mkisofs -R -r -J -joliet-long -l \
-iso-level 3 \
-cache-inodes \
-o "$CR/debian-custom-sid.iso" \
-full-iso9660-filenames \
-A "Crunch Sid" \
-volid "CRUNCHY" \
--mbr-force-bootable -partition_offset 16 \
-rational-rock \
-eltorito-alt-boot \
--efi-boot boot/efiboot.img \
-e --interval:appended_partition_2:all:: \
-no-emul-boot \
-isohybrid-gpt-basdat \
-append_partition 2 0x01 $CR/staging/boot/efiboot.img \
"$CR/staging"
```

Now you have your own custom live boot ISO.


