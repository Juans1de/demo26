# Demo 2026 (M1 & M2)
## Модуль №1 - Команды для ВМ
<details>
<summary> - ISP </summary>

```bash
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
systemctl enable --now iptables
apt-get update && apt-get reinstall tzdata
timedatectl set-timezone Asia/Yekaterinburg
timedatectl
exec bash
```

</details>

<details>
<summary> - HQ-RTR </summary>

```bash
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
```

</details>

<details>
<summary> - HQ-SRV </summary>

```bash
hostnamectl set-hostname hq-srv.au-team.irpo
mkdir /etc/net/ifaces/ens20
echo -e "BOOTPROTO=static\nCONFIG_IPV4=yes\nDISABLED=no\nTYPE=eth" > /etc/net/ifaces/ens20/options
echo 192.168.1.10/27 > /etc/net/ifaces/ens20/ipv4address
echo default via 192.168.1.1 > /etc/net/ifaces/ens20/ipv4route
echo nameserver 8.8.8.8 > /etc/resolv.conf
systemctl restart network
ip -c a
useradd sshuser -u 2026
echo "sshuser:P@ssw0rd" | chpasswd
sed -i 's/# WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL/WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL/g' /etc/sudoers
gpasswd -a "sshuser" wheel
sed -i 's/#Port 22/Port 2026\nAllowUsers sshuser\nMaxAuthTries 2\nPasswordAuthentication yes\nBanner \/etc\/openssh\/banner/' /etc/openssh/sshd_config
echo Authorized access only > /etc/openssh/banner
systemctl enable --now sshd
apt-get update && apt-get install chrony nfs-server fdisk dnsmasq -y
timedatectl set-timezone Asia/Yekaterinburg
systemctl enable --now dnsmasq
cat << EOF >> /etc/dnsmasq.conf
no-resolv
domain=au-team.irpo
server=8.8.8.8
interface=*
address=/hq-rtr.au-team.irpo/192.168.1.1
server=/au-team.irpo/192.168.3.10
ptr-record=1.1.168.192.in-addr.arpa,hq-rtr.au-team.irpo
address=/web.au-team.irpo/172.16.1.1
address=/docker.au-team.irpo/172.16.2.1
address=/br-rtr.au-team.irpo/192.168.3.1
address=/hq-srv.au-team.irpo/192.168.1.10
ptr-record=10.1.168.192.in-addr.arpa,hq-srv.au-team.irpo
address=/hq-cli.au-team.irpo/192.168.2.10
ptr-record=10.2.168.192.in-addr.arpa,hq-cli.au-team.irpo
address=/br-srv.au-team.irpo/192.168.3.10
EOF
echo -e "192.168.1.1  hq-rtr.au-team.irpo" >> /etc/hosts
systemctl restart dnsmasq
exec bash
```

</details>

<details>
<summary> - HQ-CLI  </summary>

```bash
hostnamectl set-hostname hq-cli.au-team.irpo
mkdir /etc/net/ifaces/ens20
echo -e "BOOTPROTO=static\nCONFIG_IPV4=yes\nDISABLED=no\nTYPE=eth" > /etc/net/ifaces/ens20/options
echo 192.168.2.10/28 > /etc/net/ifaces/ens20/ipv4address
echo default via 192.168.2.1 > /etc/net/ifaces/ens20/ipv4route
echo nameserver 8.8.8.8 > /etc/resolv.conf
systemctl restart network
ip -c a
useradd sshuser -u 2026
echo "sshuser:P@ssw0rd" | chpasswd
sed -i 's/# WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL/WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL/g' /etc/sudoers
gpasswd -a "sshuser" wheel
sed -i 's/#Port 22/Port 2026\nAllowUsers sshuser\nMaxAuthTries 2\nPasswordAuthentication yes\nBanner \/etc\/openssh\/banner/' /etc/openssh/sshd_config
echo Authorized access only > /etc/openssh/banner
systemctl enable --now sshd
apt-get update && apt-get install chrony nfs-clients admc bind-utils  -y
timedatectl set-timezone Asia/Yekaterinburg
timedatectl
rm -rf /etc/net/ifaces/ens20/{ipv4address,ipv4route}
echo -e "BOOTPROTO=dhcp\nCONFIG_IPV4=yes\nDISABLED=no\nTYPE=eth" > /etc/net/ifaces/ens20/options
systemctl restart network
ip -c a
exec bash
```

</details>

<details>
<summary> - BR-RTR </summary>

```bash
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
```

</details>

<details>
<summary> - BR-SRV </summary>

```bash
hostnamectl set-hostname br-srv.au-team.irpo
mkdir /etc/net/ifaces/ens20
echo -e "BOOTPROTO=static\nCONFIG_IPV4=yes\nDISABLED=no\nTYPE=eth" > /etc/net/ifaces/ens20/options
echo 192.168.3.10/28 > /etc/net/ifaces/ens20/ipv4address
echo default via 192.168.3.1 > /etc/net/ifaces/ens20/ipv4route
echo nameserver 8.8.8.8 > /etc/resolv.conf
systemctl restart network
ip -c a
useradd sshuser -u 2026
echo "sshuser:P@ssw0rd" | chpasswd
sed -i 's/# WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL/WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL/g' /etc/sudoers
gpasswd -a "sshuser" wheel
sed -i 's/#Port 22/Port 2026\nAllowUsers sshuser\nMaxAuthTries 2\nPasswordAuthentication yes\nBanner \/etc\/openssh\/banner/' /etc/openssh/sshd_config
echo Authorized access only > /etc/openssh/banner
systemctl enable --now sshd
apt-get update && apt-get install chrony docker-compose docker-engine ansible task-samba-dc sshpass wget dos2unix-y
timedatectl set-timezone Asia/Yekaterinburg
timedatectl
exec bash
```

</details>

## Модуль №2 - Команды для ВМ (ПРИОСТАНОВЛЕНО)
<details> 
<summary> - HQ-CLI </summary>

```bash
systemctl restart network
system-auth write ad AU-TEAM.IRPO cli AU-TEAM 'administrator' 'P@ssw0rd'
reboot
```

```bash
apt-get install sudo libsss_sudo -y
control sudo public
sed -i '19 a\
sudo_provider = ad' /etc/sssd/sssd.conf
sed -i 's/services = nss, pam/services = nss, pam, sudo/' /etc/sssd/sssd.conf
sed -i '28 a\
sudoers: files sss' /etc/nsswitch.conf
rm -rf /var/lib/sss/db/*
sss_cache -E
systemctl restart sssd
```

</details>

## Модуль №2 - Команды для ВМ (Без SAMBA)
<details>
<summary> - ISP </summary>

```bash
echo -e "server 127.0.0.1 iburst prefer\n\thwtimestamp *\n\tlocal stratum 5\n\tallow 0/0" > /etc/chrony.conf
systemctl enable --now chronyd
systemctl restart chronyd
chronyc sources
chronyc tracking | grep Stratum
apt-get update && apt-get install apache2-htpasswd -y
htpasswd -bc /etc/nginx/.htpasswd WEB P@ssw0rd
echo -e "server {\n\tlisten 80;\n\tserver_name web.au-team.irpo;\n\tauth_basic \"Restricted Access\";\n\tauth_basic_user_file /etc/nginx/.htpasswd;\n\tlocation / {\n\t\tproxy_pass http://172.16.1.4:8080;\n\t\tproxy_set_header Host \$host;\n\t\tproxy_set_header X-Real-IP \$remote_addr;\n\t}\n}\nserver {\n\tlisten 80;\n\tserver_name docker.au-team.irpo;\n\tlocation / {\n\t\tproxy_pass http://172.16.2.5:8080;\n\t\tproxy_set_header Host \$host;\n\t\tproxy_set_header X-Real-IP \$remote_addr;\n\t}\n}" > /etc/nginx/sites-available.d/proxy.conf
ln -s /etc/nginx/sites-available.d/proxy.conf /etc/nginx/sites-enabled.d/
mv /etc/nginx/sites-available.d/default.conf /root/
systemctl enable --now nginx
```

</details>

<details>
<summary> - HQ-RTR </summary>

```bash
ip nat source static tcp 192.168.1.10 80 172.16.1.4 8080
ip nat source static tcp 192.168.1.10 2026 172.16.1.4 2026
write
```

</details>

<details>
<summary> - HQ-SRV </summary>

```bash
lsblk
mdadm --create /dev/md0 --level=0 --raid-devices=2 /dev/sd[b-c]
mdadm --detail -scan --verbose > /etc/mdadm.conf
echo -e "n\n\n\n\n\nw" | fdisk /dev/md0
mkfs.ext4 /dev/md0p1
cat << EOF >> /etc/fstab
/dev/md0p1  /raid  ext4  defaults  0  0
EOF
mkdir /raid
mount -a
sleep 2
mkdir /raid/nfs
chown 99:99 /raid/nfs
chmod 777 /raid/nfs
echo "/raid/nfs 192.168.2.0/28(rw,sync,no_subtree_check)" >> /etc/exports
exportfs -a
exportfs -v
systemctl enable --now nfs
systemctl restart nfs
echo server 172.16.1.1 iburst prefer > /etc/chrony.conf
systemctl enable --now chronyd
timedatectl
apt-get update
apt-get install -y apache2 php8.2 apache2-mod_php8.2 mariadb-server php8.2-{opcache,curl,gd,intl,mysqli,xml,xmlrpc,ldap,zip,soap,mbstring,json,xmlreader,fileinfo,sodium}
mount -o loop /dev/sr0
systemctl enable --now httpd2 mysqld
echo -e "\ny\nP@ssw0rd\nP@ssw0rd\ny\ny\ny\ny" > mysql_secure_installation_answers.txt
sudo mysql_secure_installation < mysql_secure_installation_answers.txt
mariadb -u root -pP@ssw0rd
CREATE DATABASE webdb;
CREATE USER 'webc'@'localhost' IDENTIFIED BY 'P@ssw0rd';
GRANT ALL PRIVILEGES ON webdb.* TO 'webc'@'localhost';
FLUSH PRIVILEGES;
EXIT;
iconv -f UTF-16LE -t UTF-8 /media/ALTLinux/web/dump.sql > /tmp/dump_utf8.sql
mariadb -u root -pP@ssw0rd webdb < /tmp/dump_utf8.sql
chmod 777 /var/www/html
cp /media/ALTLinux/web/index.php /var/www/html
cp /media/ALTLinux/web/logo.png /var/www/html
rm -f /var/www/html/index.html
chown apache2:apache2 /var/www/html
systemctl restart httpd2
sed -i 's/$username = "user";/$username = "webc";/g' /var/www/html/index.php
sed -i 's/$password = "password";/$password = "P@ssw0rd";/g' /var/www/html/index.php
sed -i 's/$dbname = "db";/$dbname = "webdb";/g' /var/www/html/index.php
ip -c a
```

</details>

<details>
<summary> - HQ-CLI </summary>

```bash
systemctl restart network
system-auth write ad AU-TEAM.IRPO cli AU-TEAM 'administrator' 'P@ssw0rd'
reboot
```

```bash
apt-get install sudo libsss_sudo -y
control sudo public
sed -i '19 a\
sudo_provider = ad' /etc/sssd/sssd.conf
sed -i 's/services = nss, pam/services = nss, pam, sudo/' /etc/sssd/sssd.conf
sed -i '28 a\
sudoers: files sss' /etc/nsswitch.conf
rm -rf /var/lib/sss/db/*
sss_cache -E
systemctl restart sssd
mkdir -p /mnt/nfs
echo "192.168.1.10:/raid/nfs /mnt/nfs nfs intr,soft,_netdev,x-systemd.automount 0 0" >> /etc/fstab
mount -a
mount -v
touch /mnt/nfs
echo server 172.16.1.1 iburst prefer > /etc/chrony.conf
systemctl enable --now chronyd
systemctl restart chronyd
timedatectl
apt-get install yandex-browser -y
ip -c a
```

</details>

<details>
<summary> - BR-RTR </summary>

```bash
ip nat source static tcp 192.168.3.10 8080 172.16.2.5 8080
ip nat source static tcp 192.168.3.10 2026 172.16.2.5 2026
write
```

</details>

<details>
<summary> - BR-SRV </summary>

```bash
echo nameserver 192.168.1.10 > /etc/resolv.conf
rm -rf /etc/samba/smb.conf
echo 192.168.3.10  br-srv.au-team.irpo >> /etc/hosts
samba-tool domain provision --realm=AU-TEAM.IRPO --domain=AU-TEAM --option='dns forwarder=192.168.1.10' --adminpass=P@ssw0rd --dns-backend=SAMBA_INTERNAL --server-role=dc
mv -f /var/lib/samba/private/krb5.conf /etc/krb5.conf
systemctl enable --now samba
samba-tool user add hquser1 P@ssw0rd
samba-tool user add hquser2 P@ssw0rd
samba-tool user add hquser3 P@ssw0rd
samba-tool user add hquser4 P@ssw0rd
samba-tool user add hquser5 P@ssw0rd
samba-tool group add hq
samba-tool group addmembers hq hquser1,hquser2,hquser3,hquser4,hquser5
wget https://raw.githubusercontent.com/sudo-project/sudo/main/docs/schema.ActiveDirectory
dos2unix schema.ActiveDirectory
sed -i 's/DC=X/DC=au-team,DC=irpo/g' schema.ActiveDirectory
head -$(grep -B1 -n '^dn:$' schema.ActiveDirectory | head -1 | grep -oP '\d+') schema.ActiveDirectory > first.ldif
tail +$(grep -B1 -n '^dn:$' schema.ActiveDirectory | head -1 | grep -oP '\d+') schema.ActiveDirectory | sed '/^-/d' > second.ldif
ldbadd -H /var/lib/samba/private/sam.ldb first.ldif --option="dsdb:schema update allowed"=true
ldbmodify -v -H /var/lib/samba/private/sam.ldb second.ldif --option="dsdb:schema update allowed"=true
samba-tool ou add 'ou=sudoers'
cat << EOF > sudoRole-object.ldif
dn: CN=prava_hq,OU=sudoers,DC=au-team,DC=irpo
changetype: add
objectClass: top
objectClass: sudoRole
cn: prava_hq
name: prava_hq
sudoUser: %hq
sudoHost: ALL
sudoCommand: /bin/grep
sudoCommand: /bin/cat
sudoCommand: /usr/bin/id
sudoOption: !authenticate
EOF
ldbadd -H /var/lib/samba/private/sam.ldb sudoRole-object.ldif
echo -e "dn: CN=prava_hq,OU=sudoers,DC=au-team,DC=irpo\nchangetype: modify\nreplace: nTSecurityDescriptor" > ntGen.ldif
ldbsearch  -H /var/lib/samba/private/sam.ldb -s base -b 'CN=prava_hq,OU=sudoers,DC=au-team,DC=irpo' 'nTSecurityDescriptor' | sed -n '/^#/d;s/O:DAG:DAD:AI/O:DAG:DAD:AI\(A\;\;RPLCRC\;\;\;AU\)\(A\;\;RPWPCRCCDCLCLORCWOWDSDDTSW\;\;\;SY\)/;3,$p' | sed ':a;N;$!ba;s/\n\s//g' | sed -e 's/.\{78\}/&\n /g' >> ntGen.ldif
ldbmodify -v -H /var/lib/samba/private/sam.ldb ntGen.ldif
echo server 172.16.2.1 iburst prefer > /etc/chrony.conf
systemctl enable --now chronyd
timedatectl
echo -e "VMs:\n hosts:\n  HQ-SRV:\n    ansible_host: 192.168.1.10\n    ansible_user: sshuser\n    ansible_port: 2026\n  HQ-CLI:\n    ansible_host: 192.168.2.10\n    ansible_user: sshuser\n    ansible_port: 2026\n  HQ-RTR:\n    ansible_host: 192.168.1.1\n    ansible_user: net_admin\n    ansible_password: P@ssw0rd\n    ansible_connection: network_cli\n    ansible_network_os: ios\n  BR-RTR:\n    ansible_host: 192.168.3.1\n    ansible_user: net_admin\n    ansible_password: P@ssw0rd\n    ansible_connection: network_cli\n    ansible_network_os: ios" > /etc/ansible/hosts
sed -i 's/\[defaults\]/\[defaults\]\ninterpreter_python=auto_silent/g' /etc/ansible/ansible.cfg
ssh-keygen -t rsa -f ~/.ssh/id_rsa -N ""
grep -q "192.168.1.10:2026" ~/.ssh/known_hosts 2>/dev/null || ssh-keyscan -p 2026 192.168.1.10 >> ~/.ssh/known_hosts
grep -q "192.168.2.10:2026" ~/.ssh/known_hosts 2>/dev/null || ssh-keyscan -p 2026 192.168.2.10 >> ~/.ssh/known_hosts
sshpass -p "P@ssw0rd" ssh-copy-id -p 2026 sshuser@192.168.1.10
sshpass -p "P@ssw0rd" ssh-copy-id -p 2026 sshuser@192.168.2.10
ansible all -m ping
systemctl enable --now docker
mount -o loop /dev/sr0
docker load < /media/ALTLinux/docker/site_latest.tar
sleep 2
docker load < /media/ALTLinux/docker/mariadb_latest.tar
sleep 2
docker images
echo -e 'services:\n  db:\n    image: mariadb\n    container_name: db\n    environment:\n      MYSQL_ROOT_PASSWORD: Passw0rd\n      MYSQL_DATABASE: testdb\n      MYSQL_USER: test\n      MYSQL_PASSWORD: Passw0rd\n    volumes:\n      - db_data:/var/lib/mysql\n    restart: always\n  testapp:\n    image: site\n    container_name: testapp\n    environment:\n      DB_TYPE: maria\n      DB_HOST: db\n      DB_NAME: testdb\n      DB_USER: test\n      DB_PASS: Passw0rd\n      DB_PORT: 3306\n    ports:\n      - "8080:8000"\n    restart: always\nvolumes:\n  db_data:' > site.yml
docker compose -f site.yml up -d
sleep 5
docker exec -it db mysql -u root -pPassw0rd -e "CREATE DATABASE testdb; CREATE USER 'test'@'%' IDENTIFIED BY 'Passw0rd'; GRANT ALL PRIVILEGES ON testdb.* TO 'test'@'%'; FLUSH PRIVILEGES;"
sleep 2
docker compose restart
ip -c a
```

</details>
