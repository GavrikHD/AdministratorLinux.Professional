# Установка Zabbix

устанавлиуаем зависимости

>sudo apt install -y wget curl vim gnupg2 apt-transport-https software-properties-common

Добовляем репозиторий zabbix 

>wget https://repo.zabbix.com/zabbix/6.4/ubuntu/pool/main/z/zabbix-release/zabbix-release_6.4-1+ubuntu24.04_all.deb
>sudo dpkg -i zabbix-release_6.4-1+ubuntu24.04_all.deb
>sudo apt update

Устанавливаем Zabbix Server, Frontend и Agent

>sudo apt install -y zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-sql-scripts zabbix-agent mariadb-server

Настройка базы данных

>sudo mysql -uroot -p

<pre>CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
CREATE USER 'zabbix'@'localhost' IDENTIFIED BY 'Passw@060625';
GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost';
FLUSH PRIVILEGES;
EXIT;</pre>

импортируем схему

>sudo zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql -uzabbix -p zabbix

# Настройка конфигурации Zabbix Server

>sudo nano /etc/zabbix/zabbix_server.conf

проверим кунфигурацию и разкомунтируем некоторые параметры

<pre>DBHost=localhost
DBName=zabbix
DBUser=zabbix
DBPassword=strong_password</pre>

# Настройка веб-интерфейса

настраиваем php

>sudo nano /etc/php/8.1/apache2/php.ini

укажите следующие параметры

<pre>memory_limit = 128M
post_max_size = 16M
upload_max_filesize = 2M
max_execution_time = 300
max_input_time = 300
date.timezone = Europe/Moscow  # Укажите свою временную зону</pre>

перезапускаем apache

>sudo systemctl restart apache2

Далее заходим на страницу zabbix сервера и заканчиваем настройку.

После захода на страницу сурвера, начинаем делать наш дашбоард

Добавляем хост, привязываем шаблон к хосту, настраиваем графики в дашбоард

![alt text](<Снимок экрана от 2025-06-06 16-04-30.png>)
