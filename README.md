Поняла. Тебе нужна **не отдельная памятка**, а **полная методичка в твоём стиле**, где после каждого пункта сразу вставлена проверка: “после этого проверяем вот так”. Без длинных объяснений, без “что может сломаться”, без лекции от занудного линукс-призрака. Держи нормальный полный текст для README.

````markdown
# МОДУЛЬ 1 + МОДУЛЬ 2: БЫСТРАЯ МЕТОДИЧКА С ПРОВЕРКАМИ

Формат: как в методичке 2026, без EOF.  
Все файлы открывать через `nano` или `mcedit`.  
Сначала делаешь команды, потом сразу проверку. Если проверка не проходит — дальше не идешь.

---

# 0. Адреса, которые должны получиться

| Устройство/интерфейс | IP | Шлюз |
|---|---|---|
| ISP eth0 | DHCP | шлюз DHCP |
| ISP eth1 | 172.16.1.1/28 | - |
| ISP eth2 | 172.16.2.1/28 | - |
| HQ-RTR eth0 | 172.16.1.2/28 | 172.16.1.1 |
| HQ-RTR eth1.100 | 192.168.1.1/27 | - |
| HQ-RTR eth1.200 | 192.168.2.1/28 | - |
| HQ-RTR eth1.999 | 192.168.99.1/29 | - |
| HQ-RTR gre1 | 10.0.0.1/30 | - |
| HQ-SRV ens18 VLAN100 | 192.168.1.2/27 | 192.168.1.1 |
| HQ-CLI ens18 VLAN200 | DHCP 192.168.2.x/28 | 192.168.2.1 |
| BR-RTR eth0 | 172.16.2.2/28 | 172.16.2.1 |
| BR-RTR eth1 | 192.168.3.1/28 | - |
| BR-RTR gre1 | 10.0.0.2/30 | - |
| BR-SRV ens18 | 192.168.3.2/28 | 192.168.3.1 |

---

# 1. 40-минутный порядок выполнения

| Шаг | Где | Что сделать |
|---|---|---|
| 1 | Все ВМ | hostname + timezone + IP |
| 2 | ISP | ip_forward + NAT |
| 3 | HQ-RTR/BR-RTR | net_admin, GRE, NAT, FRR/OSPF |
| 4 | HQ-SRV/BR-SRV | sshuser, SSH 2026 на ОБОИХ серверах |
| 5 | HQ-RTR/HQ-CLI | DHCP для HQ-CLI |
| 6 | HQ-SRV | dnsmasq со всеми A/PTR записями |
| 7 | Модуль 2 | Samba, RAID, NFS, chrony, ansible, docker, LAMP, DNAT, nginx, auth |

---

# МОДУЛЬ 1. Настройка сетевой инфраструктуры

---

## 1.2 ISP: интерфейсы, маршрутизация, NAT

На ISP:

```bash
hostnamectl set-hostname isp.au-team.irpo
hostname
timedatectl set-timezone Asia/Yekaterinburg
hostname -f
timedatectl
````

Открываем файл:

```bash
nano /etc/network/interfaces
```

Оставить `lo`, ниже добавить:

```text
auto eth0
iface eth0 inet dhcp

auto eth1
iface eth1 inet static
 address 172.16.1.1
 netmask 255.255.255.240

auto eth2
iface eth2 inet static
 address 172.16.2.1
 netmask 255.255.255.240
```

Перезапускаем сеть:

```bash
systemctl restart networking
```

Включаем маршрутизацию:

```bash
nano /etc/sysctl.conf
```

Добавить:

```text
net.ipv4.ip_forward=1
```

Применить:

```bash
sysctl -p
```

Настраиваем NAT:

```bash
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables-save > /etc/rules.v4
```

Автозагрузка правил:

```bash
crontab -e
```

В конец добавить:

```text
@reboot /sbin/iptables-restore < /etc/rules.v4
@reboot /sbin/sysctl -p
```

DNS:

```bash
echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

### Проверка после 1.2

На ISP:

```bash
hostname -f
timedatectl
ip -br a
ip r
cat /proc/sys/net/ipv4/ip_forward
iptables -t nat -L -n -v
cat /etc/resolv.conf
```

Должно быть:

```text
hostname: isp.au-team.irpo
eth0: DHCP
eth1: 172.16.1.1/28
eth2: 172.16.2.1/28
ip_forward: 1
в NAT есть MASQUERADE
```

Проверка интернета:

```bash
ping -c 3 8.8.8.8
```

---

## 1.3 HQ-RTR: VLAN, GRE, NAT

На HQ-RTR:

```bash
hostnamectl set-hostname hq-rtr.au-team.irpo
hostname
timedatectl set-timezone Asia/Yekaterinburg
hostname -f
timedatectl
```

Открываем файл:

```bash
nano /etc/network/interfaces
```

Вписать:

```text
auto eth0
iface eth0 inet static
 address 172.16.1.2
 netmask 255.255.255.240
 gateway 172.16.1.1

auto eth1.100
iface eth1.100 inet static
 address 192.168.1.1
 netmask 255.255.255.224
 vlan-raw-device eth1

auto eth1.200
iface eth1.200 inet static
 address 192.168.2.1
 netmask 255.255.255.240
 vlan-raw-device eth1

auto eth1.999
iface eth1.999 inet static
 address 192.168.99.1
 netmask 255.255.255.248
 vlan-raw-device eth1

auto gre1
iface gre1 inet static
 address 10.0.0.1
 netmask 255.255.255.252
 pre-up ip tunnel del gre1 2>/dev/null || true
 pre-up ip tunnel add gre1 mode gre local 172.16.1.2 remote 172.16.2.2 ttl 255
 post-down ip tunnel del gre1 2>/dev/null || true
```

DNS:

```bash
echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

Перезапуск сети:

```bash
systemctl restart networking
```

Включаем маршрутизацию:

```bash
nano /etc/sysctl.conf
```

Добавить:

```text
net.ipv4.ip_forward=1
```

Применить:

```bash
sysctl -p
```

NAT:

```bash
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables-save > /etc/rules.v4
```

Добавляем пользователя:

```bash
adduser net_admin
usermod -aG sudo net_admin
nano /etc/sudoers
```

Строку `%sudo` привести к виду:

```text
%sudo ALL=(ALL:ALL) NOPASSWD: ALL
```

### Проверка после 1.3

На HQ-RTR:

```bash
hostname -f
ip -br a
ip r
cat /proc/sys/net/ipv4/ip_forward
iptables -t nat -L -n -v
id net_admin
```

Должно быть:

```text
hostname: hq-rtr.au-team.irpo
eth0: 172.16.1.2/28
eth1.100: 192.168.1.1/27
eth1.200: 192.168.2.1/28
eth1.999: 192.168.99.1/29
gre1: 10.0.0.1/30
gateway: 172.16.1.1
ip_forward: 1
пользователь net_admin есть
```

Проверка связи:

```bash
ping -c 3 172.16.1.1
ping -c 3 8.8.8.8
```

---

## 1.4 BR-RTR: интерфейсы, GRE, NAT

На BR-RTR:

```bash
hostnamectl set-hostname br-rtr.au-team.irpo
hostname
timedatectl set-timezone Asia/Yekaterinburg
hostname -f
timedatectl
```

Открываем файл:

```bash
nano /etc/network/interfaces
```

Вписать:

```text
auto eth0
iface eth0 inet static
 address 172.16.2.2
 netmask 255.255.255.240
 gateway 172.16.2.1

auto eth1
iface eth1 inet static
 address 192.168.3.1
 netmask 255.255.255.240

auto gre1
iface gre1 inet static
 address 10.0.0.2
 netmask 255.255.255.252
 pre-up ip tunnel del gre1 2>/dev/null || true
 pre-up ip tunnel add gre1 mode gre local 172.16.2.2 remote 172.16.1.2 ttl 255
 post-down ip tunnel del gre1 2>/dev/null || true
```

DNS:

```bash
echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

Перезапуск сети:

```bash
systemctl restart networking
```

Включаем маршрутизацию:

```bash
nano /etc/sysctl.conf
```

Добавить:

```text
net.ipv4.ip_forward=1
```

Применить:

```bash
sysctl -p
```

NAT:

```bash
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables-save > /etc/rules.v4
```

Добавляем пользователя:

```bash
adduser net_admin
usermod -aG sudo net_admin
nano /etc/sudoers
```

Строку `%sudo` привести к виду:

```text
%sudo ALL=(ALL:ALL) NOPASSWD: ALL
```

### Проверка после 1.4

На BR-RTR:

```bash
hostname -f
ip -br a
ip r
cat /proc/sys/net/ipv4/ip_forward
iptables -t nat -L -n -v
id net_admin
```

Должно быть:

```text
hostname: br-rtr.au-team.irpo
eth0: 172.16.2.2/28
eth1: 192.168.3.1/28
gre1: 10.0.0.2/30
gateway: 172.16.2.1
ip_forward: 1
пользователь net_admin есть
```

Проверка GRE:

```bash
ping -c 3 10.0.0.1
```

---

## 1.5 HQ-SRV: IP, sshuser, SSH 2026

В Proxmox на HQ-SRV поставить VLAN Tag: `100`.

На HQ-SRV:

```bash
hostnamectl set-hostname hq-srv.au-team.irpo
hostname
timedatectl set-timezone Asia/Yekaterinburg
hostname -f
timedatectl
```

Настройка IP:

```bash
mcedit /etc/net/ifaces/ens18/options
```

Содержимое:

```text
TYPE=eth
BOOTPROTO=static
CONFIG_IPV4=yes
DISABLED=no
```

```bash
mcedit /etc/net/ifaces/ens18/ipv4address
```

Содержимое:

```text
192.168.1.2/27
```

```bash
mcedit /etc/net/ifaces/ens18/ipv4route
```

Содержимое:

```text
default via 192.168.1.1
```

DNS:

```bash
echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

Перезапуск сети:

```bash
systemctl restart network
```

Добавляем пользователя:

```bash
useradd -u 2026 -m sshuser
passwd sshuser
usermod -aG wheel sshuser
mcedit /etc/sudoers
```

Раскомментировать или добавить:

```text
%wheel ALL=(ALL) NOPASSWD: ALL
```

Проверка sudo:

```bash
su - sshuser
sudo id
exit
```

Настраиваем SSH:

```bash
mcedit /etc/openssh/sshd_config
```

Добавить или проверить строки:

```text
Port 2026
AllowUsers sshuser
MaxAuthTries 2
Banner /root/banner
PermitRootLogin no
```

Баннер:

```bash
mcedit /root/banner
```

Содержимое:

```text
Authorized access only
```

Перезапуск SSH:

```bash
systemctl restart sshd
systemctl enable sshd
```

### Проверка после настройки HQ-SRV

На HQ-SRV:

```bash
hostname -f
ip -br a
ip r
id sshuser
ss -tulpn | grep 2026
systemctl status sshd
```

Должно быть:

```text
hostname: hq-srv.au-team.irpo
ens18: 192.168.1.2/27
gateway: 192.168.1.1
sshuser есть
sshd слушает 2026
```

Проверка связи:

```bash
ping -c 3 192.168.1.1
ping -c 3 172.16.1.1
ping -c 3 8.8.8.8
```

Проверка SSH:

```bash
ssh -p 2026 sshuser@192.168.1.2
```

---

## 1.5 BR-SRV: IP, sshuser, SSH 2026

BR-SRV без VLAN, если по схеме он сидит в обычной сети BR.

На BR-SRV:

```bash
hostnamectl set-hostname br-srv.au-team.irpo
hostname
timedatectl set-timezone Asia/Yekaterinburg
hostname -f
timedatectl
```

Настройка IP:

```bash
mcedit /etc/net/ifaces/ens18/options
```

Содержимое:

```text
TYPE=eth
BOOTPROTO=static
CONFIG_IPV4=yes
DISABLED=no
```

```bash
mcedit /etc/net/ifaces/ens18/ipv4address
```

Содержимое:

```text
192.168.3.2/28
```

```bash
mcedit /etc/net/ifaces/ens18/ipv4route
```

Содержимое:

```text
default via 192.168.3.1
```

DNS:

```bash
echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

Перезапуск сети:

```bash
systemctl restart network
```

Добавляем пользователя:

```bash
useradd -u 2026 -m sshuser
passwd sshuser
usermod -aG wheel sshuser
mcedit /etc/sudoers
```

Раскомментировать или добавить:

```text
%wheel ALL=(ALL) NOPASSWD: ALL
```

Проверка sudo:

```bash
su - sshuser
sudo id
exit
```

Настраиваем SSH:

```bash
mcedit /etc/openssh/sshd_config
```

Добавить или проверить строки:

```text
Port 2026
AllowUsers sshuser
MaxAuthTries 2
Banner /root/banner
PermitRootLogin no
```

Баннер:

```bash
mcedit /root/banner
```

Содержимое:

```text
Authorized access only
```

Перезапуск SSH:

```bash
systemctl restart sshd
systemctl enable sshd
```

### Проверка после настройки BR-SRV

На BR-SRV:

```bash
hostname -f
ip -br a
ip r
id sshuser
ss -tulpn | grep 2026
systemctl status sshd
```

Должно быть:

```text
hostname: br-srv.au-team.irpo
ens18: 192.168.3.2/28
gateway: 192.168.3.1
sshuser есть
sshd слушает 2026
```

Проверка связи:

```bash
ping -c 3 192.168.3.1
ping -c 3 8.8.8.8
```

Проверка SSH:

```bash
ssh -p 2026 sshuser@192.168.3.2
```

---

## 1.8 FRR/OSPF на HQ-RTR и BR-RTR

На HQ-RTR и BR-RTR:

```bash
nano /etc/apt/sources.list
apt-get update
apt-get install -y frr
nano /etc/frr/daemons
```

Изменить:

```text
ospfd=yes
```

Запуск:

```bash
systemctl enable frr --now
systemctl restart frr
```

### HQ-RTR

```bash
vtysh
```

Внутри `vtysh`:

```text
conf t
interface gre1
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 123qweR%
exit
router ospf
 passive-interface default
 no passive-interface gre1
 network 10.0.0.0/30 area 0
 network 192.168.1.0/27 area 0
 network 192.168.2.0/28 area 0
 network 192.168.99.0/29 area 0
exit
end
wr
show ip ospf neighbor
show ip route ospf
exit
```

### BR-RTR

```bash
vtysh
```

Внутри `vtysh`:

```text
conf t
interface gre1
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 123qweR%
exit
router ospf
 passive-interface default
 no passive-interface gre1
 network 10.0.0.0/30 area 0
 network 192.168.3.0/28 area 0
exit
end
wr
show ip ospf neighbor
show ip route ospf
exit
```

### Проверка после 1.8

На HQ-RTR:

```bash
vtysh -c "show ip ospf neighbor"
vtysh -c "show ip route ospf"
```

На BR-RTR:

```bash
vtysh -c "show ip ospf neighbor"
vtysh -c "show ip route ospf"
```

Должно быть:

```text
OSPF neighbor есть
состояние Full
```

Проверка с HQ-RTR:

```bash
ping -c 3 192.168.3.1
ping -c 3 192.168.3.2
```

Проверка с BR-RTR:

```bash
ping -c 3 192.168.1.1
ping -c 3 192.168.1.2
```

Проверка с HQ-SRV:

```bash
ping -c 3 192.168.3.2
```

Проверка с BR-SRV:

```bash
ping -c 3 192.168.1.2
```

---

## 1.9 DHCP на HQ-RTR для HQ-CLI

На HQ-RTR:

```bash
apt-get update && apt-get install -y isc-dhcp-server
nano /etc/default/isc-dhcp-server
```

Строка:

```text
INTERFACESv4="eth1.200"
```

Копия конфига:

```bash
cp /etc/dhcp/dhcpd.conf /etc/dhcp/dhcpd.bkp
nano /etc/dhcp/dhcpd.conf
```

В конец добавить:

```text
subnet 192.168.2.0 netmask 255.255.255.240 {
  range 192.168.2.2 192.168.2.14;
  option routers 192.168.2.1;
  option domain-name-servers 192.168.1.2;
  option domain-name "au-team.irpo";
}
```

Запуск:

```bash
systemctl restart isc-dhcp-server
systemctl enable isc-dhcp-server
systemctl status isc-dhcp-server
```

На HQ-CLI поставить получение IP по DHCP.

В Proxmox на HQ-CLI поставить VLAN Tag: `200`.

На HQ-CLI:

```bash
hostnamectl set-hostname hq-cli.au-team.irpo
hostname
timedatectl set-timezone Asia/Yekaterinburg
hostname -f
timedatectl
```

DNS на HQ-CLI:

```bash
nano /etc/resolv.conf
```

Вписать:

```text
search au-team.irpo
nameserver 192.168.1.2
nameserver 8.8.8.8
```

### Проверка после 1.9

На HQ-RTR:

```bash
systemctl status isc-dhcp-server
journalctl -u isc-dhcp-server -n 20
```

На HQ-CLI:

```bash
ip -br a
ip r
cat /etc/resolv.conf
```

Должно быть:

```text
HQ-CLI получил IP из 192.168.2.0/28
шлюз: 192.168.2.1
DNS: 192.168.1.2
```

Проверка с HQ-CLI:

```bash
ping -c 3 192.168.2.1
ping -c 3 192.168.1.2
ping -c 3 192.168.3.2
```

---

## 1.10 DNS dnsmasq на HQ-SRV

На HQ-SRV:

```bash
apt-get update && apt-get install -y dnsmasq
systemctl disable bind --now
mcedit /etc/dnsmasq.conf
```

В конец добавить:

```text
interface=*
no-resolv
no-hosts
listen-address=192.168.1.2
domain=au-team.irpo
expand-hosts
server=8.8.8.8
address=/hq-rtr.au-team.irpo/192.168.1.1
address=/br-rtr.au-team.irpo/192.168.3.1
address=/hq-srv.au-team.irpo/192.168.1.2
address=/hq-cli.au-team.irpo/192.168.2.2
address=/br-srv.au-team.irpo/192.168.3.2
address=/docker.au-team.irpo/172.16.1.1
address=/web.au-team.irpo/172.16.2.1
ptr-record=1.1.168.192.in-addr.arpa,hq-rtr.au-team.irpo
ptr-record=2.1.168.192.in-addr.arpa,hq-srv.au-team.irpo
ptr-record=2.2.168.192.in-addr.arpa,hq-cli.au-team.irpo
```

Запуск:

```bash
systemctl restart dnsmasq
systemctl enable dnsmasq
systemctl status dnsmasq
```

### Проверка после 1.10

На HQ-SRV:

```bash
systemctl status dnsmasq
ss -tulpn | grep :53
ss -ulpn | grep :53
```

С HQ-CLI:

```bash
nslookup hq-srv.au-team.irpo 192.168.1.2
nslookup hq-rtr.au-team.irpo 192.168.1.2
nslookup br-srv.au-team.irpo 192.168.1.2
nslookup web.au-team.irpo 192.168.1.2
nslookup docker.au-team.irpo 192.168.1.2
ping -c 3 hq-srv.au-team.irpo
```

Должно быть:

```text
имена разрешаются
dnsmasq active
порт 53 слушается
```

---

# МОДУЛЬ 2. Сетевое администрирование

---

## 2.1 ISP: chrony, nginx, web-auth

### Chrony на ISP

```bash
apt-get update
apt-get install -y chrony
nano /etc/chrony/chrony.conf
```

Старые `pool/server` закомментировать, в конец добавить:

```text
server 0.ru.pool.ntp.org iburst
local stratum 5
allow 172.16.0.0/16
allow 192.168.0.0/16
```

Запуск:

```bash
systemctl restart chrony || systemctl restart chronyd
systemctl enable chrony || systemctl enable chronyd
chronyc sources
```

### Nginx reverse proxy + пароль на web.au-team.irpo

```bash
apt-get install -y nginx apache2-utils
htpasswd -cb /etc/nginx/.htpasswd WEB P@ssw0rd
nano /etc/nginx/sites-available/default
```

В файл `/etc/nginx/sites-available/default` вписать:

```nginx
server {
    listen 80;
    server_name web.au-team.irpo;

    auth_basic "Restricted";
    auth_basic_user_file /etc/nginx/.htpasswd;

    location / {
        proxy_pass http://172.16.1.2:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 80;
    server_name docker.au-team.irpo;

    location / {
        proxy_pass http://172.16.2.2:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Проверка конфига и запуск:

```bash
nginx -t
systemctl enable nginx --now
systemctl restart nginx
systemctl status nginx
```

### Проверка после 2.1

На ISP:

```bash
systemctl status chrony || systemctl status chronyd
chronyc sources
nginx -t
systemctl status nginx
ss -tulpn | grep :80
cat /etc/nginx/.htpasswd
```

Должно быть:

```text
chrony active
nginx active
nginx -t без ошибок
порт 80 слушается
.htpasswd создан
```

Важно:

```text
После 2.1 проверяется только запуск nginx.
web.au-team.irpo полностью проверяется после 2.4.
docker.au-team.irpo полностью проверяется после 2.5.
```

---

## 2.2 HQ-RTR: chrony клиент + проброс портов на HQ-SRV

### Chrony клиент

На HQ-RTR:

```bash
apt-get update
apt-get install -y chrony
nano /etc/chrony/chrony.conf
```

Старые `pool/server` закомментировать, в конец добавить:

```text
server 172.16.1.1 iburst
```

Запуск:

```bash
systemctl restart chrony || systemctl restart chronyd
systemctl enable chrony || systemctl enable chronyd
chronyc sources
```

### DNAT на HQ-SRV

```bash
iptables -t nat -A PREROUTING -d 172.16.1.2 -p tcp --dport 8080 -j DNAT --to-destination 192.168.1.2:80
iptables -t nat -A PREROUTING -d 172.16.1.2 -p tcp --dport 2026 -j DNAT --to-destination 192.168.1.2:2026
iptables-save > /etc/rules.v4
iptables -t nat -L -n -v
```

### Проверка после 2.2

На HQ-RTR:

```bash
chronyc sources
iptables -t nat -L -n -v
iptables-save | grep 8080
iptables-save | grep 2026
```

Должно быть:

```text
chrony видит 172.16.1.1
есть DNAT 8080 на 192.168.1.2:80
есть DNAT 2026 на 192.168.1.2:2026
```

Проверка SSH через проброс:

```bash
ssh -p 2026 sshuser@172.16.1.2
```

---

## 2.3 BR-RTR: chrony клиент + проброс портов на BR-SRV

### Chrony клиент

На BR-RTR:

```bash
apt-get update
apt-get install -y chrony
nano /etc/chrony/chrony.conf
```

Старые `pool/server` закомментировать, в конец добавить:

```text
server 172.16.1.1 iburst
```

Запуск:

```bash
systemctl restart chrony || systemctl restart chronyd
systemctl enable chrony || systemctl enable chronyd
chronyc sources
```

### DNAT на BR-SRV

```bash
iptables -t nat -A PREROUTING -d 172.16.2.2 -p tcp --dport 8080 -j DNAT --to-destination 192.168.3.2:8080
iptables -t nat -A PREROUTING -d 172.16.2.2 -p tcp --dport 2026 -j DNAT --to-destination 192.168.3.2:2026
iptables-save > /etc/rules.v4
iptables -t nat -L -n -v
```

### Проверка после 2.3

На BR-RTR:

```bash
chronyc sources
iptables -t nat -L -n -v
iptables-save | grep 8080
iptables-save | grep 2026
```

Должно быть:

```text
chrony видит 172.16.1.1
есть DNAT 8080 на 192.168.3.2:8080
есть DNAT 2026 на 192.168.3.2:2026
```

Проверка SSH через проброс:

```bash
ssh -p 2026 sshuser@172.16.2.2
```

---

## 2.4 HQ-SRV: RAID0, NFS, chrony, LAMP

### 2.4.1 RAID0

В Proxmox заранее добавить два диска по 1 ГБ.

На HQ-SRV:

```bash
lsblk
apt-get update
apt-get install -y mdadm
mdadm --create /dev/md0 --level=0 --raid-devices=2 /dev/sdb /dev/sdc
mdadm --detail --scan > /etc/mdadm.conf
fdisk /dev/md0
```

В `fdisk` нажать:

```text
n
p
Enter
Enter
Enter
w
```

Форматирование и монтирование:

```bash
mkfs.ext4 /dev/md0p1
mkdir -p /raid
mount /dev/md0p1 /raid
df -h
```

Автомонтирование RAID через systemd:

```bash
systemctl --force --full edit raid.mount
```

Вписать:

```text
[Unit]
Description=Mount RAID

[Mount]
What=/dev/md0p1
Where=/raid
Type=ext4
Options=defaults

[Install]
WantedBy=multi-user.target
```

Запуск:

```bash
systemctl enable raid.mount --now
mount -a
df -h | grep raid
```

### Проверка RAID после 2.4.1

На HQ-SRV:

```bash
lsblk
cat /proc/mdstat
mdadm --detail /dev/md0
df -h | grep raid
systemctl status raid.mount
```

Должно быть:

```text
есть /dev/md0
есть /dev/md0p1
/raid смонтирован
raid.mount active
```

---

### 2.4.2 NFS-сервер

На HQ-SRV:

```bash
apt-get install -y nfs-server
mkdir -p /raid/nfs
chmod 777 /raid/nfs
mcedit /etc/exports
```

В `/etc/exports` добавить:

```text
/raid/nfs 192.168.2.0/28(rw,sync,no_subtree_check,no_root_squash)
```

Применить:

```bash
exportfs -a
systemctl enable nfs-server --now
exportfs -v
```

### Проверка NFS после 2.4.2

На HQ-SRV:

```bash
systemctl status nfs-server
exportfs -v
ls -ld /raid/nfs
```

Должно быть:

```text
nfs-server active
экспортируется /raid/nfs
доступ для 192.168.2.0/28
```

---

### 2.4.3 Chrony клиент на HQ-SRV

На HQ-SRV:

```bash
apt-get install -y chrony
mcedit /etc/chrony.conf
```

Старые `pool/server` закомментировать, в конец добавить:

```text
server 172.16.1.1 iburst
```

Запуск:

```bash
systemctl restart chronyd || systemctl restart chrony
systemctl enable chronyd || systemctl enable chrony
chronyc sources
```

### Проверка chrony после 2.4.3

На HQ-SRV:

```bash
chronyc sources
chronyc tracking
```

Должно быть:

```text
источник времени 172.16.1.1
```

---

### 2.4.4 LAMP-приложение

Additional.iso должен быть подключен к HQ-SRV.

На HQ-SRV:

```bash
apt-get update
apt-get install -y lamp-server
mount /dev/sr0 /mnt
ls /mnt
cp /mnt/web/index.php /var/www/html/
cp -r /mnt/web/images /var/www/html/ 2>/dev/null || cp /mnt/web/logo.png /var/www/html/
mcedit /var/www/html/index.php
```

В `index.php` проверить:

```text
база: webdb
пользователь: web
пароль: P@ssw0rd
host: localhost
```

MariaDB:

```bash
systemctl enable --now mariadb
mariadb -u root
```

В MariaDB ввести:

```sql
CREATE DATABASE webdb;
CREATE USER 'web'@'localhost' IDENTIFIED BY 'P@ssw0rd';
GRANT ALL PRIVILEGES ON webdb.* TO 'web'@'localhost' WITH GRANT OPTION;
FLUSH PRIVILEGES;
EXIT;
```

Импорт базы:

```bash
mariadb -u web -p -D webdb < /mnt/web/dump.sql
systemctl enable --now httpd2
curl http://127.0.0.1
curl http://192.168.1.2
```

### Проверка LAMP после 2.4.4

На HQ-SRV:

```bash
systemctl status mariadb
systemctl status httpd2
ss -tulpn | grep :80
curl http://127.0.0.1
curl http://192.168.1.2
```

Должно быть:

```text
mariadb active
httpd2 active
порт 80 слушается
сайт открывается локально
сайт открывается по 192.168.1.2
```

Проверка базы:

```bash
mariadb -u web -p -D webdb
```

В MariaDB:

```sql
SHOW TABLES;
EXIT;
```

Проверка через DNAT:

```bash
curl http://172.16.1.2:8080
```

Проверка через nginx:

```bash
curl -u WEB:P@ssw0rd http://web.au-team.irpo
```

После пункта 2.4 должен проверяться `web.au-team.irpo`.

---

## 2.5 BR-SRV: Samba DC, пользователи, Ansible, Docker

### 2.5.1 Samba DC

На BR-SRV:

```bash
apt-get update
apt-get install -y task-samba-dc
rm -f /etc/samba/smb.conf
samba-tool domain provision
```

При вопросах указать:

```text
Realm = AU-TEAM.IRPO
Domain = AU-TEAM
Server Role = dc
DNS backend = SAMBA_INTERNAL
DNS forwarder = 192.168.1.2
```

Копируем Kerberos:

```bash
cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
systemctl disable bind --now
mcedit /etc/samba/smb.conf
```

В блок `[global]` добавить:

```text
interfaces = lo ens18
bind interfaces only = yes
```

Запуск:

```bash
systemctl enable samba --now
systemctl restart samba
samba-tool domain info 127.0.0.1
```

На HQ-SRV добавить пересылку запросов домена на BR-SRV:

```bash
mcedit /etc/dnsmasq.conf
```

Добавить:

```text
server=/au-team.irpo/192.168.3.2
```

Перезапуск:

```bash
systemctl restart dnsmasq
```

### Проверка Samba DC после 2.5.1

На BR-SRV:

```bash
systemctl status samba
samba-tool domain info 127.0.0.1
cat /etc/krb5.conf
```

На HQ-SRV:

```bash
nslookup -type=SRV _ldap._tcp.au-team.irpo 192.168.1.2
```

Должно быть:

```text
samba active
домен AU-TEAM.IRPO создан
SRV-записи домена находятся через DNS HQ-SRV
```

---

### 2.5.2 Пользователи домена и группа hq

На BR-SRV:

```bash
samba-tool group add hq
samba-tool user create hquser1 P@ssw0rd
samba-tool user create hquser2 P@ssw0rd
samba-tool user create hquser3 P@ssw0rd
samba-tool user create hquser4 P@ssw0rd
samba-tool user create hquser5 P@ssw0rd
samba-tool group addmembers hq hquser1,hquser2,hquser3,hquser4,hquser5
samba-tool group listmembers hq
```

### Проверка пользователей после 2.5.2

На BR-SRV:

```bash
samba-tool user list | grep hquser
samba-tool group listmembers hq
```

Должно быть:

```text
есть hquser1-hquser5
все пользователи состоят в группе hq
```

---

### 2.5.3 Ansible на BR-SRV

На BR-SRV:

```bash
apt-get update
apt-get install -y ansible sshpass
mkdir -p /etc/ansible
mcedit /etc/ansible/ansible.cfg
```

Вписать:

```text
[defaults]
host_key_checking = False
inventory = /etc/ansible/hosts
```

Открыть hosts:

```bash
mcedit /etc/ansible/hosts
```

Вписать:

```text
[routers]
hq-rtr ansible_host=10.0.0.1 ansible_user=net_admin ansible_password=P@ssw0rd
br-rtr ansible_host=10.0.0.2 ansible_user=net_admin ansible_password=P@ssw0rd

[servers]
hq-srv ansible_host=192.168.1.2 ansible_user=sshuser ansible_password=P@ssw0rd ansible_port=2026

[clients]
hq-cli ansible_host=192.168.2.2 ansible_user=user ansible_password=P@ssw0rd
```

Проверка:

```bash
ansible all -m ping
```

Если HQ-CLI не отвечает pong, на HQ-CLI поставить SSH:

```bash
apt-get install -y openssh
systemctl enable sshd --now
```

### Проверка Ansible после 2.5.3

На BR-SRV:

```bash
ansible --version
cat /etc/ansible/hosts
ansible all -m ping
```

Должно быть:

```text
ansible установлен
hosts заполнен
доступные машины отвечают pong
```

---

### 2.5.4 Docker testapp

Additional.iso должен быть подключен к BR-SRV. https://disk.yandex.ru/d/0MGlkrp2B9nXDw

На BR-SRV:

```bash
systemctl disable ahttpd --now
apt-get install -y docker-engine docker-compose
systemctl enable docker --now
mkdir -p /mnt/add_cd
mount /dev/sr0 /mnt/add_cd
cp -r /mnt/add_cd/docker /root/
docker image load -i /root/docker/site_latest.tar
docker image load -i /root/docker/mariadb_latest.tar
docker images
mkdir -p /root/testapp
cd /root/testapp
mcedit docker-compose.yaml
```
<img width="663" height="903" alt="image" src="https://github.com/user-attachments/assets/2faedb96-b2e8-4ad9-9c91-11f552b88c7b" />

В `docker-compose.yaml` вписать:
services:                  0 пробелов
  testapp:                 2 пробела
    image:                 4 пробела
    depends_on:            4 пробела
      - db                 6 пробелов
    environment:           4 пробела
      DB_TYPE: postgres    6 пробелов

  db:                      2 пробела
    image:                 4 пробела
    environment:           4 пробела
      POSTGRES_DB: testdb  6 пробелов
    volumes:               4 пробела
      - db_data:...        6 пробелов

volumes:                   0 пробелов
  db_data:                 2 пробела
```yaml
version: '3'
services:
  db:
    image: mariadb_latest
    container_name: db
    environment:
      MYSQL_ROOT_PASSWORD: P@ssw0rd
      MYSQL_DATABASE: testdb
      MYSQL_USER: test
      MYSQL_PASSWORD: P@ssw0rd

  testapp:
    image: site_latest
    container_name: testapp
    depends_on:
      - db
    ports:
      - "8080:8000"
    environment:
      DB_HOST: db
      DB_NAME: testdb
      DB_USER: test
      DB_PASSWORD: P@ssw0rd
```

Запуск:

```bash
docker compose up -d
docker ps
curl http://127.0.0.1:8080
curl http://192.168.3.2:8080
```

### Проверка Docker после 2.5.4

На BR-SRV:

```bash
systemctl status docker
docker images
docker ps
docker compose ps
curl http://127.0.0.1:8080
curl http://192.168.3.2:8080
```

Должно быть:

```text
docker active
есть образы site_latest и mariadb_latest
контейнеры db и testapp запущены
testapp открывается на 127.0.0.1:8080
testapp открывается на 192.168.3.2:8080
```

Проверка через DNAT:

```bash
curl http://172.16.2.2:8080
```

Проверка через nginx:

```bash
curl http://docker.au-team.irpo
```

После пункта 2.5 должен проверяться `docker.au-team.irpo`.

---

## 2.6 HQ-CLI: домен, NFS, sudo для hq, Яндекс Браузер

### 2.6.1 Ввод HQ-CLI в домен

Перед вводом в домен проверить DNS:

```bash
cat /etc/resolv.conf
nslookup -type=SRV _ldap._tcp.au-team.irpo 192.168.1.2
ping -c 3 192.168.3.2
```

Ввести машину в домен:

```bash
acc
```

Дальше:

```text
Центр управления
Пользователи
Аутентификация
Домен Active Directory
домен: au-team.irpo
галочка: Восстановить файлы конфигурации по умолчанию
пароль администратора домена: P@ssw0rd
```

После успешного ввода в домен перезагрузить HQ-CLI и войти пользователем:

```text
hquser1
P@ssw0rd
```

### Проверка домена после 2.6.1

На HQ-CLI:

```bash
id
id 'AU-TEAM\hquser1'
```

Должно быть:

```text
вход доменным пользователем работает
пользователь hquser1 определяется
```

---

### 2.6.2 Ограниченный sudo для группы hq

На HQ-CLI:

```bash
nano /etc/sudoers
```

Добавить:

```text
%hq ALL=(ALL) /bin/cat, /bin/grep, /usr/bin/id
```

Если группа отображается как `AU-TEAM\hq`, использовать:

```text
%AU-TEAM\\hq ALL=(ALL) /bin/cat, /bin/grep, /usr/bin/id
```

### Проверка sudo после 2.6.2

Под доменным пользователем:

```bash
id
sudo -l
sudo id
sudo cat /etc/hostname
sudo grep root /etc/passwd
```

Должно быть:

```text
sudo разрешает только cat, grep, id
```

---

### 2.6.3 NFS-автомонтирование /mnt/nfs

На HQ-CLI:

```bash
apt-get update
apt-get install -y nfs-clients
mkdir -p /mnt/nfs
systemctl --force --full edit mnt-nfs.mount
```

В файл `mnt-nfs.mount` вписать:

```text
[Unit]
Description=Mount NFS
After=network-online.target

[Mount]
What=192.168.1.2:/raid/nfs
Where=/mnt/nfs
Type=nfs
Options=_netdev,rw,nolock

[Install]
WantedBy=multi-user.target
```

Запуск:

```bash
systemctl enable mnt-nfs.mount --now
mount -a
df -h | grep nfs
touch /mnt/nfs/test.txt
```

### Проверка NFS после 2.6.3

На HQ-CLI:

```bash
systemctl status mnt-nfs.mount
df -h | grep nfs
mount | grep nfs
ls -l /mnt/nfs
```

На HQ-SRV:

```bash
ls -l /raid/nfs
```

Должно быть:

```text
/mnt/nfs смонтирован
test.txt появился на HQ-SRV в /raid/nfs
```

---

### 2.6.4 Яндекс Браузер

На HQ-CLI:

```bash
apt-get install -y yandex-browser-stable
```

### Проверка браузера после 2.6.4

На HQ-CLI:

```bash
yandex-browser-stable --version
```

В браузере открыть:

```text
http://web.au-team.irpo
http://docker.au-team.irpo
```

Для `web.au-team.irpo`:

```text
логин: WEB
пароль: P@ssw0rd
```

---

# 2.7 Быстрая финальная проверка модуля 2

## ISP

```bash
systemctl status chrony || systemctl status chronyd
systemctl status nginx
nginx -t
ss -tulpn | grep :80
```

Должно быть:

```text
chrony active
nginx active
nginx -t без ошибок
порт 80 слушается
```

---

## HQ-RTR

```bash
chronyc sources
iptables -t nat -L -n -v
iptables-save | grep 8080
iptables-save | grep 2026
```

Должно быть:

```text
есть NTP-сервер
есть DNAT 8080 и 2026
```

---

## BR-RTR

```bash
chronyc sources
iptables -t nat -L -n -v
iptables-save | grep 8080
iptables-save | grep 2026
```

Должно быть:

```text
есть NTP-сервер
есть DNAT 8080 и 2026
```

---

## HQ-SRV

```bash
df -h | grep raid
exportfs -v
systemctl status nfs-server
systemctl status httpd2
systemctl status mariadb
curl http://192.168.1.2
```

Должно быть:

```text
/raid смонтирован
NFS работает
LAMP работает
сайт открывается
```

---

## BR-SRV

```bash
systemctl status samba
samba-tool domain info 127.0.0.1
systemctl status docker
docker ps
curl http://192.168.3.2:8080
ansible all -m ping
```

Должно быть:

```text
Samba active
Docker active
контейнеры db и testapp работают
testapp открывается
Ansible проверяется
```

---

## HQ-CLI

```bash
df -h | grep nfs
nslookup web.au-team.irpo
nslookup docker.au-team.irpo
curl -u WEB:P@ssw0rd http://web.au-team.irpo
curl http://docker.au-team.irpo
```

Должно быть:

```text
NFS смонтирован
DNS работает
web.au-team.irpo открывается с авторизацией
docker.au-team.irpo открывается
```

---

# Где что должно проверяться

```text
После 1.2 проверяется ISP, NAT, ip_forward, интернет.
После 1.3 проверяется HQ-RTR, VLAN, GRE-интерфейс, NAT, net_admin.
После 1.4 проверяется BR-RTR, GRE между HQ и BR, NAT, net_admin.
После 1.5 проверяются HQ-SRV и BR-SRV: IP, sshuser, SSH 2026.
После 1.8 проверяется OSPF и связность между офисами.
После 1.9 проверяется DHCP и получение IP на HQ-CLI.
После 1.10 проверяется DNS dnsmasq и nslookup с HQ-CLI.
После 2.1 проверяется chrony и nginx на ISP.
После 2.2 проверяется DNAT на HQ-RTR.
После 2.3 проверяется DNAT на BR-RTR.
После 2.4 проверяется RAID, NFS, chrony и web.au-team.irpo.
После 2.5 проверяется Samba, Ansible, Docker и docker.au-team.irpo.
После 2.6 проверяется домен, sudo, NFS на HQ-CLI и браузер.
```

---

# Минимум на оценку 3

```text
1.2 ISP
1.3 HQ-RTR
1.4 BR-RTR
1.5 HQ-SRV и BR-SRV
1.8 OSPF
1.9 DHCP
1.10 DNS
```

После 1.10 уже должен быть рабочий фундамент сети.

---

# Лучше добрать для уверенной 3

```text
2.1 chrony + nginx
2.2 DNAT HQ-RTR
2.3 DNAT BR-RTR
2.4 LAMP на HQ-SRV
```

После 2.4 уже должен проверяться `web.au-team.irpo`.

---

# Для оценки 4

```text
Полностью модуль 1
2.1 chrony + nginx
2.2 DNAT HQ-RTR
2.3 DNAT BR-RTR
2.4 RAID/NFS/LAMP
2.5 Samba/Ansible/Docker
```

После 2.5 должен проверяться `docker.au-team.irpo`.

```

Вот это уже именно **методичка с проверками внутри**, а не “отдельный справочник по всем возможным бедам человечества”.
```
