# Статическая и динамическая маршрутизация, OSPF

создаем Vagrantfile

<pre>MACHINES = {
  :router1 =&gt; {
        :box_name =&gt; &quot;ubuntu/focal64&quot;,
        :vm_name =&gt; &quot;router1&quot;,
        :net =&gt; [
                   {ip: &apos;10.0.10.1&apos;, adapter: 2, netmask: &quot;255.255.255.252&quot;, virtualbox__intnet: &quot;r1-r2&quot;},
                   {ip: &apos;10.0.12.1&apos;, adapter: 3, netmask: &quot;255.255.255.252&quot;, virtualbox__intnet: &quot;r1-r3&quot;},
                   {ip: &apos;192.168.10.1&apos;, adapter: 4, netmask: &quot;255.255.255.0&quot;, virtualbox__intnet: &quot;net1&quot;},
                   {ip: &apos;192.168.56.10&apos;, adapter: 5},  # Изменено на разрешённый диапазон
                ]
  },

  :router2 =&gt; {
        :box_name =&gt; &quot;ubuntu/focal64&quot;,
        :vm_name =&gt; &quot;router2&quot;,
        :net =&gt; [
                   {ip: &apos;10.0.10.2&apos;, adapter: 2, netmask: &quot;255.255.255.252&quot;, virtualbox__intnet: &quot;r1-r2&quot;},
                   {ip: &apos;10.0.11.2&apos;, adapter: 3, netmask: &quot;255.255.255.252&quot;, virtualbox__intnet: &quot;r2-r3&quot;},
                   {ip: &apos;192.168.20.1&apos;, adapter: 4, netmask: &quot;255.255.255.0&quot;, virtualbox__intnet: &quot;net2&quot;},
                   {ip: &apos;192.168.56.11&apos;, adapter: 5},  # Изменено на разрешённый диапазон
                ]
  },

  :router3 =&gt; {
        :box_name =&gt; &quot;ubuntu/focal64&quot;,
        :vm_name =&gt; &quot;router3&quot;,
        :net =&gt; [
                   {ip: &apos;10.0.11.1&apos;, adapter: 2, netmask: &quot;255.255.255.252&quot;, virtualbox__intnet: &quot;r2-r3&quot;},
                   {ip: &apos;10.0.12.2&apos;, adapter: 3, netmask: &quot;255.255.255.252&quot;, virtualbox__intnet: &quot;r1-r3&quot;},
                   {ip: &apos;192.168.30.1&apos;, adapter: 4, netmask: &quot;255.255.255.0&quot;, virtualbox__intnet: &quot;net3&quot;},
                   {ip: &apos;192.168.56.12&apos;, adapter: 5},  # Изменено на разрешённый диапазон
                ]
  }
}

Vagrant.configure(&quot;2&quot;) do |config|
  MACHINES.each do |boxname, boxconfig|
    config.vm.define boxname do |box|
      box.vm.box = boxconfig[:box_name]
      box.vm.host_name = boxconfig[:vm_name]

      if boxconfig[:vm_name] == &quot;router3&quot;
        box.vm.provision &quot;ansible&quot; do |ansible|
          ansible.playbook = &quot;ansible/provision.yml&quot;
          ansible.inventory_path = &quot;ansible/hosts&quot;
          ansible.host_key_checking = &quot;false&quot;
          ansible.limit = &quot;all&quot;
        end
      end

      boxconfig[:net].each do |ipconf|
        box.vm.network &quot;private_network&quot;, **ipconf
      end
    end
  end
end
</pre>

запускаем создание виртуальных машин

>vagrant up

После создания вм, созданим настройку с помошью ansible

>mkdir ansible && cd ansible

Создадим необходимые файлы с содержимым

>nano ansible.cfg

<pre>[defaults]
#Отключение проверки ключа хоста
host_key_checking = false
#Указываем имя файла инвентаризации
inventory = hosts
#Отключаем игнорирование предупреждений
command_warnings= false
</pre>

>nano hosts

<pre><span style="color:#26A269"><b>[routers]</b></span>
router1 ansible_host=192.168.56.10 ansible_user=vagrant ansible_ssh_private_key_file=/home/roman/l33/.v<span style="background-color:#D0CFCC"><span style="color:#171421">&gt;</span></span>
router2 ansible_host=192.168.56.11 ansible_user=vagrant ansible_ssh_private_key_file=/home/roman/l33/.v<span style="background-color:#D0CFCC"><span style="color:#171421">&gt;</span></span>
router3 ansible_host=192.168.56.12 ansible_user=vagrant ansible_ssh_private_key_file=/home/roman/l33/.v<span style="background-color:#D0CFCC"><span style="color:#171421">&gt;</span></span>
</pre>

>nano provision.yml

<pre><span style="color:#2AA1B3"><i>#Начало файла provision.yml</i></span>
<span style="color:#A2734C"><b>- </b></span><span style="color:#33DA7A">name</span>: OSPF
 <span style="color:#2AA1B3"><i> #Указываем имя хоста или группу, которые будем настраивать</i></span>
  <span style="color:#33DA7A">hosts</span>: all
 <span style="color:#2AA1B3"><i> #Параметр выполнения модулей от root-пользователя</i></span>
  <span style="color:#33DA7A">become</span>: <span style="color:#C061CB">yes</span>
 <span style="color:#2AA1B3"><i> #Указание файла с дополнителыми переменными (понадобится при добавлении темплейтов)</i></span>
<span style="color:#2AA1B3"><i>#  vars_files:</i></span>
<span style="color:#2AA1B3"><i>#    - defaults/main.yml</i></span>
  <span style="color:#33DA7A">tasks</span>:
 <span style="color:#2AA1B3"><i> # Обновление пакетов и установка vim, traceroute, tcpdump, net-tools</i></span>
<span style="color:#A2734C"><b>  - </b></span><span style="color:#33DA7A">name</span>: install base tools
    <span style="color:#33DA7A">apt</span>:
      <span style="color:#33DA7A">name</span>:
<span style="color:#A2734C">       </span><span style="color:#A2734C"><b> - </b></span>vim
<span style="color:#A2734C">       </span><span style="color:#A2734C"><b> - </b></span>traceroute
<span style="color:#A2734C">       </span><span style="color:#A2734C"><b> - </b></span>tcpdump
<span style="color:#A2734C">       </span><span style="color:#A2734C"><b> - </b></span>net-tools
      <span style="color:#33DA7A">state</span>: present
      <span style="color:#33DA7A">update_cache</span>: <span style="color:#C061CB">true</span>

</pre>

запускаем настройку

>ansible-playbook provision.yml

<pre>PLAY [OSPF] ********************************************************************************************

TASK [Gathering Facts] *********************************************************************************
<span style="color:#26A269">ok: [router1]</span>
<span style="color:#26A269">ok: [router3]</span>
<span style="color:#26A269">ok: [router2]</span>

TASK [install base tools] ******************************************************************************
<span style="color:#A2734C">changed: [router2]</span>
<span style="color:#A2734C">changed: [router3]</span>
<span style="color:#A2734C">changed: [router1]</span>

PLAY RECAP *********************************************************************************************
<span style="color:#A2734C">router1</span>                    : <span style="color:#26A269">ok=2   </span> <span style="color:#A2734C">changed=1   </span> unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
<span style="color:#A2734C">router2</span>                    : <span style="color:#26A269">ok=2   </span> <span style="color:#A2734C">changed=1   </span> unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
<span style="color:#A2734C">router3</span>                    : <span style="color:#26A269">ok=2   </span> <span style="color:#A2734C">changed=1   </span> unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   </pre>

# Настройка OSPF с помощью ansible

создаем каталог templates

>mkdir templates && cd templates

Создаем необходимые файлы для настройки с содержимм

>nano daemons

<pre>
zebra=yes
bgpd=no
ospfd=yes
ospf6d=no
ripd=no
ripngd=no
isisd=no
pimd=no
ldpd=no
nhrpd=no
eigrpd=no
babeld=no
sharpd=no
pbrd=no
bfdd=no
fabricd=no
vrrpd=no
pathd=no
</pre>

>nano frr.conf.j2

<pre>
frr version 8.1
frr defaults traditional
hostname {{ ansible_hostname }}
log syslog informational
no ipv6 forwarding
service integrated-vtysh-config
!
password {{ frr_passwords.vty }}
enable password {{ frr_passwords.enable }}
!
{% for interface in ospf_interfaces %}
interface {{ interface.name }}
 description {{ interface.description }}
 ip address {{ interface.ip }}/{{ interface.mask }}
 ip ospf hello-interval {{ ospf_hello_interval }}
 ip ospf dead-interval {{ ospf_dead_interval }}
!
{% endfor %}
router ospf
 router-id {{ router_id }}
 {% for network in ospf_networks %}
 network {{ network.subnet }} area {{ network.area }}
 {% endfor %}
!
log file /var/log/frr/frr.log
</pre>

создаем файл main.yml

>mkdir defaults && cd defaults && nano main.yml

<pre>
---
# OSPF Timers
ospf_hello_interval: 10
ospf_dead_interval: 40

# Common FRR Settings
frr_passwords:
  enable: "password123"
  vty: "password123"

# Router-specific variables will be in host_vars/
</pre>

Дописываем измененния в файл hosts

>nano hosts

<pre>[routers]
router1 ansible_host=192.168.56.10 ansible_user=vagrant ansible_ssh_private_key_file=/home/roman/l33/.vagrant/machines/router1/virtualbox/private_key router_id=1.1.1.1
router2 ansible_host=192.168.56.11 ansible_user=vagrant ansible_ssh_private_key_file=/home/roman/l33/.vagrant/machines/router2/virtualbox/private_key router_id=2.2.2.2
router3 ansible_host=192.168.56.12 ansible_user=vagrant ansible_ssh_private_key_file=/home/roman/l33/.vagrant/machines/router3/virtualbox/private_key router_id=3.3.3.3
</pre>

и изменяем файл provision.yml

>nano provision.yml

<pre>---
---
- name: OSPF Setup
  hosts: all
  become: yes
  vars_files:
    - defaults/main.yml

  tasks:
    - name: Install base tools
      apt:
        name:
          - vim
          - traceroute
          - tcpdump
          - net-tools
        state: present
        update_cache: yes

    - name: Add FRR repository key
      apt_key:
        url: "https://deb.frrouting.org/frr/keys.asc"
        state: present

    - name: Add FRR repository
      apt_repository:
        repo: "deb https://deb.frrouting.org/frr {{ ansible_distribution_release }} frr-stable"
        state: present

    - name: Install FRR packages
      apt:
        name:
          - frr
          - frr-pythontools
        state: present
        update_cache: yes

    - name: Enable IP forwarding
      sysctl:
        name: net.ipv4.ip_forward
        value: '1'
        state: present

    - name: Disable UFW firewall
      ufw:
        state: disabled

    - name: Configure FRR daemons
      template:
        src: daemons
        dest: /etc/frr/daemons
        owner: frr
        group: frr
        mode: '0640'

    - name: Configure FRR OSPF
      template:
        src: frr.conf.j2
        dest: /etc/frr/frr.conf
        owner: frr
        group: frr
        mode: '0640'
      notify: Restart FRR

  handlers:
    - name: Restart FRR
      service:
        name: frr
        state: restarted
        enabled: yes
</pre>

Создаем host_vars для каждого роутера

>mkdir host_vars && cd host_vars

>nano router1.yml

<pre>
---
router_id: "1.1.1.1"
ospf_interfaces:
  - { name: "enp0s8", description: "r1-r2", ip: "10.0.10.1", mask: "30" }
  - { name: "enp0s9", description: "r1-r3", ip: "10.0.12.1", mask: "30" }
  - { name: "enp0s10", description: "net_router1", ip: "192.168.10.1", mask: "24" }

ospf_networks:
  - { subnet: "10.0.10.0/30", area: "0" }
  - { subnet: "10.0.12.0/30", area: "0" }
  - { subnet: "192.168.10.0/24", area: "0" }

ospf_neighbors:
  - "10.0.10.2"  # router2
  - "10.0.12.2"  # router3
</pre>

>nano router2.yml

<pre>
---
router_id: "2.2.2.2"
ospf_interfaces:
  - { name: "enp0s8", description: "r2-r1", ip: "10.0.10.2", mask: "30" }
  - { name: "enp0s9", description: "r2-r3", ip: "10.0.11.1", mask: "30" }
  - { name: "enp0s10", description: "net_router2", ip: "192.168.20.1", mask: "24" }

ospf_networks:
  - { subnet: "10.0.10.0/30", area: "0" }
  - { subnet: "10.0.11.0/30", area: "0" }
  - { subnet: "192.168.20.0/24", area: "0" }

ospf_neighbors:
  - "10.0.10.1"  # router1
  - "10.0.11.2"  # router3
</pre>

>nano router3.yml

<pre>
---
router_id: "3.3.3.3"
ospf_interfaces:
  - { name: "enp0s8", description: "r3-r1", ip: "10.0.12.2", mask: "30" }
  - { name: "enp0s9", description: "r3-r2", ip: "10.0.11.2", mask: "30" }
  - { name: "enp0s10", description: "net_router3", ip: "192.168.30.1", mask: "24" }

ospf_networks:
  - { subnet: "10.0.12.0/30", area: "0" }
  - { subnet: "10.0.11.0/30", area: "0" }
  - { subnet: "192.168.30.0/24", area: "0" }

ospf_neighbors:
  - "10.0.12.1"  # router1
  - "10.0.11.1"  # router2
</pre>

Запускаем настройку

>ansible-playbook provision.yml

проверяем, заходим на первый роутер и пингуем третий

<pre>root@router1:~# ping 192.168.30.1
PING 192.168.30.1 (192.168.30.1) 56(84) bytes of data.
64 bytes from 192.168.30.1: icmp_seq=1 ttl=64 time=0.647 ms
64 bytes from 192.168.30.1: icmp_seq=2 ttl=64 time=1.13 ms
64 bytes from 192.168.30.1: icmp_seq=3 ttl=64 time=1.16 ms
64 bytes from 192.168.30.1: icmp_seq=4 ttl=64 time=1.35 ms
^C
--- 192.168.30.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3039ms
rtt min/avg/max/mdev = 0.647/1.071/1.351/0.259 ms
</pre>

Запустим трассировку до адреса 192.168.30.1

>traceroute 192.168.30.1

<pre><span style="color:#26A269"><b>vagrant@router1</b></span>:<span style="color:#12488B"><b>~</b></span>$ sudo -i
root@router1:~# traceroute 192.168.30.1
traceroute to 192.168.30.1 (192.168.30.1), 30 hops max, 60 byte packets
 1  192.168.30.1 (192.168.30.1)  0.553 ms  0.479 ms  0.441 ms
</pre>

Попробуем отключить интерфейс enp0s9 и немного подождем и снова запустим трассировку до ip-адреса 192.168.30.1

ifconfig enp0s9 down


>traceroute 192.168.30.1



