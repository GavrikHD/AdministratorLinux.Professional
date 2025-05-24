# Подготовка виртуальной машины Ansible

После установки ansible, проверим ее версию 

>ansible --version

<pre>aansible [core 2.14.18]
  config file = None
  configured module search path = [&apos;/home/roman/.ansible/plugins/modules&apos;, &apos;/usr/share/ansible/plugins/modules&apos;]
  ansible python module location = /usr/lib/python3/dist-packages/ansible
  ansible collection location = /home/roman/.ansible/collections:/usr/share/ansible/collections
  executable location = /usr/bin/ansible
  python version = 3.11.2 (main, Nov 30 2024, 21:22:50) [GCC 12.2.0] (/usr/bin/python3)
  jinja version = 3.1.2
  libyaml = True
</pre>

Создадим каталог ansible

>mkdir ansible && cd ansible

перейдя в него создадим vagrandfile и зальем туда конфигурацию нашей виртуальной машины

>nano vagrantfile

<pre>># -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
  :nginx => {
        :box_name => "generic/ubuntu2204",
        :vm_name => "nginx",
        :net => [
           ["192.168.11.150",  2, "255.255.255.0", "mynet"],
        ]
  }
}

Vagrant.configure("2") do |config|

  MACHINES.each do |boxname, boxconfig|

    config.vm.define boxname do |box|
   
      box.vm.box = boxconfig[:box_name]
      box.vm.host_name = boxconfig[:vm_name]
      
      box.vm.provider "virtualbox" do |v|
        v.memory = 768
        v.cpus = 1
       end

      boxconfig[:net].each do |ipconf|
        box.vm.network("private_network", ip: ipconf[0], adapter: ipconf[1], netmask: ipconf[2], virtualbox__intnet: ipconf[3])
      end

      if boxconfig.key?(:public)
        box.vm.network "public_network", boxconfig[:public]
      end

      box.vm.provision "shell", inline: <<-SHELL
        mkdir -p ~root/.ssh
        cp ~vagrant/.ssh/auth* ~root/.ssh
        sudo sed -i 's/\#PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config
        systemctl restart sshd
      SHELL
    end
  end
end
</pre>

поднимим нашу вм

>vagrant up

Далее создаем inventory файл 

>mkdir staging

>nano staging/hosts

заносим в него содержимое 

<pre><span style="color:#26A269"><b>web]</b></span>
nginx ansible_host=127.0.0.1 ansible_port=2222 ansible_user=vagrant ansible_private_key_file=/home/roma<span style="background-color:#D0CFCC"><span style="color:#171421">&gt;</span></span>
</pre>

убедимся что ansible может управлять вм

>ansible -i ~/ansible/staging/hosts web -m ping -vvv

<pre>
...
<span style="color:#26A269">nginx | SUCCESS =&gt; {</span>
<span style="color:#26A269">    &quot;ansible_facts&quot;: {</span>
<span style="color:#26A269">        &quot;discovered_interpreter_python&quot;: &quot;/usr/bin/python3&quot;</span>
<span style="color:#26A269">    },</span>
<span style="color:#26A269">    &quot;changed&quot;: false,</span>
<span style="color:#26A269">    &quot;invocation&quot;: {</span>
<span style="color:#26A269">        &quot;module_args&quot;: {</span>
<span style="color:#26A269">            &quot;data&quot;: &quot;pong&quot;</span>
<span style="color:#26A269">        }</span>
<span style="color:#26A269">    },</span>
<span style="color:#26A269">    &quot;ping&quot;: &quot;pong&quot;</span>
<span style="color:#26A269">}</span>
</pre>

создадим файл ansible.cfg со следующим содержанием:

<pre><span style="color:#26A269"><b>[defaults]</b></span>
inventory = staging/hosts
remote_user = vagrant
host_key_checking = False
retry_files_enabled = False
Теперь из инвентори можно убрать информацию о пользователе:
<span style="color:#26A269"><b>[web]</b></span>
nginx ansible_host=127.0.0.1 ansible_port=2222
ansible_private_key_file=.vagrant/machines/nginx/virtualbox/private_key
</pre>

Еще раз убедимся, что управляемый хост доступе, только теперь без
явного указаниā inventory файла:

>ansible nginx -m ping

<pre><span style="color:#26A269">nginx | SUCCESS =&gt; {</span>
<span style="color:#26A269">    &quot;ansible_facts&quot;: {</span>
<span style="color:#26A269">        &quot;discovered_interpreter_python&quot;: &quot;/usr/bin/python3&quot;</span>
<span style="color:#26A269">    },</span>
<span style="color:#26A269">    &quot;changed&quot;: false,</span>
<span style="color:#26A269">    &quot;ping&quot;: &quot;pong&quot;</span>
<span style="color:#26A269">}</span>
</pre>

Теперь начнем конфигурировать хост. Посмотрим какое ядро установлено на хосте:

>ansible nginx -m command -a "uname -r"

<pre><span style="color:#A2734C">nginx | CHANGED | rc=0 &gt;&gt;</span>
<span style="color:#A2734C">5.15.0-140-generic</span>
</pre>

Проверим статус сервиса firewalld

>ansible nginx -m systemd -a name=firewalld

<pre><span style="color:#26A269">nginx | SUCCESS =&gt; {</span>
<span style="color:#26A269">    &quot;ansible_facts&quot;: {</span>
<span style="color:#26A269">        &quot;discovered_interpreter_python&quot;: &quot;/usr/bin/python3&quot;</span>
<span style="color:#26A269">    },</span>
<span style="color:#26A269">    &quot;changed&quot;: false,</span>
<span style="color:#26A269">    &quot;name&quot;: &quot;firewalld&quot;,</span>
<span style="color:#26A269">    &quot;status&quot;: {</span>
<span style="color:#26A269">        &quot;ActiveEnterTimestamp&quot;: &quot;n/a&quot;,</span>
<span style="color:#26A269">        &quot;ActiveEnterTimestampMonotonic&quot;: &quot;0&quot;,</span>
<span style="color:#26A269">        &quot;ActiveExitTimestamp&quot;: &quot;n/a&quot;,</span>
<span style="color:#26A269">        &quot;ActiveExitTimestampMonotonic&quot;: &quot;0&quot;,</span>
<span style="color:#26A269">        &quot;ActiveState&quot;: &quot;inactive&quot;,</span>
<span style="color:#26A269">        &quot;AllowIsolate&quot;: &quot;no&quot;,</span>
<span style="color:#26A269">        &quot;AssertResult&quot;: &quot;no&quot;,</span>
<span style="color:#26A269">        &quot;AssertTimestamp&quot;: &quot;n/a&quot;,</span>
<span style="color:#26A269">        &quot;AssertTimestampMonotonic&quot;: &quot;0&quot;,</span>
<span style="color:#26A269">        &quot;BlockIOAccounting&quot;: &quot;no&quot;,</span>
...
</pre>

Теперь начнем писать Playbook для установки nginx

>nano nginx.yml 

<pre><span style="background-color:#D0CFCC"><span style="color:#171421">  GNU nano 7.2                                    nginx.yml                                             </span></span>
<span style="color:#00AFD7"><b>---</b></span>
<span style="color:#A2734C"><b>- </b></span><span style="color:#33DA7A">name</span>: NGINX | Install and start NGINX
  <span style="color:#33DA7A">hosts</span>: nginx
  <span style="color:#33DA7A">become</span>: <span style="color:#C061CB">true</span>

  <span style="color:#33DA7A">tasks</span>:
<span style="color:#A2734C"><b>    - </b></span><span style="color:#33DA7A">name</span>: update
      <span style="color:#33DA7A">apt</span>:
        update_cache=yes

<span style="color:#A2734C"><b>    - </b></span><span style="color:#33DA7A">name</span>: NGINX | Install NGINX
      <span style="color:#33DA7A">apt</span>:
        <span style="color:#33DA7A">name</span>: nginx
        <span style="color:#33DA7A">state</span>: latest


</pre>

<pre>ansible-playbook nginx.yml

PLAY [NGINX | Install and start NGINX] *****************************************************************

TASK [Gathering Facts] *********************************************************************************
<span style="color:#26A269">ok: [nginx]</span>

TASK [update] ******************************************************************************************
<span style="color:#A2734C">changed: [nginx]</span>

TASK [NGINX | Install NGINX] ***************************************************************************
<span style="color:#A2734C">changed: [nginx]</span>

PLAY RECAP *********************************************************************************************
<span style="color:#A2734C">nginx</span>                      : <span style="color:#26A269">ok=3   </span> <span style="color:#A2734C">changed=2   </span> unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

</pre>

Далее добавим шаблон для конфига NGINX и модуль, который будет
копировать этот шаблон на хост, пропишем в Playbook необходимую нам переменную. Нам нужно чтобы NGINX слушал на порту 8080:

Создадим папку templates и файл шаблона nginx.conf.j2

>mkdir templates

>nano templates/nginx.conf.j2

вот с этим содержимым

<pre># {{ ansible_managed }}
events {
    worker_connections 1024;
}

http {
    server {
        listen      {{ nginx_listen_port }} default_server;
        server_name  default_server;
        root         /usr/share/nginx/html;

        location / {
        }
    }
}
</pre>

Теперь создадим handler и добавим notify к копирования шаблона. Теперь каждый раз когда конфиг будет изменяться - сервис перезагрузиться. Весь файл теперь выглядит так

<pre><span style="color:#00AFD7"><b>---</b></span>
<span style="color:#A2734C"><b>- </b></span><span style="color:#33DA7A">name</span>: NGINX | Install and configure NGINX
  <span style="color:#33DA7A">hosts</span>: nginx
  <span style="color:#33DA7A">become</span>: <span style="color:#C061CB">true</span>
  <span style="color:#33DA7A">vars</span>:
    <span style="color:#33DA7A">nginx_listen_port</span>: <span style="color:#C061CB">8080</span>

  <span style="color:#33DA7A">tasks</span>:
<span style="color:#A2734C"><b>    - </b></span><span style="color:#33DA7A">name</span>: update
      <span style="color:#33DA7A">apt</span>:
        update_cache=yes
      <span style="color:#33DA7A">tags</span>:
<span style="color:#A2734C">       </span><span style="color:#A2734C"><b> - </b></span>update apt

<span style="color:#A2734C"><b>    - </b></span><span style="color:#33DA7A">name</span>: NGINX | Install NGINX
      <span style="color:#33DA7A">apt</span>:
        <span style="color:#33DA7A">name</span>: nginx
        <span style="color:#33DA7A">state</span>: latest
      <span style="color:#33DA7A">notify</span>:
<span style="color:#A2734C">       </span><span style="color:#A2734C"><b> - </b></span>restart nginx
      <span style="color:#33DA7A">tags</span>:
<span style="color:#A2734C">       </span><span style="color:#A2734C"><b> - </b></span>nginx-package

<span style="color:#A2734C"><b>    - </b></span><span style="color:#33DA7A">name</span>: NGINX | Create NGINX config file from template
      <span style="color:#33DA7A">template</span>:
        <span style="color:#33DA7A">src</span>: templates/nginx.conf.j2
        <span style="color:#33DA7A">dest</span>: /tmp/nginx.conf
      <span style="color:#33DA7A">notify</span>:
<span style="color:#A2734C">       </span><span style="color:#A2734C"><b> - </b></span>restart nginx
      <span style="color:#33DA7A">tags</span>:
<span style="color:#A2734C">       </span><span style="color:#A2734C"><b> - </b></span>nginx-configuration

  <span style="color:#33DA7A">handlers</span>:
<span style="color:#A2734C"><b>    - </b></span><span style="color:#33DA7A">name</span>: restart nginx
      <span style="color:#33DA7A">systemd</span>:
        <span style="color:#33DA7A">name</span>: nginx
        <span style="color:#33DA7A">state</span>: restarted
        <span style="color:#33DA7A">enabled</span>: <span style="color:#C061CB">yes</span>

<span style="color:#A2734C"><b>    - </b></span><span style="color:#33DA7A">name</span>: reload nginx
      <span style="color:#33DA7A">systemd</span>:
        <span style="color:#33DA7A">name</span>: nginx
        <span style="color:#33DA7A">state</span>: reloaded
</pre>

Запустим его

>ansible-playbook nginx.yml

<pre>PLAY [NGINX | Install and configure NGINX] *************************************************************

TASK [Gathering Facts] *********************************************************************************
<span style="color:#26A269">ok: [nginx]</span>

TASK [update] ******************************************************************************************
<span style="color:#A2734C">changed: [nginx]</span>

TASK [NGINX | Install NGINX] ***************************************************************************
<span style="color:#26A269">ok: [nginx]</span>

TASK [NGINX | Create NGINX config file from template] **************************************************
<span style="color:#26A269">ok: [nginx]</span>

PLAY RECAP *********************************************************************************************
<span style="color:#A2734C">nginx</span>                      : <span style="color:#26A269">ok=4   </span> <span style="color:#A2734C">changed=1   </span> unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
</pre>

проверяем сайт

>curl http://192.168.11.150:8080 

<pre>&lt;!DOCTYPE html&gt;
&lt;html&gt;
&lt;head&gt;
&lt;title&gt;Welcome to nginx!&lt;/title&gt;
&lt;style&gt;
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
&lt;/style&gt;
&lt;/head&gt;
&lt;body&gt;
&lt;h1&gt;Welcome to nginx!&lt;/h1&gt;
&lt;p&gt;If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.&lt;/p&gt;

&lt;p&gt;For online documentation and support please refer to
&lt;a href=&quot;http://nginx.org/&quot;&gt;nginx.org&lt;/a&gt;.&lt;br/&gt;
Commercial support is available at
&lt;a href=&quot;http://nginx.com/&quot;&gt;nginx.com&lt;/a&gt;.&lt;/p&gt;

&lt;p&gt;&lt;em&gt;Thank you for using nginx.&lt;/em&gt;&lt;/p&gt;
&lt;/body&gt;
&lt;/html&gt;
</pre>



