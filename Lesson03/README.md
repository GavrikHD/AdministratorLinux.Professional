# Файловые системы и LVM-1

посмотрим какие диски есть у нас в системе
>lsblk
<pre>NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                         8:0    0 11.2G  0 disk 
├─sda1                      8:1    0    1M  0 part 
├─sda2                      8:2    0  1.8G  0 part /boot
└─sda3                      8:3    0  9.4G  0 part 
  └─ubuntu--vg-ubuntu--lv 252:0    0  9.4G  0 lvm  /
sdb                         8:16   0   10G  0 disk 
sdc                         8:32   0    2G  0 disk 
sdd                         8:48   0    1G  0 disk 
sde                         8:64   0    1G  0 disk 
sr0                        11:0    1 1024M  0 rom  
root@u24:~# 
</pre>

>lvmdiskscan

<pre>  /dev/sda2 [       1.75 GiB] 
  /dev/sda3 [      &lt;9.45 GiB] LVM physical volume
  /dev/sdb  [      10.00 GiB] 
  /dev/sdc  [       2.00 GiB] 
  /dev/sdd  [       1.00 GiB] 
  /dev/sde  [       1.00 GiB] 
  4 disks
  1 partition
  0 LVM physical volume whole disks
  1 LVM physical volume
</pre>

разметим диск для будущего использования LVM. Создадим PV

>pvcreate /dev/sdb

<pre>Physical volume &quot;/dev/sdb&quot; successfully created.
</pre>
создаем первый уровень абстракции - VG
>vgcreate vg01 /dev/sdb
<pre>Volume group &quot;vg01&quot; successfully created</pre>
Создаем логический диск - LV
>lvcreate -l+80%FREE -n lv01 vg01
<pre>Logical volume &quot;lv01&quot; created</pre>
информация о созданном Volume Group
>vgdisplay vg01
<pre>--- Volume group ---
  VG Name               vg01
  System ID             
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  2
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                1
  Open LV               0
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               &lt;10.00 GiB
  PE Size               4.00 MiB
  Total PE              2559
  Alloc PE / Size       2047 / &lt;8.00 GiB
  Free  PE / Size       512 / 2.00 GiB
  VG UUID               jEJ5hi-swsY-TKoe-hYMc-22xy-q7Yc-oi5GHU
</pre>
Смотрим какие диски входят в VG
>vgdisplay -v vg01 | grep 'PV Name'
<pre> <span style="color:#C01C28"><b>PV Name</b></span>               /dev/sdb  </pre>
посмотрим информаицю  о созданном Logical Volume
>lvdisplay /dev/vg01/lv01
<pre>--- Logical volume ---
  LV Path                /dev/vg01/lv01
  LV Name                lv01
  VG Name                vg01
  LV UUID                E7nLhs-QftW-dxQ8-tE6p-kna7-16AK-uPHKl1
  LV Write Access        read/write
  LV Creation host, time u24, 2025-03-17 06:21:02 +0000
  LV Status              available
  # open                 0
  LV Size                &lt;8.00 GiB
  Current LE             2047
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           252:1
</pre>
>vgs
<pre>VG        #PV #LV #SN Attr   VSize   VFree
  ubuntu-vg   1   1   0 wz--n-  &lt;9.45g    0 
  vg01        1   1   0 wz--n- &lt;10.00g 2.00g
</pre>
>lvs
<pre>LV        VG        Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  ubuntu-lv ubuntu-vg -wi-ao---- &lt;9.45g                                                    
  lv01      vg01      -wi-a----- &lt;8.00g        </pre>
  Cоздаем еще один LV из свободного места с абсолютным значением в мегабайтах:
lvcreate -L150M -n small vg01
<pre>Rounding up size to full physical extent 152.00 MiB
  Logical volume &quot;small&quot; created.
</pre>
>lvs
<pre>LV        VG        Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  ubuntu-lv ubuntu-vg -wi-ao----  &lt;9.45g                                                    
  lv01      vg01      -wi-a-----  &lt;8.00g                                                    
  small     vg01      -wi-a----- 152.00m       </pre>
  Создадим на LV файловую систему и смонтируем его
>mkfs.ext4 /dev/vg01/lv01
<pre>mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 2096128 4k blocks and 524288 inodes
Filesystem UUID: 74d6723e-20ae-4061-a41f-ac14a9a0afd3
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done 
</pre>
>mkdir /data
>mount /dev/vg01/lv01 /data/
>mount | grep /data
<pre>/dev/mapper/vg01-lv01 on <span style="color:#C01C28"><b>/data</b></span> type ext4 (rw,relatime)
</pre>
Что бы добавить свободного места в директории /data. Мы можем расширить файловую систему на LV /dev/vg01/lv01 за счет нового блочного устройства /dev/sdc.
Для начала так же необходимо создать PV:
>pvcreate /dev/sdc
<pre>Physical volume &quot;/dev/sdc&quot; successfully created.</pre>
Далее необходимо расширить VG добавив в него этот диск.
>vgextend vg01 /dev/sdc
<pre>Volume group &quot;vg01&quot; successfully extended
</pre>
>vgdisplay -v vg01 | grep 'PV Name'
<pre><span style="color:#C01C28"><b>PV Name</b></span>               /dev/sdb     
<span style="color:#C01C28"><b>PV Name</b></span>               /dev/sdc   </pre>
И место в VG прибавилось
>vgs
<pre>VG        #PV #LV #SN Attr   VSize  VFree 
  ubuntu-vg   1   1   0 wz--n- &lt;9.45g     0 
  vg01        2   2   0 wz--n- 11.99g &lt;3.85g</pre>

