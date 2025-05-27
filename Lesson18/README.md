# Vagrant-стенд для обновления ядра и создания образа системы

Создадим Vagrantfile, в котором будут указаны параметры нашей ВМ:

>mkdir vagrant2 && cd vagrant2

>nano vagrantfile

зальем содержимое в файл

<pre>
# Описываем Виртуальные машины
MACHINES = {
  # Указываем имя ВМ "kernel update"
  :"kernel-update" => {
              #Какой vm box будем использовать
              :box_name => "generic/centos8s",
              #Указываем box_version
              :box_version => "4.3.4",
              #Указываем количество ядер ВМ
              :cpus => 2,
              #Указываем количество ОЗУ в мегабайтах
              :memory => 1024,
            }
}

Vagrant.configure("2") do |config|
  MACHINES.each do |boxname, boxconfig|
    # Отключаем проброс общей папки в ВМ
    config.vm.synced_folder ".", "/vagrant", disabled: true
    # Применяем конфигурацию ВМ
    config.vm.define boxname do |box|
      box.vm.box = boxconfig[:box_name]
      box.vm.box_version = boxconfig[:box_version]
      box.vm.host_name = boxname.to_s
      box.vm.provider "virtualbox" do |v|
        v.memory = boxconfig[:memory]
        v.cpus = boxconfig[:cpus]
      end
    end
  end
end
</pre>

запускаем виртуальную машину

>vagrant up

После создания и запуска подключаемся внутрь системы

>vagrant ssh

<pre>[vagrant@kernel-update ~]$ uname -r
4.18.0-516.el8.x86_64
</pre>

Далее подключим репозиторий, откуда возьмём необходимую версию ядра:
Сначала заменим стандартные репозитори на архивные

>sudo sed -i 's|mirrorlist=http://mirrorlist.centos.org|#mirrorlist=http://mirrorlist.centos.org|g' /etc/yum.repos.d/CentOS-*
sudo sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*

Обновим кэш

>sudo dnf clean all
>sudo dnf makecache

и подключим

<pre>sudo yum install -y https://www.elrepo.org/elrepo-release-8.el8.elrepo.noarch.rpm
Failed to set locale, defaulting to C.UTF-8
Last metadata expiration check: 0:01:16 ago on Tue May 27 12:03:37 2025.
elrepo-release-8.el8.elrepo.noarch.rpm                                   23 kB/s |  19 kB     00:00    
Dependencies resolved.
========================================================================================================
 Package                   Architecture      Version                      Repository               Size
========================================================================================================
Installing:
 <span style="color:#26A269">elrepo-release           </span> noarch            8.4-2.el8.elrepo             @commandline             19 k

Transaction Summary
========================================================================================================
Install  1 Package

Total size: 19 k
Installed size: 8.3 k
Downloading Packages:
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                                                1/1 
  Installing       : elrepo-release-8.4-2.el8.elrepo.noarch                                         1/1 
  Verifying        : elrepo-release-8.4-2.el8.elrepo.noarch                                         1/1 

Installed:
  elrepo-release-8.4-2.el8.elrepo.noarch                                                                

Complete!
</pre>

Установим последнее ядро из репозитория elrepo-kernel:

>sudo yum --enablerepo elrepo-kernel install kernel-ml -y

<pre>sudo yum --enablerepo elrepo-kernel install kernel-ml -y
Failed to set locale, defaulting to C.UTF-8
CentOS Stream 8 - AppStream                                             7.2 kB/s | 4.4 kB     00:00    
CentOS Stream 8 - BaseOS                                                7.2 kB/s | 3.9 kB     00:00    
CentOS Stream 8 - Extras                                                1.9 kB/s | 2.9 kB     00:01    
CentOS Stream 8 - Extras common packages                                5.8 kB/s | 3.0 kB     00:00    
ELRepo.org Community Enterprise Linux Repository - el8                  290 kB/s | 264 kB     00:00    
ELRepo.org Community Enterprise Linux Kernel Repository - el8           1.8 MB/s | 2.2 MB     00:01    
Extra Packages for Enterprise Linux 8 - x86_64                          110 kB/s |  36 kB     00:00    
Extra Packages for Enterprise Linux 8 - Next - x86_64                   3.6 kB/s | 1.9 kB     00:00    
Dependencies resolved.
========================================================================================================
 Package                    Architecture    Version                        Repository              Size
========================================================================================================
Installing:
 <span style="color:#26A269"><b>kernel-ml                 </b></span> x86_64          6.14.8-1.el8.elrepo            elrepo-kernel          150 k
Installing dependencies:
 <span style="color:#26A269"><b>kernel-ml-core            </b></span> x86_64          6.14.8-1.el8.elrepo            elrepo-kernel           66 M
 <span style="color:#26A269"><b>kernel-ml-modules         </b></span> x86_64          6.14.8-1.el8.elrepo            elrepo-kernel           62 M

Transaction Summary
========================================================================================================
Install  3 Packages

Total download size: 128 M
Installed size: 174 M
Downloading Packages:
(1/3): kernel-ml-6.14.8-1.el8.elrepo.x86_64.rpm                         435 kB/s | 150 kB     00:00    
(2/3): kernel-ml-core-6.14.8-1.el8.elrepo.x86_64.rpm                    3.0 MB/s |  66 MB     00:21    
(3/3): kernel-ml-modules-6.14.8-1.el8.elrepo.x86_64.rpm                 2.5 MB/s |  62 MB     00:24    
--------------------------------------------------------------------------------------------------------
Total                                                                   5.2 MB/s | 128 MB     00:24     
ELRepo.org Community Enterprise Linux Kernel Repository - el8           1.6 MB/s | 1.7 kB     00:00    
Importing GPG key 0xBAADAE52:
 Userid     : &quot;elrepo.org (RPM Signing Key for elrepo.org) &lt;secure@elrepo.org&gt;&quot;
 Fingerprint: 96C0 104F 6315 4731 1E0B B1AE 309B C305 BAAD AE52
 From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-elrepo.org
Key imported successfully
ELRepo.org Community Enterprise Linux Kernel Repository - el8           3.0 MB/s | 3.1 kB     00:00    
Importing GPG key 0xEAA31D4A:
 Userid     : &quot;elrepo.org (RPM Signing Key v2 for elrepo.org) &lt;secure@elrepo.org&gt;&quot;
 Fingerprint: B8A7 5587 4DA2 40C9 DAC4 E715 5160 0989 EAA3 1D4A
 From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-v2-elrepo.org
Key imported successfully
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                                                1/1 
  Installing       : kernel-ml-core-6.14.8-1.el8.elrepo.x86_64                                      1/3 
  Running scriptlet: kernel-ml-core-6.14.8-1.el8.elrepo.x86_64                                      1/3 
  Installing       : kernel-ml-modules-6.14.8-1.el8.elrepo.x86_64                                   2/3 
  Running scriptlet: kernel-ml-modules-6.14.8-1.el8.elrepo.x86_64                                   2/3 
  Installing       : kernel-ml-6.14.8-1.el8.elrepo.x86_64                                           3/3 
  Running scriptlet: kernel-ml-core-6.14.8-1.el8.elrepo.x86_64                                      3/3 
dracut: Disabling early microcode, because kernel does not support it. CONFIG_MICROCODE_[AMD|INTEL]!=y
dracut: Disabling early microcode, because kernel does not support it. CONFIG_MICROCODE_[AMD|INTEL]!=y

  Running scriptlet: kernel-ml-6.14.8-1.el8.elrepo.x86_64                                           3/3 
  Verifying        : kernel-ml-6.14.8-1.el8.elrepo.x86_64                                           1/3 
  Verifying        : kernel-ml-core-6.14.8-1.el8.elrepo.x86_64                                      2/3 
  Verifying        : kernel-ml-modules-6.14.8-1.el8.elrepo.x86_64                                   3/3 

Installed:
  kernel-ml-6.14.8-1.el8.elrepo.x86_64                 kernel-ml-core-6.14.8-1.el8.elrepo.x86_64        
  kernel-ml-modules-6.14.8-1.el8.elrepo.x86_64        

Complete!
</pre>

>uname -r

<pre>6.14.8-1.el8.elrepo.x86_64</pre>


