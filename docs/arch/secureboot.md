# Secure boot

Enable secure boot on both Windows and Linux.

Install `sbctl` from AUR.

```
$ yay -S sbctl
```

Checking current status

```
# sbctl status
```

### Prepare

1. Restart the computer and enter the BIOS. Clear the Secure Boot keys.
2. Create a new key pair:

    ```
    # sbctl create-keys
    ```

3. Enroll the keys with vendor keys:

    ```
    # sbctl enroll-keys -m
    ```

### Signing EFI binaries


####  Check what files need to be signed

```
# sbctl verify
```

#### sign the files

```
# sbctl sign -s /boot/vmlinuz-linux
# sbctl sign -s /boot/EFI/BOOT/BOOTX64.EFI
```

Sign multi files at once:

```
# sbctl verify | sed 's/✗ /sbctl sign -s /e'
```

Output for successfully signed and boot ins secure boot mode example:

```
# sbctl status

Installed:      ✓ sbctl is installed
Owner GUID:	    cd873a3e-3b18-4b70-89aa-422620c183db
Setup Mode:	    ✓ Disabled
Secure Boot:    ✓ Enabled
Vendor Keys:    microsoft
```

### Notes

Autosign with packman hooks with systemd boot

quote from [ArchWiki](https://wiki.archlinux.org/index.php/Secure_Boot#Automatic_signing_with_the_pacman_hook):

> Tip: If you use Systemd-boot and systemd-boot-update.service, the boot loader is only updated after a reboot, and the sbctl pacman hook will therefore not sign the new file. As a workaround, it can be useful to sign the boot loader directly in /usr/lib/, as bootctl install and update will automatically recognize and copy .efi.signed files to the ESP if present, instead of the normal .efi file. See bootctl(1).


```
# sbctl sign -s -o /usr/lib/systemd/boot/efi/systemd-bootx64.efi.signed /usr/lib/systemd/boot/efi/systemd-bootx64.efi
```
