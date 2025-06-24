# PAM
Создаем папку для проекта и заходим в нее
>mkdir pam && cd pam
Далее создаем файл Vagrantfile и вносим в него конфигурицию для новой вм
>nano Vagrantfile
<pre>
MACHINES = {
  :"pam" => {
              :box_name => "ubuntu/jammy64",
              :cpus => 2,
              :memory => 1024,
              :ip => "192.168.57.10",
            }
}

Vagrant.configure("2") do |config|
  MACHINES.each do |boxname, boxconfig|
    config.vm.synced_folder ".", "/vagrant", disabled: true
    config.vm.network "private_network", ip: boxconfig[:ip]
    config.vm.define boxname do |box|
      box.vm.box = boxconfig[:box_name]
      box.vm.box_version = boxconfig[:box_version]
      box.vm.host_name = boxname.to_s

      box.vm.provider "virtualbox" do |v|
        v.memory = boxconfig[:memory]
        v.cpus = boxconfig[:cpus]
      end
      box.vm.provision "shell", inline: <<-SHELL
        sed -i 's/^#\?PasswordAuthentication.*$/PasswordAuthentication yes/' /etc/ssh/sshd_config && sed -i 's/^KbdInteractiveAuthentication.*$/KbdInteractiveAuthentication yes/' /etc/ssh/sshd_config

          systemctl restart sshd.service
  	  SHELL
    end
  end
end
</pre>

Запускаем создание вм

>vagrant up

после создания вм, подключаемся к ней

>vagrant ssh

переходим в root

>sudo -i

Создаем пользователей

>sudo useradd -m -s /bin/bash otusadm  && ssudo useradd -m -s /bin/bash otus

меняем пароли к ним

>echo "otus:Otus2025!" | sudo chpasswd && echo "otusadm:Otus2025!" | sudo chpasswd

создаем группу admin

>sudo groupadd -f admin

Добавляем пользователей vagrant,root и otusadm в группу admin

>usermod otusadm -a -G admin && usermod root -a -G admin && usermod vagrant -a -G admin

Проверяем подключение по SSH для новых пользователей

>ssh otus@192.168.57.10

>ssh otusadm@192.168.57.10

После успешного входы, выходим и настроим правило по которому все пользователи кроме тех, что указаны в группе admin не смогут подключаться в выходные дни:

Для этого, проверим, что пользователи root, vagrant и otusadm есть в группе admin:

>cat /etc/group | grep admin

<pre><span style="color:#C01C28"><b>admin</b></span>:x:118:root,vagrant,otusadm
</pre>

поом создадим скрипт контроля и будем использовать модуль pam_exec

>nano /usr/local/bin/login.sh

со следующим содержимым

<pre>

#!/bin/bash
#Первое условие: если день недели суббота или воскресенье
if [ $(date +%a) = "Sat" ] || [ $(date +%a) = "Sun" ]; then
 #Второе условие: входит ли пользователь в группу admin
 if getent group admin | grep -qw "$PAM_USER"; then
        #Если пользователь входит в группу admin, то он может подключиться
        exit 0
      else
        #Иначе ошибка (не сможет подключиться)
        exit 1
    fi
  #Если день не выходной, то подключиться может любой пользователь
  else
    exit 0
fi
</pre>

Добавим права на исполнение файла

>chmod +x /usr/local/bin/login.sh

Укажем в файле /etc/pam.d/sshd модуль pam_exec и наш скрипт:

<pre># PAM configuration for the Secure Shell service

# Standard Un*x authentication.
@include common-auth

# Добавляем проверку вашего скрипта (должно быть ДО common-auth, если хотим перекрывать стандартную аутентификацию)
auth       required     pam_exec.so debug /usr/local/bin/login.sh

# Disallow non-root logins when /etc/nologin exists.
account    required     pam_nologin.so

# Uncomment and edit /etc/security/access.conf if you need to set complex
# access limits that are hard to express in sshd_config.
# account  required     pam_access.so

# Standard Un*x authorization.
@include common-account

# SELinux needs to be the first session rule.  This ensures that any
# lingering context has been cleared.  Without this it is possible that a
# module could execute code in the wrong domain.
session [success=ok ignore=ignore module_unknown=ignore default=bad]        pam_selinux.so close

# Set the loginuid process attribute.
session    required     pam_loginuid.so

# Create a new session keyring.
session    optional     pam_keyinit.so force revoke

# Standard Un*x session setup and teardown.
@include common-session

# Print the message of the day upon successful login.
# This includes a dynamically generated part from /run/motd.dynamic
# and a static (admin-editable) part from /etc/motd.
session    optional     pam_motd.so  motd=/run/motd.dynamic
session    optional     pam_motd.so noupdate

# Print the status of the user&apos;s mailbox upon successful login.
session    optional     pam_mail.so standard noenv # [1]

# Set up user limits from /etc/security/limits.conf.
session    required     pam_limits.so

# Read environment variables from /etc/environment and
# /etc/security/pam_env.conf.
session    required     pam_env.so # [1]
# In Debian 4.0 (etch), locale-related environment variables were moved to
# /etc/default/locale, so read that as well.
session    required     pam_env.so user_readenv=1 envfile=/etc/default/locale

# SELinux needs to intervene at login time to ensure that the process starts
# in the proper default security context.  Only sessions which are intended
# to run in the user&apos;s context should be run after this.
session [success=ok ignore=ignore module_unknown=ignore default=bad]        pam_selinux.so open

# Standard Un*x password updating.
@include common-password
</pre>

Что бы проверить работу скрипта, переустановим время на компьютере на выходной

>timedatectl set-ntp off

>timedatectl set-time "2022-08-27 12:30:00"

>date

<pre>Sat Aug 27 12:30:04 UTC 2022</pre>

Пробуем подключиться пользователем otus

<pre>ssh otus@192.168.57.10
(otus@192.168.57.10) Password: 
/usr/local/bin/login.sh failed: exit code 1

(otus@192.168.57.10) Password: 
(otus@192.168.57.10) Password: 
otus@192.168.57.10: Permission denied (publickey,keyboard-interactive).
</pre>

Пробуем подключиться пользователем otusadm

<pre>ssh otusadm@192.168.57.10
(otusadm@192.168.57.10) Password: 
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-142-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Tue Jun 24 14:12:43 UTC 2025

  System load:  0.01              Processes:               112
  Usage of /:   5.5% of 38.70GB   Users logged in:         1
  Memory usage: 21%               IPv4 address for enp0s3: 10.0.2.15
  Swap usage:   0%

 * Strictly confined Kubernetes makes edge and IoT secure. Learn how MicroK8s
   just raised the bar for easy, resilient and secure K8s cluster deployment.

   https://ubuntu.com/engage/secure-kubernetes-at-the-edge

Expanded Security Maintenance for Applications is not enabled.

10 updates can be applied immediately.
To see these additional updates run: apt list --upgradable

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


Last login: Tue Jun 24 14:08:50 2025 from 192.168.57.1
</pre>


