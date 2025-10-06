# Module 1 (Demo26)
## №1-2 Базовая настройка 
<details>
<summary> - ISP  </summary>
hostnamectl set-hostname ISP
mkdir /etc/net/ifaces/{ens20,ens21,ens22}
echo -e "BOOTPROTO=static/nCONFIG_IPV4=yes/nDISABLED=no/nTYPE=eth" > /etc/net/ifaces/ens20/options
cp /etc/net/ifaces/ens20
</details>
