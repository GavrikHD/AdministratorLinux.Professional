# Размещаем свой RPM в своем репозитории

Создать свой RPM пакет

Устанавливаем нужные пакеты:
>yum install -y wget rpmdevtools rpm-build createrepo \yum-utils cmake gcc git nano

Загрузим SRPM пакет Nginx для дальнейшей работы над ним:
>mkdir rpm && cd rpm
>yumdownloader --source nginx
<pre>enabling appstream-source repository
enabling baseos-source repository
enabling extras-source repository
AlmaLinux 9 - AppStream - Source                                        454 kB/s | 857 kB     00:01    
AlmaLinux 9 - BaseOS - Source                                           182 kB/s | 312 kB     00:01    
AlmaLinux 9 - Extras - Source                                           4.8 kB/s | 8.2 kB     00:01    
nginx-1.20.1-20.el9.alma.1.src.rpm </pre>

Поставим все зависимости для сборки пакета Nginx:
>rpm -Uvh nginx*.src.rpm
>yum-builddep nginx

Качаем исходный код модуля ngx_brotli — он потребуется при сборке:
>cd /root
>git clone --recurse-submodules -j8 \https://github.com/google/ngx_brotli
>cd ngx_brotli/deps/brotli

Собираем модуль ngx_brotli:

>cmake -DCMAKE_BUILD_TYPE=Release -DBUILD_SHARED_LIBS=OFF -DCMAKE_C_FLAGS="-Ofast -m64 -march=native -mtune=native -flto -funroll-loops -ffunction-sections -fdata-sections -Wl,--gc-sections" -DCMAKE_CXX_FLAGS="-Ofast -m64 -march=native -mtune=native -flto -funroll-loops -ffunction-sections -fdata-sections -Wl,--gc-sections" -DCMAKE_INSTALL_PREFIX=./installed ..

<pre>-- The C compiler identification is GNU 11.5.0
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Check for working C compiler: /usr/bin/cc - skipped
-- Detecting C compile features
-- Detecting C compile features - done
-- Build type is &apos;Release&apos;
-- Performing Test BROTLI_EMSCRIPTEN
-- Performing Test BROTLI_EMSCRIPTEN - Failed
-- Compiler is not EMSCRIPTEN
-- Looking for log2
-- Looking for log2 - not found
-- Looking for log2
-- Looking for log2 - found
-- Configuring done (0.5s)
-- Generating done (0.0s)
<span style="color:#A2734C">CMake Warning:</span>
<span style="color:#A2734C">  Manually-specified variables were not used by the project:</span>

<span style="color:#A2734C">    CMAKE_CXX_FLAGS</span>


-- Build files have been written to: /root/ngx_brotli/deps/brotli
</pre>

>cmake --build . --config Release -j 2 --target brotlienc
>cd ../../../..

Нужно поправить spec файл, чтобы Nginx собирался с необходимыми нам опциями: 
находим секцию с параметрами configure и добавляем указание на модуль

>--add-module=/root/ngx_brotli \

Начинаем сборку RPM пакета:
>cd ~/rpmbuild/SPECS/
>rpmbuild -ba nginx.spec -D 'debug_package %{nil}'

<pre>Executing(%clean): /bin/sh -e /var/tmp/rpm-tmp.sbMWiA
+ umask 022
+ cd /root/rpmbuild/BUILD
+ cd nginx-1.20.1
+ /usr/bin/rm -rf /root/rpmbuild/BUILDROOT/nginx-1.20.1-20.el9.alma.1.x86_64
+ RPM_EC=0
++ jobs -p
+ exit 0
</pre>

Убедимся, что пакеты создались:

>ll rpmbuild/RPMS/x86_64/
<pre>total 1568
-rw-r--r--. 1 root root  36242 Mar 29 03:50 <span style="color:#C01C28"><b>nginx-1.20.1-20.el9.alma.1.x86_64.rpm</b></span>
-rw-r--r--. 1 root root 593325 Mar 29 03:50 <span style="color:#C01C28"><b>nginx-core-1.20.1-20.el9.alma.1.x86_64.rpm</b></span>
-rw-r--r--. 1 root root 759753 Mar 29 03:50 <span style="color:#C01C28"><b>nginx-mod-devel-1.20.1-20.el9.alma.1.x86_64.rpm</b></span>
-rw-r--r--. 1 root root  19365 Mar 29 03:50 <span style="color:#C01C28"><b>nginx-mod-http-image-filter-1.20.1-20.el9.alma.1.x86_64.rpm</b></span>
-rw-r--r--. 1 root root  31012 Mar 29 03:50 <span style="color:#C01C28"><b>nginx-mod-http-perl-1.20.1-20.el9.alma.1.x86_64.rpm</b></span>
-rw-r--r--. 1 root root  18174 Mar 29 03:50 <span style="color:#C01C28"><b>nginx-mod-http-xslt-filter-1.20.1-20.el9.alma.1.x86_64.rpm</b></span>
-rw-r--r--. 1 root root  53821 Mar 29 03:50 <span style="color:#C01C28"><b>nginx-mod-mail-1.20.1-20.el9.alma.1.x86_64.rpm</b></span>
-rw-r--r--. 1 root root  80448 Mar 29 03:50 <span style="color:#C01C28"><b>nginx-mod-stream-1.20.1-20.el9.alma.1.x86_64.rpm</b></span>
</pre>
Копируем пакеты в общий каталог:
>cp ~/rpmbuild/RPMS/noarch/* ~/rpmbuild/RPMS/x86_64/
>cd ~/rpmbuild/RPMS/x86_64

Устанавливем наш пакет и убеждаемся, что nginx работает:
>yum localinstall *.rpm
<pre>Installed:
  almalinux-logos-httpd-90.5.1-1.1.el9.noarch                                                           
  nginx-2:1.20.1-20.el9.alma.1.x86_64                                                                   
  nginx-all-modules-2:1.20.1-20.el9.alma.1.noarch                                                       
  nginx-core-2:1.20.1-20.el9.alma.1.x86_64                                                              
  nginx-filesystem-2:1.20.1-20.el9.alma.1.noarch                                                        
  nginx-mod-devel-2:1.20.1-20.el9.alma.1.x86_64                                                         
  nginx-mod-http-image-filter-2:1.20.1-20.el9.alma.1.x86_64                                             
  nginx-mod-http-perl-2:1.20.1-20.el9.alma.1.x86_64                                                     
  nginx-mod-http-xslt-filter-2:1.20.1-20.el9.alma.1.x86_64                                              
  nginx-mod-mail-2:1.20.1-20.el9.alma.1.x86_64                                                          
  nginx-mod-stream-2:1.20.1-20.el9.alma.1.x86_64      </pre>

>systemctl start nginx

>systemctl status nginx
<pre>     Loaded: loaded (/usr/lib/systemd/system/nginx.service; <span style="color:#D7D75F"><b>disabled</b></span>; preset: <span style="color:#D7D75F"><b>disabled</b></span>)
     Active: <span style="color:#26A269"><b>active (running)</b></span> since Sat 2025-03-29 07:12:09 MSK; 20s ago
    Process: 46041 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Process: 46042 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 46043 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
   Main PID: 46044 (nginx)
      Tasks: 2 (limit: 11083)
     Memory: 3.9M
        CPU: 34ms
     CGroup: /system.slice/nginx.service
             ├─<span style="color:#8A8A8A">46044 &quot;nginx: master process /usr/sbin/nginx&quot;</span>
             └─<span style="color:#8A8A8A">46045 &quot;nginx: worker process&quot;</span>

Mar 29 07:12:09 localhost.localdomain systemd[1]: Starting The nginx HTTP and reverse proxy server...
Mar 29 07:12:09 localhost.localdomain nginx[46042]: nginx: the configuration file /etc/nginx/nginx.conf<span style="background-color:#D0CFCC"><span style="color:#171421">&gt;</span></span>
Mar 29 07:12:09 localhost.localdomain nginx[46042]: nginx: configuration file /etc/nginx/nginx.conf tes<span style="background-color:#D0CFCC"><span style="color:#171421">&gt;</span></span>
Mar 29 07:12:09 localhost.localdomain systemd[1]: Started The nginx HTTP and reverse proxy server.
</pre>

Далее мы будем использовать его для доступа к своему репозиторию.

# Создать свой репозиторий и разместить там ранее собранный RPM

Создадим каталог repo в директории для статики, у Nginx по умолчанию это /usr/share/nginx/htm

>mkdir /usr/share/nginx/html/repo

Копируем туда наши собранные RPM-пакеты:

>cp ~/rpmbuild/RPMS/x86_64/*.rpm /usr/share/nginx/html/repo/

Инициализируем репозиторий командой:

>createrepo /usr/share/nginx/html/repo/
<pre>Directory walk started
Directory walk done - 10 packages
Temporary output repo path: /usr/share/nginx/html/repo/.repodata/
Preparing sqlite DBs
Pool started (with 5 workers)
Pool finished
</pre>

Для прозрачности настроим в NGINX доступ к листингу каталога. В файле /etc/nginx/nginx.conf в блоке server добавим следующие директивы:

>nano /etc/nginx/nginx.conf

	index index.html index.htm;
	autoindex on;

Проверяем синтаксис и перезапускаем NGINX:

>nginx -t
<pre>nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
</pre>

>nginx -s reload

Смотрим с помощью curl:

 curl -a http://localhost/repo/

 <html>
<head><title>403 Forbidden</title></head>
<body>
<center><h1>403 Forbidden</h1></center>
<hr><center>nginx/1.20.1</center>
</body>
</html>
[root@localhost x86_64]# nginx -s reload
[root@localhost x86_64]#  curl -a http://localhost/repo/
<html>
<head><title>Index of /repo/</title></head>
<body>
<h1>Index of /repo/</h1><hr><pre><a href="../">../</a>
<a href="repodata/">repodata/</a>                                          29-Mar-2025 07:10                   -
<a href="nginx-1.20.1-20.el9.alma.1.x86_64.rpm">nginx-1.20.1-20.el9.alma.1.x86_64.rpm</a>              29-Mar-2025 07:07               36242
<a href="nginx-all-modules-1.20.1-20.el9.alma.1.noarch.rpm">nginx-all-modules-1.20.1-20.el9.alma.1.noarch.rpm</a>  29-Mar-2025 07:07                7357
<a href="nginx-core-1.20.1-20.el9.alma.1.x86_64.rpm">nginx-core-1.20.1-20.el9.alma.1.x86_64.rpm</a>         29-Mar-2025 07:07              593325
<a href="nginx-filesystem-1.20.1-20.el9.alma.1.noarch.rpm">nginx-filesystem-1.20.1-20.el9.alma.1.noarch.rpm</a>   29-Mar-2025 07:07                8440
<a href="nginx-mod-devel-1.20.1-20.el9.alma.1.x86_64.rpm">nginx-mod-devel-1.20.1-20.el9.alma.1.x86_64.rpm</a>    29-Mar-2025 07:07              759753
<a href="nginx-mod-http-image-filter-1.20.1-20.el9.alma.1.x86_64.rpm">nginx-mod-http-image-filter-1.20.1-20.el9.alma...&gt;</a> 29-Mar-2025 07:07               19365
<a href="nginx-mod-http-perl-1.20.1-20.el9.alma.1.x86_64.rpm">nginx-mod-http-perl-1.20.1-20.el9.alma.1.x86_64..&gt;</a> 29-Mar-2025 07:07               31012
<a href="nginx-mod-http-xslt-filter-1.20.1-20.el9.alma.1.x86_64.rpm">nginx-mod-http-xslt-filter-1.20.1-20.el9.alma.1..&gt;</a> 29-Mar-2025 07:07               18174
<a href="nginx-mod-mail-1.20.1-20.el9.alma.1.x86_64.rpm">nginx-mod-mail-1.20.1-20.el9.alma.1.x86_64.rpm</a>     29-Mar-2025 07:07               53821
<a href="nginx-mod-stream-1.20.1-20.el9.alma.1.x86_64.rpm">nginx-mod-stream-1.20.1-20.el9.alma.1.x86_64.rpm</a>   29-Mar-2025 07:07               80448
</pre><hr></body>
</html>

Все готово для того, чтобы протестировать репозиторий.

Добавим его в /etc/yum.repos.d:

>cat >> /etc/yum.repos.d/otus.repo << EOF
[otus]
name=otus-linux
baseurl=http://localhost/repo
gpgcheck=0
enabled=1
EOF

Убедимся, что репозиторий подключился и посмотрим, что в нем есть:
>yum repolist enabled | grep otus
<pre><span style="color:#C01C28"><b>otus</b></span>                             <span style="color:#C01C28"><b>otus</b></span>-linux
</pre>

Добавим пакет в наш репозиторий:

>cd /usr/share/nginx/html/repo/

>wget https://repo.percona.com/yum/percona-release-latest.noarch.rpm

<pre>--2025-03-29 10:56:47--  https://repo.percona.com/yum/percona-release-latest.noarch.rpm
Resolving repo.percona.com (repo.percona.com)... 49.12.125.205, 2a01:4f8:242:5792::2
Connecting to repo.percona.com (repo.percona.com)|49.12.125.205|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 28300 (28K) [application/x-redhat-package-manager]
Saving to: ‘percona-release-latest.noarch.rpm’

percona-release-latest.no 100%[=====================================&gt;]  27.64K  --.-KB/s    in 0s      

2025-03-29 10:56:47 (118 MB/s) - ‘percona-release-latest.noarch.rpm’ saved [28300/28300]
</pre>

Обновим список пакетов в репозитории:

>createrepo /usr/share/nginx/html/repo/
<pre>Directory walk started
Directory walk done - 11 packages
Temporary output repo path: /usr/share/nginx/html/repo/.repodata/
Preparing sqlite DBs
Pool started (with 5 workers)
Pool finished
</pre>

>yum makecache
<pre>AlmaLinux 9 - AppStream                                                 7.9 kB/s | 4.2 kB     00:00    
AlmaLinux 9 - BaseOS                                                    7.3 kB/s | 3.8 kB     00:00    
AlmaLinux 9 - Extras                                                    6.3 kB/s | 3.3 kB     00:00    
otus-linux                                                              573 kB/s | 7.2 kB     00:00    
Metadata cache created.</pre>

>yum list | grep otus
<pre>percona-release.noarch                               1.0-30                              <span style="color:#C01C28"><b>otus</b></span>  </pre>

Так как Nginx у нас уже стоит, установим репозиторий percona-release:

>yum install -y percona-release.noarch

<pre>  Verifying        : percona-release-1.0-30.noarch                                                  1/1 

Installed:
  percona-release-1.0-30.noarch                                                                         

Complete!
</pre>

Все прошло успешно.
