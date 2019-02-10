### Arch Linux Installation (UEFI LVM - LUKS encryption)

1. Confirm you are booted into EFI mode by checking if there exist files in the `efivars` folder.
```
ls /sys/firmware/efi/efivars
```
2. Connect to the desired WiFi network.
  ```
  wifi-menu -o
  ```
3. Install vim if needed with Pacman.
```
sudo pacman -S vim
```
4. Setup the drive(s)
    <br>
    i. List all drives and their corresponding partitions. Take note of the drive you wish to partition.
    ```
    lsblk
    ```
    ii. Open the partition utility for the particular device. If you are wanting to partition the whole device just provide the device as the argument to `cgdisk`. For example, run `cgdisk /dev/sda` instead of `cgdisk /dev/sda1`.
    ```
    cgdisk <device-name>
    ```
    iii. You will want to delete any existing partitions on the device to free up all the space for Arch. A typical partition for an Arch Linux LVM boot drive looks like the below.
    |Partition Name |Size|File System Code|
    |-----|------|------|
    |Boot | 512M  | ef00|
    |LVM|remaining|8e00|
    If you wish to use additional hard drives in your system you should follow the same steps for them too. However, you will not need a boot partition. Just fill the whole thing with one LVM partition.
    <br>
    iv. We can make the boot filesystem now. Run the below to do so.
    ```
    mkfs.fat -F32 <boot-partition-name>
    ```
    v. Next, encrypt the LVM partition. Note that when you are asked to confirm that you want to encrypt the drive, you will need to type YES in caps. You will be asked to set a password for decrypting the partition in the future.
    ```
    cryptsetup -y -v luksFormat --type luks2 <lvm-partition-name>
    ```
    vi. In order to work with the LVM partition we will now need to decrypt and mount it, giving the mounted partition a name. I usually call it `archlv`. You can verify this was done correctly by checking there exists a file `/dev/mapper/<lvm-mount-name>`.
    ```
    cryptsetup open <lvm-partition-name> <lvm-mount-name>
    ```
    vii. Create a physical volume for the logical volume. You will need to pass in the previosuly created LVM mount path as the argument.
    ```
    pvcreate /dev/mapper/<lv-mount-name>
    ```
    viii. Now, create a volume group to contain our logical volumes. This will need a name as well. I usually call it `archvg`.
    ```
    vgcreate <vg-name> /dev/mapper/<lv-mount-name>
    ```
    ix. Create the logical volumes. Typical sizes for each volume are outlined in the table below.
    |Volume Name |Size|
    |-----|------|
    |swap | 1.5x RAM size|
    |root|30G|
    |home|remaining|
    ```
    lvcreate -L<1.5xRAM> <vg-name> -n swap
    lvcreate -L30G <vg-name> -n root
    lvcreate -l 100%FREE <vg-name> -n home
    ```
    For any other drives, you can do the same thing. You won't need a swap and root partition. Keep it as one big logical volume, or split it up into multiple.
    <br>
    x. Make the file systems for each logical volume and set up the swap volume.
    ```
    mkfs.ext4 /dev/mapper/<vg-name>-root
    mkfs.ext4 /dev/mapper/<vg-name>-home
    mkswap /dev/mapper/<vg-name>-swap
    swapon /dev/mapper/<vg-name>-swap
    ```
    For addiitonal data drives, it's usually best to use ext4. Follow the same structure for the root and home volumes. You can verify the swap was setup correctly by checking that there exists swap memory when running `free -m`.
    <br>
    xi. Now we must mount the logical volumes.
    ```
    mkdir /mnt/boot/
    mkdir /mnt/home
    
    mount /dev/mapper/<vg-name>-root /mnt
    mount /dev/mapper/<vg-name>-boot /mnt/boot
    mount /dev/mapper/<vg-name>-home /mnt/home
    ```
5. Install and setup `reflector` to setup the mirrors for a faster install.
```
sudo pacman -S reflector
reflector -c "<country-name>" -f 12 -l 5 --verbose --sort --save /etc/pacman.d/mirrorlist
```

6. Install Arch. Make sure to include `base` and `base_devel` for basic packages, sudo for sudo use, as well as `dialog` and `wpa_supplicant` for WiFi access.
```
pacstrap /mnt base base-devel sudo dialog wpa_supplicant
```

7. After the install we can generate out `fstab` file.
```
genfstab -p -U /mnt >> /mnt/etc/fstab
```

8. After that we can change root into our newly setup Arch install.
```
arch-chroot /mnt
```
9. Now, let's create a symlink for our timezone in `/etc/localtime`.
```
ln -sf /usr/share/zoneinfo/<country-name>/<city-name> /etc/localtime
```
10. Set the hardware clock from the system clock.
```
hwclock --systohc
```
11. Set the root password.
```
passwd
```
12. Install vim on the system. In this case I install `gvim` for clipboard support, which the regular `vim` package doesn't have.
```
pacman -S gvim
```
13. Set the correct language by opening the `locale.gen` file and uncommenting. For example uncomment `en_US.UTF-8 UTF-8` if you want US English.
```
vim /etc/locale.gen
```
14. Write the output of the `locale` command to `/etc/locale.conf`.
```
locale > /etc/locale/conf
```
15. Run locale-gen to generate the locale settings setup earlier.
```
locale-gen
```
16. Write your desired hostname to `/etc/hostname`
```
echo "<hostname> > /etc/hostname"
```
26. Setup your hosts file, modifying the below template to suit your needs.
```
vim /etc/hosts
```
```
# /etc/hosts

127.0.0.1   localhost
::1         localhost
127.0.1.1   <hostname>.localdomain   <hostname>
```
27. Edit `/etc/mkinitcpio.conf` so that the `initramfs` image supports our encrypted system. The hooks section should look like the below.
```
vim /etc/mkinitcpio.conf
```
```
# /etc/mkinitcpio.conf

...

HOOKS=(base udev autodetect modconf block keyboard encrypt lvm2 filesystems fsck)
...

```
28. Make the `initramfs` image.
```
mkinitcpio -p linux
```
29. Install `systemd-boot` into the boot partition.
```
bootctl install --path=/boot/
```
30. Setup the bootloader configuration file. We use `editor 0` here to disable editing the kernel parameters at boot. With the kernel parameters editor enabled, the user can add `init=/bin/bash` as a parameter to bypass the root password and gain root access.
```
vim /boot/loader/loader.conf
```
```
# /boot/loader/loader.conf

default Arch
timeout 5 # leave this out for no boot menu
editor 0
```
31. Find the UUID of the LVM partition on the boot drive.
```
blkid <lvm-partition-name>
# example: blkid /dev/sda1
```

32. Add the arch boot entry using the the UUID from the previous step. The `resume` option is optional here. It sets the swap location so that the kernel can resume from disk, thus allowing hibernation.

```
vim /boot/loader/entries/arch.conf
```
```
# /boot/loader/entries/arch.conf

title   Arch Linux
linux   /vmlinuz-linux
initrd  /intel-ucode.img
initrd  /initramfs-linux.img
options cryptdevice=UUID=<lvm-partition-UUID>:<lvm-mount-name> root=/dev/mapper/<vg-name>-root resume=/dev/mapper/<vg-name>-swap quiet rw
```
33. Install `intel-ucode` for Intel microcode updates.
```
sudo pacman -S intel-ucode
```
34. Exit the Arch installation.
```
exit
```
35. Unmount the drives.
```
umount -R /mnt
```
36. Reboot into the new Arch installation.
```
reboot
```
