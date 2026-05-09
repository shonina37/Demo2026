Поняла, делаю **всю методичку именно под README.md**, не “красивое описание вместо дела”. Ниже текст можно **целиком скопировать в README.md**. Я поправила опасные места: убрала кривые кавычки, убрала русскую `м` в `sources.list`, вынесла комментарий из GRE-команды, добавила нормальный `docker-compose.yaml`, оформила таблицы и команды так, чтобы GitHub не превращал всё в кашу. Да, наконец-то README будет похож на README, а не на проклятый свиток из Word 🫠

````md
# МОДУЛЬ 1 + МОДУЛЬ 2: быстрая методичка на 40 минут

Формат: как в методичке 2026, без `EOF`.  
Все файлы открывать через `nano` или `mcedit`.  
Сначала выполняешь команды, потом проверяешь результат.  
Если проверка не проходит, смотри таблицу ошибок в конце.

---

## 0. Адреса, которые должны получиться

| Устройство / интерфейс | IP | Шлюз |
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

## 1. Быстрый порядок выполнения

| Шаг | Где | Что сделать |
|---|---|---|
| 1 | Все ВМ | hostname, timezone, IP |
| 2 | ISP | ip_forward, NAT, проверка интернета |
| 3 | HQ-RTR / BR-RTR | net_admin, GRE, NAT, FRR/OSPF |
| 4 | HQ-SRV / BR-SRV | sshuser, SSH на порту 2026 |
| 5 | HQ-RTR / HQ-CLI | DHCP для HQ-CLI |
| 6 | HQ-SRV | dnsmasq со всеми A/PTR-записями |
| 7 | Модуль 2 | Samba, RAID, NFS, chrony, Ansible, Docker, LAMP, DNAT, nginx, auth |

---

# МОДУЛЬ 1. Настройка сетевой инфраструктуры

---

## 1.1 ISP: интерфейсы, маршрутизация, NAT

На машине **ISP** выполнить:

```bash
hostnamectl set-hostname isp.au-team.irpo
hostname
timedatectl set-timezone Asia/Yekaterinburg
hostname -f
timedatectl
````

Открыть файл интерфейсов:

```bash
nano /etc/network/interfaces
```

Оставить `lo`, ниже добавить или проверить:

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

Перезапустить сеть:

```bash
systemctl restart networking
```

Включить маршрутизацию:

```bash
nano /etc/sysctl.conf
```

Добавить строку:

```text
net.ipv4.ip_forward=1
```

Применить:

```bash
sysctl -p
```

Настроить NAT:

```bash
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables-save > /etc/rules.v4
```

Добавить автозагрузку правил:

```bash
crontab -e
```

В конец добавить:

```text
@reboot /sbin/iptables-restore < /etc/rules.v4
@reboot /sbin/sysctl -p
```

DNS для скачивания пакетов:

```bash
echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

Проверка:

```bash
ip -br a
cat /proc/sys/net/ipv4/ip_forward
iptables -t nat -L -n -v
ping -c 3 8.8.8.8
```

---

## 1.2 HQ-RTR: VLAN, GRE, NAT

На машине **HQ-RTR**:

```bash
hostnamectl set-hostname hq-rtr.au-team.irpo
hostname
timedatectl set-timezone Asia/Yekaterinburg
hostname -f
timedatectl
```

Открыть файл:

```bash
nano /etc/network/interfaces
```

Настроить интерфейсы:

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

Строка `pre-up ip tunnel del gre1` нужна, чтобы старый криво созданный туннель удалялся перед новым запуском.

DNS для скачивания пакетов:

```bash
echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

Перезапустить сеть:

```bash
systemctl restart networking
```

Включить маршрутизацию:

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

Настроить NAT:

```bash
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables-save > /etc/rules.v4
```

Проверка:

```bash
ip -br a
ip route
ping -c 3 172.16.1.1
```

Создать пользователя `net_admin`:

```bash
adduser net_admin
usermod -aG sudo net_admin
nano /etc/sudoers
```

Строку `%sudo` привести к виду:

```text
%sudo ALL=(ALL:ALL) NOPASSWD: ALL
```

---

## 1.3 BR-RTR: интерфейсы, GRE, NAT

На машине **BR-RTR**:

```bash
hostnamectl set-hostname br-rtr.au-team.irpo
hostname
timedatectl set-timezone Asia/Yekaterinburg
hostname -f
timedatectl
```

Открыть файл:

```bash
nano /etc/network/interfaces
```

Настроить интерфейсы:

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

DNS для скачивания пакетов:

```bash
echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

Перезапустить сеть:

```bash
systemctl restart networking
```

Включить маршрутизацию:

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

Настроить NAT:

```bash
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables-save > /etc/rules.v4
```

Проверка:

```bash
ip -br a
ip route
ping -c 3 172.16.2.1
ping -c 3 172.16.1.2
ping -c 3 10.0.0.1
```

Создать пользователя `net_admin`:

```bash
adduser net_admin
usermod -aG sudo net_admin
nano /etc/sudoers
```

Строку `%sudo` привести к виду:

```text
%sudo ALL=(ALL:ALL) NOPASSWD: ALL
```

---

## 1.4 Серверы ALT: IP-адреса и SSH

В Proxmox обязательно поставить VLAN Tag:

| Машина | VLAN Tag                            |
| ------ | ----------------------------------- |
| HQ-SRV | 100                                 |
| HQ-CLI | 200                                 |
| BR-SRV | без VLAN, если он в обычной сети BR |

---

### HQ-SRV

На машине **HQ-SRV**:

```bash
hostnamectl set-hostname hq-srv.au-team.irpo
hostname
timedatectl set-timezone Asia/Yekaterinburg
hostname -f
timedatectl
```

Настроить интерфейс:

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

IP-адрес:

```bash
mcedit /etc/net/ifaces/ens18/ipv4address
```

Содержимое:

```text
192.168.1.2/27
```

Шлюз:

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

Перезапустить сеть:

```bash
systemctl restart network
ip -br a
ip route
```

Создать пользователя `sshuser`:

```bash
useradd -u 2026 -m sshuser
passwd sshuser
```

Пароль:

```text
P@ssw0rd
```

Добавить в группу `wheel`:

```bash
usermod -aG wheel sshuser
mcedit /etc/sudoers
```

Добавить или раскомментировать:

```text
%wheel ALL=(ALL) NOPASSWD: ALL
```

Проверка sudo:

```bash
su - sshuser
sudo id
exit
```

Настроить SSH:

```bash
mcedit /etc/openssh/sshd_config
```

Проверить или добавить строки:

```text
Port 2026
AllowUsers sshuser
MaxAuthTries 2
Banner /root/banner
PermitRootLogin no
```

Создать баннер:

```bash
mcedit /root/banner
```

Содержимое:

```text
Authorized access only
```

Проверить конфиг SSH:

```bash
sshd -t
```

Если ошибок нет:

```bash
systemctl restart sshd
systemctl enable sshd
ss -tulpn | grep 2026
```

---

### BR-SRV

На машине **BR-SRV**:

```bash
hostnamectl set-hostname br-srv.au-team.irpo
hostname
timedatectl set-timezone Asia/Yekaterinburg
hostname -f
timedatectl
```

Настроить интерфейс:

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

IP-адрес:

```bash
mcedit /etc/net/ifaces/ens18/ipv4address
```

Содержимое:

```text
192.168.3.2/28
```

Шлюз:

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

Перезапустить сеть:

```bash
systemctl restart network
ip -br a
ip route
```

Создать пользователя `sshuser`:

```bash
useradd -u 2026 -m sshuser
passwd sshuser
```

Пароль:

```text
P@ssw0rd
```

Добавить в группу `wheel`:

```bash
usermod -aG wheel sshuser
mcedit /etc/sudoers
```

Добавить или раскомментировать:

```text
%wheel ALL=(ALL) NOPASSWD: ALL
```

Настроить SSH:

```bash
mcedit /etc/openssh/sshd_config
```

Проверить или добавить строки:

```text
Port 2026
AllowUsers sshuser
MaxAuthTries 2
Banner /root/banner
PermitRootLogin no
```

Создать баннер:

```bash
mcedit /root/banner
```

Содержимое:

```text
Authorized access only
```

Проверка:

```bash
sshd -t
systemctl restart sshd
systemctl enable sshd
ss -tulpn | grep 2026
```

---

## 1.5 FRR / OSPF на HQ-RTR и BR-RTR

На **HQ-RTR** и **BR-RTR**:

```bash
nano /etc/apt/sources.list
```

Добавить репозиторий, если он нужен по методичке:

```text
deb [trusted=yes] https://archive.debian.org/debian buster main
```

Установить FRR:

```bash
apt-get update
apt-get install -y frr
```

Открыть файл демонов:

```bash
nano /etc/frr/daemons
```

Изменить:

```text
ospfd=yes
```

Запустить FRR:

```bash
systemctl enable frr --now
systemctl restart frr
```

---

### HQ-RTR

Зайти в FRR:

```bash
vtysh
```

Ввести:

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

---

### BR-RTR

Зайти в FRR:

```bash
vtysh
```

Ввести:

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

Проверка:

```bash
ping -c 3 10.0.0.1
ping -c 3 10.0.0.2
```

---

## 1.6 DHCP на HQ-RTR для HQ-CLI

На **HQ-RTR**:

```bash
apt-get update
apt-get install -y isc-dhcp-server
```

Открыть файл:

```bash
nano /etc/default/isc-dhcp-server
```

Указать интерфейс VLAN 200:

```text
INTERFACESv4="eth1.200"
```

Сделать копию конфига:

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

Проверить конфиг:

```bash
dhcpd -t -cf /etc/dhcp/dhcpd.conf
```

Запустить DHCP:

```bash
systemctl restart isc-dhcp-server
systemctl enable isc-dhcp-server
systemctl status isc-dhcp-server
```

---

### HQ-CLI

На **HQ-CLI**:

```bash
hostnamectl set-hostname hq-cli.au-team.irpo
hostname
timedatectl set-timezone Asia/Yekaterinburg
hostname -f
timedatectl
```

Проверить получение адреса:

```bash
ip -br a
ip route
ping -c 3 192.168.2.1
ping -c 3 192.168.1.2
```

Если DNS не прописался автоматически:

```bash
nano /etc/resolv.conf
```

Содержимое:

```text
search au-team.irpo
nameserver 192.168.1.2
```

---

## 1.7 DNS dnsmasq на HQ-SRV

На **HQ-SRV**:

```bash
apt-get update
apt-get install -y dnsmasq
systemctl disable bind --now
```

Открыть файл:

```bash
mcedit /etc/dnsmasq.conf
```

В конец добавить:

```text
no-resolv
no-hosts
listen-address=127.0.0.1,192.168.1.2
bind-interfaces
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

Проверить конфиг:

```bash
dnsmasq --test
```

Запустить:

```bash
systemctl restart dnsmasq
systemctl enable dnsmasq
systemctl status dnsmasq
ss -tulpn | grep :53
```

Проверка с **HQ-CLI**:

```bash
nslookup hq-srv.au-team.irpo 192.168.1.2
nslookup web.au-team.irpo 192.168.1.2
ping -c 3 hq-srv.au-team.irpo
```

---

# МОДУЛЬ 2. Сетевое администрирование

---

## 2.0 Что где делается

| Машина | Что настраивать                                                               |
| ------ | ----------------------------------------------------------------------------- |
| ISP    | chrony сервер времени, nginx reverse proxy, web-based аутентификация          |
| HQ-RTR | chrony клиент, проброс WEB/SSH портов на HQ-SRV                               |
| BR-RTR | chrony клиент, проброс WEB/SSH портов на BR-SRV                               |
| HQ-SRV | RAID0, автомонтирование `/raid`, NFS-сервер, chrony клиент, LAMP              |
| BR-SRV | Samba DC, пользователи домена, Ansible, Docker Compose testapp                |
| HQ-CLI | ввод в домен, вход доменным пользователем, sudo-права hq, NFS, Яндекс Браузер |

---

## 2.1 ISP: chrony, nginx, web-auth

### Chrony на ISP

На **ISP**:

```bash
apt-get update
apt-get install -y chrony
nano /etc/chrony/chrony.conf
```

Старые `pool` и `server` закомментировать.
В конец добавить:

```text
server 0.ru.pool.ntp.org iburst
local stratum 5
allow 172.16.0.0/16
allow 192.168.0.0/16
```

Перезапустить:

```bash
systemctl restart chrony || systemctl restart chronyd
systemctl enable chrony || systemctl enable chronyd
chronyc sources
```

---

### Nginx reverse proxy + web-based auth

На **ISP**:

```bash
apt-get install -y nginx apache2-utils
htpasswd -cb /etc/nginx/.htpasswd WEB P@ssw0rd
nano /etc/nginx/sites-available/default
```

В файл вписать:

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
    }
}

server {
    listen 80;
    server_name docker.au-team.irpo;

    location / {
        proxy_pass http://172.16.2.2:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Проверить и запустить:

```bash
nginx -t
systemctl enable nginx --now
systemctl restart nginx
systemctl status nginx
```

Проверка с **HQ-CLI**:

```bash
curl -I http://web.au-team.irpo
curl -I http://docker.au-team.irpo
```

`web.au-team.irpo` должен запросить логин и пароль:

```text
WEB
P@ssw0rd
```

---

## 2.2 HQ-RTR: chrony клиент и проброс портов

### Chrony клиент

На **HQ-RTR**:

```bash
apt-get update
apt-get install -y chrony
nano /etc/chrony/chrony.conf
```

Старые `pool/server` закомментировать.
В конец добавить:

```text
server 172.16.1.1 iburst
```

Перезапустить:

```bash
systemctl restart chrony || systemctl restart chronyd
systemctl enable chrony || systemctl enable chronyd
chronyc sources
```

---

### Проброс портов на HQ-SRV

На **HQ-RTR**:

```bash
iptables -t nat -A PREROUTING -d 172.16.1.2 -p tcp --dport 8080 -j DNAT --to-destination 192.168.1.2:80
iptables -t nat -A PREROUTING -d 172.16.1.2 -p tcp --dport 2026 -j DNAT --to-destination 192.168.1.2:2026
iptables-save > /etc/rules.v4
iptables -t nat -L -n -v
```

---

## 2.3 BR-RTR: chrony клиент и проброс портов

### Chrony клиент

На **BR-RTR**:

```bash
apt-get update
apt-get install -y chrony
nano /etc/chrony/chrony.conf
```

Старые `pool/server` закомментировать.
В конец добавить:

```text
server 172.16.1.1 iburst
```

Перезапустить:

```bash
systemctl restart chrony || systemctl restart chronyd
systemctl enable chrony || systemctl enable chronyd
chronyc sources
```

---

### Проброс портов на BR-SRV

На **BR-RTR**:

```bash
iptables -t nat -A PREROUTING -d 172.16.2.2 -p tcp --dport 8080 -j DNAT --to-destination 192.168.3.2:8080
iptables -t nat -A PREROUTING -d 172.16.2.2 -p tcp --dport 2026 -j DNAT --to-destination 192.168.3.2:2026
iptables-save > /etc/rules.v4
iptables -t nat -L -n -v
```

---

## 2.4 HQ-SRV: RAID0, NFS, chrony, LAMP

### RAID0

В Proxmox заранее добавить к **HQ-SRV** два диска по `1 ГБ`.

На **HQ-SRV**:

```bash
lsblk
apt-get update
apt-get install -y mdadm
```

Создать RAID0:

```bash
mdadm --create /dev/md0 --level=0 --raid-devices=2 /dev/sdb /dev/sdc
mdadm --detail --scan > /etc/mdadm.conf
```

Разметить:

```bash
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

Форматировать и смонтировать:

```bash
mkfs.ext4 /dev/md0p1
mkdir -p /raid
mount /dev/md0p1 /raid
df -h
```

---

### Автомонтирование RAID

Создать unit:

```bash
systemctl --force --full edit raid.mount
```

Вписать:

```ini
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

Сохранить и выйти.
Если открылся `vim`: `Esc`, потом `:wq`, потом `Enter`.

Запустить:

```bash
systemctl enable raid.mount --now
mount -a
df -h | grep raid
```

---

### NFS-сервер

На **HQ-SRV**:

```bash
apt-get install -y nfs-server
mkdir -p /raid/nfs
chmod 777 /raid/nfs
mcedit /etc/exports
```

Добавить строку:

```text
/raid/nfs 192.168.2.0/28(rw,sync,no_subtree_check,no_root_squash)
```

Важно: между сетью и скобкой пробела нет.

Применить:

```bash
exportfs -a
systemctl enable nfs-server --now
exportfs -v
```

---

### Chrony клиент на HQ-SRV

На **HQ-SRV**:

```bash
apt-get install -y chrony
mcedit /etc/chrony.conf
```

Старые `pool/server` закомментировать.
В конец добавить:

```text
server 172.16.1.1 iburst
```

Перезапустить:

```bash
systemctl restart chronyd || systemctl restart chrony
systemctl enable chronyd || systemctl enable chrony
chronyc sources
```

---

### LAMP-приложение

Additional.iso должен быть подключен к **HQ-SRV**.

```bash
apt-get update
apt-get install -y lamp-server
mount /dev/sr0 /mnt
ls /mnt
```

Скопировать файлы:

```bash
cp /mnt/web/index.php /var/www/html/
cp -r /mnt/web/images /var/www/html/ 2>/dev/null || cp /mnt/web/logo.png /var/www/html/
```

Проверить настройки БД в `index.php`:

```bash
mcedit /var/www/html/index.php
```

Должно быть:

```text
database: webdb
user: web
password: P@ssw0rd
host: localhost
```

Запустить MariaDB:

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

Импортировать дамп:

```bash
mariadb -u web -p -D webdb < /mnt/web/dump.sql
```

Запустить Apache:

```bash
systemctl enable --now httpd2
curl http://127.0.0.1
curl http://192.168.1.2
```

---

## 2.5 BR-SRV: Samba DC, пользователи, Ansible, Docker

### Samba DC

На **BR-SRV**:

```bash
apt-get update
apt-get install -y task-samba-dc
rm -f /etc/samba/smb.conf
samba-tool domain provision
```

При вопросах указать:

| Параметр      | Значение       |
| ------------- | -------------- |
| Realm         | AU-TEAM.IRPO   |
| Domain        | AU-TEAM        |
| Server Role   | dc             |
| DNS backend   | SAMBA_INTERNAL |
| DNS forwarder | 192.168.1.2    |

Пароль администратора домена лучше задать:

```text
P@ssw0rd
```

Скопировать Kerberos-конфиг:

```bash
cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
systemctl disable bind --now
```

Открыть Samba-конфиг:

```bash
mcedit /etc/samba/smb.conf
```

В блок `[global]` добавить:

```text
interfaces = lo ens18
bind interfaces only = yes
```

Запустить Samba:

```bash
systemctl enable samba --now
systemctl restart samba
samba-tool domain info 127.0.0.1
```

---

### Пользователи домена и группа hq

На **BR-SRV**:

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

Если **HQ-CLI** не видит домен, на **HQ-SRV** добавить пересылку запросов домена на **BR-SRV**:

```bash
mcedit /etc/dnsmasq.conf
```

В конец добавить:

```text
server=/au-team.irpo/192.168.3.2
```

Перезапустить dnsmasq:

```bash
systemctl restart dnsmasq
```

---

### Ansible на BR-SRV

На **BR-SRV**:

```bash
apt-get update
apt-get install -y ansible sshpass
mkdir -p /etc/ansible
```

Создать конфиг:

```bash
mcedit /etc/ansible/ansible.cfg
```

Содержимое:

```ini
[defaults]
host_key_checking = False
inventory = /etc/ansible/hosts
```

Создать inventory:

```bash
mcedit /etc/ansible/hosts
```

Содержимое:

```ini
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

Если **HQ-CLI** не отвечает `pong`, на **HQ-CLI** поставить SSH:

```bash
apt-get install -y openssh
systemctl enable sshd --now
```

---

### Docker testapp

Additional.iso должен быть подключен к **BR-SRV**.

```bash
systemctl disable ahttpd --now
apt-get install -y docker-engine docker-compose
systemctl enable docker --now
```

Смонтировать Additional.iso:

```bash
mkdir -p /mnt/add_cd
mount /dev/sr0 /mnt/add_cd
cp -r /mnt/add_cd/docker /root/
```

Загрузить образы:

```bash
docker image load -i /root/docker/site_latest.tar
docker image load -i /root/docker/mariadb_latest.tar
docker images
```

Создать каталог:

```bash
mkdir -p /root/testapp
cd /root/testapp
mcedit docker-compose.yaml
```

В `docker-compose.yaml` вписать:

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

Отступы только пробелами, не Tab.

Запустить:

```bash
docker compose up -d
docker ps
```

Проверить:

```bash
curl http://127.0.0.1:8080
curl http://192.168.3.2:8080
```

---

## 2.6 HQ-CLI: домен, NFS, sudo для hq, Яндекс Браузер

### Ввод машины в домен

На **HQ-CLI**:

```bash
acc
```

Дальше в графическом интерфейсе:

```text
Центр управления
Пользователи
Аутентификация
Домен Active Directory
```

Указать домен:

```text
au-team.irpo
```

Поставить галочку:

```text
Восстановить файлы конфигурации по умолчанию
```

Пароль администратора домена — тот, который задавался при `samba-tool domain provision`.

После успешного ввода в домен перезагрузить **HQ-CLI** и войти:

```text
hquser1
P@ssw0rd
```

---

### Ограниченный sudo для группы hq

На **HQ-CLI**:

```bash
nano /etc/sudoers
```

Добавить простой вариант:

```text
%hq ALL=(ALL) /bin/cat, /bin/grep, /usr/bin/id
```

Если группа отображается как `AU-TEAM\hq`, использовать:

```text
%AU-TEAM\hq ALL=(ALL) /bin/cat, /bin/grep, /usr/bin/id
```

Проверка:

```bash
sudo -l
sudo id
```

---

### NFS-автомонтирование

На **HQ-CLI**:

```bash
apt-get update
apt-get install -y nfs-clients
mkdir -p /mnt/nfs
systemctl --force --full edit mnt-nfs.mount
```

Вписать:

```ini
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

Запустить:

```bash
systemctl enable mnt-nfs.mount --now
mount -a
df -h | grep nfs
touch /mnt/nfs/test.txt
```

---

### Яндекс Браузер

На **HQ-CLI**:

```bash
apt-get install -y yandex-browser-stable
```

---

## 2.7 Быстрая финальная проверка модуля 2

| Где    | Команды проверки                                                     | Что должно быть                                            |                                                  |
| ------ | -------------------------------------------------------------------- | ---------------------------------------------------------- | ------------------------------------------------ |
| ISP    | `systemctl status chrony nginx`; `nginx -t`                          | chrony/nginx active, nginx без ошибок                      |                                                  |
| HQ-RTR | `chronyc sources`; `iptables -t nat -L -n -v`                        | виден NTP-сервер, есть DNAT 8080 и 2026                    |                                                  |
| BR-RTR | `chronyc sources`; `iptables -t nat -L -n -v`                        | виден NTP-сервер, есть DNAT 8080 и 2026                    |                                                  |
| HQ-SRV | `df -h`; `exportfs -v`; `systemctl status nfs-server httpd2 mariadb` | `/raid` смонтирован, NFS/LAMP работают                     |                                                  |
| BR-SRV | `systemctl status samba docker`; `docker ps`; `ansible all -m ping`  | Samba/Docker active, контейнеры db и testapp, ansible pong |                                                  |
| HQ-CLI | `df -h                                                               | grep nfs`; `nslookup web.au-team.irpo`; браузер web/docker | NFS смонтирован, DNS работает, сайты открываются |

---

# 3. Если сломалось

| Ошибка                                                    | Почему                                                  | Что сделать быстро                                                             |
| --------------------------------------------------------- | ------------------------------------------------------- | ------------------------------------------------------------------------------ |
| `sudo: unable to resolve host`                            | Не совпадает hostname и `/etc/hosts`                    | Проверить `hostname -f`; добавить имя в `/etc/hosts`                           |
| `networking.service failed`                               | Ошибка в `/etc/network/interfaces`                      | Проверить опечатки: `gateway`, `iface eth1.200`, `vlan-raw-device`             |
| GRE не пингуется                                          | Нет связи между внешними IP или local/remote перепутаны | Проверить `ping 172.16.1.2`, `ping 172.16.2.2`, потом `ping 10.0.0.1/10.0.0.2` |
| `add tunnel failed: File exists`                          | GRE уже был создан                                      | `ip tunnel del gre1 2>/dev/null`; потом restart networking                     |
| OSPF нет соседа                                           | GRE не работает или разные MD5-пароли                   | Проверить `ping` по GRE и пароль `123qweR%`                                    |
| DHCP не выдаёт IP                                         | Не указан интерфейс DHCP                                | В `/etc/default/isc-dhcp-server` указать `INTERFACESv4="eth1.200"`             |
| `dnsmasq bad option`                                      | Ошибка в строке конфига                                 | Выполнить `dnsmasq --test`; проверить `ptr-record`, `address`, `domain`        |
| DNS отвечает через nslookup, но ping по имени не работает | Клиент не использует нужный DNS                         | В `/etc/resolv.conf` указать `nameserver 192.168.1.2`                          |
| `sshd failed`                                             | Опечатка в sshd_config                                  | Выполнить `sshd -t`; проверить `AllowUsers`, `Banner`, `Port`                  |
| `exportfs: No host name given`                            | Пробел между сетью и скобкой                            | Писать `192.168.2.0/28(rw...)` без пробела                                     |
| NFS требует `rpc.statd`                                   | Клиент требует locking                                  | В mount-unit поставить `Options=_netdev,rw,nolock`                             |
| Samba domain info не работает                             | Samba не запущена или конфликтует bind                  | `systemctl disable bind --now`; `systemctl restart samba`                      |
| HQ-CLI не входит в домен                                  | DNS не ведёт домен на BR-SRV                            | На HQ-SRV добавить `server=/au-team.irpo/192.168.3.2`                          |
| Ansible не отвечает pong                                  | SSH недоступен или неверный порт                        | Проверить SSH вручную; для HQ-SRV нужен `ansible_port=2026`                    |
| Docker Compose ругается                                   | Ошибка отступов или не загружены images                 | `docker images`; `docker compose config`; отступы только пробелами             |
| `curl 192.168.3.2:8080` висит                             | Контейнер не поднялся или порт не проброшен             | `docker ps`; должен быть порт `0.0.0.0:8080->8000`                             |
| LAMP показывает ошибку БД                                 | Неверные данные в `index.php`                           | Проверить `web`, `P@ssw0rd`, `webdb`, `localhost`                              |
| `nginx -t` ошибка                                         | Пропущена скобка или `;`                                | `nginx -t` покажет строку ошибки                                               |
| `web.au-team.irpo` не спрашивает пароль                   | `auth_basic` не в том server-блоке                      | Проверить блок `server_name web.au-team.irpo`                                  |

---

## 4. Важно для GitHub

Не включать автоматический перевод страницы GitHub.
Если браузер переводит команды, они становятся нерабочими.

Правильно:

```bash
hostnamectl set-hostname hq-rtr.au-team.irpo
```

Неправильно:

```text
имя хоста set-hostname hq-rtr.au-team.irpo
```

Команды нужно копировать только из оригинального README без автоматического перевода.

```


