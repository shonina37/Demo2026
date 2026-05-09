 МОДУЛЬ 1 + МОДУЛЬ 2: БЫСТРАЯ МЕТОДИЧКА НА 40 МИНУТ
Формат: как в методичке 2026, без EOF. Все файлы открывать через nano или mcedit. Сначала делаешь команды, потом проверку. Если проверка не проходит, смотри таблицу ошибок в конце+

0. Адреса, которые должны получиться
Устройство/интерфейс	IP	Шлюз
ISP eth0	DHCP	шлюз DHCP
ISP eth1	172.16.1.1/28	-
ISP eth2	172.16.2.1/28	-
HQ-RTR eth0	172.16.1.2/28	172.16.1.1
HQ-RTR eth1.100	192.168.1.1/27	-
HQ-RTR eth1.200	192.168.2.1/28	-
HQ-RTR eth1.999	192.168.99.1/29	-
HQ-RTR gre1	10.0.0.1/30	-
HQ-SRV ens18 VLAN100	192.168.1.2/27	192.168.1.1
HQ-CLI ens18 VLAN200	DHCP 192.168.2.x/28	192.168.2.1
BR-RTR eth0	172.16.2.2/28	172.16.2.1
BR-RTR eth1	192.168.3.1/28	-
BR-RTR gre1	10.0.0.2/30	-
BR-SRV ens18	192.168.3.2/28	192.168.3.1
1. 40-минутный порядок выполнения
Шаг	Где	Что сделать
1	Все ВМ	hostname + timezone + IP. Без этого дальше будет цирк.
2	ISP	ip_forward + NAT. Проверить интернет с роутеров.
3	HQ-RTR/BR-RTR	net_admin, GRE, NAT, FRR/OSPF. Проверить соседство OSPF.
4	HQ-SRV/BR-SRV	sshuser, SSH 2026 на ОБОИХ серверах. Это было пропущено на листочке.
5	HQ-RTR/HQ-CLI	DHCP для HQ-CLI. Проверить IP на клиенте.
6	HQ-SRV	dnsmasq со всеми A/PTR записями.
7	Модуль 2	Samba, RAID, NFS, chrony, ansible, docker, LAMP, DNAT, nginx, auth. Делать только по порядку.
МОДУЛЬ 1. Настройка сетевой инфраструктуры
1.2 ISP: интерфейсы, маршрутизация, NAT
hostnamectl set-hostname isp.au-team.irpo 
hostname (проверяем применилось ли)
timedatectl set-timezone Asia/Yekaterinburg
hostname -f
timedatectl


nano /etc/network/interfaces

# оставить lo, ниже добавить/проверить:
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

systemctl restart networking
nano /etc/sysctl.conf
# добавить:
net.ipv4.ip_forward=1

sysctl -p
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables-save > /etc/rules.v4
crontab -e
# в конец добавить:
@reboot /sbin/iptables-restore < /etc/rules.v4
@reboot /sbin/sysctl -p

echo “nameserver 8.8.8.8” > /etc/resolv.conf
ip -br a
iptables -t nat -L -n -v

1.3 HQ-RTR: VLAN, GRE, NAT
hostnamectl set-hostname hq-rtr.au-team.irpo 
hostname (проверяем применилось ли)
timedatectl set-timezone Asia/Yekaterinburg
hostname -f
timedatectl


nano /etc/network/interfaces

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
    pre-up ip tunnel del gre1 2>/dev/null || true (эта команда нужна чтоб если вы допустили ошибку и туннель не работает все автоматически почистилось и не ломалось дальшен)
    pre-up ip tunnel add gre1 mode gre local 172.16.1.2 remote 172.16.2.2 ttl 255
    post-down ip tunnel del gre1 2>/dev/null || true

echo “nameserver 8.8.8.8” > /etc/resolv.conf

systemctl restart networking
nano /etc/sysctl.conf
net.ipv4.ip_forward=1
sysctl -p
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables-save > /etc/rules.v4
ip -br a

Добавляем пользователя

adduser net_admin
usermod -aG sudo net_admin
nano /etc/sudoers
# строку %sudo привести к виду:
%sudo ALL=(ALL:ALL) NOPASSWD: ALL

1.4 BR-RTR: интерфейсы, GRE, NAT
hostnamectl set-hostname br-rtr.au-team.irpo 
hostname (проверяем применилось ли)
timedatectl set-timezone Asia/Yekaterinburg
hostname -f
timedatectl


nano /etc/network/interfaces

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
echo “nameserver 8.8.8.8” > /etc/resolv.conf

systemctl restart networking
nano /etc/sysctl.conf
net.ipv4.ip_forward=1
sysctl -p
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables-save > /etc/rules.v4
ping 10.0.0.1

Добавляем пользователя
adduser net_admin
usermod -aG sudo net_admin
nano /etc/sudoers
# строку %sudo привести к виду:
%sudo ALL=(ALL:ALL) NOPASSWD: ALL
1.5 Серверы ALT: IP адреса
В Proxmox обязательно поставь VLAN Tag: HQ-SRV = 100, HQ-CLI = 200. BR-SRV без VLAN, если по схеме он сидит в обычной сети BR.
# HQ-SRV
hostnamectl set-hostname hq-srv.au-team.irpo 
hostname (проверяем применилось ли)
timedatectl set-timezone Asia/Yekaterinburg
hostname -f
timedatectl


mcedit /etc/net/ifaces/ens18/options
# содержимое:
TYPE=eth
BOOTPROTO=static
CONFIG_IPV4=yes
DISABLED=no

mcedit /etc/net/ifaces/ens18/ipv4address
192.168.1.2/27

mcedit /etc/net/ifaces/ens18/ipv4route
default via 192.168.1.1

echo “nameserver 8.8.8.8” > /etc/resolv.conf


systemctl restart network
ip -br a
ip r
mcedit /etc/openssh/sshd_config
# проверить/добавить строки:
Port 2026
AllowUsers sshuser
MaxAuthTries 2
Banner /root/banner
PermitRootLogin no

mcedit /root/banner
Authorized access only

systemctl restart sshd
systemctl enable sshd
ss -tulpn | grep 2026
ssh -p 2026 sshuser@192.168.1.2

Добавляем пользователя
useradd -u 2026 -m sshuser
passwd sshuser
# пароль: P@ssw0rd
usermod -aG wheel sshuser
mcedit /etc/sudoers
# раскомментировать/добавить:
%wheel ALL=(ALL) NOPASSWD: ALL

su - sshuser
sudo id
exit


# BR-SRV так же, но:
hostnamectl set-hostname br-srv.au-team.irpo 
hostname (проверяем применилось ли)
timedatectl set-timezone Asia/Yekaterinburg
hostname -f
timedatectl

echo “nameserver 8.8.8.8” > /etc/resolv.conf

# ipv4address: 192.168.3.2/28
# ipv4route:   default via 192.168.3.1
mcedit /etc/openssh/sshd_config
# проверить/добавить строки:
Port 2026
AllowUsers sshuser
MaxAuthTries 2
Banner /root/banner
PermitRootLogin no

mcedit /root/banner
Authorized access only

systemctl restart sshd
systemctl enable sshd
ss -tulpn | grep 2026
ssh -p 2026 sshuser@192.168.3.2
 


1.8 FRR/OSPF на HQ-RTR и BR-RTR
nano /etc/apt/sources.list
deb [trustd=yes] https://archive.debian.org/debian buster main 
 
apt-get update
apt-get install -y frr
nano /etc/frr/daemons
# изменить:
ospfd=yes

systemctl enable frr --now
systemctl restart frr
vtysh

На HQ-RTR внутри vtysh:
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

На BR-RTR внутри vtysh:
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

1.9 DHCP на HQ-RTR для HQ-CLI
apt-get update && apt-get install -y isc-dhcp-server
nano /etc/default/isc-dhcp-server
# строка:
INTERFACESv4="eth1.200"

cp /etc/dhcp/dhcpd.conf /etc/dhcp/dhcpd.bkp
nano /etc/dhcp/dhcpd.conf
# добавить в конец:
subnet 192.168.2.0 netmask 255.255.255.240 {
  range 192.168.2.2 192.168.2.14;
  option routers 192.168.2.1;
  option domain-name-servers 192.168.1.2;
  option domain-name "au-team.irpo";
}

systemctl restart isc-dhcp-server
systemctl enable isc-dhcp-server
systemctl status isc-dhcp-server

На HQ-CLI поставь получение IP по DHCP и проверь:
hostnamectl set-hostname hq-cli.au-team.irpo 
hostname (проверяем применилось ли)
timedatectl set-timezone Asia/Yekaterinburg
hostname -f
timedatectl
 nano /etc/resolv.conf пишем search au-team.irpo
                                                nameserver 192.168.1.2
                                                nameserver 8.8.8.8

ip -br a
ip r
ping 192.168.2.1
ping 192.168.1.2

1.10 DNS dnsmasq на HQ-SRV со всеми записями
apt-get update && apt-get install -y dnsmasq
systemctl disable bind --now
mcedit /etc/dnsmasq.conf
# в конец добавить:
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

systemctl restart dnsmasq
systemctl enable dnsmasq
systemctl status dnsmasq

Проверка с HQ-CLI:
nslookup hq-srv.au-team.irpo 192.168.1.2
nslookup web.au-team.irpo 192.168.1.2
ping hq-srv.au-team.irpo

МОДУЛЬ 2. Сетевое администрирование
2.0 Что где делается


Машина	Что настраивать
ISP	chrony сервер времени, nginx reverse proxy, web-based аутентификация
HQ-RTR	chrony клиент, проброс WEB/SSH портов на HQ-SRV
BR-RTR	chrony клиент, проброс WEB/SSH портов на BR-SRV
HQ-SRV	RAID0, автомонтирование /raid, NFS-сервер, chrony клиент, LAMP-приложение
BR-SRV	Samba DC, пользователи домена, Ansible, Docker Compose testapp
HQ-CLI	ввод в домен, вход доменным пользователем, sudo-права hq, NFS-автомонтирование, Яндекс Браузер

2.1 ISP: chrony, nginx, web-auth
1) Chrony на ISP как сервер времени.
apt-get update
apt-get install -y chrony
nano /etc/chrony/chrony.conf

# старые pool/server закомментировать, в конец добавить:
server 0.ru.pool.ntp.org iburst
local stratum 5
allow 172.16.0.0/16
allow 192.168.0.0/16

systemctl restart chrony || systemctl restart chronyd
systemctl enable chrony || systemctl enable chronyd
chronyc sources

2) Nginx reverse proxy + пароль на web.au-team.irpo.
apt-get install -y nginx apache2-utils
htpasswd -cb /etc/nginx/.htpasswd WEB P@ssw0rd
nano /etc/nginx/sites-available/default

В файл /etc/nginx/sites-available/default вписать два server-блока:
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

nginx -t
systemctl enable nginx --now
systemctl restart nginx
systemctl status nginx

Проверка с HQ-CLI: web.au-team.irpo должен спросить WEB / P@ssw0rd, docker.au-team.irpo должен открыть testapp.
2.2 HQ-RTR: chrony клиент + проброс портов на HQ-SRV
1) Chrony клиент. На Astra файл обычно /etc/chrony/chrony.conf.
apt-get update
apt-get install -y chrony
nano /etc/chrony/chrony.conf

# старые pool/server закомментировать, в конец добавить:
server 172.16.1.1 iburst

systemctl restart chrony || systemctl restart chronyd
systemctl enable chrony || systemctl enable chronyd
chronyc sources

2) Проброс портов на HQ-SRV: внешний 8080 → Apache 80, внешний 2026 → SSH 2026.
iptables -t nat -A PREROUTING -d 172.16.1.2 -p tcp --dport 8080 -j DNAT --to-destination 192.168.1.2:80
iptables -t nat -A PREROUTING -d 172.16.1.2 -p tcp --dport 2026 -j DNAT --to-destination 192.168.1.2:2026
iptables-save > /etc/rules.v4
iptables -t nat -L -n -v

2.3 BR-RTR: chrony клиент + проброс портов на BR-SRV
apt-get update
apt-get install -y chrony
nano /etc/chrony/chrony.conf

# старые pool/server закомментировать, в конец добавить:
server 172.16.1.1 iburst

systemctl restart chrony || systemctl restart chronyd
systemctl enable chrony || systemctl enable chronyd
chronyc sources

Проброс портов на BR-SRV: внешний 8080 → Docker testapp, внешний 2026 → SSH 2026.
iptables -t nat -A PREROUTING -d 172.16.2.2 -p tcp --dport 8080 -j DNAT --to-destination 192.168.3.2:8080
iptables -t nat -A PREROUTING -d 172.16.2.2 -p tcp --dport 2026 -j DNAT --to-destination 192.168.3.2:2026
iptables-save > /etc/rules.v4
iptables -t nat -L -n -v

2.4 HQ-SRV: RAID0, NFS, chrony, LAMP
1) RAID0. В Proxmox заранее добавить два диска по 1 ГБ. Имена обычно /dev/sdb и /dev/sdc, но сначала проверь lsblk.
lsblk
apt-get update
apt-get install -y mdadm
mdadm --create /dev/md0 --level=0 --raid-devices=2 /dev/sdb /dev/sdc
mdadm --detail --scan > /etc/mdadm.conf
fdisk /dev/md0

# в fdisk нажать: n, p, Enter, Enter, Enter, w
mkfs.ext4 /dev/md0p1
mkdir -p /raid
mount /dev/md0p1 /raid
df -h

Автомонтирование RAID через systemd.
systemctl --force --full edit raid.mount

В открывшийся файл raid.mount вписать:
[Unit]
Description=Mount RAID

[Mount]
What=/dev/md0p1
Where=/raid
Type=ext4
Options=defaults

[Install]
WantedBy=multi-user.target

systemctl enable raid.mount --now
mount -a
df -h | grep raid

2) NFS-сервер. Папка доступна только сети HQ-CLI 192.168.2.0/28.
apt-get install -y nfs-server
mkdir -p /raid/nfs
chmod 777 /raid/nfs
mcedit /etc/exports

В /etc/exports добавить одну строку. Важно: между сетью и скобкой пробела нет.
/raid/nfs 192.168.2.0/28(rw,sync,no_subtree_check,no_root_squash)

exportfs -a
systemctl enable nfs-server --now
exportfs -v

3) Chrony клиент на HQ-SRV. На ALT файл обычно /etc/chrony.conf.
apt-get install -y chrony
mcedit /etc/chrony.conf

# старые pool/server закомментировать, в конец добавить:
server 172.16.1.1 iburst

systemctl restart chronyd || systemctl restart chrony
systemctl enable chronyd || systemctl enable chrony
chronyc sources

4) LAMP-приложение. Additional.iso должен быть подключен к HQ-SRV.
apt-get update
apt-get install -y lamp-server
mount /dev/sr0 /mnt
ls /mnt
cp /mnt/web/index.php /var/www/html/
cp -r /mnt/web/images /var/www/html/ 2>/dev/null || cp /mnt/web/logo.png /var/www/html/
mcedit /var/www/html/index.php

# в index.php проверить:
# база: webdb
# пользователь: web
# пароль: P@ssw0rd
# host: localhost

systemctl enable --now mariadb
mariadb -u root

В MariaDB ввести по строкам:
CREATE DATABASE webdb;
CREATE USER 'web'@'localhost' IDENTIFIED BY 'P@ssw0rd';
GRANT ALL PRIVILEGES ON webdb.* TO 'web'@'localhost' WITH GRANT OPTION;
FLUSH PRIVILEGES;
EXIT;

mariadb -u web -p -D webdb < /mnt/web/dump.sql
systemctl enable --now httpd2
curl http://127.0.0.1
curl http://192.168.1.2

2.5 BR-SRV: Samba DC, пользователи, Ansible, Docker
1) Samba DC. Пароль администратора домена при provision задай P@ssw0rd, чтобы потом не путаться.
apt-get update
apt-get install -y task-samba-dc
rm -f /etc/samba/smb.conf
samba-tool domain provision

При вопросах указать: Realm = AU-TEAM.IRPO, Domain = AU-TEAM, Server Role = dc, DNS backend = SAMBA_INTERNAL, DNS forwarder = 192.168.1.2.
cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
systemctl disable bind --now
mcedit /etc/samba/smb.conf

В блок [global] добавить:
interfaces = lo ens18
bind interfaces only = yes

systemctl enable samba --now
systemctl restart samba
samba-tool domain info 127.0.0.1

2) Пользователи домена и группа hq.
samba-tool group add hq
samba-tool user create hquser1 P@ssw0rd
samba-tool user create hquser2 P@ssw0rd
samba-tool user create hquser3 P@ssw0rd
samba-tool user create hquser4 P@ssw0rd
samba-tool user create hquser5 P@ssw0rd
samba-tool group addmembers hq hquser1,hquser2,hquser3,hquser4,hquser5
samba-tool group listmembers hq

Если HQ-CLI не видит домен, на HQ-SRV в dnsmasq добавить пересылку запросов домена на BR-SRV:
mcedit /etc/dnsmasq.conf
# добавить в конец:
server=/au-team.irpo/192.168.3.2
systemctl restart dnsmasq

3) Ansible на BR-SRV. Рабочий каталог /etc/ansible.
apt-get update
apt-get install -y ansible sshpass
mkdir -p /etc/ansible
mcedit /etc/ansible/ansible.cfg

В /etc/ansible/ansible.cfg вписать:
[defaults]
host_key_checking = False
inventory = /etc/ansible/hosts

mcedit /etc/ansible/hosts

В /etc/ansible/hosts вписать:
[routers]
hq-rtr ansible_host=10.0.0.1 ansible_user=net_admin ansible_password=P@ssw0rd
br-rtr ansible_host=10.0.0.2 ansible_user=net_admin ansible_password=P@ssw0rd

[servers]
hq-srv ansible_host=192.168.1.2 ansible_user=sshuser ansible_password=P@ssw0rd ansible_port=2026

[clients]
hq-cli ansible_host=192.168.2.2 ansible_user=user ansible_password=P@ssw0rd

ansible all -m ping

Если HQ-CLI не отвечает pong, на HQ-CLI поставить SSH: apt-get install -y openssh; systemctl enable sshd --now.
4) Docker testapp. Additional.iso должен быть подключен к BR-SRV.
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

В docker-compose.yaml вписать. Отступы только пробелами, не Tab.
 
curl http://127.0.0.1:8080
curl http://192.168.3.2:8080

2.6 HQ-CLI: домен, NFS, sudo для hq, Яндекс Браузер
1) Ввести машину в домен через графический центр управления.
acc

Дальше: Центр управления → Пользователи → Аутентификация → Домен Active Directory → домен au-team.irpo. Поставить галочку “Восстановить файлы конфигурации по умолчанию”. Пароль администратора домена — тот, который задавала при Samba provision.
После успешного ввода в домен перезагрузить HQ-CLI и войти пользователем hquser1 / P@ssw0rd.
2) Ограниченный sudo для группы hq: можно только cat, grep, id.
nano /etc/sudoers

Добавить один рабочий вариант. Сначала попробуй простой:
%hq ALL=(ALL) /bin/cat, /bin/grep, /usr/bin/id

Если группа в id отображается как AU-TEAM\hq, тогда вместо строки выше использовать:
%AU-TEAM\hq ALL=(ALL) /bin/cat, /bin/grep, /usr/bin/id

sudo -l
sudo id

3) NFS-автомонтирование /mnt/nfs.
apt-get update
apt-get install -y nfs-clients
mkdir -p /mnt/nfs
systemctl --force --full edit mnt-nfs.mount

В файл mnt-nfs.mount вписать:
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

systemctl enable mnt-nfs.mount --now
mount -a
df -h | grep nfs
touch /mnt/nfs/test.txt

4) Яндекс Браузер.
apt-get install -y yandex-browser-stable

2.7 Быстрая финальная проверка модуля 2
Где	Команды проверки	Что должно быть
ISP	systemctl status chrony nginx; nginx -t	chrony/nginx active, nginx -t без ошибок
HQ-RTR	chronyc sources; iptables -t nat -L -n -v	виден NTP-сервер, есть DNAT 8080 и 2026
BR-RTR	chronyc sources; iptables -t nat -L -n -v	виден NTP-сервер, есть DNAT 8080 и 2026
HQ-SRV	df -h; exportfs -v; systemctl status nfs-server httpd2 mariadb	/raid смонтирован, NFS/LAMP работают
BR-SRV	systemctl status samba docker; docker ps; ansible all -m ping	Samba/Docker active, контейнеры db и testapp, ansible pong
HQ-CLI	df -h | grep nfs; nslookup web.au-team.irpo; браузер web/docker	NFS смонтирован, DNS работает, сайты открываются

2.8 Если сломалось во втором модуле
Ошибка	Почему	Что сделать быстро
chronyc sources пустой	клиент не видит ISP или неверный файл chrony	проверить ping 172.16.1.1; в chrony прописать server 172.16.1.1 iburst; restart chrony/chronyd
RAID не создаётся	диски называются не sdb/sdc или уже есть md0	lsblk; mdadm --stop /dev/md0; проверить реальные имена дисков
exportfs: No host name given	пробел между сетью и скобкой в /etc/exports	писать 192.168.2.0/28(rw...) без пробела
NFS: rpc.statd required	клиент требует блокировки	в mount-unit поставить Options=_netdev,rw,nolock
Samba domain info не работает	samba не запущена или конфликтует bind	systemctl disable bind --now; systemctl restart samba; samba-tool domain info 127.0.0.1
HQ-CLI не входит в домен	DNS не ведёт au-team.irpo на BR-SRV	на HQ-SRV в dnsmasq добавить server=/au-team.irpo/192.168.3.2 и restart dnsmasq
ansible не pong	SSH недоступен, неправильный порт или пользователь	с BR-SRV вручную проверить ssh; для HQ-SRV нужен ansible_port=2026
docker compose ругается	ошибка отступов или не загружены images	docker images; в yaml только пробелы; docker compose config
curl 192.168.3.2:8080 висит	контейнер не поднялся или порт не проброшен	docker ps; ports должны быть 0.0.0.0:8080->8000
LAMP показывает ошибку БД	не тот пользователь/пароль в index.php	в index.php web / P@ssw0rd / webdb / localhost; проверить mariadb import
nginx -t ошибка	скобка или ; пропущены	nginx -t показывает строку; проверить ; после proxy_set_header
web.au-team.irpo не спрашивает пароль	auth_basic не в server web	проверить server_name web.au-team.irpo и строки auth_basic/auth_basic_user_file

образования.
