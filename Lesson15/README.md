# Создаём виртуальную машину

Создаем виртуальную машину с помощью vagrant
Создаем папку для проекта

>mkdir vagrant1

>cd vagrant1

Создаем файл конфигурации

>nano Vagrantfile

В него вписываем конфигурацию

<pre>MACHINES = {
  :"selinux" => {
              :box_name => "almalinux/9",
              :box_version => "9.4.20240805",
              :cpus => 2,
              :memory => 2048
            }
}

Vagrant.configure("2") do |config|
  MACHINES.each do |boxname, boxconfig|
    config.vm.synced_folder ".", "/vagrant", disabled: true
    config.vm.define boxname do |box|
      box.vm.box = boxconfig[:box_name]
      box.vm.box_version = boxconfig[:box_version]
      box.vm.host_name = boxname.to_s
      box.vm.network "forwarded_port", guest: 4881, host: 4881
      box.vm.provider "virtualbox" do |v|
        v.memory = boxconfig[:memory]
        v.cpus = boxconfig[:cpus]
      end
      box.vm.provision "shell", inline: <<-SHELL
      yum install -y epel-release
      yum install -y nginx
      yum install -y setroubleshoot-server selinux-policy-mls setools-console policycoreutils-python-utils policycoreutils-newrole
      sed -ie 's/:80/:4881/g' /etc/nginx/nginx.conf
      sed -i 's/listen       80;/listen       4881;/' /etc/nginx/nginx.conf
      systemctl start nginx
      systemctl status nginx
      ss -tlpn | grep 4881
SHELL
    end
  end
end</pre>

разворачиваем виртуальную машину

>vagratn up

в конце установки получаем ошибку

<pre><span style="color:#C01C28">selinux: Job for nginx.service failed because the control process exited with error code.</span>
<span style="color:#C01C28">    selinux: See &quot;systemctl status nginx.service&quot; and &quot;journalctl -xeu nginx.service&quot; for details.</span>
<span style="color:#26A269">    selinux: × nginx.service - The nginx HTTP and reverse proxy server</span>
<span style="color:#26A269">    selinux:      Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; preset: disabled)</span>
<span style="color:#26A269">    selinux:      Active: failed (Result: exit-code) since Wed 2025-05-14 09:04:57 UTC; 9ms ago</span>
<span style="color:#26A269">    selinux:     Process: 6648 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)</span>
<span style="color:#26A269">    selinux:     Process: 6661 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)</span>
<span style="color:#26A269">    selinux:         CPU: 14ms</span>
<span style="color:#26A269">    selinux: </span>
<span style="color:#26A269">    selinux: May 14 09:04:57 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...</span>
<span style="color:#26A269">    selinux: May 14 09:04:57 selinux nginx[6661]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok</span>
<span style="color:#26A269">    selinux: May 14 09:04:57 selinux nginx[6661]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denied)</span>
<span style="color:#26A269">    selinux: May 14 09:04:57 selinux nginx[6661]: nginx: configuration file /etc/nginx/nginx.conf test failed</span>
<span style="color:#26A269">    selinux: May 14 09:04:57 selinux systemd[1]: nginx.service: Control process exited, code=exited, status=1/FAILURE</span>
<span style="color:#26A269">    selinux: May 14 09:04:57 selinux systemd[1]: nginx.service: Failed with result &apos;exit-code&apos;.</span>
<span style="color:#26A269">    selinux: May 14 09:04:57 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.</span>
<span style="color:#C01C28">The SSH command responded with a non-zero exit status. Vagrant</span>
<span style="color:#C01C28">assumes that this means the command failed. The output for this command</span>
<span style="color:#C01C28">should be in the log above. Please read the output to determine what</span>
<span style="color:#C01C28">went wrong.</span>
</pre>

Что бы понять причину, заходим на сервер

>vagrant ssh

<pre>[vagrant@selinux ~]$</pre>

для исправления нм понадобяться права root

>sudo -i

<pre>[root@selinux ~]#</pre>

# Запуск nginx на нестандартном порту 3-мя разными способами

Проверяем работает ли Firewall

>systemctl status firewalld

<pre>firewalld.service - firewalld - dynamic firewall daemon
     Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; preset: enabled)
     Active: inactive (dead)
       Docs: man:firewalld(1)
</pre>

Проверяем nginx на наличие ошибок

>nginx -t

<pre>nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
</pre>

Далее проверим режим работы SELinux:

>getenforce

<pre> Enforcing
</pre>

# Разрешим в SELinux работу nginx на порту TCP 4881 c помощью переключателей setsebool

ищем в логах информацию о блокировке порта

>grep src=4881 /var/log/audit/audit.log | audit2why

<pre>type=AVC msg=audit(1747213497.847:779): avc:  denied  { name_bind } for  pid=6661 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

	Was caused by:
	The boolean nis_enabled was set incorrectly. 
	Description:
	Allow nis to enabled

	Allow access by executing:
	# setsebool -P nis_enabled 1

</pre>

Утилита audit2why показывает почему трафик блокируется. 

Включим параметр nis_enabled и перезапустим nginx: 

>setsebool -P nis_enabled on

>setsebool -P nis_enabled on
>systemctl restart nginx
>systemctl status nginx
<pre><span style="color:#26A269"><b>●</b></span> nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; <span style="color:#D7D75F"><b>disabled</b></span>; preset: <span style="color:#D7D75F"><b>disabled</b></span>)
     Active: <span style="color:#26A269"><b>active (running)</b></span> since Fri 2025-05-16 11:28:41 UTC; 15s ago
    Process: 11690 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Process: 11691 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 11693 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
   Main PID: 11694 (nginx)
      Tasks: 3 (limit: 11997)
     Memory: 2.9M
        CPU: 23ms
     CGroup: /system.slice/nginx.service
             ├─<span style="color:#8A8A8A">11694 &quot;nginx: master process /usr/sbin/nginx&quot;</span>
             ├─<span style="color:#8A8A8A">11695 &quot;nginx: worker process&quot;</span>
             └─<span style="color:#8A8A8A">11696 &quot;nginx: worker process&quot;</span>

May 16 11:28:41 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
May 16 11:28:41 selinux nginx[11691]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
May 16 11:28:41 selinux nginx[11691]: nginx: configuration file /etc/nginx/nginx.conf test is successful
May 16 11:28:41 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
</pre>

Проверить статус параметра можно с помощью команды: 
>getsebool -a | grep nis_enabled

<pre><span style="color:#C01C28"><b>nis_enabled</b></span> --&gt; on
</pre>

Далее ворачиваем запрет работы nginx на порту 4881 обратно.

>setsebool -P nis_enabled off

# Теперь разрешим в SELinux работу nginx на порту TCP 4881 c помощью добавления нестандартного порта в имеющийся тип:

Поиск имеющегося типа, для http трафика:

>semanage port -l | grep http

<pre><span style="color:#C01C28"><b>http</b></span>_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
<span style="color:#C01C28"><b>http</b></span>_cache_port_t              udp      3130
<span style="color:#C01C28"><b>http</b></span>_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_<span style="color:#C01C28"><b>http</b></span>_port_t            tcp      5988
pegasus_<span style="color:#C01C28"><b>http</b></span>s_port_t           tcp      5989
</pre>

Добавим порт в тип http_port_t: 

>semanage port -a -t http_port_t -p tcp 4881

проверим

> semanage port -l | grep  http_port_t

<pre><span style="color:#C01C28"><b>http_port_t</b></span>                    tcp      4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_<span style="color:#C01C28"><b>http_port_t</b></span>            tcp      5988
</pre>

перезапускаем службу nginx и проверим её работу: 

>systemctl restart nginx

>systemctl status nginx

<pre> nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; <span style="color:#D7D75F"><b>disabled</b></span>; preset: <span style="color:#D7D75F"><b>disabled</b></span>)
     Active: <span style="color:#26A269"><b>active (running)</b></span> since Sat 2025-05-17 11:09:11 UTC; 1min 12s ago
    Process: 1223 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Process: 1224 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 1225 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
   Main PID: 1226 (nginx)
      Tasks: 3 (limit: 11997)
     Memory: 4.2M
        CPU: 34ms
     CGroup: /system.slice/nginx.service
             ├─<span style="color:#8A8A8A">1226 &quot;nginx: master process /usr/sbin/nginx&quot;</span>
             ├─<span style="color:#8A8A8A">1227 &quot;nginx: worker process&quot;</span>
             └─<span style="color:#8A8A8A">1228 &quot;nginx: worker process&quot;</span>

May 17 11:09:11 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
May 17 11:09:11 selinux nginx[1224]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
May 17 11:09:11 selinux nginx[1224]: nginx: configuration file /etc/nginx/nginx.conf test is successful
May 17 11:09:11 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
</pre>

Теперь удалим нестандартный порт из имеющегося 

>semanage port -d -t http_port_t -p tcp 4881

проверяем 

>semanage port -l | grep  http_port_t

<pre><span style="color:#C01C28"><b>http_port_t</b></span>                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_<span style="color:#C01C28"><b>http_port_t</b></span>            tcp      5988
</pre>

>systemctl restart nginx

<pre><span style="color:#C01C28"><b>Job for nginx.service failed because the control process exited with error code.</b></span>
<span style="color:#C01C28"><b>See &quot;systemctl status nginx.service&quot; and &quot;journalctl -xeu nginx.service&quot; for details.</b></span>
</pre>

>systemctl status nginx

<pre><span style="color:#C01C28"><b>×</b></span> nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; <span style="color:#D7D75F"><b>disabled</b></span>; preset: <span style="color:#D7D75F"><b>disabled</b></span>)
     Active: <span style="color:#C01C28"><b>failed</b></span> (Result: exit-code) since Sat 2025-05-17 11:17:11 UTC; 35s ago
   Duration: 7min 59.636s
    Process: 1244 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Process: 1245 ExecStartPre=/usr/sbin/nginx -t <span style="color:#C01C28"><b>(code=exited, status=1/FAILURE)</b></span>
        CPU: 28ms

May 17 11:17:11 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
May 17 11:17:11 selinux nginx[1245]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
May 17 11:17:11 selinux nginx[1245]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denie<span style="background-color:#D0CFCC"><span style="color:#171421">&gt;</span></span>
May 17 11:17:11 selinux nginx[1245]: nginx: configuration file /etc/nginx/nginx.conf test failed
May 17 11:17:11 selinux systemd[1]: <b>nginx.service: Control process exited, code=exited, status=1/FAILURE</b>
May 17 11:17:11 selinux systemd[1]: <span style="color:#D7D75F"><b>nginx.service: Failed with result &apos;exit-code&apos;.</b></span>
May 17 11:17:11 selinux systemd[1]: <span style="color:#C01C28"><b>Failed to start The nginx HTTP and reverse proxy server.</b></span>
</pre>

# Разрешим в SELinux работу nginx на порту TCP 4881 c помощью формирования и установки модуля SELinux:

Запускаем Nginx: 
>
<pre><span style="color:#C01C28"><b>Job for nginx.service failed because the control process exited with error code.</b></span>
<span style="color:#C01C28"><b>See &quot;systemctl status nginx.service&quot; and &quot;journalctl -xeu nginx.service&quot; for details.</b></span>
</pre>

смотрим логи SELinux, которые относятся к Nginx: 

>grep nginx /var/log/audit/audit.log

<pre>type=ADD_GROUP msg=audit(1747213473.164:667): pid=3200 uid=0 auid=1000 ses=3 subj=unconfined_u:unconfined_r:groupadd_t:s0-s0:c0.c1023 msg=&apos;op=add-group id=991 exe=&quot;/usr/sbin/groupadd&quot; hostname=? addr=? terminal=? res=success&apos;UID=&quot;root&quot; AUID=&quot;vagrant&quot; ID=&quot;<span style="color:#C01C28"><b>nginx</b></span>&quot;
type=GRP_MGMT msg=audit(1747213473.167:668): pid=3200 uid=0 auid=1000 ses=3 subj=unconfined_u:unconfined_r:groupadd_t:s0-s0:c0.c1023 msg=&apos;op=add-shadow-group id=991 exe=&quot;/usr/sbin/groupadd&quot; hostname=? addr=? terminal=? res=success&apos;UID=&quot;root&quot; AUID=&quot;vagrant&quot; ID=&quot;<span style="color:#C01C28"><b>nginx</b></span>&quot;
type=ADD_USER msg=audit(1747213473.227:669): pid=3207 uid=0 auid=1000 ses=3 subj=unconfined_u:unconfined_r:useradd_t:s0-s0:c0.c1023 msg=&apos;op=add-user acct=&quot;<span style="color:#C01C28"><b>nginx</b></span>&quot; exe=&quot;/usr/sbin/useradd&quot; hostname=? addr=? terminal=? res=success&apos;UID=&quot;root&quot; AUID=&quot;vagrant&quot;
type=SOFTWARE_UPDATE msg=audit(1747213473.661:695): pid=3179 uid=0 auid=1000 ses=3 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 msg=&apos;op=install sw=&quot;<span style="color:#C01C28"><b>nginx</b></span>-filesystem-2:1.20.1-20.el9.alma.1.noarch&quot; sw_type=rpm key_enforce=0 gpg_res=1 root_dir=&quot;/&quot; comm=&quot;yum&quot; exe=&quot;/usr/bin/python3.9&quot; hostname=? addr=? terminal=? res=success&apos;UID=&quot;root&quot; AUID=&quot;vagrant&quot;
type=SOFTWARE_UPDATE msg=audit(1747213473.662:696): pid=3179 uid=0 auid=1000 ses=3 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 msg=&apos;op=install sw=&quot;<span style="color:#C01C28"><b>nginx</b></span>-core-2:1.20.1-20.el9.alma.1.x86_64&quot; sw_type=rpm key_enforce=0 gpg_res=1 root_dir=&quot;/&quot; comm=&quot;yum&quot; exe=&quot;/usr/bin/python3.9&quot; hostname=? addr=? terminal=? res=success&apos;UID=&quot;root&quot; AUID=&quot;vagrant&quot;
type=SOFTWARE_UPDATE msg=audit(1747213473.662:698): pid=3179 uid=0 auid=1000 ses=3 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 msg=&apos;op=install sw=&quot;<span style="color:#C01C28"><b>nginx</b></span>-2:1.20.1-20.el9.alma.1.x86_64&quot; sw_type=rpm key_enforce=0 gpg_res=1 root_dir=&quot;/&quot; comm=&quot;yum&quot; exe=&quot;/usr/bin/python3.9&quot; hostname=? addr=? terminal=? res=success&apos;UID=&quot;root&quot; AUID=&quot;vagrant&quot;
type=AVC msg=audit(1747213497.847:779): avc:  denied  { name_bind } for  pid=6661 comm=&quot;<span style="color:#C01C28"><b>nginx</b></span>&quot; src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
type=SYSCALL msg=audit(1747213497.847:779): arch=c000003e syscall=49 success=no exit=-13 a0=6 a1=5633880c86b0 a2=10 a3=7ffdd089a330 items=0 ppid=1 pid=6661 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm=&quot;<span style="color:#C01C28"><b>nginx</b></span>&quot; exe=&quot;/usr/sbin/<span style="color:#C01C28"><b>nginx</b></span>&quot; subj=system_u:system_r:httpd_t:s0 key=(null)ARCH=x86_64 SYSCALL=bind AUID=&quot;unset&quot; UID=&quot;root&quot; GID=&quot;root&quot; EUID=&quot;root&quot; SUID=&quot;root&quot; FSUID=&quot;root&quot; EGID=&quot;root&quot; SGID=&quot;root&quot; FSGID=&quot;root&quot;
type=SERVICE_START msg=audit(1747213497.848:780): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg=&apos;unit=<span style="color:#C01C28"><b>nginx</b></span> comm=&quot;systemd&quot; exe=&quot;/usr/lib/systemd/systemd&quot; hostname=? addr=? terminal=? res=failed&apos;UID=&quot;root&quot; AUID=&quot;unset&quot;
type=SERVICE_START msg=audit(1747394921.790:1105): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg=&apos;unit=<span style="color:#C01C28"><b>nginx</b></span> comm=&quot;systemd&quot; exe=&quot;/usr/lib/systemd/systemd&quot; hostname=? addr=? terminal=? res=success&apos;UID=&quot;root&quot; AUID=&quot;unset&quot;
type=SERVICE_STOP msg=audit(1747396354.387:1126): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg=&apos;unit=<span style="color:#C01C28"><b>nginx</b></span> comm=&quot;systemd&quot; exe=&quot;/usr/lib/systemd/systemd&quot; hostname=? addr=? terminal=? res=success&apos;UID=&quot;root&quot; AUID=&quot;unset&quot;
type=SERVICE_START msg=audit(1747480151.655:96): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg=&apos;unit=<span style="color:#C01C28"><b>nginx</b></span> comm=&quot;systemd&quot; exe=&quot;/usr/lib/systemd/systemd&quot; hostname=? addr=? terminal=? res=success&apos;UID=&quot;root&quot; AUID=&quot;unset&quot;
type=SERVICE_STOP msg=audit(1747480631.312:99): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg=&apos;unit=<span style="color:#C01C28"><b>nginx</b></span> comm=&quot;systemd&quot; exe=&quot;/usr/lib/systemd/systemd&quot; hostname=? addr=? terminal=? res=success&apos;UID=&quot;root&quot; AUID=&quot;unset&quot;
type=AVC msg=audit(1747480631.375:100): avc:  denied  { name_bind } for  pid=1245 comm=&quot;<span style="color:#C01C28"><b>nginx</b></span>&quot; src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
type=SYSCALL msg=audit(1747480631.375:100): arch=c000003e syscall=49 success=no exit=-13 a0=6 a1=556c764f56b0 a2=10 a3=7fff4fffe620 items=0 ppid=1 pid=1245 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm=&quot;<span style="color:#C01C28"><b>nginx</b></span>&quot; exe=&quot;/usr/sbin/<span style="color:#C01C28"><b>nginx</b></span>&quot; subj=system_u:system_r:httpd_t:s0 key=(null)ARCH=x86_64 SYSCALL=bind AUID=&quot;unset&quot; UID=&quot;root&quot; GID=&quot;root&quot; EUID=&quot;root&quot; SUID=&quot;root&quot; FSUID=&quot;root&quot; EGID=&quot;root&quot; SGID=&quot;root&quot; FSGID=&quot;root&quot;
type=SERVICE_START msg=audit(1747480631.383:101): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg=&apos;unit=<span style="color:#C01C28"><b>nginx</b></span> comm=&quot;systemd&quot; exe=&quot;/usr/lib/systemd/systemd&quot; hostname=? addr=? terminal=? res=failed&apos;UID=&quot;root&quot; AUID=&quot;unset&quot;
type=AVC msg=audit(1747481060.891:106): avc:  denied  { name_bind } for  pid=1266 comm=&quot;<span style="color:#C01C28"><b>nginx</b></span>&quot; src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
type=SYSCALL msg=audit(1747481060.891:106): arch=c000003e syscall=49 success=no exit=-13 a0=6 a1=560fc81606b0 a2=10 a3=7ffeca73bda0 items=0 ppid=1 pid=1266 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm=&quot;<span style="color:#C01C28"><b>nginx</b></span>&quot; exe=&quot;/usr/sbin/<span style="color:#C01C28"><b>nginx</b></span>&quot; subj=system_u:system_r:httpd_t:s0 key=(null)ARCH=x86_64 SYSCALL=bind AUID=&quot;unset&quot; UID=&quot;root&quot; GID=&quot;root&quot; EUID=&quot;root&quot; SUID=&quot;root&quot; FSUID=&quot;root&quot; EGID=&quot;root&quot; SGID=&quot;root&quot; FSGID=&quot;root&quot;
type=SERVICE_START msg=audit(1747481060.896:107): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg=&apos;unit=<span style="color:#C01C28"><b>nginx</b></span> comm=&quot;systemd&quot; exe=&quot;/usr/lib/systemd/systemd&quot; hostname=? addr=? terminal=? res=failed&apos;UID=&quot;root&quot; AUID=&quot;unset&quot;
</pre>

Воспользуемся утилитой audit2allow для того, чтобы на основе логов SELinux сделать модуль, разрешающий работу nginx на нестандартном порту: 

>grep nginx /var/log/audit/audit.log | audit2allow -M nginx

<pre>******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp
</pre>

Audit2allow сформировал модуль, и сообщил нам команду, с помощью которой можно применить данный модуль: 

>semodule -i nginx.pp

Попробуем снова запустить nginx:
>systemctl start nginx

>systemctl status nginx

<pre><span style="color:#26A269"><b>●</b></span> nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; <span style="color:#D7D75F"><b>disabled</b></span>; preset: <span style="color:#D7D75F"><b>disabled</b></span>)
     Active: <span style="color:#26A269"><b>active (running)</b></span> since Sat 2025-05-17 11:33:38 UTC; 27s ago
    Process: 1296 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Process: 1297 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 1298 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
   Main PID: 1299 (nginx)
      Tasks: 3 (limit: 11997)
     Memory: 2.9M
        CPU: 32ms
     CGroup: /system.slice/nginx.service
             ├─<span style="color:#8A8A8A">1299 &quot;nginx: master process /usr/sbin/nginx&quot;</span>
             ├─<span style="color:#8A8A8A">1300 &quot;nginx: worker process&quot;</span>
             └─<span style="color:#8A8A8A">1301 &quot;nginx: worker process&quot;</span>

May 17 11:33:38 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
May 17 11:33:38 selinux nginx[1297]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
May 17 11:33:38 selinux nginx[1297]: nginx: configuration file /etc/nginx/nginx.conf test is successful
May 17 11:33:38 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
</pre>

После добавления модуля nginx запустился без ошибок

Просмотр всех установленных модулей: 

>semodule -l

<pre>
......
networkmanager
nginx
ninfod
nis
nova
nscd
nsd
......
</pre>

Удалить модуль можно командой: 

>semodule -r nginx

<pre>libsemanage.semanage_direct_remove_key: Removing last nginx module (no other nginx module exists at another priority).
</pre>

# Обеспечение работоспособности приложения при включенном SELinux

Выполним клонирование репозитория:

>git clone https://github.com/Nickmob/vagrant_selinux_dns_problems.git

<pre>Клонирование в «vagrant_selinux_dns_problems»...
remote: Enumerating objects: 32, done.
remote: Counting objects: 100% (32/32), done.
remote: Compressing objects: 100% (21/21), done.
remote: Total 32 (delta 9), reused 29 (delta 9), pack-reused 0 (from 0)
Получение объектов: 100% (32/32), 7.23 КиБ | 3.62 МиБ/с, готово.
Определение изменений: 100% (9/9), готово.
</pre>

Перейдём в каталог со стендом: 

>cd vagrant_selinux_dns_problems

Развернём 2 ВМ с помощью vagrant: 

>vagrant up

<pre>
......

PLAY [all] *********************************************************************

TASK [Gathering Facts] *********************************************************
<span style="color:#26A269">ok: [client]</span>

TASK [install packages] ********************************************************
<span style="color:#A2734C">changed: [client]</span>

PLAY [ns01] ********************************************************************
<span style="color:#2AA1B3">skipping: no hosts matched</span>

PLAY [client] ******************************************************************

TASK [Gathering Facts] *********************************************************
<span style="color:#26A269">ok: [client]</span>

TASK [copy resolv.conf to the client] ******************************************
<span style="color:#A2734C">changed: [client]</span>

TASK [copy rndc conf file] *****************************************************
<span style="color:#A2734C">changed: [client]</span>

TASK [copy motd to the client] *************************************************
<span style="color:#A2734C">changed: [client]</span>

TASK [copy transferkey to client] **********************************************
<span style="color:#A2734C">changed: [client]</span>

PLAY RECAP *********************************************************************
<span style="color:#A2734C">client</span>                     : <span style="color:#26A269">ok=7   </span> <span style="color:#A2734C">changed=5   </span> unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
</pre>

После того, как стенд развернется, проверим ВМ с помощью команды: 

>vagrant status

<pre>Current machine states:

ns01                      running (virtualbox)
client                    running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
</pre>

Подключимся к клиенту: 

>vagrant ssh client

<pre>###############################
### Welcome to the DNS lab! ###
###############################

- Use this client to test the enviroment
- with dig or nslookup. Ex:
    dig @192.168.50.10 ns01.dns.lab

- nsupdate is available in the ddns.lab zone. Ex:
    nsupdate -k /etc/named.zonetransfer.key
    server 192.168.50.10
    zone ddns.lab 
    update add www.ddns.lab. 60 A 192.168.50.15
    send

- rndc is also available to manage the servers
    rndc -c ~/rndc.conf reload

###############################
### Enjoy! ####################
###############################
Last login: Sun May 18 06:02:57 2025 from 10.0.2.2
</pre>

Попробуем внести изменения в зону: 

>nsupdate -k /etc/named.zonetransfer.key

>server 192.168.50.10

>zone ddns.lab

>update add www.ddns.lab. 60 A 192.168.50.15

>send

<pre>; Communication with 192.168.50.10#53 failed: timed out</pre>

>quit

Изменения внести не получилось. Давайте посмотрим логи SELinux, чтобы понять в чём может быть проблема. Для этого воспользуемся утилитой audit2why: 	

>sudo -i

>cat /var/log/audit/audit.log | audit2why

Тут мы видим, что на клиенте отсутствуют ошибки. 

<pre>type=AVC msg=audit(1747548075.306:721): avc:  denied  { dac_read_search } for  pid=3520 comm=&quot;20-chrony-dhcp&quot; capability=2  scontext=system_u:system_r:NetworkManager_dispatcher_chronyc_t:s0 tcontext=system_u:system_r:NetworkManager_dispatcher_chronyc_t:s0 tclass=capability permissive=0

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.

type=AVC msg=audit(1747548075.306:721): avc:  denied  { dac_override } for  pid=3520 comm=&quot;20-chrony-dhcp&quot; capability=1  scontext=system_u:system_r:NetworkManager_dispatcher_chronyc_t:s0 tcontext=system_u:system_r:NetworkManager_dispatcher_chronyc_t:s0 tclass=capability permissive=0

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.
</pre>

Не закрывая сессию на клиенте, подключимся к серверу ns01 и проверим логи SELinux:

смотрим лог

>cat /var/log/audit/audit.log | audit2why

<pre>type=AVC msg=audit(1747547641.957:720): avc:  denied  { dac_read_search } for  pid=3525 comm=&quot;20-chrony-dhcp&quot; capability=2  scontext=system_u:system_r:NetworkManager_dispatcher_chronyc_t:s0 tcontext=system_u:system_r:NetworkManager_dispatcher_chronyc_t:s0 tclass=capability permissive=0

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.

type=AVC msg=audit(1747547641.957:720): avc:  denied  { dac_override } for  pid=3525 comm=&quot;20-chrony-dhcp&quot; capability=1  scontext=system_u:system_r:NetworkManager_dispatcher_chronyc_t:s0 tcontext=system_u:system_r:NetworkManager_dispatcher_chronyc_t:s0 tclass=capability permissive=0

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.
</pre>

В логах мы видим, что ошибка в контексте безопасности. Целевой контекст named_conf_t.

Для сравнения посмотрим существующую зону (localhost) и её контекст:

>ls -alZ /var/named/named.localhost

<pre>rw-r-----. 1 root named system_u:object_r:named_zone_t:s0 152 Feb 19 16:04 /var/named/named.localhost</pre>

У наших конфигов в /etc/named вместо типа named_zone_t используется тип named_conf_t.
Проверим данную проблему в каталоге /etc/named:

>ls -laZ /etc/named

<pre>total 12
drwxr-x---.  2 root named system_u:object_r:named_conf_t:s0    6 Feb 19 16:04 <span style="color:#12488B"><b>.</b></span>
drwxr-xr-x. 83 root root  system_u:object_r:etc_t:s0        8192 May 18 08:33 <span style="color:#12488B"><b>..</b></span>
drw-rwx---. root named unconfined_u:object_r:named_conf_t:s0   dynamic<span style="color:#12488B"><b>..</b></span>
-rw-rw----. root named system_u:object_r:named_conf_t:s0       named.50.168.192.rev<span style="color:#12488B"><b>..</b></span>
-rw-rw----. root named system_u:object_r:named_conf_t:s0       named.dns.lab<span style="color:#12488B"><b>..</b></span>
-rw-rw----. root named system_u:object_r:named_conf_t:s0       named.dns.lab.view1<span style="color:#12488B"><b>..</b></span>
-rw-rw----. root named system_u:object_r:named_conf_t:s0       named.newdns.lab<span style="color:#12488B"><b>..</b></span>

</pre>

<pre>/etc/rndc.*                                        regular file       system_u:object_r:<span style="color:#C01C28"><b>named</b></span>_conf_t:s0 

/var/<span style="color:#C01C28"><b>named</b></span>(/.*)?                                   all files          system_u:object_r:<span style="color:#C01C28"><b>named</b></span>_zone_t:s0
</pre>

Изменим тип контекста безопасности для каталога /etc/named:

>sudo chcon -R -t named_zone_t /etc/named

<pre>total 12
drwxr-x---.  2 root named system_u:object_r:named_zone_t:s0    6 Feb 19 16:04 <span style="color:#12488B"><b>.</b></span>
drwxr-xr-x. 83 root root  system_u:object_r:etc_t:s0        8192 May 18 08:33 <span style="color:#12488B"><b>..</b></span>
drw-rwx---. root named unconfined_u:object_r:named_zone_t:s0 dynamic<span style="color:#12488B"><b>..</b></span>
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.50.168.192.rev<span style="color:#12488B"><b>..</b></span>
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab<span style="color:#12488B"><b>..</b></span>
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab.view1<span style="color:#12488B"><b>..</b></span>
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.newdns.lab<span style="color:#12488B"><b>..</b></span>
</pre>

Попробуем снова внести изменения с клиента: 

>nsupdate -k /etc/named.zonetransfer.key
>server 192.168.50.10
>zone ddns.lab
>update add www.ddns.lab. 60 A 192.168.50.15
>send
>quit 

>dig www.ddns.lab

<pre>;  <<>> DiG 9.16.23-RH <<>> www.ddns.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 17460
;; flags: qr rd ra ad; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
; EDE: 29: (Result synthesized from aggressive NSEC cache (RFC8198))

;; ANSWER SECTION:
www.ddns.lab.       60  IN  A   192.168.50.15

;; AUTHORITY SECTION:
ddns.lab.       3600    IN  NS  ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.       3600    IN  A   192.168.50.10

;; Query time: 1 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Sun May 18 11:00:32 UTC 2025
;; MSG SIZE  rcvd: 177
</pre>

Видим, что изменения применились. Попробуем перезагрузить хосты и ещё раз сделать запрос с помощью dig: 

>dig @192.168.50.10 www.ddns.lab
<pre>
; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.7 <<>> @192.168.50.10 www.ddns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 17453
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2


;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.ddns.lab.          IN  A


;; ANSWER SECTION:
www.ddns.lab.       60  IN  A   192.168.50.15


;; AUTHORITY SECTION:
ddns.lab.       3600    IN  NS  ns01.dns.lab.


;; ADDITIONAL SECTION:
ns01.dns.lab.       3600    IN  A   192.168.50.10


;; Query time: 2 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Sun May 18 11:34:12 UTC 2025
;; MSG SIZE  rcvd: 177
</pre>

Для того, чтобы вернуть правила обратно, можно ввести команду: 

>restorecon -v -R /etc/named

<pre>
restorecon reset /etc/named context system_u:object_r:named_zone_t:s0->system_u:object_r:named_conf_t:s0
restorecon reset /etc/named/named.dns.lab.view1 context system_u:object_r:named_zone_t:s0->system_u:object_r:named_conf_t:s0
restorecon reset /etc/named/named.dns.lab context system_u:object_r:named_zone_t:s0->system_u:object_r:named_conf_t:s0
restorecon reset /etc/named/dynamic context unconfined_u:object_r:named_zone_t:s0->unconfined_u:object_r:named_conf_t:s0
restorecon reset /etc/named/dynamic/named.ddns.lab context system_u:object_r:named_zone_t:s0->system_u:object_r:named_conf_t:s0
restorecon reset /etc/named/dynamic/named.ddns.lab.view1 context system_u:object_r:named_zone_t:s0->system_u:object_r:named_conf_t:s0
restorecon reset /etc/named/dynamic/named.ddns.lab.view1.jnl context system_u:object_r:named_zone_t:s0->system_u:object_r:named_conf_t:s0
restorecon reset /etc/named/named.newdns.lab context system_u:object_r:named_zone_t:s0->system_u:object_r:named_conf_t:s0
restorecon reset /etc/named/named.50.168.192.rev context system_u:object_r:named_zone_t:s0->system_u:object_r:named_conf_t:s0
[root@ns01 ~]#
</pre>