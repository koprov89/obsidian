---
tags:
  - linux
  - arch
  - guide
  - virtualization
date: 2026-01-13
---

# Arch Linux Installation

Берем iso пишем на флешку и тд

## Установка

#### Подключаемся к вафле
```
iwctl
station wlan0 connect BerlogaFilial5g
exit
```

#### Запускаем изи установщик
```
archinstall
```
Там всё плюс-минус понятно, так что распишу только disk configuration
Будем использовать manual. Нужно 2 раздела - EFI и остальное

EFI:
```
Размер: 1GiB
filesystem: FAT32
mountpoint: /efi
flags: esp, boot
```
Здесь важно что mountpoint именно /efi
Мы хотим чтобы директория /boot была частью корневой ФС, а EFI была отдельно. Это нужно для того чтобы в снэпшоты btrfs были включены ядра и прочее содержимое /boot

root:
```
Размер: 
filesystem: btrfs
mountpoint: /
mark as compressed
```

Subolumes btrfs:
```
@       /
@home   /home
@opt    /opt
@srv    /srv
@cache  /var/cache
@images /var/lib/libvirt/images  - виртуалки
@log    /var/log
@spool  /var/spool       - email и прочее, хз з0ачем но пусть будет
@tmp    /var/tmp
```

Чтобы сохранить конфигурацию перед установкой и потом если что ее повторить, можем сделать save configuretion.
Нажимаем install. 
## Шаги после установки перед ребутом
После завершения - chroot into installation for post-installation configurations

#### Виртуалки copy-on-write
Для того чтобы механика btrfs copy on write не конфликтовала с аналогичной в виртуальных машинах qcow, отключим фичу copy on write для конкретной директории где эти машины хранятся
```
chattr -VR +C /var/lib/libvirt/images/
```

#### Магия с загрузчиком
Мы не хотим чтобы директория /grub была в разделе /efi, который не является частью btrfs. Хотим чтобы она была в /boot, которая является
Удаляем /grub из /efi
```
rm -rf /efi/grub
```

Идем в `/etc/default/grub` и раскомментируем строку `GRUB_ENABLE_CRYPTODISK=y`

```
grub-install --target=x86_64-efi --efi-directory=/efi --boot-directory=/boot --bootloader-id=arch
```

Генерируем grub.cfg:
```
grub-mkconfig -o /boot/grub/grub.cfg
```

#### Управление efibootmgr

Показать чокак
```
efibootmgr
```

Удалить запись
```
efibootmgr -b 0001 -B
```

Добавить запись
```
efibootmgr -c -d /dev/sda -p 1 -L "NAME" -l "\EFI\arch\grub64.efi"
```

## Настройка после загрузки

Команды btrfs:
```
lsblk -pf /dev/sda
btrfs filesystem label / ARCH
btrfs filesystem show /
btrfs filesystem usage /
btrfs subvolume list /
```

Ставим yay:
```
git clone https://aur.archlinux.org/yay-git.git
cd yay-git
makepg -si
```

Ставим пакеты
```
sudo pacman -S snapper snap-pac grub-btrfs inotify-tools
yay -S btrfs-assistant
```

Делаем конфиги snapper под subvolumes
```
sudo snapper -c root create-config /
sudo snapper -c home create-config /home
```

Управление Snapper без sudo:
```
sudo snapper -c root set-config ALLOW_USERS="$USER" SYNC_ACL=yes
sudo snapper -c home set-config ALLOW_USERS="$USER" SYNC_ACL=yes
```

Чтобы locate не лагал добавляем в `/etc/updatedb.conf` в PRUNENAMES `.snapshots`

В `/etc/mkinitcpio.conf` добавляем в HOOKS `grub-btrfs-overlayfs`
```
sudo mkinitcpio -P
```

```
sudo systemctl enable --now grub-btrfsd.service
```

## Работа со snapper
Список снэпшотов
```
snapper -c root ls
```

Изменения между двумя снэпшотами
```
snapper status 1..2
```

Откат изменений между двумя снэпшотами
```
sudo snapper undochange 1..2
```

Изменения в файле
```
snapper diff 2..0 /etc/resolv.conf
```

Откат изменений в конкретном файле
```
sudo snapper undochange 2..0 /etc/resolv.conf
```

Создание ручного снэпшота перед и после изменений
```
snapper -c root create -t pre -c number -d 'pre change'
snapper -c root create -t post --pre-number 3 -c number -d 'post change '
```