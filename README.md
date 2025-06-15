# Yet another Bitpixie PoC

Another PoC for Bitpixie, a vulnerability discovered by Rairii in August 2022 which can be used to decrypt and mount partitions encrypted with default Bitlocker Device Encryption.

Many thanks to th0mas from Neodyme for his brilliant articles on the matter. See https://neodyme.io/en/blog/bitlocker_screwed_without_a_screwdriver and https://neodyme.io/en/blog/bitlocker_why_no_fix/.

This repo also features an exploit for CVE-2024-1086 on the Linux kernel, which was written by notselwyn.

Take a look at the repos from [Andreas Grasser](https://github.com/andigandhi/bitpixie) and [Marc André Tanner](https://github.com/martanne/bitpixie), who made their PoCs public before I did and probably put more thoughts into it.

## Exploit

- Put `create_modded_bcd.bat` into a USB stick which you'll plug into the target device (or use an SMB share if USB is completely disabled).
- Get a Windows Recovery shell. Can be achieved by pressing Shift while clicking "restart" from the Windows login screen, or by typing `shutdown /r /o /t 0` in a terminal if you're logged in.
- Once in the recovery shell, get into your USB disk and execute the script ("e:" is wherever your USB disk was mounted):  
    ```
    e:
    create_modded_bcd.bat
    ```
- Then, move the newly created `BCD` file from your USB disk (or SMB share) into the `TFTP-root/Boot/` folder on your Linux attacking device.
- Now plug a LAN cable between your device and the target and configure a quick LAN network with TFTP support (`$ABS_TFTP_ROOT` corresponds to the TFTP-root folder of this repo):
    ```
    sudo dnsmasq --no-daemon \
    --interface="$INTERFACE" \
    --dhcp-range=10.13.37.100,10.13.37.101,255.255.255.0,1h \
    --dhcp-boot=bootmgfw.efi \
    --enable-tftp \
    --tftp-root="$ABS_TFTP_ROOT" \
    --log-dhcp
    ```
* Make sure your Ethernet interface is configured:
    ```
    sudo ifconfig $INTERFACE 10.13.37.1
    sudo ifconfig $INTERFACE up 
    ```
* Trigger PXE boot on the target device. Can generally be done in two ways:
    * From the recovery screen, select "use a device" and boot from something that looks like "IPv4 Network" (name can vary)
    * After powering on, press F12 (HP) or whichever key triggers PXE boot
* If everything goes well, the target device will load a grub after a few seconds (up to a minute maybe) and you can choose to boot into Alpine Linux
* Then log in as "root" when the initrd login prompt appears and do the following:
    ```
    # If you want an AZERTY keyboard
    loadkeys fr

    # Bitlocker partition generally is on /dev/nvme0n1p3 but it can vary
    ./pwn.sh /dev/nvme0n1p3
    ```
* Now the decrypted partition should be under mnt/, enjoy! You could use it to copy the SAM hives, modify exe paths for privesc, rename AV/EDR folders so they stop working, ...
* **Do not forget to umount**

## PXE server
 TFTP-root/ will be your serving TFTP folder for the PXE boot. You'll only need to copy paste the modified BCD file into Boot/, everything else should be left untouched.

There are 3 subfolders:
- **EFI/** : replicates the EFI boot partition. Generally contains a bunch of useless files, here I kept it simple and only left EFI/Microsoft/Boot/bootmgfw.efi, which is the only file needed
- **Boot/** : contains the BCD registry file that you crafted
- **grub/** : contains the grub configuration which loads the Linux initramfs and kernel

As for the files:
- **alpine-initrd.xz**: Alpine Linux filesystem with everything you need
- **bootmgfw.efi**: a vulnerable Windows boot file from before 2022. The first file that the target device will load during PXE boot
- **debian-kernel-514**: vulnerable kernel that allows us to read the VMK in memory
- **grubx64.efi**: grub boot file that loads the grub/grub.cfg and enables you to boot into Linux
- **shimx64.efi**: shim boot file that loads the grubx64.efi file

## Tips
* File names ARE case sensitive when booting via PXE, so if the target device is looking for BOOTMGFW.EFI instead of bootmgfw.efi, rename it as such.
* If you can't use a keyboard on the target device (which is likely if this isn't a laptop and keyboards are external devices), you can just use SSH to connect to it and run the exploit.
* You won't have write access to the Bitlocker partition if you haven't shut down the Windows client properly (Windows hibernation system). Make sure you don't hold the power button and force the shutdown. If you want to be safe, use `shutdown /s /t 0` (shutdown) or `shutdown /r /o /t 0` (reboot) or just click "shut down" or "restart" if you aren't logged in (and wait properly)
* Some hardware makes it so NVMe disks are not recognized sometimes. This can be due to Intel Volume Management Device (VMD) or other reasons. Using a different kernel version that is still vulnerable to CVE-2024-1086 has proven useful in some cases, e.g. kernel version 6.1.0-17 (which is included in this repo as a backup somewhere).
* If PXE is working fine but you're getting an empty blue screen, just press ESC to skip to the grub. Might indicate there is an issue somewhere though.
* If absolutely no 3rd party distribution is allowed to boot (the shimx64.efi file not properly going through PXE can be diagnostic of that) because Secure Boot 3rd party certificates are not allowed, you will have to read the memory from Windows instead. This project seems to have made some progress on a WinPE exploit: https://github.com/martanne/bitpixie. 
* On AMD hardware, disabling SVM virtualization in the BIOS might be necessary to boot Linux. If you cannot turn it off, see previous point for how to exploit from a Windows partition.

## Unexploitable cases
This particular exploit will not work in the following scenarios (but TPM sniffing and lesser-known software exploits might still do):
* A PIN or key file is required to unlock the drive.
* PXE boot is completely disabled and you also cannot boot via USB (otherwise you could try using a USB-Ethernet dongle). You could theoretically reset the UEFI from the motherboard if PXE is allowed by default, but that is risky.
* Any Bitlocker configuration (e.g., Bitlocker Legacy) where other PCRs than 7 & 11 are used (run the following command with admin privileges if you can, to verify: `manage-bde -protectors -get c:`)
* KB5025885 is installed (unlikely).
* The bootloader is signed with something else than the 2011 Secure Boot certificate (unlikely). If you're logged in you can directly check the signature of bootmgr.efi in C:\Windows\Boot\EFI\ or after mounting the EFI partition from the recovery shell (via the `mountvol s: /s` command).