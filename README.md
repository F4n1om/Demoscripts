# Demoscripts
=======ISP======= 
-------Имя------- 
hostnamectl hostname ISP 
exec bash 
-------Настройка внешнего интерфейса------- 
cat /etc/net/ifaces/enp7s1/options 
-------Настройка внутренних интерфейсов------- 
mkdir /etc/net/ifaces/enp7s{2,3} 
echo -e 'BOOTPROTO=static\nTYPE=eth'>>/etc/net/ifaces/enp7s2/options 
echo -e 'BOOTPROTO=static\nTYPE=eth'>>/etc/net/ifaces/enp7s3/options 
echo '172.16.4.1/28' > /etc/net/ifaces/enp7s2/ipv4address 
echo '172.16.5.1/28' > /etc/net/ifaces/enp7s3/ipv4address 
-------NAT------- 
apt-get update && apt-get install nftables  -y 
+++
cat << EOF >/etc/nftables/nftables.nft 
#!/usr/sbin/nft -f 
flush ruleset 
table ip nat {
	chain postrouting  {
	type nat hook postrouting priority srcnat
	oifname "enp7s1" masquerade 
	}
}

EOF

+++
systemctl enable --now nftables
sed -i 's/net.ipv4.ip_forward = 0/net.ipv4.ip_forward = 1/g' /etc/net/sysctl.conf
sysctl -p 
systemctl restart network


=======BR-RTR======= 
-------Имя------- 
hostnamectl hostname br-rtr.quark.net
exec bash 

-------Настройка интерфейсов------- 
echo -e 'BOOTPROTO=static\nTYPE=eth'>>/etc/net/ifaces/enp7s1/options 
echo '172.16.5.2/28' > /etc/net/ifaces/enp7s1/ipv4address 
echo 'default via 172.16.5.1' > /etc/net/ifaces/enp7s1/ipv4route 
echo 'nameserver 8.8.8.8' > /etc/net/ifaces/enp7s1/resolv.conf
mkdir /etc/net/ifaces/{enp7s2,gre1} 
echo -e 'BOOTPROTO=static\nTYPE=eth'>>/etc/net/ifaces/enp7s2/options 
echo '172.16.6.1/26' > /etc/net/ifaces/enp7s2/ipv4address 
systemctl restart network

-------GRE-------
+++ 
cat << EOF > /etc/net/ifaces/gre1/options 
TYPE=iptun 
TUNTYPE=gre
TUNLOCAL=172.16.5.2
TUNREMOTE=172.16.4.2
TUNTTL=64 
TUNOPTIONS='ttl 64'
EOF

+++ 
echo "10.10.10.2/30" > /etc/net/ifaces/gre1/ipv4address 
systemctl restart network 

-------Установка-------
apt-get update && apt-get install sudo nftables tzdata frr -y 
-------Создание пользователя----- 
useradd -m fwferret
echo "fwferret:P@$$word" | chpasswd
usermod -aG wheel fwferret 
echo 'WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL' >> /etc/sudoers.d/fwferret
su -l fwferret
sudo id




-------DNS-------
rm -f /etc/net/ifaces/enp7s1/resolv.conf
echo $'search quark.net\nnameserver 172.16.100.2'



-------NAT------- 
+++
cat << EOF > /etc/nftables/nftables.nft 
#!/usr/sbin/nft -f 
flush ruleset 
table ip nat {
chain postrouting  {
type nat hook postrouting priority srcnat
oifname "enp7s1" masquerade 
}
}

EOF
+++
systemctl enable --now nftables
sed -i 's/net.ipv4.ip_forward = 0/net.ipv4.ip_forward = 1/g' /etc/net/sysctl.conf
sysctl -p 
-------OSPFF-------
sed -i 's/ospfd=no/ospfd=yes/' /etc/frr/daemons ; grep ospf /etc/frr/daemons ; 
+++ 
cat << 'EOF' > /etc/frr/frr.conf 
interface gre1 
	no ip ospf passive 
exit 
!
interface gre1 
	ip ospf area 0
	ip ospf authentication 
	ip ospf authentication-key P@ssw0rd
	no ip ospf passive 
exit 
!
interface enp7s2
	ip ospf area 0
exit
!
router ospf
passive-interface default
exit
EOF
+++
systemctl restart network
systemctl enable --now nftables frr
-------TimeZone-------
timedatectl set-timezone Asia/Yekaterinburg

=======HQ-RTR======= 
hostnamectl hostname hq-rtr.quark.net
exec bash 
-------Настройка внешнего интерфейса------- 
echo -e 'BOOTPROTO=static\nTYPE=eth'>/etc/net/ifaces/enp7s1/options 
echo '172.16.4.2/28' > /etc/net/ifaces/enp7s1/ipv4address 
echo 'default via 172.16.4.1' > /etc/net/ifaces/enp7s1/ipv4route 
echo 'nameserver 8.8.8.8' > /etc/net/ifaces/enp7s1/resolv.conf
systemctl restart network
-------Настройка внутренних вланов------- 
mkdir /etc/net/ifaces/{enp7s2,enp7s2.{100,200,999},gre1} 
echo -e 'BOOTPROTO=static\nTYPE=eth'>>/etc/net/ifaces/enp7s2/options 
echo -e 'BOOTPROTO=static\nTYPE=vlan\nVID=100\nHOST=enp7s2\nONBOOT=yes'>>/etc/net/ifaces/enp7s2.100/options 
echo -e 'BOOTPROTO=static\nTYPE=vlan\nVID=200\nHOST=enp7s2\nONBOOT=yes'>>/etc/net/ifaces/enp7s2.200/options 
echo -e 'BOOTPROTO=static\nTYPE=vlan\nVID=999\nHOST=enp7s2\nONBOOT=yes'>>/etc/net/ifaces/enp7s2.999/options 
echo '172.16.100.1/29'>/etc/net/ifaces/enp7s2.100/ipv4address 
echo '172.16.200.1/26' > /etc/net/ifaces/enp7s2.200/ipv4address 
echo '172.16.99.1/26'>/etc/net/ifaces/enp7s2.999/ipv4address 
systemctl restart network
-------GRE-------
+++ 
cat << EOF > /etc/net/ifaces/gre1/options 
TYPE=iptun 
TUNTYPE=gre
TUNLOCAL=172.16.4.2
TUNREMOTE=172.16.5.2
TUNTTL=64 
TUNOPTIONS='ttl 64'
EOF
+++ 
echo "10.10.10.1/30" > /etc/net/ifaces/gre1/ipv4address 
systemctl restart network 
-------Установка-------
apt-get update && apt-get install sudo nftables tzdata frr dhcp-server -y 	
-------Создание пользователя----- 
useradd -m fwferret
echo "fwferret:P@$$word" | chpasswd
usermod -aG wheel fwferret
echo 'WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL' >> /etc/sudoers.d/fwferret
su -l fwferret
sudo id



-------DNS-------
rm -f /etc/net/ifaces/enp7s1/resolv.conf
echo -e 'search quark.net\nnameserver 172.16.100.2' > /etc/resolv.conf
-------NAT------- 
+++
cat << EOF > /etc/nftables/nftables.nft 
#!/usr/sbin/nft -f 
flush ruleset 
table ip nat {
chain postrouting  {
type nat hook postrouting priority srcnat
oifname "enp7s1" masquerade 
}
}

EOF
+++
systemctl enable --now nftables
sed -i 's/net.ipv4.ip_forward = 0/net.ipv4.ip_forward = 1/g' /etc/net/sysctl.conf
sysctl -p 
-------OSPFF-------
sed -i 's/ospfd=no/ospfd=yes/' /etc/frr/daemons ; grep ospf /etc/frr/daemons ; 
+++ 
cat << 'EOF' > /etc/frr/frr.conf 
interface gre1 
	no ip ospf passive 
exit 
!
interface gre1 
	ip ospf area 0
	ip ospf authentication 
	ip ospf authentication-key P@ssw0rd
	no ip ospf passive 
exit 
!
interface enp7s2.100
	ip ospf area 0
exit
!
interface enp7s2.200
	ip ospf area 0
exit
!
interface enp7s2.999 
	ip ospf area 0
exit
!
router ospf
	passive-interface default
exit
EOF
+++
systemctl enable --now frr
-------TimeZone-------
timedatectl set-timezone Asia/Yekaterinburg
-------DHCP-------
cat << 'EOF' > /etc/dhcp/dhcpd.conf
subnet 172.16.200.0 netmask 255.255.255.192 {
range 172.16.200.2 172.16.200.30;
option routers 172.16.200.1;
option domain-name-servers 172.16.100.2;
option domain-search "quark.net";
default-lease-time 600;
max-lease-time 7200;
}
EOF
+++
sed -i 's/DHCPDARGS=/DHCPDARGS=enp7s2.200/g' /etc/sysconfig/dhcpd
systemctl enable --now dhcpd
systemctl restart network
systemctl restart dhcpd
=======HQ-SRV======= 
-------Имя------- 
hostnamectl hostname hq-srv.quark.net
exec bash 
-------Насиройка интерфейсов-------
echo -e 'BOOTPROTO=static\nTYPE=eth\nONBOOT=yes'>/etc/net/ifaces/enp7s1/options 
echo '172.16.100.2/27' > /etc/net/ifaces/enp7s1/ipv4address
echo 'default via 172.16.100.1' > /etc/net/ifaces/enp7s1/ipv4route
echo 'nameserver 8.8.8.8' > /etc/net/ifaces/enp7s1/resolv.conf
systemctl restart network
------установка необходимого ПО-------
apt-get update && apt-get install bind bind-utils tzdata -y
timedatectl set-timezone Asia/Yekaterinburg
------Настройка sshuser----- 
useradd -u 1013matrixmacaw
echo "matrixmacaw:P@ssw0rd" | chpasswd
usermod -aG wheel matrixmacaw
echo "WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL" > /etc/sudoers.d/matrixmacaw
su -l matrixmacaw
sudo id
------Настройка SSH------
echo "Authorized access only" > /etc/openssh/banner
echo -e "Port 2032\nMaxAuthTries 2\nAllowUsers matrixmacaw\nBanner /etc/openssh/banner\n">>/etc/openssh/sshd_config
systemctl restart sshd
ss -ltnp | grep sshd

ssh matrixmacaw@127.0.0.1 -p 2032
------смена DNS------
echo $'search quark.net\nnameserver 127.0.0.1' > /etc/net/ifaces/enp7s1/resolv.conf
-------BIND9--------
+++
cat > /etc/bind/options.conf << 'EOF'
options {
    directory "/etc/bind/zone";
    listen-on { 172.16.100.2; 127.0.0.1; };
    allow-query { any; };
    forwarders { 8.8.8.8; };
    recursion yes;
};
EOF
+++
cat > /etc/bind/rfc1912.conf<<'EOF'

zone "quark.net" IN {
    type master;
    file "/etc/bind/zone/quark.net";
};

zone "100.16.172.in-addr.arpa" IN {
    type master;
    file "/etc/bind/zone/100.16.172.in-addr.arpa";
};
EOF
+++
cat>/etc/bind/zone/quark.net << 'EOF'
$TTL 86400
@   IN SOA  hq-srv.quark.net. root.quark.net. (
            2025010101 ; Serial
            3600       ; Refresh
            1800       ; Retry
            604800     ; Expire
            86400 )    ; Minimum TTL

@       IN NS   hq-srv.quark.net.
hq-rtr  IN A    172.16.100.1
hq-srv  IN A    172.16.100.2
hq-cli  IN A    172.16.200.2
br-rtr  IN A    172.16.6.1
br-srv  IN A    172.16.6.2
moodle  IN CNAME hq-rtr.quark.net.
wiki    IN CNAME hq-rtr.quark.net.
EOF
+++
cat > /etc/bind/zone/100.16.172.in-addr.arpa<<'EOF'
$TTL 86400
@   IN SOA  hq-srv.quark.net. root.quark.net. (
            2025010101
            3600
            1800
            604800
            86400 )

@       IN NS   hq-srv.quark.net.

1   IN PTR  hq-rtr.quark.net.
2   IN PTR  hq-srv.quark.net.
EOF
+++
rndc-confgen > /etc/bind/rndc.key
sed -i '6,$d' /etc/bind/rndc.key
named-checkconf
named-checkconf -z
systemctl enable --now bind
systemctl status bind

host hq-srv.quark.net 172.16.100.2
nslookup hq-rtr.quark.net 172.16.100.2

=======HQ-CLI======= 
-------Имя------- 
hostnamectl hostname hq-cli.quark.net
exec bash
echo -e 'BOOTPROTO=dhcp\nTYPE=eth'>/etc/net/ifaces/enp7s1/options 
systemctl restart network
=======BR-SRV======= 
-------Имя------- 
hostnamectl hostname br-srv.quark.net
exec bash 
-------Настройка интерфейсов------- 
echo -e 'BOOTPROTO=static\nTYPE=eth'>>/etc/net/ifaces/enp7s1/options 
echo '172.16.6.2/28' > /etc/net/ifaces/enp7s1/ipv4address 
echo 'default via 172.16.6.1' > /etc/net/ifaces/enp7s1/ipv4route 
echo 'nameserver 172.16.100.2' > /etc/net/ifaces/enp7s1/resolv.conf
systemctl restart network
-------Создание пользователя----- 
useradd -u 1013matrixmacaw
echo "matrixmacaw:P@ssw0rd" | chpasswd
usermod -aG w