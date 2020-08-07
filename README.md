# home-lab-setup
This small project is to deploy Jenkins via helm chart on k8s on Raspberry Pi 4 8G

On a raspnerry pi 2 1G ram - two networks:  
wifi 192.168.25.0/24 - > gateway for the ethernet network and allow K8s cluster reach internet.   
ethernet 10.0.0.0/24 - > K8s network where Jenkins is running  

configure dns for the k8s mostly for network 10.0.0.0/24  
```
sudo apt-get install bind9 bind9utils dnsutils  
```
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
        listen-on { any; };
        //listen-on { 10.0.0.0/24; };
        listen-on-v6 { none; };
};
```
service bind9 restart

I used this to setup this DNS:   
https://www.ionos.com/digitalguide/server/configuration/how-to-make-your-raspberry-pi-into-a-dns-server/


configuere it as a router for the k8s subnet
```
echo 1 > /proc/sys/net/ipv4/ip_forward
```

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
$ grep 10.0.0 /etc/sysconfig/network-scripts/route-FreshTomato50
10.0.0.0/24 via 192.168.25.254
$ 
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
```
sudo systemctl start dhcpcd.service
```
or
```
sudo systemctl restart dhcpcd.service
sudo systemctl is-enabled dhcpcd.service
sudo systemctl enabled dhcpcd.service
```

Install Docker to have local registry:

```
sudo apt-get update;sudo apt-get install docker
sudo curl -fsSL get.docker.com -o get-docker.sh && sh get-docker.sh
sudo usermod -aG docker pi
systemctl status docker.service
docker run -d -p 10.0.0.1:5000:5000 --restart always --name registry registry
```

I have Jenkins configured with ldap auth, here is what i did to have ldap server on my raspberry pi 2

https://www.instructables.com/id/Make-Raspberry-Pi-into-a-LDAP-Server/
you can reconfigure anytime:
```
sudo dpkg-reconfigure slapd
```
then adding group users:
```
cat users.ldif
# Content of the users file

dn: ou=users,ou=People,dc=home,dc=lan
objectClass: organizationalUnit
ou: users
```

run:

```
sudo ldapadd -D "cn=admin,dc=home,dc=lan" -W -H ldapi:/// -f user.ldif
```

add the user jenkins:
```
cat admin-jenkins_user.ldif
# Content of new_users LDIF file

dn: cn=jenkins,ou=users,ou=People,dc=home,dc=lan
objectClass: top
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
cn: jenkins admin
uid: jenkins
uidNumber: 10001
gidNumber: 10001
homeDirectory: /home/jenkins
userPassword: xxxxxxxxx
loginShell: /bin/bash
```

run:
```
sudo ldapadd -D "cn=admin,dc=home,dc=lan" -W -H ldapi:/// -f admin-jenkins_user.ldif
```

and finally, add the group for uniqmembers, it is needed by groups for jenkins:

```
cat devops.ldif
# Content of the users file

dn: ou=DevOps,ou=People,dc=home,dc=lan
cn: DevOps
objectClass: groupOfUniqueNames
uniqueMember: uid=jenkins,ou=users,ou=People,dc=home,dc=lan
```

run:
```
sudo ldapadd -D "cn=admin,dc=home,dc=lan" -W -H ldapi:/// -f devops.ldif 
```

