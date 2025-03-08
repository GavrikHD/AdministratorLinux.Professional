Обновление ядра

Проверка текущей версии ядра
$ uname -r
6.8.0-54-generic

Просмотр репозитория на наличие новых версий ядра https://kernel.ubuntu.com/mainline

Последняя версия 6.13.5: https://kernel.ubuntu.com/mainline/v6.13.5/

Создаем папку куда будем скачивать новое ядро

mkdir kernel && cd kernel

wget https://kernel.ubuntu.com/mainline/v6.13.5/amd64/linux-headers-6.13.5-061305-generic_6.13.5-061305.202502271338_amd64.deb

wget https://kernel.ubuntu.com/mainline/v6.13.5/amd64/linux-headers-6.13.5-061305_6.13.5-061305.202502271338_all.deb

wget https://kernel.ubuntu.com/mainline/v6.13.5/amd64/linux-image-unsigned-6.13.5-061305-generic_6.13.5-061305.202502271338_amd64.deb

wget https://kernel.ubuntu.com/mainline/v6.13.5/amd64/linux-modules-6.13.5-061305-generic_6.13.5-061305.202502271338_amd64.deb

Устанавливаем все пакеты сразу:

sudo dpkg -i *.deb 

Проверяем, что ядро появилось в /boot.

$ ls -al /boot

total 192372

drwxr-xr-x  4 root root     4096 Mar  8 09:26 .

drwxr-xr-x 23 root root     4096 Mar  4 11:19 ..

-rw-------  1 root root 10067508 Feb 27 13:38 System.map-6.13.5-061305-generic

-rw-------  1 root root  9080742 Feb  7 21:09 System.map-6.8.0-54-generic

-rw-r--r--  1 root root   310720 Feb 27 13:38 config-6.13.5-061305-generic

-rw-r--r--  1 root root   287562 Feb  7 21:09 config-6.8.0-54-generic

drwxr-xr-x  5 root root     4096 Mar  8 09:26 grub

lrwxrwxrwx  1 root root       32 Mar  8 09:26 initrd.img -> initrd.img-6.13.5-061305-generic

-rw-r--r--  1 root root 78180034 Mar  8 09:26 initrd.img-6.13.5-061305-generic


-rw-r--r--  1 root root 68189723 Mar  4 11:19 initrd.img-6.8.0-54-generic

lrwxrwxrwx  1 root root       27 Mar  4 11:19 initrd.img.old -> initrd.img-6.8.0-54-generic

drwx------  2 root root    16384 Mar  4 09:29 lost+found

lrwxrwxrwx  1 root root       29 Mar  8 09:26 vmlinuz -> vmlinuz-6.13.5-061305-generic

-rw-------  1 root root 15847936 Feb 27 13:38 vmlinuz-6.13.5-061305-generic

-rw-------  1 root root 14985608 Feb  7 22:01 vmlinuz-6.8.0-54-generic

lrwxrwxrwx  1 root root       24 Mar  4 11:19 vmlinuz.old -> vmlinuz-6.8.0-54-generic

Обновить конфигурацию загрузчика:

sudo update-grub

Sourcing file `/etc/default/grub'

Generating grub configuration file ...

Found linux image: /boot/vmlinuz-6.13.5-061305-generic

Found initrd image: /boot/initrd.img-6.13.5-061305-generic

Found linux image: /boot/vmlinuz-6.8.0-54-generic

Found initrd image: /boot/initrd.img-6.8.0-54-generic

Warning: os-prober will not be executed to detect other bootable partitions.

Systems on them will not be added to the GRUB boot configuration.

Check GRUB_DISABLE_OS_PROBER documentation entry.

Adding boot menu entry for UEFI Firmware Settings ...

done

Выбрать загрузку нового ядра по-умолчанию:

sudo grub-set-default 0


Перегружаем виртуальную машину

reboot

После перезагрузки проверяем версию ядра

uname -r 

6.13.5-061305-generic

На этом обновление ядра закончено.
