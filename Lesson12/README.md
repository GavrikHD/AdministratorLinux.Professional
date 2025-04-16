# Управление процессами

Напишем небольшой скрипт bash который показывает с помощью lsof статус процессов связанных с сервером 1С платформы 8.3.25

<pre>
#!/bin/bash

# Проверяем, установлен ли lsof
if ! command -v lsof &> /dev/null; then
    echo "Ошибка: lsof не установлен. Установите его для работы скрипта."
    exit 1
fi

# Ищем процессы, содержащие "8.3.25" в имени
pids=$(pgrep -f '8.3.25')

if [ -z "$pids" ]; then
    echo "Процессы с '8.3.25' в имени не найдены."
    exit 0
fi

# Выводим информацию о каждом процессе
for pid in $pids; do
    echo "Информация для процесса PID: $pid"
    echo "Имя процесса: $(ps -p $pid -o comm=)"
    echo "Статус процесса: $(ps -p $pid -o stat=)"
    echo "Открытые файлы (lsof):"
    lsof -p $pid
    echo "----------------------------------------"
done
</pre>

смотрим на вывод


<pre>
Информация для процесса PID: 733
Имя процесса: ras
Статус процесса: Ssl
Открытые файлы (lsof):
COMMAND PID    USER   FD      TYPE             DEVICE SIZE/OFF    NODE NAME
ras     733 usr1cv8  cwd       DIR              254,2     4096       2 /
ras     733 usr1cv8  rtd       DIR              254,2     4096       2 /
ras     733 usr1cv8  txt       REG              254,2  1634496 1048752 /opt/1cv8/x86_64/8.3.25.1501/ras
ras     733 usr1cv8  mem       REG              254,2 15559456 1048693 /opt/1cv8/x86_64/8.3.25.1501/backbas.so
ras     733 usr1cv8  mem       REG              254,2   731888 1048703 /opt/1cv8/x86_64/8.3.25.1501/libnghttp2.so.14.20.1
ras     733 usr1cv8  mem       REG              254,2   628384 1048818 /opt/1cv8/x86_64/8.3.25.1501/sas.so
ras     733 usr1cv8  mem       REG              254,2  1026952 1048820 /opt/1cv8/x86_64/8.3.25.1501/sqlite.so
ras     733 usr1cv8  mem       REG              254,2  7252992 1048697 /opt/1cv8/x86_64/8.3.25.1501/inet.so
ras     733 usr1cv8  mem       REG              254,2   982544 1048757 /opt/1cv8/x86_64/8.3.25.1501/anion.so
ras     733 usr1cv8  mem       REG              254,2 28426088 1048817 /opt/1cv8/x86_64/8.3.25.1501/rtrsrvc.so
ras     733 usr1cv8  mem       REG              254,2   579160 1048764 /opt/1cv8/x86_64/8.3.25.1501/cas.so
ras     733 usr1cv8  mem       REG              254,2 14765424 1048783 /opt/1cv8/x86_64/8.3.25.1501/ext.so
ras     733 usr1cv8  mem       REG              254,2   473128 1048765 /opt/1cv8/x86_64/8.3.25.1501/cci.so
ras     733 usr1cv8  mem       REG              254,2 18632616 1048702 /opt/1cv8/x86_64/8.3.25.1501/xml2.so
ras     733 usr1cv8  mem       REG              254,2  1760424 1048895 /opt/1cv8/x86_64/8.3.25.1501/core83_root.res
ras     733 usr1cv8  mem       REG              254,2  1575619 1048715 /opt/1cv8/x86_64/8.3.25.1501/libicuuc.so.46.1
ras     733 usr1cv8  mem       REG              254,2   685984 1048822 /opt/1cv8/x86_64/8.3.25.1501/swp.so
ras     733 usr1cv8  mem       REG              254,2 15431828 1048713 /opt/1cv8/x86_64/8.3.25.1501/libicudata.so.46.1
ras     733 usr1cv8  mem       REG              254,2  2403877 1048714 /opt/1cv8/x86_64/8.3.25.1501/libicui18n.so.46.1
ras     733 usr1cv8  mem       REG              254,2   752824 1048706 /opt/1cv8/x86_64/8.3.25.1501/libgcc_s.so.1
ras     733 usr1cv8  mem       REG              254,2  1922136  656894 /usr/lib/x86_64-linux-gnu/libc.so.6
ras     733 usr1cv8  mem       REG              254,2  2531984 1048708 /opt/1cv8/x86_64/8.3.25.1501/libstdc++.so.6.0.28
ras     733 usr1cv8  mem       REG              254,2   709592 1048806 /opt/1cv8/x86_64/8.3.25.1501/pack.so
ras     733 usr1cv8  mem       REG              254,2   904952 1048700 /opt/1cv8/x86_64/8.3.25.1501/techsys.so
ras     733 usr1cv8  mem       REG              254,2  4111616 1048709 /opt/1cv8/x86_64/8.3.25.1501/libtcmalloc.so.4.5.9
ras     733 usr1cv8  mem       REG              254,2   128996 1048889 /opt/1cv8/x86_64/8.3.25.1501/backbas_root.res
ras     733 usr1cv8  mem       REG              254,2    67820 1049040 /opt/1cv8/x86_64/8.3.25.1501/backbas_ru.res
ras     733 usr1cv8  mem       REG              254,2   188352 1048742 /opt/1cv8/x86_64/8.3.25.1501/cciswp.so
ras     733 usr1cv8  mem       REG              254,2   162832 1048743 /opt/1cv8/x86_64/8.3.25.1501/secure.so
ras     733 usr1cv8  mem       REG              254,2   384540 1049050 /opt/1cv8/x86_64/8.3.25.1501/core83_ru.res
ras     733 usr1cv8  mem       REG              254,2   684464 1048707 /opt/1cv8/x86_64/8.3.25.1501/libunwind.so.8
ras     733 usr1cv8  mem       REG              254,2   907784  656897 /usr/lib/x86_64-linux-gnu/libm.so.6
ras     733 usr1cv8  mem       REG              254,2 10370760 1048694 /opt/1cv8/x86_64/8.3.25.1501/core83.so
ras     733 usr1cv8  mem       REG              254,2    14480  657605 /usr/lib/x86_64-linux-gnu/libpthread.so.0
ras     733 usr1cv8  mem       REG              254,2    14480  656896 /usr/lib/x86_64-linux-gnu/libdl.so.2
ras     733 usr1cv8  mem       REG              254,2    14640  657608 /usr/lib/x86_64-linux-gnu/librt.so.1
ras     733 usr1cv8  mem       REG              254,2     2448 1048978 /opt/1cv8/x86_64/8.3.25.1501/ras_root.res
ras     733 usr1cv8  mem       REG              254,2     2616 1049129 /opt/1cv8/x86_64/8.3.25.1501/ras_ru.res
ras     733 usr1cv8  mem       REG              254,2    30800 1048691 /opt/1cv8/x86_64/8.3.25.1501/nuke83.so
ras     733 usr1cv8  mem       REG              254,2   210904  656891 /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
ras     733 usr1cv8    0r      CHR                1,3      0t0       4 /dev/null
ras     733 usr1cv8    1u     unix 0x000000005d905aa1      0t0   15428 type=STREAM (CONNECTED)
ras     733 usr1cv8    2u     unix 0x000000005d905aa1      0t0   15428 type=STREAM (CONNECTED)
ras     733 usr1cv8    3r     FIFO               0,13      0t0   15432 pipe
ras     733 usr1cv8    4w     FIFO               0,13      0t0   15432 pipe
ras     733 usr1cv8    5u  a_inode               0,14        0      55 [eventpoll:8,10]
ras     733 usr1cv8    6r     FIFO               0,13      0t0   15433 pipe
ras     733 usr1cv8    7w     FIFO               0,13      0t0   15433 pipe
ras     733 usr1cv8    8r     FIFO               0,13      0t0   15434 pipe
ras     733 usr1cv8    9w     FIFO               0,13      0t0   15434 pipe
ras     733 usr1cv8   10u  a_inode               0,14        0      55 [eventfd:1]
ras     733 usr1cv8   11u  a_inode               0,14        0      55 [eventfd:2]
ras     733 usr1cv8   12u  a_inode               0,14        0      55 [eventpoll:11,13,14]
ras     733 usr1cv8   13u  a_inode               0,14        0      55 [timerfd]
ras     733 usr1cv8   14u     IPv4              15436      0t0     TCP *:1545 (LISTEN)
----------------------------------------
Информация для процесса PID: 734
Имя процесса: ragent
Статус процесса: Ssl
Открытые файлы (lsof):
COMMAND PID    USER   FD      TYPE             DEVICE SIZE/OFF    NODE NAME
ragent  734 usr1cv8  cwd       DIR              254,2     4096       2 /
ragent  734 usr1cv8  rtd       DIR              254,2     4096       2 /
ragent  734 usr1cv8  txt       REG              254,2    39712 1048738 /opt/1cv8/x86_64/8.3.25.1501/ragent
ragent  734 usr1cv8  mem       REG              254,2   731888 1048703 /opt/1cv8/x86_64/8.3.25.1501/libnghttp2.so.14.20.1
ragent  734 usr1cv8  mem       REG              254,2  7252992 1048697 /opt/1cv8/x86_64/8.3.25.1501/inet.so
ragent  734 usr1cv8  mem       REG              254,2  1026952 1048820 /opt/1cv8/x86_64/8.3.25.1501/sqlite.so
ragent  734 usr1cv8  mem       REG              254,2 15559456 1048693 /opt/1cv8/x86_64/8.3.25.1501/backbas.so
ragent  734 usr1cv8  mem       REG              254,2 14765424 1048783 /opt/1cv8/x86_64/8.3.25.1501/ext.so
ragent  734 usr1cv8  mem       REG              254,2 18632616 1048702 /opt/1cv8/x86_64/8.3.25.1501/xml2.so
ragent  734 usr1cv8  mem       REG              254,2 20083208 1048816 /opt/1cv8/x86_64/8.3.25.1501/rserver.so
ragent  734 usr1cv8  mem       REG              254,2   721560 1048831 /opt/1cv8/x86_64/8.3.25.1501/fts-core-tools.so
ragent  734 usr1cv8  mem       REG              254,2 28426088 1048817 /opt/1cv8/x86_64/8.3.25.1501/rtrsrvc.so
ragent  734 usr1cv8  mem       REG              254,2  1575619 1048715 /opt/1cv8/x86_64/8.3.25.1501/libicuuc.so.46.1
ragent  734 usr1cv8  mem       REG              254,2   709592 1048806 /opt/1cv8/x86_64/8.3.25.1501/pack.so
ragent  734 usr1cv8  mem       REG              254,2 15431828 1048713 /opt/1cv8/x86_64/8.3.25.1501/libicudata.so.46.1
ragent  734 usr1cv8  mem       REG              254,2  2403877 1048714 /opt/1cv8/x86_64/8.3.25.1501/libicui18n.so.46.1
ragent  734 usr1cv8  mem       REG              254,2   752824 1048706 /opt/1cv8/x86_64/8.3.25.1501/libgcc_s.so.1
ragent  734 usr1cv8  mem       REG              254,2    37880 1048911 /opt/1cv8/x86_64/8.3.25.1501/rserver_root.res
ragent  734 usr1cv8  mem       REG              254,2  1760424 1048895 /opt/1cv8/x86_64/8.3.25.1501/core83_root.res
ragent  734 usr1cv8  mem       REG              254,2  2531984 1048708 /opt/1cv8/x86_64/8.3.25.1501/libstdc++.so.6.0.28
ragent  734 usr1cv8  mem       REG              254,2    41960 1049059 /opt/1cv8/x86_64/8.3.25.1501/rserver_ru.res
ragent  734 usr1cv8  mem       REG              254,2   341424 1048815 /opt/1cv8/x86_64/8.3.25.1501/rscalls.so
ragent  734 usr1cv8  mem       REG              254,2   128996 1048889 /opt/1cv8/x86_64/8.3.25.1501/backbas_root.res
ragent  734 usr1cv8  mem       REG              254,2   904952 1048700 /opt/1cv8/x86_64/8.3.25.1501/techsys.so
ragent  734 usr1cv8  mem       REG              254,2  4111616 1048709 /opt/1cv8/x86_64/8.3.25.1501/libtcmalloc.so.4.5.9
ragent  734 usr1cv8  mem       REG              254,2  1922136  656894 /usr/lib/x86_64-linux-gnu/libc.so.6
ragent  734 usr1cv8  mem       REG              254,2 10370760 1048694 /opt/1cv8/x86_64/8.3.25.1501/core83.so
ragent  734 usr1cv8  mem       REG              254,2    67820 1049040 /opt/1cv8/x86_64/8.3.25.1501/backbas_ru.res
ragent  734 usr1cv8  mem       REG              254,2   384540 1049050 /opt/1cv8/x86_64/8.3.25.1501/core83_ru.res
ragent  734 usr1cv8  mem       REG              254,2   684464 1048707 /opt/1cv8/x86_64/8.3.25.1501/libunwind.so.8
ragent  734 usr1cv8  mem       REG              254,2   907784  656897 /usr/lib/x86_64-linux-gnu/libm.so.6
ragent  734 usr1cv8  mem       REG              254,2    14480  656896 /usr/lib/x86_64-linux-gnu/libdl.so.2
ragent  734 usr1cv8  mem       REG              254,2    14640  657608 /usr/lib/x86_64-linux-gnu/librt.so.1
ragent  734 usr1cv8  mem       REG              254,2    14480  657605 /usr/lib/x86_64-linux-gnu/libpthread.so.0
ragent  734 usr1cv8  mem       REG              254,2    30800 1048691 /opt/1cv8/x86_64/8.3.25.1501/nuke83.so
ragent  734 usr1cv8  mem       REG              254,2   210904  656891 /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
ragent  734 usr1cv8    0r      CHR                1,3      0t0       4 /dev/null
ragent  734 usr1cv8    1u     unix 0x000000009fef8b30      0t0   12806 type=STREAM (CONNECTED)
ragent  734 usr1cv8    2u     unix 0x000000009fef8b30      0t0   12806 type=STREAM (CONNECTED)
ragent  734 usr1cv8    3r     FIFO               0,13      0t0   16518 pipe
ragent  734 usr1cv8    4w     FIFO               0,13      0t0   16518 pipe
ragent  734 usr1cv8    5ur     REG              254,2      176  131086 /home/usr1cv8/.1cv8/1C/1cv8/1cv8wsrv.lst
ragent  734 usr1cv8    6ur     REG              254,2      176  131086 /home/usr1cv8/.1cv8/1C/1cv8/1cv8wsrv.lst
ragent  734 usr1cv8    7u  a_inode               0,14        0      55 [eventfd:3]
ragent  734 usr1cv8    8u  a_inode               0,14        0      55 [eventpoll:7,9,13,14]
ragent  734 usr1cv8    9u  a_inode               0,14        0      55 [timerfd]
ragent  734 usr1cv8   10u  a_inode               0,14        0      55 [eventfd:4]
ragent  734 usr1cv8   11u  a_inode               0,14        0      55 [eventpoll:10,12]
ragent  734 usr1cv8   12u  a_inode               0,14        0      55 [timerfd]
ragent  734 usr1cv8   13u     IPv6                887      0t0     TCP *:1540 (LISTEN)
ragent  734 usr1cv8   14u     IPv4                888      0t0     TCP *:1540 (LISTEN)
----------------------------------------
</pre>