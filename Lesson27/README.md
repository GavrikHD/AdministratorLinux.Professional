# Резервное копирование

Разворачиваем две виртуальные машины с именами backup и client

На обеих вм установливаем borgbackup

>apt install borgbackup

В сервер Backup монтируем диск на 2 Гб
Сначала определим UUID или имя диска

>lsblk -f

<pre>
NAME      FSTYPE      FSVER    LABEL UUID                                   FSAVAIL FSUSE% MOUNTPOINTS
loop0     squashfs    4.0                                                         0   100% /snap/core20/2599
loop1     squashfs    4.0                                                         0   100% /snap/core20/2318
loop2     squashfs    4.0                                                         0   100% /snap/snapd/24792
loop3     squashfs    4.0                                                         0   100% /snap/lxd/31333
loop4     squashfs    4.0                                                         0   100% /snap/lxd/29351
loop5     squashfs    4.0                                                         0   100% /snap/snapd/21759
sda                                                                                        
├─sda1                                                                                     
├─sda2    ext4        1.0            b977d100-af7b-4b3e-852f-2d9cbc1ad665      1.7G     7% /boot
└─sda3    LVM2_member LVM2 001       K13nJk-R1eY-YOdt-k2eB-zOge-DjQ4-d1RO92                
  └─ubuntu--vg-ubuntu--lv
          ext4        1.0            2d6e36da-e695-43e4-87bd-f2bd05624444     23.9G    16% /
sdb </pre>

Создадим раздел и файловую систему на sdb

>sudo fdisk /dev/sdb

<pre><span style="color:#26A269">Welcome to fdisk (util-linux 2.37.2).</span>
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0x3d2dc3ff.

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1): 
First sector (2048-4211815, default 2048): 
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-4211815, default 4211815): 

Created a new partition 1 of type &apos;Linux&apos; and of size 2 GiB.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
</pre>

Cоздадим файловую систему

>sudo mkfs.ext4 /dev/sdb1

<pre>mke2fs 1.46.5 (30-Dec-2021)
Creating filesystem with 526221 4k blocks and 131648 inodes
Filesystem UUID: 160f8b16-5c07-4e20-a118-75c4180c16c9
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376, 294912

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done 
</pre>

Определим UUID нового раздела

>sudo blkid /dev/sdb1

<pre>/dev/sdb1: UUID=&quot;160f8b16-5c07-4e20-a118-75c4180c16c9&quot; BLOCK_SIZE=&quot;4096&quot; TYPE=&quot;ext4&quot; PARTUUID=&quot;3d2dc3ff-01&quot;</pre>

Создадим точку монтирования

>sudo mkdir /mnt/backup

Откроем файл  /etc/fstab

>nano /etc/fstab

и добавим строку

UUID=160f8b16-5c07-4e20-a118-75c4180c16c9 /mnt/backup  ext4  defaults  0  2

Сохраним и перегрузимся

После перезагрузки создадим пользователя

>sudo adduser borg

<pre>Adding user `borg&apos; ...
Adding new group `borg&apos; (1001) ...
Adding new user `borg&apos; (1001) with group `borg&apos; ...
Creating home directory `/home/borg&apos; ...
Copying files from `/etc/skel&apos; ...
New password: 
Retype new password: 
passwd: password updated successfully
Changing the user information for borg
Enter the new value, or press ENTER for the default
	Full Name []: Backup User
	Room Number []: 
	Work Phone []: 
	Home Phone []: 
	Other []: 
Is the information correct? [Y/n] y</pre>

Дадим пользователю права на примонтированный диск

>sudo chown borg:borg /mnt/backup/

Создаем каталог ~/.ssh/authorized_keys в каталоге /home/borg

>su - borg

>mkdir .ssh

>touch .ssh/authorized_keys

>chmod 700 .ssh

>chmod 600 .ssh/authorized_keys

Переходим на сервер client

>vagrant ssh client

создаем ключи 

>ssh-keygen

<pre>Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/id_rsa
Your public key has been saved in /root/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:CTmO6to1Qq8MJB5qdz79Y5tS+qWF5YDH/+H8JwAPL+A root@client
The key&apos;s randomart image is:
+---[RSA 3072]----+
|                 |
|       .         |
|      +          |
|     o o+.o      |
|.o. . .oS= *     |
|=..o    E.B +    |
|ooo.+.. o. * o   |
|.=.+oo + o= + o .|
|..=  .. =*o  +.o.|
+----[SHA256]-----+
</pre>

копируем ключ на сервер backup

>ssh-copy-id borg@192.168.56.160

Инициализируем репозиторий borg на backup сервере с client сервера:

>borg init --encryption=repokey borg@192.168.56.160:/mnt/backup/

<pre>borg@192.168.56.160&apos;s password: 
Enter new passphrase: 
Enter same passphrase again: 
Do you want your passphrase to be displayed for verification? [yN]: Y
Your passphrase (between double-quotes): &quot;Backup&quot;
Make sure the passphrase displayed above is exactly what you wanted.

By default repositories initialized with this version will produce security
errors if written to with an older version (up to and including Borg 1.0.8).

If you want to use these older versions, you can disable the check by running:
borg upgrade --disable-tam ssh://borg@192.168.56.160/mnt/backup

See https://borgbackup.readthedocs.io/en/stable/changes.html#pre-1-0-9-manifest-spoofing-vulnerability for details about the security implications.

IMPORTANT: you will need both KEY AND PASSPHRASE to access this repo!
If you used a repokey mode, the key is stored in the repo, but you should back it up separately.
Use &quot;borg key export&quot; to export the key, optionally in printable format.
Write down the passphrase. Store both at safe place(s).
</pre>

Запускаем для проверки создания бэкапа

>borg create --stats --list borg@192.168.56.160:/mnt/backup/::"etc-{now:%Y-%m-%d_%H:%M:%S}" /etc

<pre>...
A /etc/group
A /etc/passwd
d /etc
------------------------------------------------------------------------------
Repository: ssh://borg@192.168.56.160/mnt/backup
Archive name: etc-2025-07-13_13:24:58
Archive fingerprint: c260726289d4cf7bcf1b661e623787ce6fc9d884196bf31e22d6c44616bb9229
Time (start): Sun, 2025-07-13 13:25:19
Time (end):   Sun, 2025-07-13 13:25:20
Duration: 0.98 seconds
Number of files: 799
Utilization of max. archive size: 0%
------------------------------------------------------------------------------
                       Original size      Compressed size    Deduplicated size
This archive:                2.28 MB              1.03 MB            990.70 kB
All archives:                2.28 MB              1.03 MB              1.07 MB

                       Unique chunks         Total chunks
Chunk index:                     750                  787
------------------------------------------------------------------------------
</pre>

Проверяем:

>borg list borg@192.168.56.160:/mnt/backup/

<pre>Enter passphrase for key ssh://borg@192.168.56.160/mnt/backup: 
etc-2025-07-13_13:24:58              Sun, 2025-07-13 13:25:19 [c260726289d4cf7bcf1b661e623787ce6fc9d884196bf31e22d6c44616bb9229]
</pre>

Смотрим список файлов

>borg list borg@192.168.56.160:/mnt/backup/::etc-2025-07-13_13:24:58

<pre>...
-rw-r--r-- root   root       2068 Wed, 2022-03-23 13:52:38 etc/fonts/conf.avail/60-latin.conf
-rw-r--r-- root   root      10293 Wed, 2022-03-23 13:52:38 etc/fonts/conf.avail/65-fonts-persian.conf
-rw-r--r-- root   root        464 Wed, 2022-03-23 13:52:38 etc/fonts/conf.avail/65-khmer.conf
-rw-r--r-- root   root       8008 Wed, 2022-03-23 13:52:38 etc/fonts/conf.avail/65-nonlatin.conf
-rw-r--r-- root   root        847 Wed, 2022-03-23 13:52:38 etc/fonts/conf.avail/69-unifont.conf
-rw-r--r-- root   root        487 Wed, 2022-03-23 13:52:38 etc/fonts/conf.avail/70-force-bitmaps.conf
-rw-r--r-- root   root        487 Wed, 2022-03-23 13:52:38 etc/fonts/conf.avail/70-no-bitmaps.conf
-rw-r--r-- root   root         77 Wed, 2022-03-23 13:52:38 etc/fonts/conf.avail/70-yes-bitmaps.conf
-rw-r--r-- root   root        597 Wed, 2022-03-23 13:52:38 etc/fonts/conf.avail/80-delicious.conf
-rw-r--r-- root   root       1917 Wed, 2022-03-23 13:52:38 etc/fonts/conf.avail/90-synthetic.conf
drwxr-xr-x root   root          0 Thu, 2025-02-20 21:11:21 etc/fonts/conf.d
-rw-r--r-- root   root        978 Wed, 2022-03-23 13:52:38 etc/fonts/conf.d/README
-rw-r--r-- root   root       2808 Wed, 2022-03-23 13:52:38 etc/fonts/fonts.conf
-rw-r--r-- root   root        832 Thu, 2025-02-20 21:10:53 etc/group
-rw-r--r-- root   root       1881 Thu, 2025-02-20 21:10:53 etc/passwd
</pre>

Достаем файл из бекапа

>borg extract borg@192.168.56.160:/mnt/backup/::etc-2025-07-13_13:24:58 
 etc/hostname

Автоматизируем создание бэкапов с помощью systemd
Создаем сервис и таймер в каталоге /etc/systemd/system/

>nano /etc/systemd/system/borg-backup.service

со следующим содержимым

<pre>
[Unit]
Description=Borg Backup

[Service]
Type=oneshot

# Парольная фраза
Environment="BORG_PASSPHRASE=Backup"
# Репозиторий
Environment=REPO=borg@192.168.56.160:/mnt/backup/
# Что бэкапим
Environment=BACKUP_TARGET=/etc

# Создание бэкапа
ExecStart=/bin/borg create \
    --stats                \
    ${REPO}::etc-{now:%%Y-%%m-%%d_%%H:%%M:%%S} ${BACKUP_TARGET}

# Проверка бэкапа
ExecStart=/bin/borg check ${REPO}

# Очистка старых бэкапов
ExecStart=/bin/borg prune \
    --keep-daily  90      \
    --keep-monthly 12     \
    --keep-yearly  1       \
    ${REPO}
    </pre>

>nano /etc/systemd/system/borg-backup.timer

<pre>
[Unit]
Description=Borg Backup

[Timer]
OnUnitActiveSec=5min

[Install]
WantedBy=timers.target
</pre>

Включаем и запускаем службу таймера

>systemctl enable borg-backup.timer 

<pre>Created symlink /etc/systemd/system/timers.target.wants/borg-backup.timer → /etc/systemd/system/borg-backup.timer.
</pre>

>systemctl start borg-backup.timer

Проверяем работу таймера

systemctl list-timers --all

<pre><u style="text-decoration-style:single">NEXT                        LEFT          LAST                        PASSED       UNIT                         ACTIVATES                     </u>
Sun 2025-07-13 14:22:14 UTC 4min 23s left Sun 2025-07-13 14:17:14 UTC 36s ago      borg-backup.timer            borg-backup.service
Sun 2025-07-13 16:34:58 UTC 2h 17min left Sun 2025-07-13 04:14:01 UTC 10h ago      fwupd-refresh.timer          fwupd-refresh.service
Sun 2025-07-13 17:16:51 UTC 2h 59min left Sun 2025-07-13 02:46:22 UTC 11h ago      motd-news.timer              motd-news.service
Mon 2025-07-14 00:00:00 UTC 9h left       Sun 2025-07-13 00:00:01 UTC 14h ago      dpkg-db-backup.timer         dpkg-db-backup.service
Mon 2025-07-14 00:00:00 UTC 9h left       Sun 2025-07-13 00:00:01 UTC 14h ago      logrotate.timer              logrotate.service
Mon 2025-07-14 00:40:17 UTC 10h left      Sun 2025-07-13 11:01:00 UTC 3h 16min ago man-db.timer                 man-db.service
Mon 2025-07-14 01:36:37 UTC 11h left      Sat 2025-07-12 14:21:10 UTC 23h ago      fstrim.timer                 fstrim.service
Mon 2025-07-14 14:03:15 UTC 23h left      Sun 2025-07-13 14:03:15 UTC 14min ago    systemd-tmpfiles-clean.timer systemd-tmpfiles-clean.service
Sun 2025-07-20 03:10:36 UTC 6 days left   Sun 2025-07-13 03:10:10 UTC 11h ago      e2scrub_all.timer            e2scrub_all.service
n/a                         n/a           n/a                         n/a          apport-autoreport.timer      apport-autoreport.service
n/a                         n/a           n/a                         n/a          snapd.snap-repair.timer      snapd.snap-repair.service
n/a                         n/a           n/a                         n/a          ua-timer.timer               ua-timer.service
</pre>