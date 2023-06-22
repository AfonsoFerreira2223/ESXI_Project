# Router configuration:


You can go to root as most commands here require high-level permissions

```
su -
```

## -----------------------------


### At any point during installations:

It may ask you to insert the CD-ROM in order to download the packages

Go to esxi- edit- then pick the connect box on the datastore iso section (this is where the package is included)


![image](https://github.com/AfonsoFerreira2223/ESXI_Project/assets/114146560/1a186dad-89c3-46d4-a66b-09bd58600d5a)


### Configuring the router:


First of all, make sure you can download packages:

```
apt-cdrom add
```

Now go to


```
nano /etc/apt/sources.list
```

And allow the machine to trust packages from the cd-rom by adding ```[trusted=yes]```


For example:

```
# deb cdrom:[Debian GNU/Linux 11.7.0 _Bullseye_ - Official amd64 BD Binary-1 20230429-11:50]/ bullseye contrib main

deb [trusted=yes] cdrom:[Debian GNU/Linux 11.7.0 _Bullseye_ - Official amd64 BD Binary-4 20230429-11:50]/ bullseye contrib main
deb [trusted=yes] cdrom:[Debian GNU/Linux 11.7.0 _Bullseye_ - Official amd64 BD Binary-3 20230429-11:50]/ bullseye contrib main
deb [trusted=yes] cdrom:[Debian GNU/Linux 11.7.0 _Bullseye_ - Official amd64 BD Binary-2 20230429-11:50]/ bullseye contrib main
deb [trusted=yes] cdrom:[Debian GNU/Linux 11.7.0 _Bullseye_ - Official amd64 BD Binary-1 20230429-11:50]/ bullseye contrib main

deb http://security.debian.org/debian-security bullseye-security main contrib
deb-src http://security.debian.org/debian-security bullseye-security main contrib

# bullseye-updates, to get updates before a point release is made;
# see https://www.debian.org/doc/manuals/debian-reference/ch02.en.html#_updates_and_backports
# A network mirror was not selected during install.  The following entries
# are provided as examples, but you should amend them as appropriate
# for your mirror of choice.
#
# deb http://deb.debian.org/debian/ bullseye-updates main contrib
# deb-src http://deb.debian.org/debian/ bullseye-updates main contrib
```



Now update the machine



```
apt update
apt upgrade

```


## -----------------------------


Configuring the interfaces on the router:

```
nano /etc/network/interfaces
```

Once you are here, copy and paste this (alter the Ip addresses to suit your defined ranges):



<details>
  <summary>Network Interfaces Configuration</summary>

```
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
allow-hotplug ens192
iface ens192 inet static
        address 192.168.15.173/24
        gateway 192.168.15.1
        # dns-* options are implemented by the resolvconf package, if installed
        dns-nameservers 8.8.8.8
        dns-search enta.pt

# Inside network interface
allow-hotplug ens224
iface ens224 inet static
        address 192.168.31.1/24
        # dns-* options are implemented by the resolvconf package, if installed
        dns-search enta.pt

# DMZ network interface
allow-hotplug ens256
iface ens256 inet static
        address 172.31.0.1/24
        # dns-* options are implemented by the resolvconf package, if installed
        dns-search enta.pt
```

</details>


## -----------------------------

```
systemctl restart networking
reboot
ip a
```


Your interfaces should now have the respective IP add assigned to them

![Captura de ecrã de 2023-06-22 10-03-27](https://github.com/AfonsoFerreira2223/ESXI_Project/assets/114146560/a7e9c44d-a21c-444c-8887-fbd67bd0335a)


Make sure to note down the MAC addresses of your interfaces and match them with the correct IP addresses in the ESXi configuration.


Enable IP forwarding by editing the `/etc/sysctl.conf` file. Uncomment the following line:

> net.ipv4.ip_forward=1


Then verify the changes by running the command:
```
sysctl -p
```

Install the DHCP server package by running the following command:
```
apt install -y isc-dhcp-server
```

Edit the DHCP server configuration file `/etc/dhcp/dhcpd.conf`. Change the domain name to your desired value using the following line:
```
option domain-name "your_domain_name";
```

Specify the IP addresses of the domain name servers and interfaces by adding the following line:
```
option domain-name-servers (YOUR IP addresses);
```

Add the HMAC-MD5 key for secure DHCP updates by inserting the following lines:

```
key DHCP_UPDATER {
    algorithm HMAC-MD5.SIG-ALG.REG.INT;
    secret "pRP5FapFoJ95JEL06sv4PQ==";
};
```


Configure DHCP for each subnet by adding the appropriate subnet configuration. Here's an example for `192.168.31.0/24`:
```
subnet 192.168.31.0 netmask 255.255.255.0 {
    range 192.168.31.128 192.168.31.191;
    option routers 192.168.31.1;
    option broadcast-address 192.168.31.255;
}
```




Open the `/etc/default/isc-dhcp-server` file and add the interfaces you want the DHCP server to listen on. Modify the following line:


```
INTERFACESv4="(Include the names of your desired interfaces)"
```

In this case, I am using interfaces ens192, 224 and 256


## Configuring NFtables

```
nano /etc/nftables.conf
```

Configure the nftables appropriately

An example would be:

```
table ip filter {
        chain INPUT {
                type filter hook input priority filter; policy drop;
                iifname "lo" counter packets 14 bytes 1160 accept
                tcp dport { 22, 80, 443, 2222, 3389 } ct state established,related,new counter packets 3194 bytes 277452 accept
                icmp type echo-reply ct state established,related counter packets 6 bytes 504 accept
                ip saddr 172.31.0.0/30 icmp type echo-request ct state established,related,new counter packets 13 bytes 1092 accept
                iifname "ens224" icmp type echo-request ct state established,related,new counter packets 2 bytes 120 accept
                meta l4proto tcp ct state established,related counter packets 4 bytes 786 accept
                meta l4proto udp ct state established,related counter packets 3 bytes 282 accept
                limit rate 30/minute counter packets 613 bytes 129009 log prefix "IPT:INP:Denied: " level debug
                iifname "lo" counter packets 0 bytes 0 accept
                tcp dport { 22, 80, 443, 2222, 3389 } ct state established,related,new counter packets 0 bytes 0 accept
                icmp type echo-reply ct state established,related counter packets 0 bytes 0 accept
                ip saddr 172.31.0.0/30 icmp type echo-request ct state established,related,new counter packets 0 bytes 0 accept
                iifname "ens224" icmp type echo-request ct state established,related,new counter packets 0 bytes 0 accept
                ct state established,related counter packets 0 bytes 0 accept
                ct state established,related counter packets 0 bytes 0 accept
                limit rate 30/minute counter packets 283 bytes 25016 log prefix "IPT:INP:Denied: " level debug
                iifname "ens256" udp sport 68 udp dport 67 counter packets 2 bytes 656 accept
                iifname "ens224" udp sport 68 udp dport 67 counter packets 2 bytes 688 accept
                iifname { "ens224", "ens256" } udp dport 53 ct state established,related,new counter packets 0 bytes 0 accept
                counter packets 1866 bytes 221533 reject
        }

        chain FORWARD {
                type filter hook forward priority filter; policy accept;
                iifname "ens192" oifname "ens256" tcp dport { 80, 443 } ct state established,related,new counter packets 47 bytes 5991 accept
                iifname "ens256" oifname "ens192" tcp sport { 80, 443 } ct state established,related counter packets 35 bytes 24314 accept
                iifname "ens224" oifname "ens256" tcp dport { 20, 21, 22, 80, 443 } ct state established,related,new counter packets 13 bytes 1085 accept
                iifname "ens256" oifname "ens224" tcp sport { 20, 21, 22, 80, 443 } ct state established,related counter packets 11 bytes 3868 accept
                iifname "ens224" oifname "ens256" icmp type echo-request ct state established,related,new counter packets 4 bytes 240 accept
                iifname "ens256" oifname "ens224" icmp type echo-reply ct state established,related counter packets 4 bytes 240 accept
                iifname "ens224" oifname "ens192" tcp dport { 22, 80, 443 } ct state established,related,new counter packets 5816 bytes 612532 accept
                iifname "ens192" oifname "ens224" tcp sport { 22, 80, 443 } ct state established,related counter packets 7907 bytes 11780790 accept
                iifname "ens224" oifname "ens192" icmp type echo-request ct state established,related,new counter packets 2 bytes 120 accept
                iifname "ens192" oifname "ens224" icmp type echo-reply ct state established,related counter packets 2 bytes 120 accept
                iifname "ens256" oifname "ens192" icmp type echo-request ct state established,related,new counter packets 6 bytes 504 accept
                iifname "ens192" oifname "ens256" icmp type echo-reply ct state established,related counter packets 6 bytes 504 accept
                iifname "ens224" oifname "ens192" udp dport { 53, 443 } ct state established,related,new counter packets 110 bytes 14236 accept
                iifname "ens192" oifname "ens224" udp sport { 53, 443 } ct state established,related counter packets 126 bytes 32073 accept
                limit rate 30/minute counter packets 42 bytes 2884 log prefix "IPT:INP:Denied: " level debug
                iifname "ens192" oifname "ens256" tcp dport { 80, 443 } ct state established,related,new counter packets 0 bytes 0 accept
                iifname "ens256" oifname "ens192" tcp sport { 80, 443 } ct state established,related counter packets 0 bytes 0 accept
                iifname "ens224" oifname "ens256" tcp dport { 20, 21, 22, 80, 443 } ct state established,related,new counter packets 0 bytes 0 accept
                iifname "ens256" oifname "ens224" tcp sport { 20, 21, 22, 80, 443 } ct state established,related counter packets 0 bytes 0 accept
                iifname "ens224" oifname "ens256" icmp type echo-request ct state established,related,new counter packets 0 bytes 0 accept
                iifname "ens256" oifname "ens224" icmp type echo-reply ct state established,related counter packets 0 bytes 0 accept
                iifname "ens224" oifname "ens192" tcp dport { 22, 80, 443 } ct state established,related,new counter packets 0 bytes 0 accept
                iifname "ens192" oifname "ens224" tcp sport { 22, 80, 443 } ct state established,related counter packets 0 bytes 0 accept
                iifname "ens224" oifname "ens192" icmp type echo-request ct state established,related,new counter packets 0 bytes 0 accept
                iifname "ens192" oifname "ens224" icmp type echo-reply ct state established,related counter packets 0 bytes 0 accept
                iifname "ens256" oifname "ens192" icmp type echo-request ct state established,related,new counter packets 0 bytes 0 accept
                iifname "ens192" oifname "ens256" icmp type echo-reply ct state established,related counter packets 0 bytes 0 accept
                iifname "ens224" oifname "ens192" udp dport { 53, 443 } ct state established,related,new counter packets 0 bytes 0 accept
                iifname "ens192" oifname "ens224" udp sport { 53, 443 } ct state established,related counter packets 0 bytes 0 accept
                limit rate 30/minute counter packets 0 bytes 0 log prefix "IPT:INP:Denied: " level debug
                counter packets 0 bytes 0 reject
        }

        chain OUTPUT {
                type filter hook output priority filter; policy accept;
                oifname "lo" counter packets 14 bytes 1160 accept
                icmp type echo-request ct state established,related,new counter packets 6 bytes 504 accept
                meta l4proto tcp ct state established,related,new counter packets 2774 bytes 485222 accept
                meta l4proto udp ct state established,related,new counter packets 9 bytes 1624 accept
                limit rate 30/minute counter packets 355 bytes 33853 log prefix "IPT:INP:Denied: " level debug
                oifname "lo" counter packets 0 bytes 0 accept
                icmp type echo-request ct state established,related,new counter packets 0 bytes 0 accept
                ct state established,related,new counter packets 764 bytes 73718 accept
                ct state established,related,new counter packets 0 bytes 0 accept
                limit rate 30/minute counter packets 0 bytes 0 log prefix "IPT:INP:Denied: " level debug
                oifname "ens224" udp sport 68 udp dport 67 counter packets 0 bytes 0 accept
                oifname "ens256" udp sport 68 udp dport 67 counter packets 0 bytes 0 accept
                oifname { "ens224", "ens256" } udp sport 53 ct state established,related counter packets 0 bytes 0 accept
                counter packets 78 bytes 7120 reject

        }
}
table ip nat {
        chain PREROUTING {
                type nat hook prerouting priority dstnat; policy accept;
                iifname "ens192" tcp dport { 80, 443 } counter packets 6 bytes 360 dnat to 172.31.0.2
                iifname "ens192" tcp dport 2222 counter packets 0 bytes 0 dnat to 172.31.0.2:22
                iifname "ens192" tcp dport 3389 counter packets 0 bytes 0 dnat to 192.168.31.2
                iifname "ens192" tcp dport { 80, 443 } counter packets 0 bytes 0 dnat to 172.31.0.2
                iifname "ens192" tcp dport 2222 counter packets 0 bytes 0 dnat to 172.31.0.2:22
                iifname "ens192" tcp dport 3389 counter packets 0 bytes 0 dnat to 192.168.31.2
        }

        chain INPUT {
                type nat hook input priority 100; policy accept;
        }

        chain OUTPUT {
                type nat hook output priority -100; policy accept;
        }

        chain POSTROUTING {
                type nat hook postrouting priority srcnat; policy accept;
                oifname "ens192" counter packets 93 bytes 8000 masquerade
                oifname "ens192" counter packets 0 bytes 0 masquerade
        }
}
```




