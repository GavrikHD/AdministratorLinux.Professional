


# Уменьшить том под / до 8G

Подготовим временный том для / раздела:

>pvcreate /dev/sdb
  

<pre>Physical volume "/dev/sdb" successfully created.</pre>


>vgcreate vg_root /dev/sdb
 <pre>
Volume group "vg_root" successfully created</pre>

>lvcreate -n lv_root -l +100%FREE /dev/vg_root
<pre> Logical volume &quot;lv_root&quot; created.
</pre>

Создадим на нем файловую систему и смонтируем его, чтобы перенести туда данные:

>mkfs.ext4 /dev/vg_root/lv_root
<pre>mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 2620416 4k blocks and 655360 inodes
Filesystem UUID: 4a6857f6-1ecb-4fcc-a63e-9d161150d744
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done 
</pre>

>mount /dev/vg_root/lv_root /mnt

Устанавливаем программу rsync если ее нет
>apt install rsync

Этой командой копируем все данные с / раздела в /mnt:

>rsync -avxHAX --progress / /mnt/
<pre>…

sent 3,443,968,557 bytes  received 785,647 bytes  222,242,206.71 bytes/sec
total size is 3,444,501,261  speedup is 1.00</pre>

Проверить что скопировалось можно командой 
>ls /mnt.

Затем сконфигурируем grub для того, чтобы при старте перейти в новый /.
Сымитируем текущий root, сделаем в него chroot и обновим grub:

>for i in /proc/ /sys/ /dev/ /run/ /boot/; \
 do mount --bind $i /mnt/$i; done

>chroot /mnt/

>grub-mkconfig -o /boot/grub/grub.cfg
<pre>Sourcing file `/etc/default/grub'
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-6.8.0-31-generic
Found initrd image: /boot/initrd.img-6.8.0-31-generic
Warning: os-prober will not be executed to detect other bootable partitions.
Systems on them will not be added to the GRUB boot configuration.
Check GRUB_DISABLE_OS_PROBER documentation entry.
Adding boot menu entry for UEFI Firmware Settings ...</pre>

Обновим образ initrd. Что это такое и зачем нужно вы узнаете из следующей лекции.

>update-initramfs -u

Перезагружаемся, чтобы работать с новым разделом.
>reboot

Посмотрим картину с дисками после перезагрузки:

>lsblk
<pre>root@lvm:/home/vagrant# lsblk
NAME                      MAJ:MIN RM SIZE RO TYPE MOUNTPOINTS
sda                         8:0    0  64G  0 disk 
├─sda1                      8:1    0   1M  0 part 
├─sda2                      8:2    0   2G  0 part /boot
└─sda3                      8:3    0  62G  0 part 
  └─ubuntu--vg-ubuntu--lv 252:1    0  31G  0 lvm  
sdb                         8:16   0  10G  0 disk 
└─vg_root-lv_root         252:0    0  10G  0 lvm  /
sdc                         8:32   0   2G  0 disk 
sdd                         8:48   0   1G  0 disk 
sde                         8:64   0   1G  0 disk </pre>


Теперь нам нужно изменить размер старой VG и вернуть на него рут. Для этого удаляем старый LV размером в 40G и создаём новый на 8G:

>lvremove /dev/ubuntu-vg/ubuntu-lv
<pre>Do you really want to remove and DISCARD active logical volume ubuntu-vg/ubuntu-lv? [y/n]: y
  Logical volume "ubuntu-lv" successfully removed.</pre>

>lvcreate -n ubuntu-vg/ubuntu-lv -L 8G /dev/ubuntu-vg
<pre>WARNING: ext4 signature detected on /dev/ubuntu-vg/ubuntu-lv at offset 1080. Wipe it? [y/n]: y
  Wiping ext4 signature on /dev/ubuntu-vg/ubuntu-lv.
  Logical volume "ubuntu-lv" created.</pre>

Проделываем на нем те же операции, что и в первый раз:

>mkfs.ext4 /dev/ubuntu-vg/ubuntu-lv

>mount /dev/ubuntu-vg/ubuntu-lv /mnt

>rsync -avxHAX --progress / /mnt/
<pre>sent 3,469,398,892 bytes  received 785,702 bytes  210,314,217.82 bytes/sec
total size is 3,469,930,023  speedup is 1.00</pre>

Так же как в первый раз cконфигурируем grub.

>for i in /proc/ /sys/ /dev/ /run/ /boot/; \
 do mount --bind $i /mnt/$i; done
>chroot /mnt/
>grub-mkconfig -o /boot/grub/grub.cfg
<pre>Sourcing file `/etc/default/grub'
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-6.8.0-31-generic
Found initrd image: /boot/initrd.img-6.8.0-31-generic
Warning: os-prober will not be executed to detect other bootable partitions.
Systems on them will not be added to the GRUB boot configuration.
Check GRUB_DISABLE_OS_PROBER documentation entry.
Adding boot menu entry for UEFI Firmware Settings ...
done
</pre>
>update-initramfs -u
<pre>update-initramfs: Generating /boot/initrd.img-6.8.0-31-generic
W: Couldn't identify type of root file system for fsck hook</pre>

Пока не перезагружаемся и не выходим из под chroot - мы можем заодно перенести /var.

Выделить том под /var в зеркало

На свободных дисках создаем зеркало:

>pvcreate /dev/sdc /dev/sdd
 <pre> Physical volume "/dev/sdc" successfully created.
  Physical volume "/dev/sdd" successfully created.</pre>

>vgcreate vg_var /dev/sdc /dev/sdd
<pre>  Volume group "vg_var" successfully created</pre>

>lvcreate -L 950M -m1 -n lv_var vg_var
<pre>  Rounding up size to full physical extent 952.00 MiB
  Logical volume "lv_var" created.</pre>

Создаем на нем ФС и перемещаем туда /var:

>mkfs.ext4 /dev/vg_var/lv_var
<pre>mke2fs 1.42.9 (28-Dec-2013)
Writing superblocks and filesystem accounting information: done</pre>

>mount /dev/vg_var/lv_var /mnt
>cp -aR /var/* /mnt/

На всякий случай сохраняем содержимое старого var (или же можно его просто удалить):

>mkdir /tmp/oldvar && mv /var/* /tmp/oldvar

Ну и монтируем новый var в каталог /var:

>umount /mnt
>mount /dev/vg_var/lv_var /var

Правим fstab для автоматического монтирования /var:

>echo "`blkid | grep var: | awk '{print $2}'` \
 /var ext4 defaults 0 0" >> /etc/fstab

После чего можно успешно перезагружаться в новый (уменьшенный root) и удалять
временную Volume Group:

>lvremove /dev/vg_root/lv_root
<pre>Do you really want to remove and DISCARD active logical volume vg_root/lv_root? [y/n]: y
  Logical volume "lv_root" successfully removed.</pre>

>vgremove /dev/vg_root
<pre>  Volume group "vg_root" successfully removed</pre>

>pvremove /dev/sdb
<pre>  Labels on physical volume "/dev/sdb" successfully wiped.</pre>

# Выделить том под /home

Выделяем том под /home по тому же принципу что делали для /var:

>lvcreate -n LogVol_Home -L 2G /dev/ubuntu-vg
<pre>  Logical volume "LogVol_Home" created.</pre>

>mkfs.ext4 /dev/ubuntu-vg/LogVol_Home
>mount /dev/ubuntu-vg/LogVol_Home /mnt/
>cp -aR /home/* /mnt/
>rm -rf /home/*
>umount /mnt
>mount /dev/ubuntu-vg/LogVol_Home /home/

Правим fstab для автоматического монтирования /home:

>echo "`blkid | grep Home | awk '{print $2}'` \
 /home xfs defaults 0 0" >> /etc/fstab


# Работа со снапшотами

Генерируем файлы в /home/:

>touch /home/file{1..20}

Снять снапшот:

>lvcreate -L 100MB -s -n home_snap \
 /dev/ubuntu-vg/LogVol_Home

Удалить часть файлов:

>rm -f /home/file{11..20}

Процесс восстановления из снапшота:

>umount /home
>lvconvert --merge /dev/ubuntu-vg/home_snap
<pre>  Merging of volume ubuntu-vg/home_snap started.
  ubuntu-vg/LogVol_Home: Merged: 100.00%</pre>
>mount /dev/mapper/ubuntu--vg-LogVol_Home /home
>ls -al /home
<pre>total 28
drwxr-xr-x  4 root    root     4096 Dec 23 11:50 .
drwxr-xr-x 24 root    root     4096 Dec 23 09:49 ..
-rw-r--r--  1 root    root        0 Dec 23 11:50 file1
-rw-r--r--  1 root    root        0 Dec 23 11:50 file10
-rw-r--r--  1 root    root        0 Dec 23 11:50 file11
-rw-r--r--  1 root    root        0 Dec 23 11:50 file12
-rw-r--r--  1 root    root        0 Dec 23 11:50 file13
-rw-r--r--  1 root    root        0 Dec 23 11:50 file14
-rw-r--r--  1 root    root        0 Dec 23 11:50 file15</pre>
…
Файлы успешно восстановлены с помощью снапшота.

