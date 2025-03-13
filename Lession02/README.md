# Дисковая подсистема
Добовляем в виртуальную машину пять дисков, размером по 1Gb

После запуска виртуальной машины проверяем наличие новых дисков в системе
```htm
lsblk
```
>NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS  
sda                         8:0    0 11.2G  0 disk  
├─sda1                      8:1    0    1M  0 part  
├─sda2                      8:2    0  1.8G  0 part /boot  
└─sda3                      8:3    0  9.4G  0 part   
  └─ubuntu--vg-ubuntu--lv 252:0    0  9.4G  0 lvm  /  
sdb                         8:16   0    1G  0 disk   
sdc                         8:32   0    1G  0 disk   
sdd                         8:48   0    1G  0 disk   
sde                         8:64   0    1G  0 disk   
sdf                         8:80   0    1G  0 disk   
sr0                        11:0    1 1024M  0 rom    

Занулим на всякий случай суперблоки:
```htm
mdadm --zero-superblock --force /dev/sd{b,c,d,e,f}
```
>mdadm: Unrecognised md component device - /dev/sdb  
mdadm: Unrecognised md component device - /dev/sdc   
mdadm: Unrecognised md component device - /dev/sdd  
mdadm: Unrecognised md component device - /dev/sde  
mdadm: Unrecognised md component device - /dev/sdf  

Создаем райд10 следующей командой
```htm
mdadm --create --verbose /dev/md0 -l 10 -n 4 /dev/sd{b,c,d,e}
```
>mdadm: layout defaults to n2  
mdadm: layout defaults to n2  
mdadm: chunk size defaults to 512K  
mdadm: size set to 1046528K  
mdadm: Defaulting to version 1.2 metadata  
mdadm: array /dev/md0 started.  

проверяем результат
```htm
cat /proc/mdstat
```
>Personalities : [linear] [raid0] [raid1] [raid6] [raid5] [raid4] [raid10]   
md0 : active raid10 sde[3] sdd[2] sdc[1] sdb[0]  
      2093056 blocks super 1.2 512K chunks 2 near-copies [4/4] [UUUU]

более в развернутом виде
```htm
mdadm -D /dev/md0
```
<pre>/dev/md0:
           Version : 1.2
     Creation Time : Wed Mar 12 09:25:40 2025
        Raid Level : raid10
        Array Size : 2093056 (2044.00 MiB 2143.29 MB)
     Used Dev Size : 1046528 (1022.00 MiB 1071.64 MB)
      Raid Devices : 4
     Total Devices : 4
       Persistence : Superblock is persistent

       Update Time : Wed Mar 12 09:25:51 2025
             State : clean 
    Active Devices : 4
   Working Devices : 4
    Failed Devices : 0
     Spare Devices : 0

            Layout : near=2
        Chunk Size : 512K

Consistency Policy : resync

              Name : u24:0  (local to host u24)
              UUID : fefa8612:102c3c57:c8791931:2830b571
            Events : 17
</pre>

# Создать таблицу, пять разделов и смонтировать их в системе

Устанавливаем программу parted
```htm
apt install parted
```
Создаем рздел GPT на RAID
```htm
parted -s /dev/md0 mklabel gpt
```
Создаем партиции
```htm
parted /dev/md0 mkpart primary ext4 0% 20%
```
получаем сообщение о необходимости обновить файл /etc/fstab, Продолжаем
```htm
parted /dev/md0 mkpart primary ext4 20% 40%
```
```htm
parted /dev/md0 mkpart primary ext4 40% 60%
```
```htm
parted /dev/md0 mkpart primary ext4 60% 80%
```
```htm
parted /dev/md0 mkpart primary ext4 80% 100%
```
Далее cоздаем на этих партициях файловую систему
```htm
for i in $(seq 1 5); do sudo mkfs.ext4 /dev/md0p$i; 
>done
```
создадим каталоги
```htm
mkdir -p /raid/part{1,2,3,4,5}
```
и примонтируем их в созданные папки
```htm
for i in $(seq 1 5); do mount /dev/md0p$i /raid/part$i; done
```
Создадим в каждом разделе по файлу для проверки сохраности файлов
```htm
for i in $(seq 1 5); do touch /raid/part$i/test.txt
```
# Сломать и починить RAID
Попробуем пометить один из дисков как fail 
```htm
mdadm /dev/md0 --fail /dev/sdb
```
>mdadm: set /dev/sdb faulty in /dev/md0

проверим состояние райда
```htm
cat /proc/mdstat
```
<pre>Personalities : [linear] [raid0] [raid1] [raid6] [raid5] [raid4] [raid10] 
md0 : active raid10 sdf[3] sde[2] sdd[1] sdb[0](F)
      2093056 blocks super 1.2 512K chunks 2 near-copies [4/3] [_UUU]
</pre>

```htm
mdadm -D /dev/md0
```
<pre>/dev/md0:
           Version : 1.2
     Creation Time : Wed Mar 12 15:23:24 2025
        Raid Level : raid10
        Array Size : 2093056 (2044.00 MiB 2143.29 MB)
     Used Dev Size : 1046528 (1022.00 MiB 1071.64 MB)
      Raid Devices : 4
     Total Devices : 4
       Persistence : Superblock is persistent

       Update Time : Thu Mar 13 12:10:38 2025
             State : clean, degraded 
    Active Devices : 3
   Working Devices : 3
    Failed Devices : 1
     Spare Devices : 0

            Layout : near=2
        Chunk Size : 512K

Consistency Policy : resync

              Name : u24:0  (local to host u24)
              UUID : 38eb9f5b:8f762859:40c39243:a39b51ed
            Events : 19

    Number   Major   Minor   RaidDevice State
       -       0        0        0      removed
       1       8       48        1      active sync set-B   /dev/sdd
       2       8       64        2      active sync set-A   /dev/sde
       3       8       80        3      active sync set-B   /dev/sdf

       0       8       16        -      faulty   /dev/sdb
</pre>

Удаляем "плохой" диск из массива:
```htm
mdadm /dev/md0 --remove /dev/sdb
```
>mdadm: hot removed /dev/sdb from /dev/md0

Добовляем новый диск в райд
```htm
mdadm /dev/md0 --add /dev/sdc
```
>mdadm: added /dev/sdf

Проверяем результат

<pre>/dev/md0:
           Version : 1.2
     Creation Time : Wed Mar 12 15:23:24 2025
        Raid Level : raid10
        Array Size : 2093056 (2044.00 MiB 2143.29 MB)
     Used Dev Size : 1046528 (1022.00 MiB 1071.64 MB)
      Raid Devices : 4
     Total Devices : 4
       Persistence : Superblock is persistent

       Update Time : Thu Mar 13 12:23:28 2025
             State : clean 
    Active Devices : 4
   Working Devices : 4
    Failed Devices : 0
     Spare Devices : 0

            Layout : near=2
        Chunk Size : 512K

Consistency Policy : resync

              Name : u24:0  (local to host u24)
              UUID : 38eb9f5b:8f762859:40c39243:a39b51ed
            Events : 39

    Number   Major   Minor   RaidDevice State
       4       8       32        0      active sync set-A   /dev/sdc
       1       8       48        1      active sync set-B   /dev/sdd
       2       8       64        2      active sync set-A   /dev/sde
       3       8       80        3      active sync set-B   /dev/sdf
</pre>

Проверяем, сохранились ли файлы на созданных и примонтированных фс
```htm
ls /raid/part1/
```
>lost+found  test.txt

файл на месте
