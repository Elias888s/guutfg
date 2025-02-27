# demo2025
## <img src="3.png" width="500">
## Модуль 1
### Распределение IP:
| Имя устройства | Интерфейс | IP          | Маска           | Шлюз        |
| -------------- | --------- | ----------  | --------------- | ----------- |
| ISP            | gi1/0/1   | DHCP        |                 |             |
|                | gi1/0/2   | 172.16.5.1  | 255.255.255.240 |             |
|                | gi1/0/3   | 172.16.4.1  | 255.255.255.240 |             |
| HQ-RTR         | ge0       | 172.16.4.2  | 255.255.255.240 | 172.16.4.1  |      
|                | ge1.100   | 192.168.0.1 | 255.255.255.192 |             |      
|                | ge1.200   | 192.168.0.65| 255.255.255.240 |             |      
|                | ge1.999   | 192.168.0.81| 255.255.255.248 |             |
|                | tunnel.1  | 172.16.1.1  | 255.255.255.252 |             |      
| BR-RTR         | ge0       | 172.16.5.2  | 255.255.255.240 | 172.16.5.1  |      
|                | te0       | 192.168.1.1 | 255.255.255.224 |             |
|                | tunnel.1  | 172.16.1.2  | 255.255.255.252 |             |
| HQ-SRV         | ens33     | 192.168.0.2 | 255.255.255.192 | 192.168.0.1 |      
| HQ-CLI         | ens192    | DHCP        | 255.255.255.240 | 192.168.0.65|      
| BR-SRV         | ens192    | 192.168.1.2 | 255.255.255.224 | 192.168.1.1 |
### Выдача имени устройству:
```
ISP - hostnamectl set-hostname ISP; exec bash
HQ-RTR - hostname HQ-RTR.au-team.irpo
BR-RTR - hostname BR-RTR.au-team.irpo
HQ-SRV - hostnamectl set-hostname HQ-SRV.au-team.irpo; exec bash
BR-SRV - hostnamectl set-hostname BR-SRV.au-team.irpo; exec bash
HQ-CLI - hostnamectl set-hostname HQ-CLI.au-team.irpo; exec bash
```
### 1 Назначение IP:
#### ISP
Просмотр и создание директорий для интерфейсов
```
ip -c a
mkdir /etc/net/ifaces/ens**
mkdir /etc/net/ifaces/ens**
```
Создание настроек для интерфейсов
```
vim /etc/net/ifaces/ens**/options
TYPE=eth
DISABLED=no
NM_CONTROLLED=no
BOOTPROTO=static
CONFIG_IPv4=yes
выйти и сохранить изменения в vim :wq
cp /etc/net/ifaces/ens34/options /etc/net/ifaces/ens**
```
Выдача IP
```
echo 172.16.4.1/28 > /etc/net/ifaces/ens**/ipv4address
echo 172.16.5.1/28 > /etc/net/ifaces/ens**/ipv4address
```
включение forwarding, в строке net.ipv4.ip_forward поменять 0 на 1
```
vim /etc/net/sysctl.conf
```
Перезагрузка службы network
```
systemctl restart network
```
Проверка:
<img src="1.jpg" width="500">
### 2 Настройка NAT
Обновление и установка firewalld
```

#### ISP
```
systemctl disable NetworkManager
```
Настройки интерфейсов должны быть такими:
...
NM_CONTROLLED=no
DISABLED=no
```
Установка firewalld:
```
apt-get update && apt-get -y install firewalld && systemctl enable --now firewalld
```
Правила к исходящим пакетам (в сторону провайдера):
```
firewall-cmd --permanent --zone=public --add-interface=ens33
```
Правила к входящим пакетам (к локальной сети):
```
firewall-cmd --permanent --zone=trusted --add-interface=ens34
firewall-cmd --permanent --zone=trusted --add-interface=ens35
```
Включение NAT:
```
firewall-cmd --permanent --zone=public --add-masquerade
```
Сохранение правил:
```
firewall-cmd --complete-reload
```
включим forward
```
vim /etc/net/sysctl.conf
net.ipv4.ip_forward = 1
```
#### HQ-RTR
Выбор заводской прошивки на роутере, чтоб работали порты
```
en
no boot b-image active
no boot b-image stable
no boot a-image active
no boot a-image stable
```
перезагружаем машину
```
Включение - en
Вход в конфигурационный режим - conf t
Просмотр портов - do sh port br
просмотр истории команд - do sh run
```
Создание интерфейса и назначение ему IP
```
int TO-ISP
ip address 172.16.4.14/28
no shutdown
ex
ip name-server "DNS от ISP который в /etc/resolv.conf"
```
Создание интерфейсов для VLAN
```
interface 100
 ip address 192.168.1.1/26
!
interface 200
 ip address 192.168.1.65/28
!
interface 999
 ip address 192.168.1.81/29
!
```
Создание для каждого VLAN своего service-instance
```
port te0
 service-instance isp
  encapsulation untagged
  connect ip interface isp
ex
ex
port te1
 service-instance 100
  encapsulation dot1q 100
  rewrite pop 1
  connect ip interface 100
 service-instance 200
  encapsulation dot1q 200
  rewrite pop 1
  connect ip interface 200
service-instance 999
  encapsulation dot1q 999
  rewrite pop 1
  connect ip interface 999
ip route 0.0.0.0/0 172.16.4.1
ip name-server
do wr
```
Создание nat
```
ip nat pool INTERNET 192.168.1.1-192.168.1.78
ip nat source dynamic inside-to-outside pool INTERNET overload 172.16.4.14
int isp
ip nat outside
ex
int 100
ip nat instide
ex
int 200
ip nat instide
ex
int 999
ip nat instide
ex
```
Создание GRE тоннеля
```
interface tunnel.1
 ip mtu 1400
 ip address 172.16.0.1/30
 ip tunnel 172.16.4.14 172.16.5.14 mode gre
```
ospf
HQ-RTR
```
en
conf t
router ospf 1
network 172.16.0.1/30 area 0
network 192.168.1.0/26 area 0
passive-interface default
no passive-interface tunnel0
area 0 authentication
exit
write memory
Маршрут в сторону ISP
```
ip route 0.0.0.0/0 172.16.4.1
do wr
```
Проверка:
<img src="2.jpg" width="500">
```
#### HQ-SRV
Просмотр интерфейсов
```
ip -c a
```
Создание настроек интерфейсов
```
vim /etc/net/ifaces/ens**/options
возможность делать изменения в vim клавиша I
TYPE=eth
DISABLED=no
NM_CONTROLLED=no
BOOTPROTO=static
CONFIG_IPv4=yes
выйти и сохранить изменения в vim :wq
```
Выдача DNS
```
vim /etc/resolv.conf
возможность делать изменения в vim клавиша I
nameserver "DNS от ISP который в /etc/resolv.conf"
выйти и сохранить изменения в vim :wq
```
Выдача IP
```
echo 192.168.0.2/26 > /etc/net/ifaces/ens**/ipv4address
```
Выдача шлюза
```
echo default via 192.168.0.1 > /etc/net/ifaces/ens**/ipv4route
```
включение forwarding, в строке net.ipv4.ip_forward поменять 0 на 1
```
vim /etc/net/sysctl.conf
```
Перезагрузка службы network
```
systemctl restart network
```
#### BR-RTR
Выбор заводской прошивки на роутере, чтоб работали порты
```
en
no boot b-image active
no boot b-image stable
no boot a-image active
no boot a-image stable
перезагружаем машину
```
```
Включение - en
Вход в конфигурационный режим - conf t
Просмотр портов - do sh port br
```
Создание интерфейса и назначение ему IP
```
int isp
ip address 172.16.5.14/28
no shutdown
int lan
ip address 192.168.2.1/27
no shutdown
```
Привязка созданных интерфейсов к портам
```
port te0
service-instance isp
encapsulation untagged
connect ip interface isp
ex
port te1
service-instance lan
encapsulation untagged
connect ip interface lan
ex
ex
ip route 0.0.0.0/0 172.16.5.1
ip nameserver

```
Создание nat
```
ip nat pool INTERNET 192.168.2.1-192.168.2.30
ip nat source dynamic inside-to-outside pool INTERNET overload 172.16.5.14
int isp
ip nat outside
ex
int lan
ip nat instide
ex
```
Создание GRE тоннеля
```
interface tunnel.1
 ip mtu 1400
 ip address 172.16.0.2/30
 ip tunnel 172.16.5.14 172.16.4.14 mode gre
 ```
 BR-RTR OSPF
```
en
conf t
router ospf 1
network 172.16.0.2/30 area 0
network 192.168.2.0/27 area 0
passive-interface default
no passive-interface tunnel.0
area 0 authentication
exit
write memory
```
Маршрут в сторону ISP
```
ip route 0.0.0.0/0 172.16.5.1
do wr
```
#### BR-SRV
Просмотр интерфейсов
```
ip -c a
```
Создание настроек интерфейсов
```
vim /etc/net/ifaces/ens**/options
возможность делать изменения в vim клавиша I
TYPE=eth
DISABLED=no
NM_CONTROLLED=no
BOOTPROTO=static
CONFIG_IPv4=yes
выйти и сохранить изменения в vim :wq
```
Выдача DNS
```
vim /etc/resolv.conf
возможность делать изменения в vim клавиша I
nameserver "DNS от ISP который в /etc/resolv.conf"
выйти и сохранить изменения в vim :wq
```
Выдача IP
```
echo 192.168.1.2/27 > /etc/net/ifaces/ens**/ipv4address
```
Выдача шлюза
```
echo default via 192.168.1.1 > /etc/net/ifaces/ens**/ipv4route
```
включение forwarding, в строке net.ipv4.ip_forward поменять 0 на 1
```
vim /etc/net/sysctl.conf
```
Перезагрузка службы network
```
systemctl restart network
```
Проверка, должен пинговаться DNS от ISP:
```
do ping 'DNS от ISP'
Прекратить пинговать сочетание клавиш Ctrl+C
```
### 3 Создание локальных учетных записей
#### HQ-SRV и BR-SRV
```
useradd -m -u 1010 sshuser
passwd sshuser
nano /etc/sudoers.d/sshuser
sshuser ALL=(ALL) NOPASSWD:ALL
```
Проверка
```
su - sshuser
sudo whoami
```
#### HQ-RTR и BR-RTR
```
conf t
username net_admin
password P@ssw0rd
role admin
do wr
```
### 4 Настройка SSH на HQ-SRV и BR-SRV
Делаем бэкап конфига:
```
cp /etc/openssh/sshd_config /etc/openssh/sshd_config.bak
```
Редактируем файл конфигурации SSH
```
vim /etc/openssh/sshd_config
```
Изменяем следующие параметры, раскомменчиваем, если параметр не находится то добавляем
```
Port 2024
MaxAuthTries 3
Banner /etc/openssh/banner
AllowUsers sshuser
```
Создаем файл с баннером
```
vim /etc/openssh/banner
```
```
WARNING!!!!

AUTHORIZED ACCESS ONLY!!!!
```
Перезагружаем SSH
```
systemctl restart sshd
```
### 5 Настраиваем OSPF
#### HQ-RTR
```
router ospf 1
 network 172.16.1.0/30 area 0
 network 192.168.0.0/28 area 0
!
interface tunnel.1
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 Demo2025
 ip ospf network point-to-point
do wr
```
#### OSPF
```
router ospf 1
 network 172.16.1.0/30 area 0
 network 192.168.1.0/27 area 0
!
interface tunnel.1
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 Demo2025
 ip ospf network point-to-point
do wr
```
Проверка:
```
do sh ip ospf neighbor 
do sh ip ospf interface brief
do sh ip route
```
### 6 Настройка DHCP
#### HQ-RTR
```
ip pool HQ-NET200 192.168.0.66-192.168.0.70
!
dhcp-server 1
 lease 86400
 mask 255.255.255.0
 pool HQ-NET200 1
  dns 192.168.0.2
  domain-name au-team.irpo
  gateway 192.168.0.65
  mask 255.255.255.240
!
interface HQ-CLI
 dhcp-server 1
!
do wr
```
#### HQ-CLI
<img src="5.png" width="500">
<img src="6.png" width="500">
<img src="7.png" width="500">
<img src="8.png" width="500">
Если сработало то в ip -c a будет так
<img src="9.png" width="500">
### 7 Настройка DNS с помощью bind
#### HQ-SRV
Устанавливаем bind:

```
apt-get install bind bind-utils
```

Редактируем конфиг:

```
vim /var/lib/bind/etc/options.conf
```

Изменяем следующие параметры
<img src="10.png" width='500'>
<img src="11.png" width='500'>

Проверяем на ошибки

```
named-checkconf
```

Если ошибок нет, то запускаем `bind`

```
systemctl enable --now bind
```
Проверяем что `bind` работает

```
systemctl status bind
```

Редактируем `resolv.conf`

```
vim /etc/resolv.conf 
search au-team.irpo
nameserver 127.0.0.1
nameserver 192.168.0.2
nameserver 'DNS от ISP'
```

Перезагружаем сеть

```
systemctl restart network
```
Создаем зону прямого просмотра
```
vim  /var/lib/bind/etc/local.conf
```

```
zone "au-team.irpo" {
        type master;
        file "au-team.irpo.db";
};
```
Создаем копию файла-шаблона прямой зоны `/var/lib/bind/etc/zone/localdomain`

```
cp /var/lib/bind/etc/zone/localdomain  /var/lib/bind/etc/zone/au-team.irpo.db
```

Задаем права на файл
```
chown named. /var/lib/bind/etc/zone/au-team.irpo.db

chmod 600 /var/lib/bind/etc/zone/au-team.irpo.db
```

Открываем для редактирования

```
vim /var/lib/bind/etc/zone/au-team.irpo.db
```

```
$TTL    1D
@       IN      SOA     au-team.irpo. root.au-team.irpo. (
                                2024102200      ; serial
                                12H             ; refresh
                                1H              ; retry
                                1W              ; expire
                                1H              ; ncache
                        )
        IN      NS      au-team.irpo.
        IN      A       192.168.0.2
hq-rtr  IN      A       192.168.0.1
br-rtr  IN      A       192.168.1.1
hq-srv  IN      A       192.168.0.2
hq-cli  IN      A       192.168.0.66
br-srv  IN      A       192.168.1.2
moodle  IN      CNAME   hq-rtr
wiki    IN      CNAME   hq-rtr
```

Проверяем, что зона настроена Предварительно

```
named-checkconf -z
```
Создаем зону обратного просмотра и PTR записи
```
vim  /var/lib/bind/etc/local.conf
```

```
zone "0.168.192.in-addr.arpa" {
        type master;
        file "au-team.irpo_rev.db";
};
```
Копируем шаблон файла

```
cp /var/lib/bind/etc/zone/{127.in-addr.arpa,au-team.irpo_rev.db}
```

Задаем права на файл
```
chown named. /var/lib/bind/etc/zone/au-team.irpo_rev.db

chmod 600 /var/lib/bind/etc/zone/au-team.irpo_rev.db
```

Открываем для редактирования

```
vim /var/lib/bind/etc/zone/au-team.irpo_rev.db
```

Вставляем в него следующее содержимое.

```
$TTL    1D
@       IN      SOA     au-team.irpo. root.au-team.irpo. (
                                2024102200      ; serial
                                12H             ; refresh
                                1H              ; retry
                                1W              ; expire
                                1H              ; ncache
                        )
        IN      NS      au-team.irpo.
1       IN      PTR     hq-rtr.au-team.irpo.
2       IN      PTR     hq-srv.au-team.irpo.
66      IN      PTR     hq-cli.au-team.irpo.
```

Проверяем

```
named-checkconf -z
```
### 8 Настройка часового пояса
#### HQ-SRV, HQ-CLI, BR-SRV

Проверяем какой часовой пояс установлен

```
timedatectl status
```
Если time-zone не Asia/Yekaterinburg, то устанавливаем его
```
timedatectl set-timezone Asia/Yekaterinburg
```
#### HQ-RTR, BR-RTR
```
conf t
ntp timezone utc+5
do wr
```
Проверяем:

```
do show ntp timezone
```
