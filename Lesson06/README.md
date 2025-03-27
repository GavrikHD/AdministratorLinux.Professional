# Настраиваем сервер NFS
Создаём 2 виртуальные машины с сетевыми интерфейсами, которые позволяют связь между ними. 
Далее будем называть ВМ с NFS сервером nfss (IP 192.168.1.65), а ВМ с клиентом nfsc (IP 192.168.1.221).

Заходим на сервер c NFS-сервером.

Установим сервер NFS:
>apt install nfs-kernel-server

Проверяем наличие слушающих портов 2049/udp, 2049/tcp,111/udp, 111/tcp 

>ss -tnplu 

Создаём и настраиваем директорию, которая будет экспортирована в будущем 

>mkdir -p /srv/share/upload

>chown -R nobody:nogroup /srv/share

>chmod 0777 /srv/share/upload

Cоздаём в файле /etc/exports структуру, которая позволит экспортировать ранее созданную директорию:

>cat << EOF > /etc/exports 
/srv/share 192.168.1.61/24(rw,sync,root_squash)
EOF

Экспортируем ранее созданную директорию:
>exportfs -r
<pre>exportfs: /etc/exports [1]: Neither &apos;subtree_check&apos; or &apos;no_subtree_check&apos; specified for export &quot;192.168.1.61/24:/srv/share&quot;.
  Assuming default behaviour (&apos;no_subtree_check&apos;).
  NOTE: this default has changed since nfs-utils version 1.0.x
</pre>
Проверяем экспортированную директорию следующей командой

>exportfs -s
<pre>/srv/share  192.168.1.61/24(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)
</pre>

# Настраиваем клиент NFS 

Заходим на сервер с клиентом.

Установим пакет с NFS-клиентом
>sudo apt install nfs-common

Добавляем в /etc/fstab строку
>echo "192.168.1.221:/srv/share/ /mnt nfs vers=3,noauto,x-systemd.automount 0 0" >> /etc/fstab

и выполняем команды:
>systemctl daemon-reload
>systemctl restart remote-fs.target

Заходим в директорию /mnt/ и проверяем успешность монтирования:
>mount | grep mnt
<pre>systemd-1 on /<span style="color:#C01C28"><b>mnt</b></span> type autofs (rw,relatime,fd=63,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=4036)
192.168.1.221:/srv/share/ on /<span style="color:#C01C28"><b>mnt</b></span> type nfs (rw,relatime,vers=3,rsize=262144,wsize=262144,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,mountaddr=192.168.1.221,mountvers=3,mountport=54982,mountproto=udp,local_lock=none,addr=192.168.1.221)
</pre>

# Проверка работоспособности

Заходим на сервер. 

Заходим в каталог /srv/share/upload.
>cd /srv/share/upload/
Создаём тестовый файл
>touch check_file

Заходим на клиент.

Заходим в каталог /mnt/upload. 
>cd /mnt/upload

Проверяем наличие ранее созданного файла.
>ls
<pre>check_file</pre>
Создаём тестовый файл
>touch client_file

Проверяем, что файл успешно создан.
>ls
<pre>check_file  client_file</pre>

Предварительно проверяем клиент: 

перезагружаем и заходим  на клиент;

заходим в каталог /mnt/upload и проверяем наличие ранее созданных файлов.
>ls /mnt/upload/
<pre>check_file  client_file</pre>

Проверяем сервер: 

перегружаем сервер, заходим на него и проверяем наличие файлов в каталоге /srv/share/upload/
>ls /srv/share/upload/
<pre>check_file  client_file</pre>

проверяем экспорты 
>exportfs -s
<pre>/srv/share  192.168.1.61/24(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)
</pre>
проверяем работу RPC
>showmount -a 192.168.1.221
<pre>All mount points on 192.168.1.221:
192.168.1.65:/srv/share
</pre>

Проверяем клиент: 

перезагружаем клиент, заходим на него и проверяем работу RPC
>showmount -a 192.168.1.221
<pre>192.168.1.65:/srv/share</pre>
заходим в каталог /mnt/upload
>cd /mnt/upload
проверяем статус монтирования
>mount | grep mnt
<pre>systemd-1 on /<span style="color:#C01C28"><b>mnt</b></span> type autofs (rw,relatime,fd=57,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=4022)
192.168.1.221:/srv/share/ on /<span style="color:#C01C28"><b>mnt</b></span> type nfs (rw,relatime,vers=3,rsize=262144,wsize=262144,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,mountaddr=192.168.1.221,mountvers=3,mountport=50021,mountproto=udp,local_lock=none,addr=192.168.1.221)
</pre>
проверяем наличие ранее созданных файлов;
>ls
<pre>check_file  client_file</pre>
создаём тестовый файл
>touch final_check

проверяем, что файл успешно создан.
>ls
<pre>check_file  client_file  final_check</pre>

Все провперки пройдены, демонстрационный стенд работоспособен и готов к работе. 

# Создание скрипта для сервера и клиента

Заходим на сервер и создаем файл скрипта

>nano nfss_script.sh

вставляем в файл и запоминаем его

<pre>#!/bin/bash

# Установим сервер NFS:
apt-get -y install nfs-kernel-server

# Создаём и настраиваем директорию, которая будет экспортирована в будущем 

mkdir -p /srv/share/upload

chown -R nobody:nogroup /srv/share

chmod 0777 /srv/share/upload

# Cоздаём в файле /etc/exports структуру, которая позволит экспортировать ранее созданную директорию:

echo "/srv/share 192.168.1.61/24(rw,sync,root_squash)" >> /etc/exports

# Экспортируем ранее созданную директорию:

exportfs -r
</pre>

Запускаем скрипт
>. nfss_script.sh

Проверяем экспортированную директорию следующей командой
>exportfs -s
<pre>/srv/share  192.168.1.61/24(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)
</pre>

Заходим на сервер с клиентом и создаем файл скрипта

>nano nfsс_script.sh

вставляем в файл и запоминаем его

<pre>#!/bin/bash

# Установим пакет с NFS-клиентом
apt-get -y install nfs-common

# Добавляем в /etc/fstab строку
echo "192.168.1.221:/srv/share/ /mnt nfs vers=3,noauto,x-systemd.automount 0 0" >> /etc/fstab

# Обновляем и перезапускаем службу
systemctl daemon-reload
systemctl restart remote-fs.target
</pre>
Запускаем скрипт
>. nfsc_script.sh

Проверяем работу RPC
>showmount -a 192.168.1.221
<pre>192.168.1.65:/srv/share</pre>
заходим в каталог /mnt/upload
>cd /mnt/upload
проверяем статус монтирования
>mount | grep mnt
<pre>systemd-1 on /<span style="color:#C01C28"><b>mnt</b></span> type autofs (rw,relatime,fd=57,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=4022)
192.168.1.221:/srv/share/ on /<span style="color:#C01C28"><b>mnt</b></span> type nfs (rw,relatime,vers=3,rsize=262144,wsize=262144,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,mountaddr=192.168.1.221,mountvers=3,mountport=50021,mountproto=udp,local_lock=none,addr=192.168.1.221)
</pre>

Все работает.