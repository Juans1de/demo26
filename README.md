# Module 1 (Demo26)
## №1-2 Базовая настройка 
<details>
<summary> - ISP  </summary>
hostnamectl set-hostname ISP
mkdir /etc/net/ifaces/{ens20,ens21,ens22}
echo -e "BOOTPROTO=static/nCONFIG_IPV4=yes/nDISABLED=no/nTYPE=eth" > /etc/net/ifaces/ens20/options
cp /etc/net/ifaces/ens20/options > /etc/net/ifaces/ens21/options
cp /etc/net/ifaces/ens20/options > /etc/net/ifaces/ens22/options
echo -e "BOOTPROTO=dhcp/nCONFIG_IPV4=yes/nDISABLED=no/nTYPE=eth" > /etc/net/ifaces/ens20/options
echo 172.16.1.1/28 > /etc/net/ifaces/ens21/ipv4address
echo 172.16.2.1/28 > /etc/net/ifaces/ens22/ipv4address
echo nameserver 8.8.8.8 > /etc/resosl.conf
systemctl restart network
ip -c a
</details>
