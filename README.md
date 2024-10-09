
# stuck-in-reboot-blackarch-linux


> ## "Reboot Into Firmware Interface" instead of "Arch Linux" on boot manager menu

**1. Boot into a Live Environment :** 
the easiest way to troubleshoot is to use an Arch Linux live USB environment. This allows you to access your filesystem without restrictions.
**2. Boot from the Live USB :**
Insert the live USB and boot your computer from it. You may need to change the boot order in your BIOS/UEFI settings to boot from the USB.
**3. Open a Terminal :**
Use `lsblk` to identify your partitions. Look for your encrypted root partition and the EFI partition.

    lsblk

**4. Unlock the Encrypted Partition :**
If your root partition is encrypted, you will need to unlock it:

    sudo cryptsetup luksOpen /dev/sdXn decrypted_root

 **5. Mount the Partitions :**
 Create directories to mount your root and EFI partitions:

     sudo mkdir -p /mnt/root 
     sudo mkdir -p /mnt/efi

Mount your root partition:

    sudo mount /dev/mapper/decrypted_root /mnt/root

Mount the EFI Partition:

    sudo mount /dev/sdXZ /mnt/efi

 **6. Change Root into Your Installed System :**
 Change root into your installed Arch Linux system:

    sudo arch-chroot /mnt/root

 **7. Reinstall GRUB and Update Configuration :**
 If you are using UEFI:

    grub-install --target=x86_64-efi --efi-directory=/mnt/efi --bootloader-id=GRUB

If you’re using a BIOS system:

    grub-install --target=i386-pc /dev/sdX
Finally, generate the GRUB configuration file:

    grub-mkconfig -o /boot/grub/grub.cfg

## 

> ## `erorr: attempt to install to encrypted disk without cryptodisk enabled`

this erorr ndicates that GRUB is trying to install to an encrypted disk without being configured to handle encryption. GRUB needs to be configured with `cryptodisk` support to boot from encrypted partitions

## Enable `GRUB_ENABLE_CRYPTODISK` in the GRUB Configuration File :
**1. Open the GRUB Configuration File :**

    nano /etc/default/grub

**2. Add the Following Line :**

    GRUB_ENABLE_CRYPTODISK=y

**Save and Exit then try to reinstall GRUB and Regenerate the GRUB Configuration File**

> ## `efi bootmgr not found`


The error `efi bootmgr not found` typically means that the system is unable to locate the necessary bootloader files in the EFI partition, or that the boot entries for GRUB are missing or improperly configured in the UEFI firmware. 
 **1. Ensure the EFI Partition is Correctly Mounted :**

     ls /mnt/efi

 **2. Check and Recreate the EFI Boot Entry :**
 

    efibootmgr

**3. Create the GRUB Boot Entry :**

    efibootmgr --create --disk /dev/sda --part 1 --label "GRUB" --loader /EFI/GRUB/grubx64.efi

-   `--disk /dev/sda`: This is the disk where your EFI partition resides.
-   `--part 1`: This is the partition number of your EFI partition.
-   `--label "GRUB"`: A name for your boot entry.
-   `--loader /EFI/GRUB/grubx64.efi`: Path to the GRUB binary in the EFI partition. Make sure this path is correct, as it should point to the EFI GRUB binary (`grubx64.efi`).

**4. Verify Boot Entries :**

    efibootmgr

> ##  os-prober will not be executed to detect other bootable partition  systems on them will not be added to the grub boot

The warning you're seeing is informing you that `os-prober` is not running automatically to detect other operating systems on your system (e.g., if you had a dual-boot setup with another OS like Windows). This is a common security measure in newer versions of GRUB, and it doesn’t directly affect your Arch Linux boot, but it could affect multi-boot configurations
**1. Enable `os-prober` in the GRUB Configuration :**

    nano /etc/default/grub

    GRUB_DISABLE_OS_PROBER=false

This tells GRUB to run `os-prober` during the boot configuration process.

**2. Save and Exit then Re-generate the GRUB Configuration File**

    grub-mkconfig -o /boot/grub/grub.cfg

> ## adding boot menu for EFI firmware settings only
this means that GRUB is not detecting an actual bootable operating system, and it is only adding an option to boot into the EFI firmware settings (BIOS/UEFI setup). This typically occurs if:

1.  The GRUB installation is incomplete or misconfigured.
2.  The root partition is not correctly detected by GRUB.
3.  GRUB is not finding the kernel and initramfs files (e.g., `vmlinuz-linux` and `initramfs-linux.img`).

 **1. Ensure the Kernel and Initramfs Files are in the Correct Location :**
Check if your kernel (`vmlinuz-linux`) and `initramfs` files are properly located in the `/boot` directory.

    ls /boot
You should see something like:

> vmlinuz-linux 
> initramfs-linux.img 
> initramfs-linux-fallback.img

If these files are missing, you need to reinstall the `linux` package to get them back:

    pacman -S linux

 **2. Check Your GRUB Configuration :**

     nano /boot/grub/grub.cfg
**Look for Entries**: Ensure that there's a valid entry for Arch Linux, which should point to the correct files, like so:

     menuentry 'Arch Linux' {
     load_video
    set gfxpayload=keep
    insmod gzio
    insmod part_gpt
    insmod ext2
    set root='hdX,gptY' # Ensure this points to the correct root partition
    linux /vmlinuz-linux root=UUID=<your_root_partition_UUID> rw initrd /initramfs-linux.img
    }

-   Replace `hdX,gptY` with the correct partition.
    
-   Replace `<your_root_partition_UUID>` with the UUID of your root partition.

**Save and Exit then try to reinstall GRUB and Regenerate the GRUB Configuration File**

  
> ## "invalid passphrase" 
 >##  "disk cryptouuid/root UUID not found"
 GRUB is not able to decrypt your root partition This usually happens if GRUB isn't properly set up to handle LUKS-encrypted partitions
 **1. Enable GRUB Cryptodisk Support :**

      nano /etc/default/grub

Add the following line if it's missing, or modify it if needed:

    GRUB_ENABLE_CRYPTODISK=y
 **2. Rebuild the GRUB Configuration :**

     grub-mkconfig -o /boot/grub/grub.cfg
    grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB

 **3. Verify Correct UUID for Root Partition :**
  Run `blkid` and verify the UUID for your root partition, which should be encrypted (`crypto_LUKS`).
  

    blkid

 Open `/boot/grub/grub.cfg` and verify that the UUID for your root partition is correctly referenced:

>  menuentry 'Arch Linux' {   
>  ... 
>  cryptomount -u <your_crypto_UUID> # UUID for the LUKS partition  
> set root='cryptouuid/<your_root_UUID>' #UUID for the decrypted partition
>  linux /vmlinuz-linuxroot=UUID=<your_root_partition_UUID> 
>  rw initrd /initramfs-linux.img 
>  }

Make sure both the `cryptomount` and `root=UUID` options are using the correct UUIDs.

 **4. Update the Initramfs :**

     nano /etc/mkinitcpio.conf

Ensure that the `encrypt` hook is included in the `HOOKS` array before `filesystems`. For example:

> `HOOKS=(base udev autodetect modconf block encrypt filesystems
> keyboard fsck)`

Rebuild the Initramfs:

    mkinitcpio -p linux

 **Reboot and Test**
> ## "vmlinuz-linux is not recognized as an internal command"
The error indicates that the UEFI Shell is not able to directly run the kernel file (`vmlinuz-linux`) as an executable. This can occur if the UEFI Shell can't access or locate the file correctly, or if the UEFI shell lacks support for directly executing ELF (Executable and Linkable Format) binaries like the Linux kernel.
**1. List Available Filesystems :**

    map -r
This will list all the available file systems like `FS0:`, `FS1:`, etc.
**2. Switch to the Correct Filesystem :**

    FS0:
If `FS0:` doesn’t contain the kernel, try other file systems like `FS1:`, etc., until you locate the partition containing `/vmlinuz-linux` and `/initramfs-linux.img`.
**3. Navigate to the Directory Containing the Kernel :**

    ls
    cd <directory>
  
   **4. Correct the File Paths :**
   
Ensure the paths to `vmlinuz-linux` and `initramfs-linux.img` are correct.

-   If the `vmlinuz-linux` and `initramfs-linux.img` are located in a subdirectory (like `/boot`), adjust your command accordingly:

> \boot\vmlinuz-linux root=PARTUUID= 'your-partuuid' rw
> initrd=\boot\initramfs-linux.img

**5. Ensure Secure Boot is Disabled**
-   Reboot your system and enter the BIOS/UEFI settings.
-   Look for an option to disable **Secure Boot** (usually under "Security" or "Boot" settings).
-   After disabling Secure Boot, try booting into the UEFI shell again and retry launching the kernel.

**6. Use `bootx64.efi` Instead of Kernel Directly :** 
In many cases, UEFI Shell will not allow you to run the kernel (`vmlinuz-linux`) directly because it’s not a standard EFI executable. Instead, you need to run a boot loader like GRUB or `systemd-boot`.

- You should have an EFI binary in the EFI partition, such as `grubx64.efi` or `bootx64.efi`. You can navigate to it from the UEFI Shell and run it directly:

 

>    FS0:\EFI\GRUB\grubx64.efi

or

> FS0:\EFI\BOOT\bootx64.efi
