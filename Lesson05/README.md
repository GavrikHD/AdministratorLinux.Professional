# Определение алгоритма с наилучшим сжатием
>lsblk
<pre>NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                         8:0    0 11.2G  0 disk 
├─sda1                      8:1    0    1M  0 part 
├─sda2                      8:2    0  1.8G  0 part /boot
└─sda3                      8:3    0  9.4G  0 part 
  └─ubuntu--vg-ubuntu--lv 252:0    0  9.4G  0 lvm  /
sdb                         8:16   0  512M  0 disk 
sdc                         8:32   0  512M  0 disk 
sdd                         8:48   0  512M  0 disk 
sde                         8:64   0  512M  0 disk 
sdf                         8:80   0  512M  0 disk 
sdg                         8:96   0  512M  0 disk 
sdh                         8:112  0  512M  0 disk 
sdi                         8:128  0  512M  0 disk 
</pre>
Установим пакет утилит для ZFS:
>sudo apt install zfsutils-linux
Создаём четыре пула из двух дисков в режиме RAID 1:
>zpool create otus1 mirror /dev/sdb /dev/sdc

>zpool create otus2 mirror /dev/sdd /dev/sde

>zpool create otus3 mirror /dev/sdf /dev/sdg

>zpool create otus4 mirror /dev/sdh /dev/sdi
Смотрим информацию о пулах: zpool list
<pre>otus1   480M   111K   480M        -         -     0%     0%  1.00x    ONLINE  -
otus2   480M   111K   480M        -         -     0%     0%  1.00x    ONLINE  -
otus3   480M   111K   480M        -         -     0%     0%  1.00x    ONLINE  -
otus4   480M   111K   480M        -         -     0%     0%  1.00x    ONLINE  -
</pre>
Команда zpool status показывает информацию о каждом диске, состоянии сканирования и об ошибках чтения, записи и совпадения хэш-сумм. 
>zpool status
<pre>  pool: otus1
 state: ONLINE
config:

	NAME        STATE     READ WRITE CKSUM
	otus1       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdb     ONLINE       0     0     0
	    sdc     ONLINE       0     0     0

errors: No known data errors

  pool: otus2
 state: ONLINE
config:

	NAME        STATE     READ WRITE CKSUM
	otus2       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdd     ONLINE       0     0     0
	    sde     ONLINE       0     0     0

errors: No known data errors

  pool: otus3
 state: ONLINE
config:

	NAME        STATE     READ WRITE CKSUM
	otus3       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdf     ONLINE       0     0     0
	    sdg     ONLINE       0     0     0

errors: No known data errors

  pool: otus4
 state: ONLINE
config:

	NAME        STATE     READ WRITE CKSUM
	otus4       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdh     ONLINE       0     0     0
	    sdi     ONLINE       0     0     0

errors: No known data errors
</pre>
Добавим разные алгоритмы сжатия в каждую файловую систему:
>zfs set compression=lzjb otus1

>zfs set compression=lz4 otus2

>zfs set compression=gzip-9 otus3

>zfs set compression=zle otus4

Проверим, что все файловые системы имеют разные методы сжатия:
>zfs get all | grep compression
<pre>otus1  <span style="color:#C01C28"><b>compression</b></span>           lzjb                   local
otus2  <span style="color:#C01C28"><b>compression</b></span>           lz4                    local
otus3  <span style="color:#C01C28"><b>compression</b></span>           gzip-9                 local
otus4  <span style="color:#C01C28"><b>compression</b></span>           zle                    local
</pre>
Скачаем один и тот же текстовый файл во все пулы: 
>for i in {1..4}; do wget -P /otus$i https://gutenberg.org/cache/epub/2600/pg2600.converter.log; done

Проверим, что файл был скачан во все пулы:
>ls -l /otus*
<pre>/otus1:
total 22098
-rw-r--r-- 1 root root 41130189 Mar  2 08:31 pg2600.converter.log

/otus2:
total 18008
-rw-r--r-- 1 root root 41130189 Mar  2 08:31 pg2600.converter.log

/otus3:
total 10966
-rw-r--r-- 1 root root 41130189 Mar  2 08:31 pg2600.converter.log

/otus4:
total 40196
-rw-r--r-- 1 root root 41130189 Mar  2 08:31 pg2600.converter.log
</pre>
Уже на этом этапе видно, что самый оптимальный метод сжатия у нас используется в пуле otus3.

Проверим, сколько места занимает один и тот же файл в разных пулах и проверим степень сжатия файлов:
>zfs list
<pre>NAME    USED  AVAIL  REFER  MOUNTPOINT
otus1  21.8M   330M  21.6M  /otus1
otus2  17.8M   334M  17.6M  /otus2
otus3  10.9M   341M  10.7M  /otus3
otus4  39.4M   313M  39.3M  /otus4
</pre>
>zfs get all | grep compressratio | grep -v ref
<pre>otus1  compressratio         1.81x                  -
otus2  compressratio         2.23x                  -
otus3  compressratio         3.65x                  -
otus4  compressratio         1.00x                  -
</pre>
Таким образом, у нас получается, что алгоритм gzip-9 самый эффективный по сжатию. 

#  Определение настроек пула
Скачиваем архив в домашний каталог: 
>wget -O archive.tar.gz --no-check-certificate 'https://drive.usercontent.google.com/download?id=1MvrcEp-WgAQe57aDEzxSRalPAwbNN1Bb&export=download'
<pre>--2025-03-21 07:20:57--  https://drive.usercontent.google.com/download?id=1MvrcEp-WgAQe57aDEzxSRalPAwbNN1Bb&amp;export=download
Resolving drive.usercontent.google.com (drive.usercontent.google.com)... 2a00:1450:4010:c05::84, 64.233.163.132
Connecting to drive.usercontent.google.com (drive.usercontent.google.com)|2a00:1450:4010:c05::84|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 7275140 (6.9M) [application/octet-stream]
Saving to: ‘archive.tar.gz’

archive.tar.gz            100%[=====================================&gt;]   6.94M  1.57MB/s    in 4.4s    

2025-03-21 07:21:13 (1.57 MB/s) - ‘archive.tar.gz’ saved [7275140/7275140]
</pre>
Разархивируем его:
>tar -xzvf archive.tar.gz
<pre>zpoolexport/
zpoolexport/filea
zpoolexport/fileb
</pre>
Проверим, возможно ли импортировать данный каталог в пул:
>zpool import -d zpoolexport/
<pre> pool: otus
     id: 6554193320433390805
  state: ONLINE
status: Some supported features are not enabled on the pool.
	(Note that they may be intentionally disabled if the
	&apos;compatibility&apos; property is set.)
 action: The pool can be imported using its name or numeric identifier, though
	some features will not be available without an explicit &apos;zpool upgrade&apos;.
 config:

	otus                         ONLINE
	  mirror-0                   ONLINE
	    /root/zpoolexport/filea  ONLINE
	    /root/zpoolexport/fileb  ONLINE
</pre>
Данный вывод показывает нам имя пула, тип raid и его состав. 

Сделаем импорт данного пула к нам в ОС:

>zpool import -d zpoolexport/ otus

>zpool status
<pre>  pool: otus
 state: ONLINE
status: Some supported and requested features are not enabled on the pool.
	The pool can still be used, but some features are unavailable.
action: Enable all features using &apos;zpool upgrade&apos;. Once this is done,
	the pool may no longer be accessible by software that does not support
	the features. See zpool-features(7) for details.
config:

	NAME                         STATE     READ WRITE CKSUM
	otus                         ONLINE       0     0     0
	  mirror-0                   ONLINE       0     0     0
	    /root/zpoolexport/filea  ONLINE       0     0     0
	    /root/zpoolexport/fileb  ONLINE       0     0     0

errors: No known data errors

  pool: otus1
 state: ONLINE
config:

	NAME        STATE     READ WRITE CKSUM
	otus1       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdb     ONLINE       0     0     0
	    sdc     ONLINE       0     0     0

errors: No known data errors

  pool: otus2
 state: ONLINE
config:

	NAME        STATE     READ WRITE CKSUM
	otus2       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdd     ONLINE       0     0     0
	    sde     ONLINE       0     0     0

errors: No known data errors

  pool: otus3
 state: ONLINE
config:

	NAME        STATE     READ WRITE CKSUM
	otus3       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdf     ONLINE       0     0     0
	    sdg     ONLINE       0     0     0

errors: No known data errors

  pool: otus4
 state: ONLINE
config:

	NAME        STATE     READ WRITE CKSUM
	otus4       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdh     ONLINE       0     0     0
	    sdi     ONLINE       0     0     0

errors: No known data errors
</pre>
Команда zpool status выдаст нам информацию о составе импортированного пула.

Запрашиваем все параметры файловой системы

<pre>NAME  PROPERTY              VALUE                  SOURCE
otus  type                  filesystem             -
otus  creation              Fri May 15  4:00 2020  -
otus  used                  2.05M                  -
otus  available             350M                   -
otus  referenced            24K                    -
otus  compressratio         1.00x                  -
otus  mounted               yes                    -
otus  quota                 none                   default
otus  reservation           none                   default
otus  recordsize            128K                   local
otus  mountpoint            /otus                  default
otus  sharenfs              off                    default
otus  checksum              sha256                 local
otus  compression           zle                    local
otus  atime                 on                     default
otus  devices               on                     default
otus  exec                  on                     default
otus  setuid                on                     default
otus  readonly              off                    default
otus  zoned                 off                    default
otus  snapdir               hidden                 default
otus  aclmode               discard                default
otus  aclinherit            restricted             default
otus  createtxg             1                      -
otus  canmount              on                     default
otus  xattr                 on                     default
otus  copies                1                      default
otus  version               5                      -
otus  utf8only              off                    -
otus  normalization         none                   -
otus  casesensitivity       sensitive              -
otus  vscan                 off                    default
otus  nbmand                off                    default
otus  sharesmb              off                    default
otus  refquota              none                   default
otus  refreservation        none                   default
otus  guid                  14592242904030363272   -
otus  primarycache          all                    default
otus  secondarycache        all                    default
otus  usedbysnapshots       0B                     -
otus  usedbydataset         24K                    -
otus  usedbychildren        2.02M                  -
otus  usedbyrefreservation  0B                     -
otus  logbias               latency                default
otus  objsetid              54                     -
otus  dedup                 off                    default
otus  mlslabel              none                   default
otus  sync                  standard               default
otus  dnodesize             legacy                 default
otus  refcompressratio      1.00x                  -
otus  written               24K                    -
otus  logicalused           1024K                  -
otus  logicalreferenced     12K                    -
otus  volmode               default                default
otus  filesystem_limit      none                   default
otus  snapshot_limit        none                   default
otus  filesystem_count      none                   default
otus  snapshot_count        none                   default
otus  snapdev               hidden                 default
otus  acltype               off                    default
otus  context               none                   default
otus  fscontext             none                   default
otus  defcontext            none                   default
otus  rootcontext           none                   default
otus  relatime              on                     default
otus  redundant_metadata    all                    default
otus  overlay               on                     default
otus  encryption            off                    default
otus  keylocation           none                   default
otus  keyformat             none                   default
otus  pbkdf2iters           0                      default
otus  special_small_blocks  0                      default
</pre>



C помощью команды get можно уточнить конкретный параметр, например:

Размер: 
>zfs get available otus
<pre>NAME  PROPERTY   VALUE  SOURCE
otus  available  350M   -
</pre>
Тип: 
>zfs get readonly otus
<pre>NAME  PROPERTY  VALUE   SOURCE
otus  readonly  off     default
</pre>
По типу ФС мы можем понять, что позволяет выполнять чтение и запись
Значение recordsize: 
>zfs get recordsize otus
<pre>NAME  PROPERTY    VALUE    SOURCE
otus  recordsize  128K     local
</pre>
Тип сжатия: 
>zfs get compression otus
<pre>NAME  PROPERTY     VALUE           SOURCE
otus  compression  zle             local
</pre>
Тип контрольной суммы: 
>zfs get checksum otus
<pre>NAME  PROPERTY  VALUE      SOURCE
otus  checksum  sha256     local
</pre>

# Работа со снапшотом, поиск сообщения от преподавателя

Скачаем файл, указанный в задании:
>wget -O otus_task2.file --no-check-certificate https://drive.usercontent.google.com/download?id=1wgxjih8YZ-cqLqaZVa0lA3h3Y029c3oI&export=download

Восстановим файловую систему из снапшота:
>zfs receive otus/test@today < otus_task2.file

Далее, ищем в каталоге /otus/test файл с именем “secret_message”:

>find /otus/test -name "secret_message" 
<pre>/otus/test/task1/file_mess/secret_message
</pre>
Смотрим содержимое найденного файла:

>cat /otus/test/task1/file_mess/secret_message
<pre>https://otus.ru/lessons/linux-hl/
</pre>
