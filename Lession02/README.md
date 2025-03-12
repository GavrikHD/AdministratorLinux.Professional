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
Создадим в каждом разделе по файлу для проерки сохраности файлов
```htm
for i in $(seq 1 5); do touch /raid/part$i/test.txt
```
# Сломать и починить RAID
Попробуем пометить один из дисков как fail 
```htm
mdadm /dev/md0 --fail /dev/sdc
```
>mdadm: set /dev/sdc faulty in /dev/md0

проверим состояние райда
```htm
cat /proc/mdstat
```
<pre>Personalities : [linear] [raid0] [raid1] [raid6] [raid5] [raid4] [raid10] 
md0 : active raid10 sde[3] sdd[2] sdc[1](F) sdb[0]
      2093056 blocks super 1.2 512K chunks 2 near-copies [4/3] [U_UU]
      
unused devices: &lt;none&gt;
</pre>

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

       Update Time : Wed Mar 12 14:01:42 2025
             State : clean, degraded 
    Active Devices : 3
   Working Devices : 3
    Failed Devices : 1
     Spare Devices : 0

            Layout : near=2
        Chunk Size : 512K

Consistency Policy : resync

              Name : u24:0  (local to host u24)
              UUID : fefa8612:102c3c57:c8791931:2830b571
            Events : 19

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync set-A   /dev/sdb
       -       0        0        1      removed
       2       8       48        2      active sync set-A   /dev/sdd
       3       8       64        3      active sync set-B   /dev/sde

       1       8       32        -      faulty   /dev/sdc
</pre>

Пробуем удалить "плохой" диск из массива:
```htm
mdadm /dev/md0 --remove /dev/sdс
```
но получаем ошибку
```htm
mdadm: stat failed for /dev/sdс: No such file or directory
```
Добовляем новый диск в райд
```htm
mdadm /dev/md0 --add /dev/sdf
```
>mdadm: added /dev/sdf

райд собрался, но диск остался в райд. Для того что бы его убрать, сначала размотируем фс
```htm
for i in $(seq 1 5); do umount /raid/part$i; done
```
потом останавливаем райд
```htm
mdadm --stop /dev/md0
```
>mdadm: stopped /dev/md0

```htm
mdadm --zero-superblock --force /dev/sdc
```
>mdadm: Unrecognised md component device - /dev/sdc

Заново создем райд10, только уже с другим диском
```htm
mdadm --create --verbose /dev/md0 -l 10 -n 4 /dev/sd{b,d,e,f}
```
<pre>mdadm: layout defaults to n2
mdadm: layout defaults to n2
mdadm: chunk size defaults to 512K
mdadm: /dev/sdb appears to be part of a raid array:
       level=raid10 devices=4 ctime=Wed Mar 12 09:25:40 2025
mdadm: /dev/sdd appears to be part of a raid array:
       level=raid10 devices=4 ctime=Wed Mar 12 09:25:40 2025
mdadm: /dev/sde appears to be part of a raid array:
       level=raid10 devices=4 ctime=Wed Mar 12 09:25:40 2025
mdadm: /dev/sdf appears to be part of a raid array:
       level=raid10 devices=4 ctime=Wed Mar 12 09:25:40 2025
mdadm: size set to 1046528K
Continue creating array? yes
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md0 started.
</pre>
Монтируем обратно фс
```htm
for i in $(seq 1 5); do mount /dev/md0p$i /raid/part$i; done
```
проверяем наши файлы