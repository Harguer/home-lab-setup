# home-lab-setup

two networks:
wifi 192.168.25.0/24
ethernet 10.0.0.0/24

configure dns for the k8s on network 10.0.0.0/24
sudo apt-get install bind9 bind9utils dnsutils
vim /etc/bind/named.conf.local

```
zone "home.lan" IN {
        type master;
        file "/etc/bind/db.home.lan";
  };
zone "0.0.10.in-addr.arpa" {
        type master;
        file "/etc/bind/db.rev.0.0.10.in-addr.arpa";
  };
```

vim  /etc/bind/db.rev.0.0.10.in-addr.arpa
```
@ IN SOA matrix.home.lan. hostmaster.home.lan. (
    2017081401 ; serial
    8H ; refresh
    4H ; retry
    4W ; expire
    1D ; minimum
)
          IN NS matrix.home.lan.
1         IN PTR matrix.home.lan.
2         IN PTR k8-pi-master1.home.lan.
3         IN PTR k8-pi-node1.home.lan.
```

since I have two dns (pi-hole and this for K8s, i'm using listen { 10.0.0.1}, but it can be any
vim /etc/bind/named.conf.options 
```
        listen-on { 10.0.0.1; };
        listen-on-v6 { none; };
};
```
service bind9 restart

I used this to setup this DNS: 
https://www.ionos.com/digitalguide/server/configuration/how-to-make-your-raspberry-pi-into-a-dns-server/


configuer it as a router
echo 1 > /proc/sys/net/ipv4/ip_forward
add this rule to /etc/iptables/rules.v4
```
iptables -A POSTROUTING -o wlan0 -j MASQUERADE
iptables-save
```
now in your computer/laptop add static route (I'm using fedora)
```
ip route add 10.0.0.0/24 via 192.168.25.254
```
make it permanent
```
[harguer@Hyrule-HP-Spectre-x360-Convertible-13-ae0xx ~]$ grep 10.0.0 /etc/sysconfig/network-scripts/route-FreshTomato50
10.0.0.0/24 via 192.168.25.254
[harguer@Hyrule-HP-Spectre-x360-Convertible-13-ae0xx ~]$ 
```

now you can connect to ssh to any node on network 10.0.0./24


Optional - Next is configure DHCP, just for 10.0.0.0/24

vim /etc/dhcpcd.conf
```
interface eth0
        #static ip_address=10.0.0.0/24
        static routers=10.0.0.1
        static domain_name_servers=10.0.0.1
```
start/restart the service
sudo systemctl start dhcpcd.service
or
sudo systemctl restart dhcpcd.service
sudo systemctl is-enabled dhcpcd.service
sudo systemctl enabled dhcpcd.service


