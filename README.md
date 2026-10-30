# Arch Linux с RAID1

VM в QEMU: 2 ядра, 2 ГБ памяти, два диска virtio по 8 ГБ.
Сначала использовал SeaBIOS, затем переключил VM на UEFI с отдельным файлом переменных прошивки.
Диски в системе — `/dev/vda` и `/dev/vdb`.

На каждом диске выделил 1 МБ для BIOS GRUB, 512 МБ для ESP и остальное для RAID1.
Корень находится на массиве, ESP у дисков отдельные.
Ниже команды установки из Arch ISO, без archinstall:

```bash
timedatectl set-ntp true
printf 'Server = https://geo.mirror.pkgbuild.com/$repo/os/$arch\n' > /etc/pacman.d/mirrorlist
sed -i 's/^ParallelDownloads.*/ParallelDownloads = 1/' /etc/pacman.conf
for disk in /dev/vda /dev/vdb; do
    sfdisk "$disk" <<EOF
label: gpt
size=1M, type=21686148-6449-6E6F-744E-656564454649
size=512M, type=C12A7328-F81F-11D2-BA4B-00A0C93EC93B
type=A19D880F-05FC-4D3B-A006-743F0F84911E
EOF
done
udevadm settle
mdadm --create /dev/md0 --metadata=1.2 --level=1 --raid-devices=2 --run /dev/vda3 /dev/vdb3
mkfs.ext4 /dev/md0
mkfs.fat -F32 /dev/vda2
mkfs.fat -F32 /dev/vdb2
mount /dev/md0 /mnt
mkdir -p /mnt/boot/efi
mount /dev/vda2 /mnt/boot/efi
pacstrap -K /mnt base linux mdadm grub efibootmgr dhcpcd
genfstab -U /mnt > /mnt/etc/fstab
awk '$3 == "vfat" {$4 = $4 ",nofail,x-systemd.device-timeout=3"} {print}' /mnt/etc/fstab > /mnt/etc/fstab.new
mv /mnt/etc/fstab.new /mnt/etc/fstab
mdadm --detail --scan > /mnt/etc/mdadm.conf
printf 'arch-lab\n' > /mnt/etc/hostname
printf 'LANG=en_US.UTF-8\n' > /mnt/etc/locale.conf
sed -i 's/^#en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /mnt/etc/locale.gen
arch-chroot /mnt locale-gen
arch-chroot /mnt ln -sf /usr/share/zoneinfo/Asia/Novosibirsk /etc/localtime
cat > /mnt/etc/initcpio/install/raid1root <<'EOF'
build() {
    add_runscript
}
EOF
cat > /mnt/etc/initcpio/hooks/raid1root <<'EOF'
run_hook() {
    udevadm settle --timeout=10 || true
    mdadm --assemble --scan --run 2>/dev/null || true
    mdadm --run /dev/md0 2>/dev/null || true
}
EOF
sed -i 's/^HOOKS=.*/HOOKS=(base udev autodetect modconf block mdadm_udev raid1root filesystems fsck)/' /mnt/etc/mkinitcpio.conf
arch-chroot /mnt mkinitcpio -P
printf '\nGRUB_TERMINAL="console serial"\nGRUB_SERIAL_COMMAND="serial --speed=115200"\nGRUB_CMDLINE_LINUX="console=tty0 console=ttyS0,115200"\n' >> /mnt/etc/default/grub
arch-chroot /mnt grub-install --target=i386-pc /dev/vda
arch-chroot /mnt grub-install --target=i386-pc /dev/vdb
arch-chroot /mnt grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ArchA --no-nvram
arch-chroot /mnt grub-install --target=x86_64-efi --efi-directory=/boot/efi --removable --no-nvram
mkdir -p /mnt/esp-b
mount /dev/vdb2 /mnt/esp-b
arch-chroot /mnt grub-install --target=x86_64-efi --efi-directory=/esp-b --bootloader-id=ArchB --no-nvram
arch-chroot /mnt grub-install --target=x86_64-efi --efi-directory=/esp-b --removable --no-nvram
arch-chroot /mnt grub-mkconfig -o /boot/grub/grub.cfg
arch-chroot /mnt systemctl enable dhcpcd
mkdir -p /mnt/etc/systemd/system/serial-getty@ttyS0.service.d
printf '[Service]\nExecStart=\nExecStart=-/sbin/agetty --autologin root --noclear %%I $TERM\n' > /mnt/etc/systemd/system/serial-getty@ttyS0.service.d/lab.conf
sync
mdadm --wait /dev/md0 || true
cat /proc/mdstat
grep -q '\[UU\]' /proc/mdstat
umount /mnt/esp-b
umount -R /mnt
echo LAB_INSTALL_COMPLETE
poweroff
```

У ESP добавил `nofail`, чтобы отсутствие первого диска не останавливало загрузку.
Хук `raid1root` запускает массив, даже когда доступен только один его участник.

После проверки BIOS переключил VM на UEFI. Загрузился через `EFI/BOOT/BOOTX64.EFI`
и добавил записи для обоих ESP:

```bash
efibootmgr -c -d /dev/vda -p 2 -L ArchA -l '\EFI\ArchA\grubx64.efi'
efibootmgr -c -d /dev/vdb -p 2 -L ArchB -l '\EFI\ArchB\grubx64.efi'
efibootmgr -v
cat /proc/mdstat
```

Затем выключил VM и отключил первый диск. Система загрузилась со второго,
массив показал `[_U]`. После возвращения первого диска выполнил:

```bash
mdadm --manage /dev/md0 --add /dev/vda3
watch -n 1 cat /proc/mdstat
```

Дождался `[UU]`. Также проверил загрузку без второго диска: состояние `[U_]`.
После его возвращения добавил `/dev/vdb3` и снова дождался `[UU]`.
Выводы проверок находятся в [checks.txt](checks.txt).
