############ RTR-HQ ###############

configure

#Начальная настройка

hostname rtr-hq
domain name company.prof
do com
do con

#Настройка адрессации

do show interface status

#gi1/0/1 - ISP
#gi1/0/2 - HQ-SW

interface gi1/0/1
ip address 100.0.1.2/24
no shutdown
ip firewall disable
exit
interface gi1/0/2
no shutdown
ip firewall disable
exit
interface gi1/0/2.100
ip address 10.0.10.1/27
ip firewall disable
exit
interface gi1/0/2.200
ip address 10.0.10.33/27
ip firewall disable
exit
interface gi1/0/2.300
ip address 10.0.10.65/27
ip firewall disable
exit

ip route 0.0.0.0/0 100.0.1.1
do com
do con

#Настройка NAT

object-group network LOCAL
ip prefix 10.0.10.0/24
exit

nat source
ruleset MASQUERADE
to interface gigabitethernet 1/0/1
rule 1
match source-address LOCAL
action source-nat interface
enable
exit
exit
exit
do com
do con

#Настройка туннеля

tunnel vti 1
ip firewall disable
local address 100.0.1.2
remote address 100.0.2.2
ip address 10.10.10.1/30
enable
exit
do com
do con

#Шифрование туннеля

object-group service ISAKMP
port-range 500
exit

security ike proposal ike_prop1
authentication algorithm md5
encryption algorithm aes128
dh-group 2
exit

security ike policy ike_pol1
pre-shared-key hexadecimal encrypted CDE6504B9629
proposal ike_prop1
exit

security ike gateway ike_gw1
version v2-only
ike-policy ike_pol1
mode route-based
bind-interface vti 1
exit

security ipsec proposal ipsec_prop1
authentication algorithm md5
encryption algorithm aes128
exit

security ipsec policy ipsec_pol1
proposal ipsec_prop1
exit

security ipsec vpn ipsec1
ike establish-tunnel route
ike gateway ike_gw1
ike ipsec-policy ipsec_pol1
enable
exit

do com
do con

#Настройка OSPF

router ospf log-adjacency-changes
router ospf 1
area 0.0.0.0
network 10.0.10.0/27
network 10.0.10.32/27
network 10.0.10.64/27
enable
exit
enable
exit

tunnel vti 1
ip ospf instance 1
ip ospf
exit

do com
do con


############ RTR-BR ###############

configure

#Начальная настройка

hostname rtr-br
domain name company.prof
do com
do con

#Настройка адрессации

interface gi1/0/1
ip address 100.0.2.2/24
no shutdown
ip firewall disable
exit
interface gi1/0/2
no shutdown
ip firewall disable
exit
interface gi1/0/2.100
ip address 10.0.20.1/27
ip firewall disable
exit
interface gi1/0/2.200
ip address 10.0.20.33/27
ip firewall disable
exit
interface gi1/0/2.300
ip address 10.0.20.65/27
ip firewall disable
exit

ip route 0.0.0.0/0 100.0.2.1
do com
do con

#Настройка NAT

object-group network LOCAL
ip prefix 10.0.20.0/24
exit

nat source
ruleset MASQUERADE
to interface gigabitethernet 1/0/1
rule 1
match source-address LOCAL
action source-nat interface
enable
exit
exit
exit
do com
do con

#Настройка туннеля

tunnel vti 1
ip firewall disable
local address 100.0.2.2
remote address 100.0.1.2
ip address 10.10.10.2/30
enable
exit
do com
do con

#Шифрование туннеля

object-group service ISAKMP
port-range 500
exit

security ike proposal ike_prop1
authentication algorithm md5
encryption algorithm aes128
dh-group 2
exit

security ike policy ike_pol1
pre-shared-key hexadecimal encrypted CDE6504B9629
proposal ike_prop1
exit

security ike gateway ike_gw1
version v2-only
ike-policy ike_pol1
mode route-based
bind-interface vti 1
exit

security ipsec proposal ipsec_prop1
authentication algorithm md5
encryption algorithm aes128
exit

security ipsec policy ipsec_pol1
proposal ipsec_prop1
exit

security ipsec vpn ipsec1
ike establish-tunnel route
ike gateway ike_gw1
ike ipsec-policy ipsec_pol1
enable
exit

do com
do con

#Настройка OSPF

router ospf log-adjacency-changes
router ospf 1
area 0.0.0.0
network 10.0.20.0/27
network 10.0.20.32/27
network 10.0.20.64/27
enable
exit
enable
exit

tunnel vti 1
ip ospf instance 1
ip ospf
exit

do com
do con


########### SW-HQ ###########

hostnamectl set-hostname sw-hq.company.prof

#Вставляем диск addons2024.iso

mount /dev/sr1 /media
cp -R /media/* ~
ls
apt-get install ./lib*
apt-get install ./log*
apt-get install ./open*

systemctl enable --now openvswitch
systemctl status openvswitch

ip add

#Интерфейсы:
#ens192 - RTR-HQ;
#ens224 - SRV-HQ - vlan100;
#ens256 - CLI-HQ - vlan200;

mkdir /etc/net/ifaces/ens{192,224,256}
mkdir /etc/net/ifaces/SW-HQ
mkdir /etc/net/ifaces/vlan300

sed -i "s/OVS_REMOVE=yes/OVS_REMOVE=no/g" /etc/net/ifaces/default/options

vim /etc/net/ifaces/SW-HQ/options
TYPE=ovsbr
HOST='ens192 ens224 ens256'

vim /etc/net/ifaces/vlan300/options
TYPE=ovsport
BOOTPROTO=static
CONFIG_IPV4=yes
BRIDGE=SW-HQ
VID=300

cp /etc/net/ifaces/ens192/options /etc/net/ifaces/ens224/
cp /etc/net/ifaces/ens192/options /etc/net/ifaces/ens256/

echo 10.0.10.66/27 > /etc/net/ifaces/vlan300/ipv4address
echo default via 10.0.10.65 > /etc/net/ifaces/vlan300/ipv4route
echo nameserver 100.0.1.1 > /etc/net/ifaces/vlan300/resolv.conf

systemctl restart network

ovs-vsctl show

ovs-vsctl set port ens192 trunk=100,200,300
ovs-vsctl set port ens224 tag=100
ovs-vsctl set port ens256 tag=200

reboot

ovs-vsctl show


########### SW-BR ###########

hostnamectl set-hostname sw-br.company.prof

#Вставляем диск addons2024.iso

mount /dev/sr1 /media
cp -R /media/* ~
ls
apt-get install ./lib*
apt-get install ./log*
apt-get install ./open*

systemctl enable --now openvswitch
systemctl status openvswitch

ip add

#Интерфейсы:
#ens192 - RTR-BR;
#ens224 - SRV-BR - vlan100;
#ens256 - CLI-BR - vlan200;

mkdir /etc/net/ifaces/ens{192,224,256}
mkdir /etc/net/ifaces/SW-BR
mkdir /etc/net/ifaces/vlan300

sed -i "s/OVS_REMOVE=yes/OVS_REMOVE=no/g" /etc/net/ifaces/default/options

vim /etc/net/ifaces/SW-BR/options
TYPE=ovsbr
HOST='ens192 ens224 ens256'

vim /etc/net/ifaces/vlan300/options
TYPE=ovsport
BOOTPROTO=static
CONFIG_IPV4=yes
BRIDGE=SW-BR
VID=300

cp /etc/net/ifaces/ens192/options /etc/net/ifaces/ens224/
cp /etc/net/ifaces/ens192/options /etc/net/ifaces/ens256/

echo 10.0.20.66/27 > /etc/net/ifaces/vlan300/ipv4address
echo default via 10.0.20.65 > /etc/net/ifaces/vlan300/ipv4route
echo nameserver 100.0.2.1 > /etc/net/ifaces/vlan300/resolv.conf

systemctl restart network

ovs-vsctl show

ovs-vsctl set port ens192 trunk=100,200,300
ovs-vsctl set port ens224 tag=100
ovs-vsctl set port ens256 tag=200

reboot

ovs-vsctl show


################ SRV-HQ #####################


hostnamectl set-hostname srv-hq.company.prof
bash
echo 10.0.10.2/27 > /etc/net/ifaces/ens192/ipv4address
echo default via 10.0.10.1 > /etc/net/ifaces/ens192/ipv4route
echo nameserver 100.0.1.1 > /etc/net/ifaces/ens192/resolv.conf

systemctl restart network 


################ SRV-BR #####################


hostnamectl set-hostname srv-br.company.prof
bash
echo 10.0.20.2/27 > /etc/net/ifaces/ens192/ipv4address
echo default via 10.0.20.1 > /etc/net/ifaces/ens192/ipv4route
echo nameserver 100.0.2.1 > /etc/net/ifaces/ens192/resolv.conf

systemctl restart network 


################ SSH user ################

useradd -p пароль имя_пользователя - Создаем shhuser с паролем P@ssw0rd
visudo
WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL - меняем wheel_users на sshuser
apt-get update
после скачиваем пакет openssh
а после vim /etc/openssh/sshd_config
ssh sshuser@10.0.10.2 - подключиться к сереверу через клиента
ssh-keygen(на клиенте)
ssh-copy-id sshuser@10.0.10.2
ssh-copy-id sshuser@10.0.20.2  -
Аутентификация пользователя sshuserпри помощи ключей(https://losst.pro/avtorizatsiya-po-klyuchu-ssh?ysclid=ltnz93ea4y355951136)


################ Настройка дисковой подсистемы  SRV-HQ################
1)Создание зеркалируемого LVM тома:
bash
lsblk - посмотреть названия дисков(мы работаем в sdb, sdc)
vgcreate name(mirror_vg) /dev/sdb /dev/sdc - создаем VG
lvcreate -m1 -n name(mirror_lv) -l 100%FREE mirror_vg - создаем зеркалированный LV

2)Форматирование и монтирование LVM тома:
bash
mkfs.ext4 /dev/mirror_vg/mirror_lv - Форматирование LV
mkdir -p /opt/data - Создаем точку монтирования
echo "/dev/mirror_vg/mirror_lv /opt/data ext4 default 0 0" >> /etc/fstab - добавление записи в fstab

3)Монтирование LVM тома:
bash
mount -a - Монтирование всех точук из fstab

4)Проверка
bash
pvs - Показать информацию о физических томах
vgs - Показать информацию о группы LVM
lvs - Показать информацию о логических томах 

################ Настройка дисковой подсистемы  SRV-BR################
1)Шифрование:
bash

lsblk - посмотреть названия дисков(мы работаем в sdb, sdc)

cryptsetup -y -s 256 luksFormat /dev/sdb - Шифрование первого диска(пишем YES(обязательно большими буквами)) - пароль: umwzM@i1

cryptsetup -y -s 256 luksFormat /dev/sdb - Шифрование второго диска(пишем YES(обязательно большими буквами)) - пароль: umwzM@i1

2)Создаем Striped LVM тома:
bash
pvcreate /dev/sdb - создаем PV на 1 зашифрованном диске
pvcreate /dev/sdc - создаем PV на 2 зашифрованном диске

vgcreate name(stripped_vg) /dev/sdb /dev/sdc - Создаем VG

lvcreate -m1 -n stripped_lv -1 100%FREE stripped_vg - Создаем STRIPPED LV

3)Форматирование и монтирование тома:
bash
mkfs.ext4 /dev/stripped_vg/stripped_lv - Форматирование LV

mkdir -p /opt/data - Создание точки монтирования

echo "/dev/stripped_vg/stripped_lv /opt/data ext4 defaults 0 0" >> /etc/fstab - Добавление записи в fstab

4) Настройка автоматического монтирования без запроса пароля:
bash
vim /etc/rc.d/init.d/cryptdisks.function - зайти в файл
blkid /dev/sdb - Посмотреть UUID диска sdb
blkid /dev/sdc - Посмотреть UUID диска sdb
vim /etc/fstab - Внизу прописать: /dev/stripped_vg/stripped_lv /opt/data ext4 defaults 0 0
cryptsetup status crypt_sdb - Статус диска
cryptsetup status crypt_sdc - Статус диска
