# Systemd — создание unit-файла

Cоздаём файл с конфигурацией для сервиса в директории /etc/default - из неё сервис будет брать необходимые переменные.

>nano /etc/default/watchlog

<pre># Configuration file for my watchlog service
# Place it to /etc/default

# File and word in that file that we will be monit
WORD="ALERT"
LOG=/var/log/watchlog.log</pre>

Cоздаем /var/log/watchlog.log и пишем туда строки на своё усмотрение,
плюс ключевое слово ‘ALERT’

>nano /var/log/watchlog.log

ALERT
NGINX

Создадим скрипт:
nano /opt/watchlog.sh

вставляем
<pre>#!/bin/bash

WORD=$1
LOG=$2
DATE=`date`

if grep $WORD $LOG &> /dev/null
then
logger "$DATE: I found word, Master!"
else
exit 0
fi
</pre>

Команда logger отправляет лог в системный журнал.
Добавим права на запуск файла:

>chmod +x /opt/watchlog.sh

Создадим юнит для сервиса:

>nano /etc/systemd/system/watchlog.service

вставим в файл

<pre>[Unit]
Description=My watchlog service

[Service]
Type=oneshot
EnvironmentFile=/etc/default/watchlog
ExecStart=/opt/watchlog.sh $WORD $LOG
</pre>

Создадим юнит для таймера:

nano /etc/systemd/system/watchlog.timer

вставим в файл

<pre>
[Unit]
Description=Run watchlog script every 30 second

[Timer]
# Run every 30 second
OnUnitActiveSec=30
Unit=watchlog.service

[Install]
WantedBy=multi-user.target
</pre>

Запустим timer:

>systemctl start watchlog.timer

Убедимся в результате:

>tail -n 1000 /var/log/syslog  | grep word

<pre>2025-04-03T17:33:06.843034+00:00 u24 systemd[1]: Started systemd-ask-pass<span style="color:#C01C28"><b>word</b></span>-console.path - Dispatch Pass<span style="color:#C01C28"><b>word</b></span> Requests to Console Directory Watch.
2025-04-03T17:33:06.843037+00:00 u24 systemd[1]: systemd-ask-pass<span style="color:#C01C28"><b>word</b></span>-plymouth.path - Forward Pass<span style="color:#C01C28"><b>word</b></span> Requests to Plymouth Directory Watch was skipped because of an unmet condition check (ConditionPathExists=/run/plymouth/pid).
2025-04-03T17:33:06.846770+00:00 u24 kernel: systemd[1]: Started systemd-ask-pass<span style="color:#C01C28"><b>word</b></span>-wall.path - Forward Pass<span style="color:#C01C28"><b>word</b></span> Requests to Wall Directory Watch.
2025-04-03T17:33:06.846847+00:00 u24 kernel: audit: type=1400 audit(1743701585.879:2): apparmor=&quot;STATUS&quot; operation=&quot;profile_load&quot; profile=&quot;unconfined&quot; name=&quot;1pass<span style="color:#C01C28"><b>word</b></span>&quot; pid=473 comm=&quot;apparmor_parser&quot;
</pre>

# Установить spawn-fcgi и создать unit-файл (spawn-fcgi.sevice) с помощью переделки init-скрипта

устанавливаем необходимые программы

>apt install spawn-fcgi php php-cgi php-cli \apache2 libapache2-mod-fcgid -y

Создаем каталог
>mkdir /etc/spawn-fcgi

создаем файл с настройками для будущего сервиса

nano /etc/spawn-fcgi/fcgi.conf

Он должен получится следующего вида:
# You must set some working options before the "spawn-fcgi" service will work.
# If SOCKET points to a file, then this file is cleaned up by the init script.
#
# See spawn-fcgi(1) for all possible options.
#
# Example :
SOCKET=/var/run/php-fcgi.sock
OPTIONS="-u www-data -g www-data -s $SOCKET -S -M 0600 -C 32 -F 1 -- /usr/bin/php-cgi"

Создаем сам юнит-файл

>nano /etc/systemd/system/spawn-fcgi.service

и прописываем его содержимое

<pre>[Unit]
Description=Spawn-fcgi startup service by Otus
After=network.target

[Service]
Type=simple
PIDFile=/var/run/spawn-fcgi.pid
EnvironmentFile=/etc/spawn-fcgi/fcgi.conf
ExecStart=/usr/bin/spawn-fcgi -n $OPTIONS
KillMode=process

[Install]
WantedBy=multi-user.target
</pre>

Убеждаемся, что все успешно работает:

>systemctl start spawn-fcgi

>systemctl status spawn-fcgi

<pre><span style="color:#26A269"><b>●</b></span> spawn-fcgi.service - Spawn-fcgi startup service by Otus
     Loaded: loaded (/etc/systemd/system/spawn-fcgi.
service; <span style="color:#D7D75F"><b>disabled</b></span>; preset: <span style="color:#26A269"><b>enabled</b></span>)
     Active: <span style="color:#26A269"><b>active (running)</b></span> since Sun 2025-04-06 05:00:35 UTC; 11s ago
   Main PID: 12630 (php-cgi)
      Tasks: 33 (limit: 2273)
     Memory: 14.7M (peak: 14.9M)
        CPU: 27ms
     CGroup: /system.slice/spawn-fcgi.service
             ├─<span style="color:#8A8A8A">12630 /usr/bin/php-cgi</span>
             ├─<span style="color:#8A8A8A">12631 /usr/bin/php-cgi</span>
             ├─<span style="color:#8A8A8A">12632 /usr/bin/php-cgi</span>
             ├─<span style="color:#8A8A8A">12633 /usr/bin/php-cgi</span>
             ├─<span style="color:#8A8A8A">12634 /usr/bin/php-cgi</span>
             ├─<span style="color:#8A8A8A">12635 /usr/bin/php-cgi</span>
             ├─<span style="color:#8A8A8A">12636 /usr/bin/php-cgi</span>
             ├─<span style="color:#8A8A8A">12637 /usr/bin/php-cgi</span>
             ├─<span style="color:#8A8A8A">12638 /usr/bin/php-cgi</span>
             ├─<span style="color:#8A8A8A">12639 /usr/bin/php-cgi</span>
             ├─<span style="color:#8A8A8A">12640 /usr/bin/php-cgi</span>
             ├─<span style="color:#8A8A8A">12641 /usr/bin/php-cgi</span>
             ├─<span style="color:#8A8A8A">12642 /usr/bin/php-cgi</span>
             ├─<span style="color:#8A8A8A">12643 /usr/bin/php-cgi</span>
             ├─<span style="color:#8A8A8A">12644 /usr/bin/php-cgi</span>
             ├─<span style="color:#8A8A8A">12645 /usr/bin/php-cgi</span>
             ├─<span style="color:#8A8A8A">12646 /usr/bin/php-cgi</span>
...
</pre>

# Доработать unit-файл Nginx (nginx.service) для запуска нескольких инстансов сервера с разными конфигурационными файлами одновременно

Установим Nginx из стандартного репозитория:

>apt install nginx -y

Для запуска нескольких экземпляров сервиса модифицируем исходный service для использования различной конфигурации, а также PID-файлов. Для этого создадим новый Unit для работы с шаблонами (/etc/systemd/system/nginx@.service):

>nano /etc/systemd/system/nginx@.service

прописываем в него

<pre>
# Stop dance for nginx
# =======================
#
# ExecStop sends SIGSTOP (graceful stop) to the nginx process.
# If, after 5s (--retry QUIT/5) nginx is still running, systemd takes control
# and sends SIGTERM (fast shutdown) to the main process.
# After another 5s (TimeoutStopSec=5), and if nginx is alive, systemd sends
# SIGKILL to all the remaining processes in the process group (KillMode=mixed).
#
# nginx signals reference doc:
# http://nginx.org/en/docs/control.html
#
[Unit]
Description=A high performance web server and a reverse proxy server
Documentation=man:nginx(8)
After=network.target nss-lookup.target

[Service]
Type=forking
PIDFile=/run/nginx-%I.pid
ExecStartPre=/usr/sbin/nginx -t -c /etc/nginx/nginx-%I.conf -q -g 'daemon on; master_process on;'
ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx-%I.conf -g 'daemon on; master_process on;'
ExecReload=/usr/sbin/nginx -c /etc/nginx/nginx-%I.conf -g 'daemon on; master_process on;' -s reload
ExecStop=-/sbin/start-stop-daemon --quiet --stop --retry QUIT/5 --pidfile /run/nginx-%I.pid
TimeoutStopSec=5
KillMode=mixed

[Install]
WantedBy=multi-user.target
</pre>

Далее необходимо создать два файла конфигурации. Их можно сформировать из стандартного конфига /etc/nginx/nginx.conf, с модификацией путей до PID-файлов и разделением по портам:

Открываем стандартный конфиг

>nano /etc/nginx/nginx.conf

и копируем его настройки в 

>nano /etc/nginx/nginx-first.conf

с изменением путей к PID  и сменой порта
<pre>
pid /run/nginx-first.pid;

http {
…
	server {
		listen 9001;
	}
#include /etc/nginx/sites-enabled/*;
….
}
</pre>

Тоже самое и в 

>nano /etc/nginx/nginx-second.conf

<pre>
pid /run/nginx-second.pid;

http {
…
	server {
		listen 9002;
	}
#include /etc/nginx/sites-enabled/*;
….
}
</pre>

Запустим сервисы

>systemctl start nginx@first

>systemctl start nginx@second

Проверим их работу

>Systemctl status nginx@second

<pre><span style="color:#26A269"><b>●</b></span> nginx@second.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/etc/systemd/system/nginx@.service; <span style="color:#D7D75F"><b>disabled</b></span>; preset: <span style="color:#26A269"><b>enabled</b></span>)
     Active: <span style="color:#26A269"><b>active (running)</b></span> since Sun 2025-04-06 09:27:43 UTC; 10s ago
       Docs: man:nginx(8)
    Process: 13905 ExecStartPre=/usr/sbin/nginx -t -c /etc/nginx/nginx-second.conf -q -g daemon on; mast
er_process on; (code=exited, status=0/SUCCESS)
    Process: 13906 ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx-second.conf -g daemon on; master_proces
s on; (code=exited, status=0/SUCCESS)
   Main PID: 13908 (nginx)
      Tasks: 2 (limit: 2273)
     Memory: 1.7M (peak: 1.9M)
        CPU: 9ms
     CGroup: /system.slice/system-nginx.slice/nginx@second.service
             ├─<span style="color:#8A8A8A">13908 &quot;nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx-second.conf </span>
<span style="color:#8A8A8A">-g daemon on; master_process on;&quot;</span>
             └─<span style="color:#8A8A8A">13909 &quot;nginx: worker process&quot;</span>

Apr 06 09:27:43 u24 systemd[1]: Starting nginx@second.service - A high performance web server and a reve
rse proxy server...
Apr 06 09:27:43 u24 systemd[1]: Started nginx@second.service - A high performance web server and a rever
se proxy server.
</pre>

Проверить можно несколькими способами, например, посмотреть, какие порты слушаются:

>ss -tnulp | grep nginx

<pre>tcp   LISTEN 0      511                 0.0.0.0:9002      0.0.0.0:*    users:((&quot;<span style="color:#C01C28"><b>nginx</b></span>&quot;,pid=13909,fd=5),(&quot;<span style="color:#C01C28"><b>nginx</b></span>&quot;,pid=13908,fd=5))                                                                                                                                           
tcp   LISTEN 0      511                 0.0.0.0:9001      0.0.0.0:*    users:((&quot;<span style="color:#C01C28"><b>nginx</b></span>&quot;,pid=13901,fd=5),(&quot;<span style="color:#C01C28"><b>nginx</b></span>&quot;,pid=13898,fd=5))  </pre>

Просмотреть список процессов:

<pre> 14636 pts/0    S+     0:00          \_ grep --color=auto <span style="color:#C01C28"><b>nginx</b></span>
  13898 ?        Ss     0:00 <span style="color:#C01C28"><b>nginx</b></span>: master process /usr/sbin/<span style="color:#C01C28"><b>nginx</b></span> -c /etc/<span style="color:#C01C28"><b>nginx</b></span>/<span style="color:#C01C28"><b>nginx</b></span>-first.conf -g daemon on; master_process on;
  13901 ?        S      0:00  \_ <span style="color:#C01C28"><b>nginx</b></span>: worker process
  13908 ?        Ss     0:00 <span style="color:#C01C28"><b>nginx</b></span>: master process /usr/sbin/<span style="color:#C01C28"><b>nginx</b></span> -c /etc/<span style="color:#C01C28"><b>nginx</b></span>/<span style="color:#C01C28"><b>nginx</b></span>-second.conf -g daemon on; master_process on;
  13909 ?        S      0:00  \_ <span style="color:#C01C28"><b>nginx</b></span>: worker process
</pre>


