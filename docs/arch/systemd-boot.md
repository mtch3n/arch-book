# Systemd boot

## TODO

- [ ] Automatically create boot entries. [systemd-boot Manager for Manjaro](https://gitlab.com/dalto.8/systemd-boot-manager)
- [ ] Unify wordings and proofread.

## Configuration

### Sample Config

/*esp*/loader/loader.conf

```
default @saved
timeout 3
console-mode max
```

/*esp*/loader/entries/arch.conf

```
title Arch Linux
linux /vmlinuz-linux
initrd /intel-ucode.img
initrd /initramfs-linux.img
options root=UUID=2729d3e6-4542-46fb-829d-44789d2670a1 rw rootflags=subvol=@active/root rootfstype=btrfs sysrq_always_enabled=1 nvidia_drm.modeset=1
```

or run this command via terminal.

> Replace `esp` and `/dev/nvme1n1p2` with your EFI partition path and root partition respectively.

```
cat <<EOF >> /esp/loader/entries/arch.conf
title Arch Linux
linux /vmlinuz-linux
initrd /intel-ucode.img
initrd /initramfs-linux.img
options root=UUID=$(blkid /dev/nvme1n1p2 -s UUID -o value) rw rootflags=subvol=@active/root rootfstype=btrfs sysrq_always_enabled=1 nvidia_drm.modeset=1
EOF

```

### Automatic update

Use `systemd-boot-pacman-hook` to automatically update systemd-boot after a kernel update.

### Save Entry

```
default @saved
...
```

### Boot Windows on other disk

> In order to detact disk correctly, `fastboot` should be disabled in BIOS.

1. Install `edk2-shell` and add the following entry to `/boot/loader/entries/windows.conf`:

    ```
    title Windows
    efi edk2-shellx64.efi
    ```

2. Copy `edk2-shellx64.efi` to `/boot`

    ```
    # cp /usr/share/edk2-shell/x64/Shell.efi /boot/edk2-shellx64.efi
    ```

3. Verify Windows EFI partition UUID

    ```
    # blkid | grep vfat
    ```

The disk alias has not verified yet. After create a new entry we can boot to EFI shell and verify the disk alias.

Reboot and boot to EFI shell. If shell prompt a countdown to run custom script, press any key to stop it.

If the alias is not shown, run `map` to list all block devices.

4. Identify the disk alias with UUID. Example output:

    ```
    fs0: Alias(s):HD1a0a:;BLK1: ...
    fs1: Alias(s):HD1c:;BLK2: ...
    ...
    ```

5. Modify `/boot/loader/entries/windows.conf`

    ```
    title Windows
    efi edk2-shellx64.efi
    options -nointerrupt -noconsoleout windows.nsh
    ```

6. Add the following to `/boot/windows.nsh`. Replace `HD1c` with the actual disk alias.
    > Windows EFI path usually is located at `EFI\Microsoft\Boot\Bootmgfw.efi`

    ```
    HD1c:EFI\Microsoft\Boot\Bootmgfw.efi
    ```