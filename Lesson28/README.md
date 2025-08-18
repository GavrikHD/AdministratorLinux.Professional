# Теоретическая часть

## 1. Топология сети

Нам нужно построить следующую сетевую архитектуру:

```
[Internet]
   |
[inetRouter] (192.168.255.1/30)
   |
[centralRouter] (192.168.255.2/30)
   ├── [directors] (192.168.0.0/28)
   ├── [office hardware] (192.168.0.32/28)
   ├── [wifi] (192.168.0.64/26)
   ├── [office1Router] (через office1 network)
   └── [office2Router] (через office2 network)
```

Для офисов:
- **Office1**:
  - dev (192.168.2.0/26)
  - test servers (192.168.2.64/26)
  - managers (192.168.2.128/26)
  - office hardware (192.168.2.192/26)

- **Office2**:
  - dev (192.168.1.0/25)
  - test servers (192.168.1.128/26)
  - office hardware (192.168.1.192/26)

## 2. Свободные подсети

После анализа заданных подсетей, выявлены следующие свободные подсети:

1. В центральной сети:
   - 192.168.0.16/28
   - 192.168.0.48/28
   - 192.168.0.128/25

2. В сети между inetRouter и centralRouter (192.168.255.0/30):
   - 192.168.255.4/30
   - 192.168.255.8/29
   - 192.168.255.16/28
   - 192.168.255.32/27
   - 192.168.255.64/26

## 3. Количество узлов в каждой подсети

### Центральная сеть
| Подсеть            | Маска            | Кол-во узлов | Используемые адреса        |
|---------------------|------------------|--------------|----------------------------|
| 192.168.0.0/28      | 255.255.255.240  | 14           | directors                  |
| 192.168.0.16/28     | 255.255.255.240  | 14           | свободная                  |
| 192.168.0.32/28     | 255.255.255.240  | 14           | office hardware            |
| 192.168.0.48/28     | 255.255.255.240  | 14           | свободная                  |
| 192.168.0.64/26     | 255.255.255.192  | 62           | wifi                       |
| 192.168.0.128/25    | 255.255.255.128  | 126          | свободная                  |

### Сеть Office1
| Подсеть            | Маска            | Кол-во узлов | Используемые адреса        |
|---------------------|------------------|--------------|----------------------------|
| 192.168.2.0/26      | 255.255.255.192  | 62           | dev                        |
| 192.168.2.64/26     | 255.255.255.192  | 62           | test servers               |
| 192.168.2.128/26    | 255.255.255.192  | 62           | managers                   |
| 192.168.2.192/26    | 255.255.255.192  | 62           | office hardware            |

### Сеть Office2
| Подсеть            | Маска            | Кол-во узлов | Используемые адреса        |
|---------------------|------------------|--------------|----------------------------|
| 192.168.1.0/25      | 255.255.255.128  | 126          | dev                        |
| 192.168.1.128/26    | 255.255.255.192  | 62           | test servers               |
| 192.168.1.192/26    | 255.255.255.192  | 62           | office hardware            |

### Сеть между роутерами
| Подсеть            | Маска            | Кол-во узлов | Используемые адреса        |
|---------------------|------------------|--------------|----------------------------|
| 192.168.255.0/30    | 255.255.255.252  | 2            | inetRouter-centralRouter   |
| 192.168.255.4/30    | 255.255.255.252  | 2            | свободная                  |
| 192.168.255.8/29    | 255.255.255.248  | 6            | свободная                  |
| 192.168.255.16/28   | 255.255.255.240  | 14           | свободная                  |
| 192.168.255.32/27   | 255.255.255.224  | 30           | свободная                  |
| 192.168.255.64/26   | 255.255.255.192  | 62           | свободная                  |

## 4. Broadcast-адреса для каждой подсети

### Центральная сеть
| Подсеть            | Broadcast        |
|---------------------|------------------|
| 192.168.0.0/28      | 192.168.0.15     |
| 192.168.0.16/28     | 192.168.0.31     |
| 192.168.0.32/28     | 192.168.0.47     |
| 192.168.0.48/28     | 192.168.0.63     |
| 192.168.0.64/26     | 192.168.0.127    |
| 192.168.0.128/25    | 192.168.0.255    |

### Сеть Office1
| Подсеть            | Broadcast        |
|---------------------|------------------|
| 192.168.2.0/26      | 192.168.2.63     |
| 192.168.2.64/26     | 192.168.2.127    |
| 192.168.2.128/26    | 192.168.2.191    |
| 192.168.2.192/26    | 192.168.2.255    |

### Сеть Office2
| Подсеть            | Broadcast        |
|---------------------|------------------|
| 192.168.1.0/25      | 192.168.1.127    |
| 192.168.1.128/26    | 192.168.1.191    |
| 192.168.1.192/26    | 192.168.1.255    |

### Сеть между роутерами
| Подсеть            | Broadcast        |
|---------------------|------------------|
| 192.168.255.0/30    | 192.168.255.3    |
| 192.168.255.4/30    | 192.168.255.7    |
| 192.168.255.8/29    | 192.168.255.15   |
| 192.168.255.16/28   | 192.168.255.31   |
| 192.168.255.32/27   | 192.168.255.63   |
| 192.168.255.64/26   | 192.168.255.127  |

## 5. Проверка на ошибки при разбиении

После тщательного анализа разбиения сети можно сделать следующие выводы:

1. **Нет перекрытия подсетей**: Все подсети имеют четкие границы и не пересекаются между собой.
   
2. **Эффективное использование адресного пространства**:
   - В центральной сети выделены подсети с разным размером в зависимости от потребностей (от /28 до /25)
   - В office1 все подсети одинакового размера (/26)
   - В office2 использованы подсети разного размера в зависимости от потребностей (/25 для dev, /26 для остальных)

3. **Достаточно свободного пространства** для расширения:
   - В центральной сети остались свободные подсети 192.168.0.16/28, 192.168.0.48/28 и 192.168.0.128/25
   - В сети между роутерами много свободного пространства (от /30 до /26)

4. **Логическое разделение**:
   - Разные офисы имеют свои диапазоны (192.168.1.0/24 для office2, 192.168.2.0/24 для office1)
   - Центральная сеть использует 192.168.0.0/24
   - Соединение между роутерами выделено в отдельный диапазон 192.168.255.0/24

**Вывод**: Ошибок в разбиении сети нет, все подсети корректно разделены и не пересекаются, есть достаточный запас для расширения сети.

# Практическая часть

## создание вм

<pre>
MACHINES = {
  :inetRouter => {
        :box_name => "generic/ubuntu2204",
        :vm_name => "inetRouter",
        #:public => {:ip => "10.10.10.1", :adapter => 1},
        :net => [   
                    #ip, adpter, netmask, virtualbox__intnet
                    ["192.168.255.1", 2, "255.255.255.252",  "router-net"], 
                    ["192.168.56.10", 8, "255.255.255.0"],
                ]
  },

  :centralRouter => {
        :box_name => "generic/ubuntu2204",
        :vm_name => "centralRouter",
        :net => [
                   ["192.168.255.2",  2, "255.255.255.252",  "router-net"],
                   ["192.168.0.1",    3, "255.255.255.240",  "dir-net"],
                   ["192.168.0.33",   4, "255.255.255.240",  "hw-net"],
                   ["192.168.0.65",   5, "255.255.255.192",  "mgt-net"],
                   ["192.168.255.9",  6, "255.255.255.252",  "office1-central"],
                   ["192.168.255.5",  7, "255.255.255.252",  "office2-central"],
                   ["192.168.56.11",  8, "255.255.255.0"],
                ]
  },

  :centralServer => {
        :box_name => "generic/ubuntu2204",
        :vm_name => "centralServer",
        :net => [
                   ["192.168.0.2",    2, "255.255.255.240",  "dir-net"],
                   ["192.168.56.12",  8, "255.255.255.0"],
                ]
  },

  :office1Router => {
        :box_name => "generic/ubuntu2204",
        :vm_name => "office1Router",
        :net => [
                   ["192.168.255.10",  2,  "255.255.255.252",  "office1-central"],
                   ["192.168.2.1",     3,  "255.255.255.192",  "dev1-net"],
                   ["192.168.2.65",    4,  "255.255.255.192",  "test1-net"],
                   ["192.168.2.129",   5,  "255.255.255.192",  "managers-net"],
                   ["192.168.2.193",   6,  "255.255.255.192",  "office1-net"],
                   ["192.168.56.20",   8,  "255.255.255.0"],
                ]
  },

  :office1Server => {
        :box_name => "generic/ubuntu2204",
        :vm_name => "office1Server",
        :net => [
                   ["192.168.2.130",  2,  "255.255.255.192",  "managers-net"],
                   ["192.168.56.21",  8,  "255.255.255.0"],
                ]
  },

  :office2Router => {
       :box_name => "generic/ubuntu2204",
       :vm_name => "office2Router",
       :net => [
                   ["192.168.255.6",  2,  "255.255.255.252",  "office2-central"],
                   ["192.168.1.1",    3,  "255.255.255.128",  "dev2-net"],
                   ["192.168.1.129",  4,  "255.255.255.192",  "test2-net"],
                   ["192.168.1.193",  5,  "255.255.255.192",  "office2-net"],
                   ["192.168.56.30",  8,  "255.255.255.0"],
               ]
  },

  :office2Server => {
       :box_name => "generic/ubuntu2204",
       :vm_name => "office2Server",
       :net => [
                  ["192.168.1.2",    2,  "255.255.255.128",  "dev2-net"],
                  ["192.168.56.31",  8,  "255.255.255.0"],
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
      SHELL
    end
  end
end
</pre>

запускаем создание вм

>vagranr up

##  Настройка NAT

 Подключаемся к роутеру

 >vagrant ssh inetRouter

 >sudo -i

 отключаем файрвол

 >systemctl stop ufw

 >systemctl disable ufw

Создаём файл /etc/iptables_rules.ipv4

>nano /etc/iptables_rules.ipv4

<pre>
# Generated by iptables-save v1.8.7 on Sat Oct 14 16:14:36 2023
*filter
:INPUT ACCEPT [90:8713]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [54:7429]
-A INPUT -p icmp -j ACCEPT
-A INPUT -i lo -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 22 -j ACCEPT
COMMIT
# Completed on Sat Oct 14 16:14:36 2023
# Generated by iptables-save v1.8.7 on Sat Oct 14 16:14:36 2023
*nat
:PREROUTING ACCEPT [1:44]
:INPUT ACCEPT [1:44]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING ! -d 192.168.0.0/16 -o eth0 -j MASQUERADE
COMMIT
# Completed on Sat Oct 14 16:14:36 2023
</pre>

Создаём файл, в который добавим скрипт автоматического восстановления правил при перезапуске системы:

>nano /etc/network/if-pre-up.d/iptables

<pre>
#!/bin/sh
/sbin/iptables-restore < /etc/iptables_rules.ipv4
</pre>

Добавляем права на выполнение файла /etc/network/if-pre-up.d/iptables

>chmod +x /etc/network/if-pre-up.d/iptables

Перезагружаем сервер

>reboot

После перезагрузки сервера проверяем правила iptables

>iptables-save

<pre><span style="color:#26A269"><b>root@inetRouter</b></span>:<span style="color:#12488B"><b>~</b></span># iptables-save
# Generated by iptables-save v1.8.7 on Fri Aug 15 13:35:51 2025
*filter
:INPUT ACCEPT [48:6108]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [32:5877]
-A INPUT -p icmp -j ACCEPT
-A INPUT -i lo -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 22 -j ACCEPT
COMMIT
# Completed on Fri Aug 15 13:35:51 2025
# Generated by iptables-save v1.8.7 on Fri Aug 15 13:35:51 2025
*nat
:PREROUTING ACCEPT [1:44]
:INPUT ACCEPT [1:44]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING ! -d 192.168.0.0/16 -o eth0 -j MASQUERADE
COMMIT
# Completed on Fri Aug 15 13:35:51 2025
</pre>

## Маршрутизация транзитных пакетов (IP forward)

Включаем маршрутизацию транзитных пакетов на всех роутерах

>echo "net.ipv4.conf.all.forwarding = 1" >> /etc/sysctl.conf
sysctl -p

Проверяем форвардинг

>sysctl net.ipv4.ip_forward

<pre><span style="color:#26A269"><b>root@inetRouter</b></span>:<span style="color:#12488B"><b>~</b></span># sysctl net.ipv4.ip_forward
net.ipv4.ip_forward = 1
</pre>

<pre><span style="color:#26A269"><b>root@centralRouter</b></span>:<span style="color:#12488B"><b>~</b></span># sysctl net.ipv4.ip_forward
net.ipv4.ip_forward = 1
</pre>

<pre><span style="color:#26A269"><b>root@office1Router</b></span>:<span style="color:#12488B"><b>~</b></span># sysctl net.ipv4.ip_forward
net.ipv4.ip_forward = 1
</pre>

<pre><span style="color:#26A269"><b>root@office2Router</b></span>:<span style="color:#12488B"><b>~</b></span># sysctl net.ipv4.ip_forward
net.ipv4.ip_forward = 1
</pre>

## Отключение маршрута по умолчанию на интерфейсе eth0

Отключаем маршрут по умолчанию в файле /etc/netplan/00-installer-config.yaml и добавляем отключение маршрутов, полученных через DHCP на всех роутерах и серверах, кроме inetRouter

>nano /etc/netplan/00-installer-config.yaml

<pre>
network:
  ethernets:
    eth0:
      dhcp4: true
      dhcp4-overrides:
          use-routes: false
      dhcp6: false
  version: 2
</pre>

После внесения данных изменений перезапускаем сетевую службу: 

>netplan try

## Настройка статических маршрутов

для настройки статических маршрутов надо поправить netplan-конфиг на всем роутерах и серверах 

>nano /etc/netplan/50-vagrant.yaml 

Для добавления маршрута, после раздела addresses нужно добавить блок routes:
вот как будет выглядеть файл на всех вм

inetRouter:
<pre>
  ethernets:
    eth1:
      addresses:
        - 192.168.255.1/30
      routes:
        - to: 192.168.0.0/28
          via: 192.168.255.2
        - to: 192.168.0.16/28
          via: 192.168.255.2
        - to: 192.168.0.32/28
          via: 192.168.255.2
        - to: 192.168.0.48/28
          via: 192.168.255.2
        - to: 192.168.0.64/26
          via: 192.168.255.2
        - to: 192.168.0.128/25
          via: 192.168.255.2
        - to: 192.168.2.0/26
          via: 192.168.255.2
        - to: 192.168.2.64/26
          via: 192.168.255.2
        - to: 192.168.2.128/26
          via: 192.168.255.2
        - to: 192.168.2.192/26
          via: 192.168.255.2
        - to: 192.168.1.0/25
          via: 192.168.255.2
        - to: 192.168.1.128/26
          via: 192.168.255.2
        - to: 192.168.1.192/26
          via: 192.168.255.2
        - to: 192.168.255.0/30
          via: 192.168.255.2
        - to: 192.168.255.4/30
          via: 192.168.255.2
        - to: 192.168.255.8/29
          via: 192.168.255.2
        - to: 192.168.255.16/28
          via: 192.168.255.2
        - to: 192.168.255.32/27
          via: 192.168.255.2
        - to: 192.168.255.64/26
          via: 192.168.255.2
    eth2:
      addresses:
        - 192.168.56.10/24
</pre>


centralRouter:
<pre>
  ethernets:
    eth1:
      addresses:
      - 192.168.255.2/30
      routes:
      - to: 0.0.0.0/0
        via: 192.168.255.1
    eth2:
      addresses:
      - 192.168.0.1/28
    eth3:
      addresses:
      - 192.168.0.33/28
    eth4:
      addresses:
      - 192.168.0.65/26
    eth5:
      addresses:
      - 192.168.255.9/30
      routes:
      - to: 192.168.2.128/26
        via: 192.168.255.10
      - to: 192.168.2.64/28
        via: 192.168.255.10
      - to: 192.168.2.0/26
        via: 192.168.255.10
      - to: 192.168.2.19/26
        via: 192.168.255.10
    eth6:
      addresses:
      - 192.168.255.5/30
      routes:
      - to: 192.168.1.0/25
        via: 192.168.255.6
      - to: 192.168.1.128/26
        via: 192.168.255.6
      - to: 192.168.1.192/26
        via: 192.168.255.6
    eth7:
      addresses:
      - 192.168.56.11/24
</pre>

centralServer
<pre>
  ethernets:
    eth1:
      addresses:
      - 192.168.0.2/28
      routes:
      - to: 0.0.0.0/0
        via: 192.168.0.1
    eth2:
      addresses:
      - 192.168.56.12/24
</pre>

offuce1Router
<pre>
  ethernets:
    eth1:
      addresses:
      - 192.168.255.10/30
      routes:
      - to: 0.0.0.0/0
        via: 192.168.255.9
    eth2:
      addresses:
      - 192.168.2.1/26
    eth3:
      addresses:
      - 192.168.2.65/26
    eth4:
      addresses:
      - 192.168.2.129/26
    eth5:
      addresses:
      - 192.168.2.193/26
    eth6:
      addresses:
      - 192.168.56.20/24
</pre>

office2Router
<pre>
  ethernets:
    eth1:
      addresses:
      - 192.168.255.6/30
      routes:
      - to: 0.0.0.0/0
        via: 192.168.255.5
    eth2:
      addresses:
      - 192.168.1.1/25
    eth3:
      addresses:
      - 192.168.1.129/26
    eth4:
      addresses:
      - 192.168.1.193/26
    eth5:
      addresses:
      - 192.168.56.30/24
</pre>

office1Server
<pre>
  ethernets:
    eth1:
      addresses:
      - 192.168.2.130/26
      routes:
      - to: 0.0.0.0/0
        via: 192.168.2.129
    eth2:
      addresses:
      - 192.168.56.21/24
</pre>

office2Server
<pre>
  ethernets:
    eth1:
      addresses:
      - 192.168.1.2/25
      routes:
      - to: 0.0.0.0/0
        via: 192.168.1.1
    eth2:
      addresses:
      - 192.168.56.31/24
</pre>

Теперь установим traceroute и проверим правильность наших маршрутов

>apt install -y traceroute

<pre><span style="color:#26A269"><b>root@office1Server</b></span>:<span style="color:#12488B"><b>~</b></span># traceroute 8.8.8.8
traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets
 1  _gateway (192.168.2.129)  0.713 ms  0.615 ms  0.818 ms
 2  192.168.255.9 (192.168.255.9)  1.202 ms  0.993 ms  0.912 ms
 3  192.168.255.1 (192.168.255.1)  1.587 ms  1.628 ms  1.553 ms
 4  10.0.2.2 (10.0.2.2)  1.559 ms  1.497 ms  1.340 ms
 5  * * *
 6  * * *
 7  * * *
 8  mag9-cr03-be12.51.msk.mts-internet.net (212.188.6.44)  9.426 ms  9.380 ms  9.218 ms
 9  72.14.223.72 (72.14.223.72)  9.749 ms 72.14.223.74 (72.14.223.74)  9.704 ms 72.14.223.72 (72.14.223.72)  9.587 ms
10  * * *
11  72.14.233.96 (72.14.233.96)  8.636 ms * 192.178.241.66 (192.178.241.66)  8.741 ms
12  192.178.241.232 (192.178.241.232)  8.008 ms * *
13  * * *
14  142.250.235.68 (142.250.235.68)  29.878 ms  30.955 ms 142.251.238.70 (142.251.238.70)  28.266 ms
15  216.239.46.243 (216.239.46.243)  24.597 ms 172.253.51.247 (172.253.51.247)  26.017 ms *
16  * * *
17  * * *
18  * * *
19  * * *
20  * * *
21  * * *
22  * * *
23  * * *
24  * * *
25  * * dns.google (8.8.8.8)  26.820 ms
</pre>

Маршрутизация работает
