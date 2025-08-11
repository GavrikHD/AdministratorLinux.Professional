# VTUN/TAP режимы VPN

Создаем Vagrantfile

>nano vagrant

<pre>
Vagrant.configure("2") do |config|
config.vm.box = "ubuntu/jammy64" 
config.vm.define "server" do |server| 
server.vm.hostname = "server.loc" 
  	server.vm.network "private_network", ip: "192.168.56.10" 
end 

config.vm.define "client" do |client| 
client.vm.hostname = "client.loc" 
client.vm.network "private_network", ip: "192.168.56.20" 
end 
end
</pre>


После запуска машин из Vagrantfile необходимо выполнить следующие действия на server и client машинах:

>sudo -i

>apt update

>apt install openvpn iperf3 selinux-utils

>setenforce 0

Настройка хоста 1:

>sudo -i

Cоздаем файл-ключ 

>openvpn --genkey secret /etc/openvpn/static.key

Cоздаем конфигурационный файл OpenVPN 

>nano /etc/openvpn/server.conf
	
Содержимое файла server.conf

<pre>
dev tap 
ifconfig 10.10.10.1 255.255.255.0 
topology subnet 
secret /etc/openvpn/static.key 
comp-lzo 
status /var/log/openvpn-status.log 
log /var/log/openvpn.log  
verb 3 
</pre>

Создаем service unit для запуска OpenVPN
     
>nano /etc/systemd/system/openvpn@.service

Содержимое файла-юнита

<pre>
[Unit] 
Description=OpenVPN Tunneling Application On %I 
After=network.target 
[Service] 
Type=notify 
PrivateTmp=true 
ExecStart=/usr/sbin/openvpn --cd /etc/openvpn/ --config %i.conf 
[Install] 
WantedBy=multi-user.target
</pre>

Запускаем сервис 

>systemctl start openvpn@server 

>systemctl enable openvpn@server

Настройка хоста 2: 

>sudo -i

Cоздаем конфигурационный файл OpenVPN 

nano /etc/openvpn/server.conf

Содержимое конфигурационного файла  

<pre>
dev tap 
remote 192.168.56.10 
ifconfig 10.10.10.2 255.255.255.0 
topology subnet 
route 192.168.56.0 255.255.255.0 
secret /etc/openvpn/static.key
comp-lzo
status /var/log/openvpn-status.log 
log /var/log/openvpn.log 
verb 3 
<pre>

скопируем ключ который создали на хосте server

>cat /etc/openvpn/static.key

На хост client в созданный файл 

>nano /etc/openvpn/static.key

#
# 2048 bit OpenVPN static key
#
-----BEGIN OpenVPN Static key V1-----
9b652eca2aa560a5b53425bcadc0d27b
3d9f6e5d90302de36da28d08bd7f7518
4810332c5e1a99aae555cbf05d8f8cf1
08e0221eae7b5d44a42f4c8d9f0a41be
7c847d837094161cd497df3e5e7efa60
2fba9bd2facb9617384406fffd2128ef
bcacbdf9bc5bd26e8803a84b51c8448f
fcfbbae1600cf64264ec036ca981a9d6
4dd0089089090025bcc8d0f85eb574a6
474450a436b50ecc13c5f1b4710ee120
40cfaa557c2685dd8b06ba6d94b3f695
4fdd09a2c36af6c96cc6bece2c1fa346
6fef5c4884b4c37c95123b648cc9399a
29e85438073a8ab13514d8c6872e5977
651c23795e413c593ba5bc37b312e1d2
051b5c67bf718d9ab6f2c961159e0910
-----END OpenVPN Static key V1-----


Создаем service unit для запуска OpenVPN

>nano /etc/systemd/system/openvpn@.service
     
<pre>
[Unit] 
Description=OpenVPN Tunneling Application On %I 
After=network.target 
[Service] 
Type=notify 
PrivateTmp=true 
ExecStart=/usr/sbin/openvpn --cd /etc/openvpn/ --config %i.conf 
[Install] 
WantedBy=multi-user.target
</pre>     
     
Запускаем сервис 

>systemctl start openvpn@server 

>systemctl enable openvpn@server


Далее необходимо замерить скорость в туннеле: 

На хосте server запускаем iperf3 в режиме сервера: iperf3 -s

>iperf3 -s

<pre>root@server:~# iperf3 -s
-----------------------------------------------------------
Server listening on 5201
-----------------------------------------------------------
</pre>

На хосте client запускаем iperf3 в режиме клиента и замеряем  скорость в туннеле: 

>iperf3 -c 10.10.10.1 -t 40 -i 5 

<pre>Connecting to host 10.10.10.1, port 5201
[  5] local 10.10.10.2 port 37268 connected to 10.10.10.1 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-5.00   sec   141 MBytes   237 Mbits/sec  350    642 KBytes       
[  5]   5.00-10.00  sec   130 MBytes   218 Mbits/sec  254    301 KBytes       
[  5]  10.00-15.00  sec   131 MBytes   220 Mbits/sec   91    477 KBytes       
[  5]  15.00-20.00  sec   132 MBytes   222 Mbits/sec  282    294 KBytes       
[  5]  20.00-25.00  sec   114 MBytes   191 Mbits/sec  363    126 KBytes       
[  5]  25.00-30.00  sec   106 MBytes   178 Mbits/sec  123    164 KBytes       
[  5]  30.00-35.00  sec   106 MBytes   178 Mbits/sec   82    141 KBytes       
[  5]  35.00-40.00  sec   105 MBytes   176 Mbits/sec   70    157 KBytes       
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-40.00  sec   966 MBytes   203 Mbits/sec  1615             sender
[  5]   0.00-40.06  sec   963 MBytes   202 Mbits/sec                  receiver
</pre>

теперь меняем на обеих хостах dev tap на dev tun в 
перезапускаем сервисы openvpn

>systemctl restart openvpn@server

и Повторяем пункты 1-2 для режима работы tun.

<pre>Connecting to host 10.10.10.1, port 5201
[  5] local 10.10.10.2 port 39818 connected to 10.10.10.1 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-5.00   sec   116 MBytes   194 Mbits/sec  241    120 KBytes       
[  5]   5.00-10.00  sec   118 MBytes   197 Mbits/sec  136    140 KBytes       
[  5]  10.00-15.00  sec   116 MBytes   194 Mbits/sec  180    145 KBytes       
[  5]  15.00-20.00  sec   115 MBytes   193 Mbits/sec  164    102 KBytes       
[  5]  20.00-25.00  sec   105 MBytes   176 Mbits/sec  119    127 KBytes       
[  5]  25.00-30.00  sec   116 MBytes   195 Mbits/sec  180    141 KBytes       
[  5]  30.00-35.00  sec   109 MBytes   182 Mbits/sec  160    181 KBytes       
[  5]  35.00-40.00  sec   112 MBytes   188 Mbits/sec  161    141 KBytes       
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-40.00  sec   906 MBytes   190 Mbits/sec  1341             sender
[  5]   0.00-40.05  sec   905 MBytes   190 Mbits/sec                  receiver
</pre>

В нашем случае TAP оказался чуть быстрее, но TUN — стабильнее

# RAS на базе OpenV

устанавливаем вм с помощью Vagrantfile

<pre>
Vagrant.configure("2") do |config|
     config.vm.box = "ubuntu/jammy64" 
     config.vm.define "server" do |server| 
          server.vm.hostname = "server.loc" 
  	  server.vm.network "private_network", ip: "192.168.56.10" 
     end 
end
</pre>

>vagrant up


Подключаемся и Установим необходимые пакеты 

>vagrant ssh server

>sudo -i

>apt update

>apt install openvpn easy-rsa selinux-utils

>setenforce 0

Переходим в директорию /etc/openvpn и инициализируем PKI

>cd /etc/openvpn

>/usr/share/easy-rsa/easyrsa init-pki

Генерируем необходимые ключи и сертификаты для сервера 

>echo 'server' | /usr/share/easy-rsa/easyrsa gen-req server nopass

>echo 'yes' | /usr/share/easy-rsa/easyrsa sign-req server server

>/usr/share/easy-rsa/easyrsa gen-dh

>openvpn --genkey secret /etc/openvpn/pki/ta.key

Генерируем необходимые ключи и сертификаты для клиента

>echo 'client' | /usr/share/easy-rsa/easyrsa gen-req client nopass

>echo 'yes' | /usr/share/easy-rsa/easyrsa sign-req client client

Создаем конфигурационный файл сервера 

>nano /etc/openvpn/server.conf

<pre>
port 1207
proto udp
dev tun
topology subnet
ca /etc/openvpn/pki/ca.crt
cert /etc/openvpn/pki/issued/server.crt
key /etc/openvpn/pki/private/server.key
dh /etc/openvpn/pki/dh.pem
tls-auth /etc/openvpn/pki/ta.key 0
server 10.10.10.0 255.255.255.0
ifconfig-pool-persist ipp.txt
push "route 10.10.10.0 255.255.255.0"
client-to-client
client-config-dir /etc/openvpn/client
keepalive 10 120
data-ciphers AES-256-GCM:AES-128-GCM
persist-key
persist-tun
status /var/log/openvpn-status.log
log /var/log/openvpn.log
verb 3
</pre>

Создаем service unit для запуска OpenVPN

>nano /etc/systemd/system/openvpn@.service
     
<pre>
[Unit] 
Description=OpenVPN Tunneling Application On %I 
After=network.target 
[Service] 
Type=notify 
PrivateTmp=true 
ExecStart=/usr/sbin/openvpn --cd /etc/openvpn/ --config %i.conf 
[Install] 
WantedBy=multi-user.target
</pre>     

Запускаем сервис

>systemctl start openvpn@server

>systemctl enable openvpn@server


На хост-машине создаем файл client.conf со следующим содержимым: 

<pre>
dev tun
proto udp
remote 192.168.56.10 1207
client
resolv-retry infinite
remote-cert-tls server
ca ./ca.crt
cert ./client.crt
key ./client.key
tls-auth ./ta.key 1
route 10.10.10.0 255.255.255.0
persist-key
persist-tun
data-ciphers AES-256-GCM:AES-128-GCM
verb 3
</pre>

Скопировали в одну директорию с client.conf файлы с сервера:     	   

/etc/openvpn/pki/ca.crt 
/etc/openvpn/pki/issued/client.crt 
/etc/openvpn/pki/private/client.key
/etc/openvpn/pki/ta.key

Проверяем подключение с помощью: openvpn --config client.conf

<pre>2025-08-11 19:17:03 Note: --cipher is not set. OpenVPN versions before 2.5 defaulted to BF-CBC as fallback when cipher negotiation failed in this case. If you need this fallback please add &apos;--data-ciphers-fallback BF-CBC&apos; to your configuration and/or add BF-CBC to --data-ciphers.
2025-08-11 19:17:03 Note: Kernel support for ovpn-dco missing, disabling data channel offload.
2025-08-11 19:17:03 WARNING: file &apos;./client.key&apos; is group or others accessible
2025-08-11 19:17:03 OpenVPN 2.6.3 x86_64-pc-linux-gnu [SSL (OpenSSL)] [LZO] [LZ4] [EPOLL] [PKCS11] [MH/PKTINFO] [AEAD] [DCO]
2025-08-11 19:17:03 library versions: OpenSSL 3.0.16 11 Feb 2025, LZO 2.10
2025-08-11 19:17:03 DCO version: N/A
2025-08-11 19:17:03 TCP/UDP: Preserving recently used remote address: [AF_INET]192.168.56.10:1207
2025-08-11 19:17:03 Socket Buffers: R=[212992-&gt;212992] S=[212992-&gt;212992]
2025-08-11 19:17:03 UDPv4 link local: (not bound)
2025-08-11 19:17:03 UDPv4 link remote: [AF_INET]192.168.56.10:1207
2025-08-11 19:17:03 TLS: Initial packet from [AF_INET]192.168.56.10:1207, sid=eaf45581 7a828e06
2025-08-11 19:17:03 VERIFY OK: depth=1, CN=rasvpn
2025-08-11 19:17:03 VERIFY KU OK
2025-08-11 19:17:03 Validating certificate extended key usage
2025-08-11 19:17:03 ++ Certificate has EKU (str) TLS Web Server Authentication, expects TLS Web Server Authentication
2025-08-11 19:17:03 VERIFY EKU OK
2025-08-11 19:17:03 VERIFY OK: depth=0, CN=server
2025-08-11 19:17:03 Control Channel: TLSv1.3, cipher TLSv1.3 TLS_AES_256_GCM_SHA384, peer certificate: 2048 bit RSA, signature: RSA-SHA256
2025-08-11 19:17:03 [server] Peer Connection Initiated with [AF_INET]192.168.56.10:1207
2025-08-11 19:17:03 TLS: move_session: dest=TM_ACTIVE src=TM_INITIAL reinit_src=1
2025-08-11 19:17:03 TLS: tls_multi_process: initial untrusted session promoted to trusted
2025-08-11 19:17:03 PUSH: Received control message: &apos;PUSH_REPLY,route-gateway 10.10.10.1,topology subnet,ping 10,ping-restart 120,ifconfig 10.10.10.2 255.255.255.0,peer-id 0,cipher AES-256-GCM&apos;
2025-08-11 19:17:03 OPTIONS IMPORT: --ifconfig/up options modified
2025-08-11 19:17:03 OPTIONS IMPORT: route-related options modified
2025-08-11 19:17:03 net_route_v4_best_gw query: dst 0.0.0.0
2025-08-11 19:17:03 net_route_v4_best_gw result: via 192.168.1.254 dev wlo1
2025-08-11 19:17:03 ROUTE_GATEWAY 192.168.1.254/255.255.255.0 IFACE=wlo1 HWADDR=e4:c7:67:0c:b7:8c
2025-08-11 19:17:03 TUN/TAP device tun0 opened
2025-08-11 19:17:03 net_iface_mtu_set: mtu 1500 for tun0
2025-08-11 19:17:03 net_iface_up: set tun0 up
2025-08-11 19:17:03 net_addr_v4_add: 10.10.10.2/24 dev tun0
2025-08-11 19:17:03 net_route_v4_add: 10.10.10.0/24 via 10.10.10.1 dev [NULL] table 0 metric -1
2025-08-11 19:17:03 Initialization Sequence Completed
2025-08-11 19:17:03 Data Channel: cipher &apos;AES-256-GCM&apos;, peer-id: 0
2025-08-11 19:17:03 Timers: ping 10, ping-restart 120
</pre>

Проверяем пинг по внутреннему IP адресу  сервера в туннеле: ping -c 4 10.10.10.1 

<pre>ping -c 4 10.10.10.1
PING 10.10.10.1 (10.10.10.1) 56(84) bytes of data.
64 bytes from 10.10.10.1: icmp_seq=1 ttl=64 time=0.924 ms
64 bytes from 10.10.10.1: icmp_seq=2 ttl=64 time=1.63 ms
64 bytes from 10.10.10.1: icmp_seq=3 ttl=64 time=1.52 ms
^C
--- 10.10.10.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2018ms
rtt min/avg/max/mdev = 0.924/1.358/1.629/0.310 ms
</pre>

Также проверяем командой ip r на хостовой машине что сеть туннеля импортирована в таблицу маршрутизации. 

<pre>ip route show
default via 192.168.1.254 dev wlo1 proto dhcp src 192.168.1.119 metric 600 
10.0.85.2 dev outline-tun0 scope link src 10.0.85.1 linkdown 
10.10.10.0/24 via 10.10.10.1 dev tun0 
10.10.10.0/24 dev tun0 proto kernel scope link src 10.10.10.2 
169.254.0.0/16 dev outline-tun0 scope link metric 1000 linkdown 
192.168.1.0/24 dev wlo1 proto kernel scope link src 192.168.1.119 metric 600 
192.168.56.0/24 dev vboxnet0 proto kernel scope link src 192.168.56.1 
</pre>
