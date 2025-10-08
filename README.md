# Module 1 (Demo26)
## Команды для выполнения Модуля 1 - Демоэкзамен 2026. 
<details>
<summary> - ISP  </summary>
hostnamectl set-hostname ISP
mkdir /etc/net/ifaces/{ens20,ens21,ens22}
echo -e "BOOTPROTO=static\nCONFIG_IPV4=yes\nDISABLED=no\nTYPE=eth" > /etc/net/ifaces/ens20/options
cp /etc/net/ifaces/ens20/options /etc/net/ifaces/ens21/options
cp /etc/net/ifaces/ens20/options /etc/net/ifaces/ens22/options
echo -e "BOOTPROTO=dhcp\nCONFIG_IPV4=yes\nDISABLED=no\nTYPE=eth" > /etc/net/ifaces/ens20/options
echo 172.16.1.1/28 > /etc/net/ifaces/ens21/ipv4address
echo 172.16.2.1/28 > /etc/net/ifaces/ens22/ipv4address
echo nameserver 8.8.8.8 > /etc/resolv.conf
sed -i 's/net.ipv4.ip_forward = 0/net.ipv4.ip_forward = 1/g' /etc/net/sysctl.conf
sysctl -p
systemctl restart network
ip -c a
apt-get update && apt-get install chrony iptables nginx -y
iptables -t nat -A POSTROUTING -o ens20 -s 172.16.1.0/28 -j MASQUERADE
iptables -t nat -A POSTROUTING -o ens20 -s 172.16.2.0/28 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
systemctl restart iptables
apt-get update && apt-get reinstall tzdata
timedatectl set-timezone Asia/Yekaterinburg
timedatectl
</details>

<details>
<summary> - HQ-RTR </summary>
en
conf t
hostname hq-rtr
ip domain-name au-team.irpo
int int0
description "to isp"
ip address 172.16.1.4/28
ip nat outside
exit
port te0
service-instance te0/int0
encapsulation untagged
exit
exit
int int0
connect port te0 service-instance te0/int0
exit
int int1
description "to hq-srv"
ip address 192.168.1.1/27
ip nat inside
exit
int int2
description "to hq-cli"
ip address 192.168.2.1/28
ip nat inside
exit
int int3
description "999"
ip address 192.168.1.99/29
exit
port te1
service-instance te1/int1
encapsulation dot1q 100
rewrite pop 1
exit
service-instance te1/int2
encapsulation dot1q 200
rewrite pop 1
exit
service-instance te1/int3
encapsulation dot1q 999
rewrite pop 1
exit
exit
int int1
connect port te1 service-instance te1/int1
exit
int int2
connect port te1 service-instance te1/int2
exit
int int3
connect port te1 service-instance te1/int3
exit
ip route 0.0.0.0 0.0.0.0 172.16.1.1
write
username net_admin
password P@ssw0rd
role admin
exit
int tunnel.0
ip address 172.16.0.1/30
ip mtu 1400
ip tunnel 172.16.1.4 172.16.2.5 mode gre
ip ospf authentication-key ecorouter
exit
router ospf 1
net 172.16.0.0/30 ar 0
net 192.168.1.0/27 ar 0
net 192.168.2.0/28 ar 0
passive-interface default
no passive-interface tunnel.0
ar 0 auth
exit
write
ip name-server 8.8.8.8
ip nat pool NAT_POOL 192.168.1.1-192.168.1.254,192.168.2.1-192.168.2.254
ip nat source dynamic inside-to-outside pool NAT_POOL overload int int0
ip pool cli_pool 192.168.2.10-192.168.2.10
dhcp-server 1
pool cli_pool 1
mask 255.255.255.240
gateway 192.168.2.1
dns 192.168.1.10
domain-name au-team.irpo
exit
int int2
dhcp-server 1
exit
ntp timezone utc+5
ntp server 172.16.1.1
write
exit
show run
</details>

<details>
<summary> - BR-RTR </summary>
en
conf t
hostname br-rtr
ip domain-name au-team.irpo
int int0
description "to isp"
ip address 172.16.2.5/28
ip nat outside
exit
port te0
service-instance te0/int0
encapsulation untagged
exit
exit
int int0
connect port te0 service-instance te0/int0
exit
int int1
description "to br-srv"
ip address 192.168.3.1/28
ip nat inside
exit
port te1
service-instance te1/int1
encapsulation untagged
exit
exit
int int1
connect port te1 service-instance te1/int1
exit
ip route 0.0.0.0 0.0.0.0 172.16.2.1
write
username net_admin
password P@ssw0rd
role admin
exit
int tunnel.0
ip address 172.16.0.2/30
ip mtu 1400
ip tunnel 172.16.2.5 172.16.1.4 mode gre
ip ospf authentication-key ecorouter
exit
router ospf 1
net 172.16.0.0/30 ar 0
net 192.168.3.0/28 ar 0
passive-interface default
no passive-interface tunnel.0
ar 0 auth
exit
write
ip name-server 8.8.8.8
ip nat pool NAT_POOL 192.168.3.1-192.168.3.254
ip nat source dynamic inside-to-outside pool NAT_POOL overload int int0
ntp timezone utc+5
ntp server 172.16.2.1
write
exit
show run
</details>

<details>
<summary> - HQ-SRV  </summary>
hostnamectl set-hostname hq-srv.au-team.irpo
mkdir /etc/net/ifaces/ens20
echo -e "BOOTPROTO=static\nCONFIG_IPV4=yes\nDISABLED=no\nTYPE=eth" > /etc/net/ifaces/ens20/options
echo 192.168.1.10/27 > /etc/net/ifaces/ens20/ipv4address
echo default via 192.168.1.1 > /etc/net/ifaces/ens20/ipv4route
echo nameserver 8.8.8.8 > /etc/resolv.conf
systemctl restart network
ip -c a
useradd remote_user -u 2026
echo "remote_user:P@ssw0rd" | chpasswd
sed -i 's/# WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL/WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL/g' /etc/sudoers
gpasswd -a "remote_user" wheel
sed -i 's/#Port 22/Port 2026\nAllowUsers remote_user\nMaxAuthTries 2\nPasswordAuthentication yes\nBanner /etc/openssh/banner/g' /etc/openssh/sshd_config
echo Authorized access only > /etc/openssh/banner
systemctl restart sshd
apt-get update && apt-get install chrony nfs-server fdisk dnsmasq -y
timedatectl set-timezone Asia/Yekaterinburg
systemctl enable --now dnsmasq
 ---
echo -e "no-resolv\ndomain=au-team.irpo\nserver=8.8.8.8\ninterface=ens20\naddress=/hq-rtr.au-team.irpo/192.168.1.1\nptr-record=1.1.168.192.in-addr.arpa,hq-rtr.au-team.irpo\naddress=/docker.au-team.irpo/172.16.1.1\naddress=/web.au-team.irpo/172.16.2.1\naddress=hq-srv.au-team.irpo/192.168.1.10\nptr-record=10.1.168.192.in-addr.arpa,hq-srv.au-team.irpo\naddress=/hq-cli.au-team.irpo/192.168.2.10\nptr-record=10.2.168.192.in-addr.arpa,hq-cli.au-team.irpo\naddress=/br-rtr.au-team.irpo/192.168.3.1\naddress=/br-srv.au-team.irpo/192.168.3.10" /etc/dnsmasq.conf
 ---
echo -e "192.168.1.1  hq-rtr.au-team.irpo" >> /etc/hosts
 ---
systemctl restart dnsmasq
</details>


<details>
<summary> - HQ-CLI  </summary>
hostnamectl set-hostname hq-cli.au-team.irpo
mkdir /etc/net/ifaces/ens20
echo -e "BOOTPROTO=static\nCONFIG_IPV4=yes\nDISABLED=no\nTYPE=eth" > /etc/net/ifaces/ens20/options
echo 192.168.2.10/28 > /etc/net/ifaces/ens20/ipv4address
echo default via 192.168.2.1 > /etc/net/ifaces/ens20/ipv4route
echo nameserver 8.8.8.8 > /etc/resolv.conf
systemctl restart network
ip -c a
useradd remote_user -u 2026
echo "remote_user:P@ssw0rd" | chpasswd
sed -i 's/# WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL/WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL/g' /etc/sudoers
gpasswd -a "remote_user" wheel
  ---
echo -e "Port 2026\nAllowUsers remote_user\nMaxAuthTries 2\nPasswordAuthentication yes\nBanner /etc/openssh/banner" > /etc/openssh/sshd_config
  ---
echo Authorized access only > /etc/openssh/banner
systemctl restart sshd
apt-get update && apt-get install chrony nfs-clients admc  -y
timedatectl set-timezone Asia/Yekaterinburg
timedatectl
ip -c a
</details>
