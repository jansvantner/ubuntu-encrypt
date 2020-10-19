# ubuntu-encrypt
based on https://help.ubuntu.com/community/Full_Disk_Encryption_Howto_2019

```bash
 mount | grep efivars
 sudo -i
 lsblk
 export DEV="/dev/nvme0n1"
 export DM="${DEV##*/}"
 export DM="${DM}$( if [[ "$DM" =~ "nvme" ]]; then echo "p"; fi )"
```
```bash
 sgdisk --print $DEV
 # !!! this deletes all partitions !!!
 sgdisk --zap-all $DEV
```
```bash
 sgdisk --new=1:0:+768M $DEV
 sgdisk --new=2:0:+2M $DEV
 sgdisk --new=3:0:+128M $DEV
 sgdisk --new=5:0:0 $DEV
 sgdisk --typecode=1:8301 --typecode=2:ef02 --typecode=3:ef00 --typecode=5:8301 $DEV
 sgdisk --change-name=1:/boot --change-name=2:GRUB --change-name=3:EFI-SP --change-name=5:rootfs $DEV
 sgdisk --hybrid 1:2:3 $DEV

 sgdisk --print $DEV
```
```bash
 cryptsetup luksFormat --type=luks1 ${DEV}p1
 cryptsetup luksFormat ${DEV}p5
 cryptsetup open ${DEV}p1 LUKS_BOOT
 cryptsetup open ${DEV}p5 ${DM}5_crypt
 ls /dev/mapper/
```
```bash
mkfs.ext4 -L boot /dev/mapper/LUKS_BOOT
mkfs.vfat -F 16 -n EFI-SP ${DEV}p3
pvcreate /dev/mapper/${DM}5_crypt
vgcreate ubuntu-vg /dev/mapper/${DM}5_crypt
lvcreate -L 4G -n swap_1 ubuntu-vg
lvcreate -l 80%FREE -n root ubuntu-vg
```
```bash
echo "GRUB_ENABLE_CRYPTODISK=y" >> /target/etc/default/grub
```
```bash
 mount /dev/mapper/ubuntu--vg-root /target
 for n in proc sys dev etc/resolv.conf; do mount --rbind /$n /target/$n; done 
 chroot /target

 mount -a
 apt install -y cryptsetup-initramfs
 
 echo "KEYFILE_PATTERN=/etc/luks/*.keyfile" >> /etc/cryptsetup-initramfs/conf-hook 
 echo "UMASK=0077" >> /etc/initramfs-tools/initramfs.conf 
 
 mkdir /etc/luks
 dd if=/dev/urandom of=/etc/luks/boot_os.keyfile bs=4096 count=1
 
 chmod u=rx,go-rwx /etc/luks
 chmod u=r,go-rwx /etc/luks/boot_os.keyfile
 cryptsetup luksAddKey ${DEV}p1 /etc/luks/boot_os.keyfile 
 cryptsetup luksAddKey ${DEV}p5 /etc/luks/boot_os.keyfile 
 
 echo "LUKS_BOOT UUID=$(blkid -s UUID -o value ${DEV}p1) /etc/luks/boot_os.keyfile luks,discard" >> /etc/crypttab
 echo "${DM}5_crypt UUID=$(blkid -s UUID -o value ${DEV}p5) /etc/luks/boot_os.keyfile luks,discard" >> /etc/crypttab
 
 update-initramfs -u -k all
```
