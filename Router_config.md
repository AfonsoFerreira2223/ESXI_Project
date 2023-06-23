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


## DHCP:


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


This is an example of my configuration


```
subnet 192.168.31.0 netmask 255.255.255.0 {
  range 192.168.31.128 192.168.31.191;
  option routers 192.168.31.1;
  option broadcast-address 192.168.31.255;
}

subnet 172.31.0.0 netmask 255.255.255.0 {
  range 172.31.0.64 172.31.0.95;
  option routers 172.31.0.1;
  option broadcast-address 172.31.0.255;
}

host 172.31.0.64 {
  hardware ethernet 00:50:56:86:13:6f;
  fixed-address 172.31.0.64;
  option routers 172.31.0.1;
  option broadcast-address 172.31.0.255;
}

zone enta.pt {
         primary 127.0.0.1;
         key DHCP_UPDATER;
}

zone 31.172.in-addr.arpa. {
         primary 127.0.0.1;
         key DHCP_UPDATER;
}

zone 31.168.192.in-addr.arpa. {
         primary 127.0.0.1;
         key DHCP_UPDATER;
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

## DNS

```
apt install bind9 bind9-dnsutils bind9-doc bind9-utils bind9utils
```


Now that bind9 is installed, create the following files inside the directory (you can use nano as a text editor)

> enta.pt will be the domain used here

```
cd /etc/var/cache
```

```
nano db.enta.pt
```

```
;
; BIND data file for local loopback interface
;
$TTL	604800
@	IN	SOA	enta.pt. root.enta.pt. (
			      2		; Serial
			 604800		; Refresh
			  86400		; Retry
			2419200		; Expire
			 604800 )	; Negative Cache TTL
;
@	IN	NS	enta.pt.
@       IN      NS      ns.enta.pt.
@       IN      MX      10      smtp.enta.pt.
@       IN      A       192.168.15.180
@       IN      A       172.31.0.1
@       IN      A       192.168.31.1
ns      IN      A       172.31.0.1
smtp    IN      A       172.31.0.1
pop     IN      A       172.31.0.1
imap    IN      A       172.31.0.1
www     IN      A       172.31.0.100
#@	IN	A	127.0.0.1
#@	IN	AAAA	::1
```

```
nano db.192.168.15.enta.pt
```


```
;
; BIND reverse data file for local loopback interface
;
$TTL	604800
@	IN	SOA	enta.pt. root.enta.pt. (
			      1		; Serial
			 604800		; Refresh
			  86400		; Retry
			2419200		; Expire
			 604800 )	; Negative Cache TTL
;
@	IN	NS	enta.pt.
173	IN	PTR	enta.pt.
```

```
nano db.192.168.31.enta.pt
```

```
;
; BIND reverse data file for local loopback interface
;
$TTL	604800
@	IN	SOA	enta.pt. root.enta.pt. (
			      1		; Serial
			 604800		; Refresh
			  86400		; Retry
			2419200		; Expire
			 604800 )	; Negative Cache TTL
;
@	IN	NS	enta.pt.
1	IN	PTR	enta.pt.
```

```
nano db.172.31.enta.pt
```

```
;
; BIND reverse data file for local loopback interface
;
$TTL	604800
@	IN	SOA	enta.pt. root.enta.pt. (
			      1		; Serial
			 604800		; Refresh
			  86400		; Retry
			2419200		; Expire
			 604800 )	; Negative Cache TTL
;
@	IN	NS	enta.pt.
1	IN	PTR	enta.pt.
```

```
nano named.conf.options
```

```
options {
        directory "/var/cache/bind";

        // If there is a firewall between you and nameservers you want
        // to talk to, you may need to fix the firewall to allow multiple
        // ports to talk.  See http://www.kb.cert.org/vuls/id/800113

        // If your ISP provided one or more IP addresses for stable
        // nameservers, you probably want to use them as forwarders.
        // Uncomment the following block, and insert the addresses replacing
        // the all-0's placeholder.

        forwarders {
          8.8.8.8;
        };

        //========================================================================
        // If BIND logs error messages about the root key being expired,
        // you will need to update your keys.  See https://www.isc.org/bind-keys
        //========================================================================
        dnssec-validation no;
        auth-nxdomain no;
        allow-recursion { any; };
        listen-on-v6 { any; };
};
```

```
nano named.conf.local 
```


```
//
// Do any local configuration here
//

// Consider adding the 1918 zones here, if they are not used in your
// organization
//include "/etc/bind/zones.rfc1918";



key DHCP_UPDATER {
         algorithm HMAC-MD5.SIG-ALG.REG.INT;
         secret pRP5FapFoJ95JEL06sv4PQ==;
};

zone "enta.pt" {
        type master;
        file "/var/cache/bind/db.enta.pt";
        allow-update { key DHCP_UPDATER; };
};

zone "31.172.in-addr.arpa" {
        type master;
        file "/var/cache/bind/db.172.31.enta.pt";
        allow-update { key DHCP_UPDATER; };
};

zone "15.168.192.in-addr.arpa" {
        type master;
        file "/var/cache/bind/db.192.168.15.enta.pt";
        allow-update { key DHCP_UPDATER; };
};

zone "31.168.192.in-addr.arpa" {
        type master;
        file "/var/cache/bind/db.192.168.31.enta.pt";
        allow-update { key DHCP_UPDATER; };
};
```


```
nano /etc/resolv.conf
```


```
# Dynamic resolv.conf(5) file for glibc resolver(3) generated by resolvconf(8)
#     DO NOT EDIT THIS FILE BY HAND -- YOUR CHANGES WILL BE OVERWRITTEN
# 127.0.0.53 is the systemd-resolved stub resolver.
# run "resolvectl status" to see details about the actual nameservers.

nameserver 192.168.15.173
search enta.pt
```

## Easy-RSA


```
apt install easy-rsa
```

```
cd /etc/easy-rsa/
cp vars.example vars
nano vars
```

Once you open vars, do the following changes:

> Change set_var EASYRSA_DN "cn_only" to "org"

![Captura de ecrã de 2023-06-22 12-14-47](https://github.com/AfonsoFerreira2223/ESXI_Project/assets/114146560/fbba8a6b-cb70-43ab-a463-aa2bd16213b8)


Set the variables according to your domain


Now to create the certificates run the following commands


```
cd /etc/easy-rsa/
./easyrsa --subject-alt-name="DNS:smtp.enta.pt" gen-req smtp.enta.pt nopass
./easyrsa --subject-alt-name="DNS:pop.enta.pt" gen-req pop.enta.pt nopass
./easyrsa --subject-alt-name="DNS:smtp.enta.pt" sign-req server smtp.enta.pt nopass
./easyrsa --subject-alt-name="DNS:pop.enta.pt" sign-req server pop.enta.pt nopass
cd pki/issued/
cp pop.enta.pt.crt /etc/ssl/crts
cp smtp.enta.pt.crt /etc/ssl/crts
cd ../private/
cp pop.enta.pt.key /etc/ssl/private/
cp smtp.enta.pt.key /etc/ssl/private/
chown -R root:ssl-cert /etc/ssl/private/pop.enta.pt.key
apt-get install exim4-daemon-heavy sasl2-bin dovecot-pop3d dovecot-imapd easy-rsa
cd ..
cd ..
addgroup --system ssl-cert
addgroup --system ssl-cert
chown -R root:ssl-cert /etc/ssl/private
chmod 710 /etc/ssl/private
chmod 440 /etc/ssl/private/*
adduser Debian-exim ssl-cert
adduser Debian-exim sasl
adduser dovecot ssl-cert
dpkg-reconfigure exim4-config
```


Now open these files with a text editor, and copy paste the following configurations (change the domain names values accordingly)



```
nano /etc/default/saslauthd
```

```
# Configuration for networking init script being run during
# the boot sequence

# Set to 'no' to skip interfaces configuration on boot
#CONFIGURE_INTERFACES=yes

# Don't configure these interfaces. Shell wildcards supported/
#EXCLUDE_INTERFACES=

# Set to 'yes' to enable additional verbosity
#VERBOSE=no

# Method to wait for the network to become online,
# for services that depend on a working network:
# - ifup: wait for ifup to have configured an interface.
# - route: wait for a route to a given address to appear.
# - ping/ping6: wait for a host to respond to ping packets.
# - none: don't wait.
#WAIT_ONLINE_METHOD=ifup

# Which interface to wait for.
# If none given, wait for all auto interfaces, or if there are none,
# wait for at least one hotplug interface.
#WAIT_ONLINE_IFACE=

# Which address to wait for for route, ping and ping6 methods.
# If none is given for route, it waits for a default gateway.
#WAIT_ONLINE_ADDRESS=

# Timeout in seconds for waiting for the network to come online.
#WAIT_ONLINE_TIMEOUT=300
```



```
nano /etc/dovecot/conf.d/10-ssl.conf
```



```
##
## SSL settings
##

# SSL/TLS support: yes, no, required. <doc/wiki/SSL.txt>
ssl = yes

# PEM encoded X.509 SSL/TLS certificate and private key. They're opened before
# dropping root privileges, so keep the key file unreadable by anyone but
# root. Included doc/mkcert.sh can be used to easily generate self-signed
# certificate, just make sure to update the domains in dovecot-openssl.cnf
ssl_cert = </etc/ssl/certs/smtp.enta.pt.crt
ssl_key = </etc/ssl/private/smtp.enta.pt.key

# If key file is password protected, give the password here. Alternatively
# give it when starting dovecot with -p parameter. Since this file is often
# world-readable, you may want to place this setting instead to a different
# root owned 0600 file by using ssl_key_password = <path.
#ssl_key_password =

# PEM encoded trusted certificate authority. Set this only if you intend to use
# ssl_verify_client_cert=yes. The file should contain the CA certificate(s)
# followed by the matching CRL(s). (e.g. ssl_ca = </etc/ssl/certs/ca.pem)
#ssl_ca = 

# Require that CRL check succeeds for client certificates.
#ssl_require_crl = yes

# Directory and/or file for trusted SSL CA certificates. These are used only
# when Dovecot needs to act as an SSL client (e.g. imapc backend or
# submission service). The directory is usually /etc/ssl/certs in
# Debian-based systems and the file is /etc/pki/tls/cert.pem in
# RedHat-based systems. Note that ssl_client_ca_file isn't recommended with
# large CA bundles, because it leads to excessive memory usage.
#ssl_client_ca_dir =
ssl_client_ca_dir = /etc/ssl/certs
#ssl_client_ca_file =

# Require valid cert when connecting to a remote server
#ssl_client_require_valid_cert = yes

# Request client to send a certificate. If you also want to require it, set
# auth_ssl_require_client_cert=yes in auth section.
#ssl_verify_client_cert = no

# Which field from certificate to use for username. commonName and
# x500UniqueIdentifier are the usual choices. You'll also need to set
# auth_ssl_username_from_cert=yes.
#ssl_cert_username_field = commonName

# SSL DH parameters
# Generate new params with `openssl dhparam -out /etc/dovecot/dh.pem 4096`
# Or migrate from old ssl-parameters.dat file with the command dovecot
# gives on startup when ssl_dh is unset.
ssl_dh = </usr/share/dovecot/dh.pem

# Minimum SSL protocol version to use. Potentially recognized values are SSLv3,
# TLSv1, TLSv1.1, and TLSv1.2, depending on the OpenSSL version used.
#ssl_min_protocol = TLSv1

# SSL ciphers to use, the default is:
#ssl_cipher_list = ALL:!kRSA:!SRP:!kDHd:!DSS:!aNULL:!eNULL:!EXPORT:!DES:!3DES:!MD5:!PSK:!RC4:!ADH:!LOW@STRENGTH
# To disable non-EC DH, use:
#ssl_cipher_list = ALL:!DH:!kRSA:!SRP:!kDHd:!DSS:!aNULL:!eNULL:!EXPORT:!DES:!3DES:!MD5:!PSK:!RC4:!ADH:!LOW@STRENGTH

# Colon separated list of elliptic curves to use. Empty value (the default)
# means use the defaults from the SSL library. P-521:P-384:P-256 would be an
# example of a valid value.
#ssl_curve_list =

# Prefer the server's order of ciphers over client's.
#ssl_prefer_server_ciphers = no

# SSL crypto device to use, for valid values run "openssl engine"
#ssl_crypto_device =

# SSL extra options. Currently supported options are:
#   compression - Enable compression.
#   no_ticket - Disable SSL session tickets.
#ssl_options =
```




```
nano /etc/dovecot/conf.d/10-auth.conf
```


```

##
## Authentication processes
##

# Disable LOGIN command and all other plaintext authentications unless
# SSL/TLS is used (LOGINDISABLED capability). Note that if the remote IP
# matches the local IP (ie. you're connecting from the same computer), the
# connection is considered secure and plaintext authentication is allowed.
# See also ssl=required setting.
disable_plaintext_auth = yes

# Authentication cache size (e.g. 10M). 0 means it's disabled. Note that
# bsdauth and PAM require cache_key to be set for caching to be used.
#auth_cache_size = 0
# Time to live for cached data. After TTL expires the cached record is no
# longer used, *except* if the main database lookup returns internal failure.
# We also try to handle password changes automatically: If user's previous
# authentication was successful, but this one wasn't, the cache isn't used.
# For now this works only with plaintext authentication.
#auth_cache_ttl = 1 hour
# TTL for negative hits (user not found, password mismatch).
# 0 disables caching them completely.
#auth_cache_negative_ttl = 1 hour

# Space separated list of realms for SASL authentication mechanisms that need
# them. You can leave it empty if you don't want to support multiple realms.
# Many clients simply use the first one listed here, so keep the default realm
# first.
#auth_realms =

# Default realm/domain to use if none was specified. This is used for both
# SASL realms and appending @domain to username in plaintext logins.
#auth_default_realm = 

# List of allowed characters in username. If the user-given username contains
# a character not listed in here, the login automatically fails. This is just
# an extra check to make sure user can't exploit any potential quote escaping
# vulnerabilities with SQL/LDAP databases. If you want to allow all characters,
# set this value to empty.
#auth_username_chars = abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ01234567890.-_@

# Username character translations before it's looked up from databases. The
# value contains series of from -> to characters. For example "#@/@" means
# that '#' and '/' characters are translated to '@'.
#auth_username_translation =

# Username formatting before it's looked up from databases. You can use
# the standard variables here, eg. %Lu would lowercase the username, %n would
# drop away the domain if it was given, or "%n-AT-%d" would change the '@' into
# "-AT-". This translation is done after auth_username_translation changes.
#auth_username_format = %Lu

# If you want to allow master users to log in by specifying the master
# username within the normal username string (ie. not using SASL mechanism's
# support for it), you can specify the separator character here. The format
# is then <username><separator><master username>. UW-IMAP uses "*" as the
# separator, so that could be a good choice.
#auth_master_user_separator =

# Username to use for users logging in with ANONYMOUS SASL mechanism
#auth_anonymous_username = anonymous

# Maximum number of dovecot-auth worker processes. They're used to execute
# blocking passdb and userdb queries (eg. MySQL and PAM). They're
# automatically created and destroyed as needed.
#auth_worker_max_count = 30

# Host name to use in GSSAPI principal names. The default is to use the
# name returned by gethostname(). Use "$ALL" (with quotes) to allow all keytab
# entries.
#auth_gssapi_hostname =

# Kerberos keytab to use for the GSSAPI mechanism. Will use the system
# default (usually /etc/krb5.keytab) if not specified. You may need to change
# the auth service to run as root to be able to read this file.
#auth_krb5_keytab = 

# Do NTLM and GSS-SPNEGO authentication using Samba's winbind daemon and
# ntlm_auth helper. <doc/wiki/Authentication/Mechanisms/Winbind.txt>
#auth_use_winbind = no

# Path for Samba's ntlm_auth helper binary.
#auth_winbind_helper_path = /usr/bin/ntlm_auth

# Time to delay before replying to failed authentications.
#auth_failure_delay = 2 secs

# Require a valid SSL client certificate or the authentication fails.
#auth_ssl_require_client_cert = no

# Take the username from client's SSL certificate, using 
# X509_NAME_get_text_by_NID() which returns the subject's DN's
# CommonName. 
#auth_ssl_username_from_cert = no

# Space separated list of wanted authentication mechanisms:
#   plain login digest-md5 cram-md5 ntlm rpa apop anonymous gssapi otp
#   gss-spnego
# NOTE: See also disable_plaintext_auth setting.
auth_mechanisms = plain

##
## Password and user databases
##

#
# Password database is used to verify user's password (and nothing more).
# You can have multiple passdbs and userdbs. This is useful if you want to
# allow both system users (/etc/passwd) and virtual users to login without
# duplicating the system users into virtual database.
#
# <doc/wiki/PasswordDatabase.txt>
#
# User database specifies where mails are located and what user/group IDs
# own them. For single-UID configuration use "static" userdb.
#
# <doc/wiki/UserDatabase.txt>

#!include auth-deny.conf.ext
#!include auth-master.conf.ext

!include auth-system.conf.ext
#!include auth-sql.conf.ext
#!include auth-ldap.conf.ext
#!include auth-passwdfile.conf.ext
#!include auth-checkpassword.conf.ext
#!include auth-static.conf.ext

```


```
nano /etc/dovecot/conf.d/10-mail.conf
```


```

##
## Mailbox locations and namespaces
##

# Location for users' mailboxes. The default is empty, which means that Dovecot
# tries to find the mailboxes automatically. This won't work if the user
# doesn't yet have any mail, so you should explicitly tell Dovecot the full
# location.
#
# If you're using mbox, giving a path to the INBOX file (eg. /var/mail/%u)
# isn't enough. You'll also need to tell Dovecot where the other mailboxes are
# kept. This is called the "root mail directory", and it must be the first
# path given in the mail_location setting.
#
# There are a few special variables you can use, eg.:
#
#   %u - username
#   %n - user part in user@domain, same as %u if there's no domain
#   %d - domain part in user@domain, empty if there's no domain
#   %h - home directory
#
# See doc/wiki/Variables.txt for full list. Some examples:
#
   mail_location = maildir:~/Maildir
#   mail_location = mbox:~/mail:INBOX=/var/mail/%u
#   mail_location = mbox:/var/mail/%d/%1n/%n:INDEX=/var/indexes/%d/%1n/%n
#
# <doc/wiki/MailLocation.txt>
#
#mail_location = mbox:~/mail:INBOX=/var/mail/%u

# If you need to set multiple mailbox locations or want to change default
# namespace settings, you can do it by defining namespace sections.
#
# You can have private, shared and public namespaces. Private namespaces
# are for user's personal mails. Shared namespaces are for accessing other
# users' mailboxes that have been shared. Public namespaces are for shared
# mailboxes that are managed by sysadmin. If you create any shared or public
# namespaces you'll typically want to enable ACL plugin also, otherwise all
# users can access all the shared mailboxes, assuming they have permissions
# on filesystem level to do so.
namespace inbox {
  # Namespace type: private, shared or public
  #type = private

  # Hierarchy separator to use. You should use the same separator for all
  # namespaces or some clients get confused. '/' is usually a good one.
  # The default however depends on the underlying mail storage format.
  #separator = 

  # Prefix required to access this namespace. This needs to be different for
  # all namespaces. For example "Public/".
  #prefix = 

  # Physical location of the mailbox. This is in same format as
  # mail_location, which is also the default for it.
  #location =

  # There can be only one INBOX, and this setting defines which namespace
  # has it.
  inbox = yes

  # If namespace is hidden, it's not advertised to clients via NAMESPACE
  # extension. You'll most likely also want to set list=no. This is mostly
  # useful when converting from another server with different namespaces which
  # you want to deprecate but still keep working. For example you can create
  # hidden namespaces with prefixes "~/mail/", "~%u/mail/" and "mail/".
  #hidden = no

  # Show the mailboxes under this namespace with LIST command. This makes the
  # namespace visible for clients that don't support NAMESPACE extension.
  # "children" value lists child mailboxes, but hides the namespace prefix.
  #list = yes

  # Namespace handles its own subscriptions. If set to "no", the parent
  # namespace handles them (empty prefix should always have this as "yes")
  #subscriptions = yes

  # See 15-mailboxes.conf for definitions of special mailboxes.
}

# Example shared namespace configuration
#namespace {
  #type = shared
  #separator = /

  # Mailboxes are visible under "shared/user@domain/"
  # %%n, %%d and %%u are expanded to the destination user.
  #prefix = shared/%%u/

  # Mail location for other users' mailboxes. Note that %variables and ~/
  # expands to the logged in user's data. %%n, %%d, %%u and %%h expand to the
  # destination user's data.
  #location = maildir:%%h/Maildir:INDEX=~/Maildir/shared/%%u

  # Use the default namespace for saving subscriptions.
  #subscriptions = no

  # List the shared/ namespace only if there are visible shared mailboxes.
  #list = children
#}
# Should shared INBOX be visible as "shared/user" or "shared/user/INBOX"?
#mail_shared_explicit_inbox = no

# System user and group used to access mails. If you use multiple, userdb
# can override these by returning uid or gid fields. You can use either numbers
# or names. <doc/wiki/UserIds.txt>
#mail_uid =
#mail_gid =

# Group to enable temporarily for privileged operations. Currently this is
# used only with INBOX when either its initial creation or dotlocking fails.
# Typically this is set to "mail" to give access to /var/mail.
mail_privileged_group = mail

# Grant access to these supplementary groups for mail processes. Typically
# these are used to set up access to shared mailboxes. Note that it may be
# dangerous to set these if users can create symlinks (e.g. if "mail" group is
# set here, ln -s /var/mail ~/mail/var could allow a user to delete others'
# mailboxes, or ln -s /secret/shared/box ~/mail/mybox would allow reading it).
#mail_access_groups =

# Allow full filesystem access to clients. There's no access checks other than
# what the operating system does for the active UID/GID. It works with both
# maildir and mboxes, allowing you to prefix mailboxes names with eg. /path/
# or ~user/.
#mail_full_filesystem_access = no

# Dictionary for key=value mailbox attributes. This is used for example by
# URLAUTH and METADATA extensions.
#mail_attribute_dict =

# A comment or note that is associated with the server. This value is
# accessible for authenticated users through the IMAP METADATA server
# entry "/shared/comment". 
#mail_server_comment = ""

# Indicates a method for contacting the server administrator. According to
# RFC 5464, this value MUST be a URI (e.g., a mailto: or tel: URL), but that
# is currently not enforced. Use for example mailto:admin@example.com. This
# value is accessible for authenticated users through the IMAP METADATA server
# entry "/shared/admin".
#mail_server_admin = 

##
## Mail processes
##

# Don't use mmap() at all. This is required if you store indexes to shared
# filesystems (NFS or clustered filesystem).
#mmap_disable = no

# Rely on O_EXCL to work when creating dotlock files. NFS supports O_EXCL
# since version 3, so this should be safe to use nowadays by default.
#dotlock_use_excl = yes

# When to use fsync() or fdatasync() calls:
#   optimized (default): Whenever necessary to avoid losing important data
#   always: Useful with e.g. NFS when write()s are delayed
#   never: Never use it (best performance, but crashes can lose data)
#mail_fsync = optimized

# Locking method for index files. Alternatives are fcntl, flock and dotlock.
# Dotlocking uses some tricks which may create more disk I/O than other locking
# methods. NFS users: flock doesn't work, remember to change mmap_disable.
#lock_method = fcntl

# Directory where mails can be temporarily stored. Usually it's used only for
# mails larger than >= 128 kB. It's used by various parts of Dovecot, for
# example LDA/LMTP while delivering large mails or zlib plugin for keeping
# uncompressed mails.
#mail_temp_dir = /tmp

# Valid UID range for users, defaults to 500 and above. This is mostly
# to make sure that users can't log in as daemons or other system users.
# Note that denying root logins is hardcoded to dovecot binary and can't
# be done even if first_valid_uid is set to 0.
#first_valid_uid = 500
#last_valid_uid = 0

# Valid GID range for users, defaults to non-root/wheel. Users having
# non-valid GID as primary group ID aren't allowed to log in. If user
# belongs to supplementary groups with non-valid GIDs, those groups are
# not set.
#first_valid_gid = 1
#last_valid_gid = 0

# Maximum allowed length for mail keyword name. It's only forced when trying
# to create new keywords.
#mail_max_keyword_length = 50

# ':' separated list of directories under which chrooting is allowed for mail
# processes (ie. /var/mail will allow chrooting to /var/mail/foo/bar too).
# This setting doesn't affect login_chroot, mail_chroot or auth chroot
# settings. If this setting is empty, "/./" in home dirs are ignored.
# WARNING: Never add directories here which local users can modify, that
# may lead to root exploit. Usually this should be done only if you don't
# allow shell access for users. <doc/wiki/Chrooting.txt>
#valid_chroot_dirs = 

# Default chroot directory for mail processes. This can be overridden for
# specific users in user database by giving /./ in user's home directory
# (eg. /home/./user chroots into /home). Note that usually there is no real
# need to do chrooting, Dovecot doesn't allow users to access files outside
# their mail directory anyway. If your home directories are prefixed with
# the chroot directory, append "/." to mail_chroot. <doc/wiki/Chrooting.txt>
#mail_chroot = 

# UNIX socket path to master authentication server to find users.
# This is used by imap (for shared users) and lda.
#auth_socket_path = /var/run/dovecot/auth-userdb

# Directory where to look up mail plugins.
#mail_plugin_dir = /usr/lib/dovecot/modules

# Space separated list of plugins to load for all services. Plugins specific to
# IMAP, LDA, etc. are added to this list in their own .conf files.
#mail_plugins = 

##
## Mailbox handling optimizations
##

# Mailbox list indexes can be used to optimize IMAP STATUS commands. They are
# also required for IMAP NOTIFY extension to be enabled.
#mailbox_list_index = yes

# Trust mailbox list index to be up-to-date. This reduces disk I/O at the cost
# of potentially returning out-of-date results after e.g. server crashes.
# The results will be automatically fixed once the folders are opened.
#mailbox_list_index_very_dirty_syncs = yes

# Should INBOX be kept up-to-date in the mailbox list index? By default it's
# not, because most of the mailbox accesses will open INBOX anyway.
#mailbox_list_index_include_inbox = no

# The minimum number of mails in a mailbox before updates are done to cache
# file. This allows optimizing Dovecot's behavior to do less disk writes at
# the cost of more disk reads.
#mail_cache_min_mail_count = 0

# When IDLE command is running, mailbox is checked once in a while to see if
# there are any new mails or other changes. This setting defines the minimum
# time to wait between those checks. Dovecot can also use inotify and
# kqueue to find out immediately when changes occur.
#mailbox_idle_check_interval = 30 secs

# Save mails with CR+LF instead of plain LF. This makes sending those mails
# take less CPU, especially with sendfile() syscall with Linux and FreeBSD.
# But it also creates a bit more disk I/O which may just make it slower.
# Also note that if other software reads the mboxes/maildirs, they may handle
# the extra CRs wrong and cause problems.
#mail_save_crlf = no

# Max number of mails to keep open and prefetch to memory. This only works with
# some mailbox formats and/or operating systems.
#mail_prefetch_count = 0

# How often to scan for stale temporary files and delete them (0 = never).
# These should exist only after Dovecot dies in the middle of saving mails.
#mail_temp_scan_interval = 1w

# How many slow mail accesses sorting can perform before it returns failure.
# With IMAP the reply is: NO [LIMIT] Requested sort would have taken too long.
# The untagged SORT reply is still returned, but it's likely not correct.
#mail_sort_max_read_count = 0

protocol !indexer-worker {
  # If folder vsize calculation requires opening more than this many mails from
  # disk (i.e. mail sizes aren't in cache already), return failure and finish
  # the calculation via indexer process. Disabled by default. This setting must
  # be 0 for indexer-worker processes.
  #mail_vsize_bg_after_count = 0
}

##
## Maildir-specific settings
##

# By default LIST command returns all entries in maildir beginning with a dot.
# Enabling this option makes Dovecot return only entries which are directories.
# This is done by stat()ing each entry, so it causes more disk I/O.
# (For systems setting struct dirent->d_type, this check is free and it's
# done always regardless of this setting)
#maildir_stat_dirs = no

# When copying a message, do it with hard links whenever possible. This makes
# the performance much better, and it's unlikely to have any side effects.
#maildir_copy_with_hardlinks = yes

# Assume Dovecot is the only MUA accessing Maildir: Scan cur/ directory only
# when its mtime changes unexpectedly or when we can't find the mail otherwise.
#maildir_very_dirty_syncs = no

# If enabled, Dovecot doesn't use the S=<size> in the Maildir filenames for
# getting the mail's physical size, except when recalculating Maildir++ quota.
# This can be useful in systems where a lot of the Maildir filenames have a
# broken size. The performance hit for enabling this is very small.
#maildir_broken_filename_sizes = no

# Always move mails from new/ directory to cur/, even when the \Recent flags
# aren't being reset.
#maildir_empty_new = no

##
## mbox-specific settings
##

# Which locking methods to use for locking mbox. There are four available:
#  dotlock: Create <mailbox>.lock file. This is the oldest and most NFS-safe
#           solution. If you want to use /var/mail/ like directory, the users
#           will need write access to that directory.
#  dotlock_try: Same as dotlock, but if it fails because of permissions or
#               because there isn't enough disk space, just skip it.
#  fcntl  : Use this if possible. Works with NFS too if lockd is used.
#  flock  : May not exist in all systems. Doesn't work with NFS.
#  lockf  : May not exist in all systems. Doesn't work with NFS.
#
# You can use multiple locking methods; if you do the order they're declared
# in is important to avoid deadlocks if other MTAs/MUAs are using multiple
# locking methods as well. Some operating systems don't allow using some of
# them simultaneously.
#
# The Debian value for mbox_write_locks differs from upstream Dovecot. It is
# changed to be compliant with Debian Policy (section 11.6) for NFS safety.
#       Dovecot: mbox_write_locks = dotlock fcntl
#       Debian:  mbox_write_locks = fcntl dotlock
#
#mbox_read_locks = fcntl
#mbox_write_locks = fcntl dotlock

# Maximum time to wait for lock (all of them) before aborting.
#mbox_lock_timeout = 5 mins

# If dotlock exists but the mailbox isn't modified in any way, override the
# lock file after this much time.
#mbox_dotlock_change_timeout = 2 mins

# When mbox changes unexpectedly we have to fully read it to find out what
# changed. If the mbox is large this can take a long time. Since the change
# is usually just a newly appended mail, it'd be faster to simply read the
# new mails. If this setting is enabled, Dovecot does this but still safely
# fallbacks to re-reading the whole mbox file whenever something in mbox isn't
# how it's expected to be. The only real downside to this setting is that if
# some other MUA changes message flags, Dovecot doesn't notice it immediately.
# Note that a full sync is done with SELECT, EXAMINE, EXPUNGE and CHECK 
# commands.
#mbox_dirty_syncs = yes

# Like mbox_dirty_syncs, but don't do full syncs even with SELECT, EXAMINE,
# EXPUNGE or CHECK commands. If this is set, mbox_dirty_syncs is ignored.
#mbox_very_dirty_syncs = no

# Delay writing mbox headers until doing a full write sync (EXPUNGE and CHECK
# commands and when closing the mailbox). This is especially useful for POP3
# where clients often delete all mails. The downside is that our changes
# aren't immediately visible to other MUAs.
#mbox_lazy_writes = yes

# If mbox size is smaller than this (e.g. 100k), don't write index files.
# If an index file already exists it's still read, just not updated.
#mbox_min_index_size = 0

# Mail header selection algorithm to use for MD5 POP3 UIDLs when
# pop3_uidl_format=%m. For backwards compatibility we use apop3d inspired
# algorithm, but it fails if the first Received: header isn't unique in all
# mails. An alternative algorithm is "all" that selects all headers.
#mbox_md5 = apop3d

##
## mdbox-specific settings
##

# Maximum dbox file size until it's rotated.
#mdbox_rotate_size = 10M

# Maximum dbox file age until it's rotated. Typically in days. Day begins
# from midnight, so 1d = today, 2d = yesterday, etc. 0 = check disabled.
#mdbox_rotate_interval = 0

# When creating new mdbox files, immediately preallocate their size to
# mdbox_rotate_size. This setting currently works only in Linux with some
# filesystems (ext4, xfs).
#mdbox_preallocate_space = no

##
## Mail attachments
##

# sdbox and mdbox support saving mail attachments to external files, which
# also allows single instance storage for them. Other backends don't support
# this for now.

# Directory root where to store mail attachments. Disabled, if empty.
#mail_attachment_dir =

# Attachments smaller than this aren't saved externally. It's also possible to
# write a plugin to disable saving specific attachments externally.
#mail_attachment_min_size = 128k

# Filesystem backend to use for saving attachments:
#  posix : No SiS done by Dovecot (but this might help FS's own deduplication)
#  sis posix : SiS with immediate byte-by-byte comparison during saving
#  sis-queue posix : SiS with delayed comparison and deduplication
#mail_attachment_fs = sis posix

# Hash format to use in attachment filenames. You can add any text and
# variables: %{md4}, %{md5}, %{sha1}, %{sha256}, %{sha512}, %{size}.
# Variables can be truncated, e.g. %{sha256:80} returns only first 80 bits
#mail_attachment_hash = %{sha1}

# Settings to control adding $HasAttachment or $HasNoAttachment keywords.
# By default, all MIME parts with Content-Disposition=attachment, or inlines
# with filename parameter are consired attachments.
#   add-flags - Add the keywords when saving new mails or when fetching can
#      do it efficiently.
#   content-type=type or !type - Include/exclude content type. Excluding will
#     never consider the matched MIME part as attachment. Including will only
#     negate an exclusion (e.g. content-type=!foo/* content-type=foo/bar).
#   exclude-inlined - Exclude any Content-Disposition=inline MIME part.
#mail_attachment_detection_options =

```



```
nano /etc/exim4/exim4.conf.template
```




<details>
  <summary>(Warning this file is REALLY big)</summary>
  
```

#####################################################
### main/01_exim4-config_listmacrosdefs
#####################################################
######################################################################
#      Runtime configuration file for Exim 4 (Debian Packaging)      #
######################################################################

######################################################################
# /etc/exim4/exim4.conf.template is only used with the non-split
#   configuration scheme.
# /etc/exim4/conf.d/main/01_exim4-config_listmacrosdefs is only used
#   with the split configuration scheme.
# If you find this comment anywhere else, somebody copied it there.
# Documentation about the Debian exim4 configuration scheme can be
# found in /usr/share/doc/exim4-base/README.Debian.gz.
######################################################################

######################################################################
#                    MAIN CONFIGURATION SETTINGS                     #
######################################################################


MAIN_TLS_ENABLE=yes
MAIN_TLS_CERTIFICATE=/etc/ssl/certs/smtp.enta.pt.crt
MAIN_TLS_PRIVATEKEY=/etc/ssl/private/smtp.enta.pt.key


# Just for reference and scripts. 
# On Debian systems, the main binary is installed as exim4 to avoid
# conflicts with the exim 3 packages.
exim_path = /usr/sbin/exim4

# Macro defining the main configuration directory.
# We do not use absolute paths.
.ifndef CONFDIR
CONFDIR = /etc/exim4
.endif

# debconf-driven macro definitions get inserted after this line
UPEX4CmacrosUPEX4C = 1

# Create domain and host lists for relay control
# '@' refers to 'the name of the local host'

# List of domains considered local for exim. Domains not listed here
# need to be deliverable remotely.
domainlist local_domains = MAIN_LOCAL_DOMAINS

# List of recipient domains to relay _to_. Use this list if you're -
# for example - fallback MX or mail gateway for domains.
domainlist relay_to_domains = MAIN_RELAY_TO_DOMAINS

# List of sender networks (IP addresses) to _unconditionally_ relay
# _for_. If you intend to be SMTP AUTH server, you do not need to enter
# anything here.
hostlist relay_from_hosts = MAIN_RELAY_NETS


# Decide which domain to use to add to all unqualified addresses.
# If MAIN_PRIMARY_HOSTNAME_AS_QUALIFY_DOMAIN is defined, the primary
# hostname is used. If not, but MAIN_QUALIFY_DOMAIN is set, the value
# of MAIN_QUALIFY_DOMAIN is used. If both macros are not defined,
# the first line of /etc/mailname is used.
.ifndef MAIN_PRIMARY_HOSTNAME_AS_QUALIFY_DOMAIN
.ifndef MAIN_QUALIFY_DOMAIN
qualify_domain = ETC_MAILNAME
.else
qualify_domain = MAIN_QUALIFY_DOMAIN
.endif
.endif

# listen on all all interfaces?
.ifdef MAIN_LOCAL_INTERFACES
local_interfaces = MAIN_LOCAL_INTERFACES
.endif

.ifndef LOCAL_DELIVERY
# The default transport, set in /etc/exim4/update-exim4.conf.conf,
# defaulting to mail_spool. See CONFDIR/conf.d/transport/ for possibilities
LOCAL_DELIVERY=mail_spool
.endif

# The gecos field in /etc/passwd holds not only the name. see passwd(5).
gecos_pattern = ^([^,:]*)
gecos_name = $1

# always log tls_peerdn as we use TLS for outgoing connects by default
.ifndef MAIN_LOG_SELECTOR
MAIN_LOG_SELECTOR = +smtp_protocol_error +smtp_syntax_error +tls_certificate_verified +tls_peerdn
.endif
#####################################################
### end main/01_exim4-config_listmacrosdefs
#####################################################
#####################################################
### main/02_exim4-config_options
#####################################################

### main/02_exim4-config_options
#################################


# Defines the access control list that is run when an
# SMTP MAIL command is received.
#
.ifndef MAIN_ACL_CHECK_MAIL
MAIN_ACL_CHECK_MAIL = acl_check_mail
.endif
acl_smtp_mail = MAIN_ACL_CHECK_MAIL


# Defines the access control list that is run when an
# SMTP RCPT command is received.
#
.ifndef MAIN_ACL_CHECK_RCPT
MAIN_ACL_CHECK_RCPT = acl_check_rcpt
.endif
acl_smtp_rcpt = MAIN_ACL_CHECK_RCPT


# Defines the access control list that is run when an
# SMTP DATA command is received.
#
.ifndef MAIN_ACL_CHECK_DATA
MAIN_ACL_CHECK_DATA = acl_check_data
.endif
acl_smtp_data = MAIN_ACL_CHECK_DATA


# Message size limit. The default (used when MESSAGE_SIZE_LIMIT
# is unset) is 50 MB
.ifdef MESSAGE_SIZE_LIMIT
message_size_limit = MESSAGE_SIZE_LIMIT
.endif


# If you are running exim4-daemon-heavy or a custom version of Exim that
# was compiled with the content-scanning extension, you can cause incoming
# messages to be automatically scanned for viruses. You have to modify the
# configuration in two places to set this up. The first of them is here,
# where you define the interface to your scanner. This example is typical
# for ClamAV; see the manual for details of what to set for other virus
# scanners. The second modification is in the acl_check_data access
# control list.

# av_scanner = clamd:/run/clamav/clamd.ctl


# For spam scanning, there is a similar option that defines the interface to
# SpamAssassin. You do not need to set this if you are using the default, which
# is shown in this commented example. As for virus scanning, you must also
# modify the acl_check_data access control list to enable spam scanning.

# spamd_address = 127.0.0.1 783

# Domain used to qualify unqualified recipient addresses
# If this option is not set, the qualify_domain value is used.
# qualify_recipient = <value of qualify_domain>


# Allow Exim to recognize addresses of the form "user@[10.11.12.13]",
# where the domain part is a "domain literal" (an IP address) instead
# of a named domain. The RFCs require this facility, but it is disabled
# in the default config since it is rarely used and frequently abused.
# Domain literal support also needs a special router, which is automatically
# enabled if you use the enable macro MAIN_ALLOW_DOMAIN_LITERALS.
# Additionally, you might want to make your local IP addresses (or @[])
# local domains.
.ifdef MAIN_ALLOW_DOMAIN_LITERALS
allow_domain_literals
.endif


# Do a reverse DNS lookup on all incoming IP calls, in order to get the
# true host name. If you feel this is too expensive, the networks for
# which a lookup is done can be listed here.
.ifndef DC_minimaldns
.ifndef MAIN_HOST_LOOKUP
MAIN_HOST_LOOKUP = *
.endif
host_lookup = MAIN_HOST_LOOKUP
.endif

# The setting below causes Exim to try to initialize the system resolver
# library with DNSSEC support.  It has no effect if your library lacks
# DNSSEC support.
dns_dnssec_ok = 1

# In a minimaldns setup, update-exim4.conf guesses the hostname and
# dumps it here to avoid DNS lookups being done at Exim run time.
.ifdef MAIN_HARDCODE_PRIMARY_HOSTNAME
primary_hostname = MAIN_HARDCODE_PRIMARY_HOSTNAME
.endif

# The settings below cause Exim to make RFC 1413 (ident) callbacks
# for all incoming SMTP calls. You can limit the hosts to which these
# calls are made, and/or change the timeout that is used. If you set
# the timeout to zero, all RFC 1413 calls are disabled. RFC 1413 calls
# are cheap and can provide useful information for tracing problem
# messages, but some hosts and firewalls have problems with them.
# This can result in a timeout instead of an immediate refused
# connection, leading to delays on starting up SMTP sessions.
# (The default was reduced from 30s to 5s for release 4.61. and to
# disabled for release 4.86)
#
#rfc1413_hosts = *
#rfc1413_query_timeout = 5s


# Enable an efficiency feature.  We advertise the feature; clients
# may request to use it.  For multi-recipient mails we then can
# reject or accept per-user after the message is received.
# This supports recipient-dependent content filtering; without it
# you have to temp-reject any recipients after the first that have
# incompatible filtering, and do the filtering in the data ACL.
# Even with this enabled, you must support the old style for peers
# not flagging support for PRDR (visible via $prdr_requested).
prdr_enable = true

# When using an external relay tester (such as rt.njabl.org and/or the
# currently defunct relay-test.mail-abuse.org, the test may be aborted
# since exim complains about "too many nonmail commands". If you want
# the test to complete, add the host from where "your" relay tester
# connects from to the MAIN_SMTP_ACCEPT_MAX_NOMAIL_HOSTS macro.
# Please note that a non-empty setting may cause extra DNS lookups to
# happen, which is the reason why this option is commented out in the
# default settings.
# MAIN_SMTP_ACCEPT_MAX_NOMAIL_HOSTS = !rt.njabl.org
.ifdef MAIN_SMTP_ACCEPT_MAX_NOMAIL_HOSTS
smtp_accept_max_nonmail_hosts = MAIN_SMTP_ACCEPT_MAX_NOMAIL_HOSTS
.endif

# By default, exim forces a Sender: header containing the local
# account name at the local host name in all locally submitted messages
# that don't have the local account name at the local host name in the
# From: header, deletes any Sender: header present in the submitted
# message and forces the envelope sender of all locally submitted
# messages to the local account name at the local host name.
# The following settings allow local users to specify their own envelope sender
# in a locally submitted message. Sender: headers existing in a locally
# submitted message are not removed, and no automatic Sender: headers
# are added. These settings are fine for most hosts.
# If you run exim on a classical multi-user systems where all users
# have local mailboxes that can be reached via SMTP from the Internet
# with the local FQDN as the domain part of the address, you might want
# to disable the following three lines for traceability reasons.
.ifndef MAIN_FORCE_SENDER
local_from_check = false
local_sender_retain = true
untrusted_set_sender = *
.endif


# By default, Exim expects all envelope addresses to be fully qualified, that
# is, they must contain both a local part and a domain. Configure exim
# to accept unqualified addresses from certain hosts. When this is done,
# unqualified addresses are qualified using the settings of qualify_domain
# and/or qualify_recipient (see above).
# sender_unqualified_hosts = <unset>
# recipient_unqualified_hosts = <unset>


# Configure Exim to support the "percent hack" for certain domains.
# The "percent hack" is the feature by which mail addressed to x%y@z
# (where z is one of the domains listed) is locally rerouted to x@y
# and sent on. If z is not one of the "percent hack" domains, x%y is
# treated as an ordinary local part. The percent hack is rarely needed
# nowadays but frequently abused. You should not enable it unless you
# are sure that you really need it.
# percent_hack_domains = <unset>


# Bounce handling
.ifndef MAIN_IGNORE_BOUNCE_ERRORS_AFTER
MAIN_IGNORE_BOUNCE_ERRORS_AFTER = 2d
.endif
ignore_bounce_errors_after = MAIN_IGNORE_BOUNCE_ERRORS_AFTER

.ifndef MAIN_TIMEOUT_FROZEN_AFTER
MAIN_TIMEOUT_FROZEN_AFTER = 7d
.endif
timeout_frozen_after = MAIN_TIMEOUT_FROZEN_AFTER

.ifndef MAIN_FREEZE_TELL
MAIN_FREEZE_TELL = postmaster
.endif
freeze_tell = MAIN_FREEZE_TELL


# Define spool directory
.ifndef SPOOLDIR
SPOOLDIR = /var/spool/exim4
.endif
spool_directory = SPOOLDIR


# trusted users can set envelope-from to arbitrary values
.ifndef MAIN_TRUSTED_USERS
MAIN_TRUSTED_USERS = uucp
.endif
trusted_users = MAIN_TRUSTED_USERS
.ifdef MAIN_TRUSTED_GROUPS
trusted_groups = MAIN_TRUSTED_GROUPS
.endif


# users in admin group can do many other things
# admin_groups = <unset>


# SMTP Banner. The example includes the Debian version in the SMTP dialog
# MAIN_SMTP_BANNER = "${primary_hostname} ESMTP Exim ${version_number} (Debian package MAIN_PACKAGE_VERSION) ${tod_full}"
# smtp_banner = $smtp_active_hostname ESMTP Exim $version_number $tod_full

.ifdef MAIN_KEEP_ENVIRONMENT
keep_environment = MAIN_KEEP_ENVIRONMENT
.else
# set option to empty value to avoid warning.
keep_environment =
.endif
.ifdef MAIN_ADD_ENVIRONMENT
add_environment = MAIN_ADD_ENVIRONMENT
.endif

.ifdef _OPT_MAIN_SMTPUTF8_ADVERTISE_HOSTS
.ifndef MAIN_SMTPUTF8_ADVERTISE_HOSTS
MAIN_SMTPUTF8_ADVERTISE_HOSTS =
.endif
smtputf8_advertise_hosts = MAIN_SMTPUTF8_ADVERTISE_HOSTS
.endif
#####################################################
### end main/02_exim4-config_options
#####################################################
#####################################################
### main/03_exim4-config_tlsoptions
#####################################################

### main/03_exim4-config_tlsoptions
#################################

# TLS/SSL configuration for exim as an SMTP server.
# See /usr/share/doc/exim4-base/README.Debian.gz for explanations.

.ifdef MAIN_TLS_ENABLE
# Defines what hosts to 'advertise' STARTTLS functionality to. The
# default, *, will advertise to all hosts that connect with EHLO.
.ifndef MAIN_TLS_ADVERTISE_HOSTS
MAIN_TLS_ADVERTISE_HOSTS = *
.endif
tls_advertise_hosts = MAIN_TLS_ADVERTISE_HOSTS


# Full paths to Certificate and Private Key. The Private Key file
# must be kept 'secret' and should be owned by root.Debian-exim mode
# 640 (-rw-r-----). exim-gencert takes care of these prerequisites.
# Normally, exim4 looks for certificate and key in different files:
#   MAIN_TLS_CERTIFICATE - path to certificate file,
#                          CONFDIR/exim.crt if unset
#   MAIN_TLS_PRIVATEKEY  - path to private key file
#                          CONFDIR/exim.key if unset
# You can also configure exim to look for certificate and key in the
# same file, set MAIN_TLS_CERTKEY to that file to enable. This takes
# precedence over all other settings regarding certificate and key file.
.ifdef MAIN_TLS_CERTKEY
tls_certificate = MAIN_TLS_CERTKEY
.else
.ifndef MAIN_TLS_CERTIFICATE
MAIN_TLS_CERTIFICATE = CONFDIR/exim.crt
.endif
tls_certificate = MAIN_TLS_CERTIFICATE

.ifndef MAIN_TLS_PRIVATEKEY
MAIN_TLS_PRIVATEKEY = CONFDIR/exim.key
.endif
tls_privatekey = MAIN_TLS_PRIVATEKEY
.endif

# Pointer to the CA Certificates against which client certificates are
# checked. This is controlled by the `tls_verify_hosts' and
# `tls_try_verify_hosts' lists below.
# If you want to check server certificates, you need to add an
# tls_verify_certificates statement to the smtp transport.
# /etc/ssl/certs/ca-certificates.crt is generated by
# the "ca-certificates" package's update-ca-certificates(8) command.
.ifndef MAIN_TLS_VERIFY_CERTIFICATES
MAIN_TLS_VERIFY_CERTIFICATES = ${if exists{/etc/ssl/certs/ca-certificates.crt}\
                                    {/etc/ssl/certs/ca-certificates.crt}\
				    {/dev/null}}
.endif
tls_verify_certificates = MAIN_TLS_VERIFY_CERTIFICATES


# A list of hosts which are constrained by `tls_verify_certificates'. A host
# that matches `tls_verify_host' must present a certificate that is
# verifyable through `tls_verify_certificates' in order to be accepted as an
# SMTP client. If it does not, the connection is aborted.
.ifdef MAIN_TLS_VERIFY_HOSTS
tls_verify_hosts = MAIN_TLS_VERIFY_HOSTS
.endif

# A weaker form of checking: if a client matches `tls_try_verify_hosts' (but
# not `tls_verify_hosts'), request a certificate and check it against
# `tls_verify_certificates' but do not abort the connection if there is no
# certificate or if the certificate presented does not match. (This
# condition can be tested for in ACLs through `verify = certificate')
# By default, this check is done for all hosts. It is known that some
# clients (including incredimail's version downloadable in February
# 2008) choke on this. To disable, set MAIN_TLS_TRY_VERIFY_HOSTS to an
# empty value.
.ifdef MAIN_TLS_TRY_VERIFY_HOSTS
tls_try_verify_hosts = MAIN_TLS_TRY_VERIFY_HOSTS
.endif

.else
# Use upstream defaults
.endif
#####################################################
### end main/03_exim4-config_tlsoptions
#####################################################
#####################################################
### main/90_exim4-config_log_selector
#####################################################

### main/90_exim4-config_log_selector
#################################

# uncomment this for debugging
# MAIN_LOG_SELECTOR == MAIN_LOG_SELECTOR +all -subject -arguments

.ifdef MAIN_LOG_SELECTOR
log_selector = MAIN_LOG_SELECTOR
.endif
#####################################################
### end main/90_exim4-config_log_selector
#####################################################
#####################################################
### acl/00_exim4-config_header
#####################################################

######################################################################
#                       ACL CONFIGURATION                            #
#         Specifies access control lists for incoming SMTP mail      #
######################################################################
begin acl


#####################################################
### end acl/00_exim4-config_header
#####################################################
#####################################################
### acl/20_exim4-config_local_deny_exceptions
#####################################################

### acl/20_exim4-config_local_deny_exceptions
#################################

# This is used to determine whitelisted senders and hosts.
# It checks for CONFDIR/host_local_deny_exceptions and
# CONFDIR/sender_local_deny_exceptions.
#
# It is meant to be used from some other acl entry.
#
# See exim4-config_files(5) for details.
#
# If the files do not exist, the white list never matches, which is
# the desired behaviour.
#
# The old file names CONFDIR/local_host_whitelist and
# CONFDIR/local_sender_whitelist will continue to be honored for a
# transition period. Their use is deprecated.

acl_local_deny_exceptions:
  accept
    hosts = ${if exists{CONFDIR/host_local_deny_exceptions}\
                 {CONFDIR/host_local_deny_exceptions}\
                 {}}
  accept
    senders = ${if exists{CONFDIR/sender_local_deny_exceptions}\
                   {CONFDIR/sender_local_deny_exceptions}\
                   {}}
  accept
    hosts = ${if exists{CONFDIR/local_host_whitelist}\
                 {CONFDIR/local_host_whitelist}\
                 {}}
  accept
    senders = ${if exists{CONFDIR/local_sender_whitelist}\
                   {CONFDIR/local_sender_whitelist}\
                   {}}

  # This hook allows you to hook in your own ACLs without having to
  # modify this file. If you do it like we suggest, you'll end up with
  # a small performance penalty since there is an additional file being
  # accessed. This doesn't happen if you leave the macro unset.
  .ifdef LOCAL_DENY_EXCEPTIONS_LOCAL_ACL_FILE
  .include LOCAL_DENY_EXCEPTIONS_LOCAL_ACL_FILE
  .endif
  
  # this is still supported for a transition period and is deprecated.
  .ifdef WHITELIST_LOCAL_DENY_LOCAL_ACL_FILE
  .include WHITELIST_LOCAL_DENY_LOCAL_ACL_FILE
  .endif
#####################################################
### end acl/20_exim4-config_local_deny_exceptions
#####################################################
#####################################################
### acl/30_exim4-config_check_mail
#####################################################

### acl/30_exim4-config_check_mail
#################################

# This access control list is used for every MAIL command in an incoming
# SMTP message. The tests are run in order until the address is either
# accepted or denied.
#
acl_check_mail:

  accept
#####################################################
### end acl/30_exim4-config_check_mail
#####################################################
#####################################################
### acl/30_exim4-config_check_rcpt
#####################################################

### acl/30_exim4-config_check_rcpt
#################################

# define macros to be used below in this file to check recipient
# local parts for strange characters. Documentation below.
# This blocks local parts that begin with a dot or contain a quite
# broad range of non-alphanumeric characters.

.ifndef CHECK_RCPT_LOCAL_LOCALPARTS
CHECK_RCPT_LOCAL_LOCALPARTS = ^[.] : ^.*[@%!/|`#&?]
.endif

.ifndef CHECK_RCPT_REMOTE_LOCALPARTS
CHECK_RCPT_REMOTE_LOCALPARTS = ^[./|] : ^.*[@%!`#&?] : ^.*/\\.\\./
.endif

# This access control list is used for every RCPT command in an incoming
# SMTP message. The tests are run in order until the address is either
# accepted or denied.
#
acl_check_rcpt:

  # Accept if the source is local SMTP (i.e. not over TCP/IP). We do this by
  # testing for an empty sending host field.
  accept
    hosts = :
    control = dkim_disable_verify

  # Do not try to verify DKIM signatures of incoming mail if DC_minimaldns
  # or DISABLE_DKIM_VERIFY are set.
.ifdef DC_minimaldns
  warn
    control = dkim_disable_verify
.else
.ifdef DISABLE_DKIM_VERIFY
  warn
    control = dkim_disable_verify
.endif
.endif

  # The following section of the ACL is concerned with local parts that contain
  # certain non-alphanumeric characters. Dots in unusual places are
  # handled by this ACL as well.
  #
  # Non-alphanumeric characters other than dots are rarely found in genuine
  # local parts, but are often tried by people looking to circumvent
  # relaying restrictions. Therefore, although they are valid in local
  # parts, these rules disallow certain non-alphanumeric characters, as
  # a precaution.
  #
  # Empty components (two dots in a row) are not valid in RFC 2822, but Exim
  # allows them because they have been encountered. (Consider local parts
  # constructed as "firstinitial.secondinitial.familyname" when applied to
  # a name without a second initial.) However, a local part starting
  # with a dot or containing /../ can cause trouble if it is used as part of a
  # file name (e.g. for a mailing list). This is also true for local parts that
  # contain slashes. A pipe symbol can also be troublesome if the local part is
  # incorporated unthinkingly into a shell command line.
  #
  # These ACL components will block recipient addresses that are valid
  # from an RFC5322 point of view. We chose to have them blocked by
  # default for security reasons.
  #
  # If you feel that your site should have less strict recipient
  # checking, please feel free to change the default values of the macros
  # defined in main/01_exim4-config_listmacrosdefs or override them from a
  # local configuration file.
  # 
  # Two different rules are used. The first one has a quite strict
  # default, and is applied to messages that are addressed to one of the
  # local domains handled by this host.

  # The default value of CHECK_RCPT_LOCAL_LOCALPARTS is defined
  # at the top of this file.
  .ifdef CHECK_RCPT_LOCAL_LOCALPARTS
  deny
    domains = +local_domains
    local_parts = CHECK_RCPT_LOCAL_LOCALPARTS
    message = restricted characters in address
  .endif


  # The second rule applies to all other domains, and its default is
  # considerably less strict.
  
  # The default value of CHECK_RCPT_REMOTE_LOCALPARTS is defined in
  # main/01_exim4-config_listmacrosdefs:
  # CHECK_RCPT_REMOTE_LOCALPARTS = ^[./|] : ^.*[@%!`#&?] : ^.*/\\.\\./

  # It allows local users to send outgoing messages to sites
  # that use slashes and vertical bars in their local parts. It blocks
  # local parts that begin with a dot, slash, or vertical bar, but allows
  # these characters within the local part. However, the sequence /../ is
  # barred. The use of some other non-alphanumeric characters is blocked.
  # Single quotes might probably be dangerous as well, but they're
  # allowed by the default regexps to avoid rejecting mails to Ireland.
  # The motivation here is to prevent local users (or local users' malware)
  # from mounting certain kinds of attack on remote sites.
  .ifdef CHECK_RCPT_REMOTE_LOCALPARTS
  deny
    domains = !+local_domains
    local_parts = CHECK_RCPT_REMOTE_LOCALPARTS
    message = restricted characters in address
  .endif


  # Accept mail to postmaster in any local domain, regardless of the source,
  # and without verifying the sender.
  #
  accept
    .ifndef CHECK_RCPT_POSTMASTER
    local_parts = postmaster
    .else
    local_parts = CHECK_RCPT_POSTMASTER
    .endif
    domains = +local_domains : +relay_to_domains


  # Deny unless the sender address can be verified.
  #
  # This is disabled by default so that DNSless systems don't break. If
  # your system can do DNS lookups without delay or cost, you might want
  # to enable this feature.
  #
  # This feature does not work in smarthost and satellite setups as
  # with these setups all domains pass verification. See spec.txt section
  # "Access control lists" subsection "Address verification" with the added
  # information that a smarthost/satellite setup routes all non-local e-mail
  # to the smarthost.
  .ifdef CHECK_RCPT_VERIFY_SENDER
  deny
    !acl = acl_local_deny_exceptions
    !verify = sender
    message = Sender verification failed
  .endif

  # Verify senders listed in local_sender_callout with a callout.
  #
  # In smarthost and satellite setups, this causes the callout to be
  # done to the smarthost. Verification will thus only be reliable if the
  # smarthost does reject illegal addresses in the SMTP dialog.
  deny
    !acl = acl_local_deny_exceptions
    senders = ${if exists{CONFDIR/local_sender_callout}\
                         {CONFDIR/local_sender_callout}\
                   {}}
    !verify = sender/callout

  .ifndef CHECK_RCPT_NO_FAIL_TOO_MANY_BAD_RCPT
  # Reject all RCPT commands after too many bad recipients
  # This is partly a defense against spam abuse and partly attacker abuse.
  # Real senders should manage, by the time they get to 10 RCPT directives,
  # to have had at least half of them be real addresses.
  #
  # This is a lightweight check and can protect you against repeated
  # invocations of more heavy-weight checks which would come after it.

  deny    condition     = ${if and {\
                        {>{$rcpt_count}{10}}\
                        {<{$recipients_count}{${eval:$rcpt_count/2}}} }}
          message       = Rejected for too many bad recipients
          logwrite      = REJECT [$sender_host_address]: bad recipient count high [${eval:$rcpt_count-$recipients_count}]
  .endif

  # Accept if the message comes from one of the hosts for which we are an
  # outgoing relay. It is assumed that such hosts are most likely to be MUAs,
  # so we set control=submission to make Exim treat the message as a
  # submission. It will fix up various errors in the message, for example, the
  # lack of a Date: header line. If you are actually relaying out out from
  # MTAs, you may want to disable this. If you are handling both relaying from
  # MTAs and submissions from MUAs you should probably split them into two
  # lists, and handle them differently.

  # Recipient verification is omitted here, because in many cases the clients
  # are dumb MUAs that don't cope well with SMTP error responses. If you are
  # actually relaying out from MTAs, you should probably add recipient
  # verification here.

  # Note that, by putting this test before any DNS black list checks, you will
  # always accept from these hosts, even if they end up on a black list. The
  # assumption is that they are your friends, and if they get onto black
  # list, it is a mistake.
  accept
    hosts = +relay_from_hosts
    control = submission/sender_retain
    control = dkim_disable_verify


  # Accept if the message arrived over an authenticated connection, from
  # any host. Again, these messages are usually from MUAs, so recipient
  # verification is omitted, and submission mode is set. And again, we do this
  # check before any black list tests.
  accept
    authenticated = *
    control = submission/sender_retain
    control = dkim_disable_verify

  # Insist that a HELO/EHLO was accepted.

  require 
    condition	= ${if def:sender_helo_name}
    message	= nice hosts say HELO first

  # Insist that any other recipient address that we accept is either in one of
  # our local domains, or is in a domain for which we explicitly allow
  # relaying. Any other domain is rejected as being unacceptable for relaying.
  require
    message = relay not permitted
    domains = +local_domains : +relay_to_domains


  # We also require all accepted addresses to be verifiable. This check will
  # do local part verification for local domains, but only check the domain
  # for remote domains.
  require
    verify = recipient


  # Verify recipients listed in local_rcpt_callout with a callout.
  # This is especially handy for forwarding MX hosts (secondary MX or
  # mail hubs) of domains that receive a lot of spam to non-existent
  # addresses.  The only way to check local parts for remote relay
  # domains is to use a callout (add /callout), but please read the
  # documentation about callouts before doing this.
  deny
    !acl = acl_local_deny_exceptions
    recipients = ${if exists{CONFDIR/local_rcpt_callout}\
                            {CONFDIR/local_rcpt_callout}\
                      {}}
    !verify = recipient/callout


  # CONFDIR/local_sender_blacklist holds a list of envelope senders that
  # should have their access denied to the local host. Incoming messages
  # with one of these senders are rejected at RCPT time.
  #
  # The explicit white lists are honored as well as negative items in
  # the black list. See exim4-config_files(5) for details.
  deny
    !acl = acl_local_deny_exceptions
    senders = ${if exists{CONFDIR/local_sender_blacklist}\
                   {CONFDIR/local_sender_blacklist}\
                   {}}
    message = sender envelope address $sender_address is locally blacklisted here. If you think this is wrong, get in touch with postmaster
    log_message = sender envelope address is locally blacklisted.


  # deny bad sites (IP address)
  # CONFDIR/local_host_blacklist holds a list of host names, IP addresses
  # and networks (CIDR notation)  that should have their access denied to
  # The local host. Messages coming in from a listed host will have all
  # RCPT statements rejected.
  #
  # The explicit white lists are honored as well as negative items in
  # the black list. See exim4-config_files(5) for details.
  deny
    !acl = acl_local_deny_exceptions
    hosts = ${if exists{CONFDIR/local_host_blacklist}\
                 {CONFDIR/local_host_blacklist}\
                 {}}
    message = sender IP address $sender_host_address is locally blacklisted here. If you think this is wrong, get in touch with postmaster
    log_message = sender IP address is locally blacklisted.


  # Warn if the sender host does not have valid reverse DNS.
  # 
  # If your system can do DNS lookups without delay or cost, you might want
  # to enable this.
  # If sender_host_address is defined, it's a remote call.  If
  # sender_host_name is not defined, then reverse lookup failed.  Use
  # this instead of !verify = reverse_host_lookup to catch deferrals
  # as well as outright failures.
  .ifdef CHECK_RCPT_REVERSE_DNS
  warn
    condition = ${if and{{def:sender_host_address}{!def:sender_host_name}}\
                      {yes}{no}}
    add_header = X-Host-Lookup-Failed: Reverse DNS lookup failed for $sender_host_address (${if eq{$host_lookup_failed}{1}{failed}{deferred}})
  .endif


  # Use spfquery to perform a pair of SPF checks.
  #
  # This is quite costly in terms of DNS lookups (~6 lookups per mail).  Do not
  # enable if that's an issue.  Also note that if you enable this, you must
  # install "spf-tools-perl" which provides the spfquery command.
  # Missing spf-tools-perl will trigger the "Unexpected error in
  # SPF check" warning.
  .ifdef CHECK_RCPT_SPF
  deny
    !acl = acl_local_deny_exceptions
    condition = ${run{/usr/bin/spfquery.mail-spf-perl --ip \
                   ${quote:$sender_host_address} --identity \
                   ${if def:sender_address_domain \
                       {--scope mfrom  --identity ${quote:$sender_address}}\
                       {--scope helo --identity ${quote:$sender_helo_name}}}}\
                   {no}{${if eq {$runrc}{1}{yes}{no}}}}
    message = [SPF] $sender_host_address is not allowed to send mail from \
              ${if def:sender_address_domain {$sender_address_domain}{$sender_helo_name}}.
    log_message = SPF check failed.

  defer
    !acl = acl_local_deny_exceptions
    condition = ${if eq {$runrc}{5}{yes}{no}}
    message = Temporary DNS error while checking SPF record.  Try again later.

  warn
    condition = ${if <={$runrc}{6}{yes}{no}}
    add_header = Received-SPF: ${if eq {$runrc}{0}{pass}\
                                {${if eq {$runrc}{2}{softfail}\
                                 {${if eq {$runrc}{3}{neutral}\
				  {${if eq {$runrc}{4}{permerror}\
				   {${if eq {$runrc}{6}{none}{error}}}}}}}}}\
				} client-ip=$sender_host_address; \
				${if def:sender_address_domain \
				   {envelope-from=${sender_address}; }{}}\
				helo=$sender_helo_name

  warn
    condition = ${if >{$runrc}{6}{yes}{no}}
    log_message = Unexpected error in SPF check.
  .endif


  # Check against classic DNS "black" lists (DNSBLs) which list
  # sender IP addresses
  .ifdef CHECK_RCPT_IP_DNSBLS
  warn
    dnslists = CHECK_RCPT_IP_DNSBLS
    add_header = X-Warning: $sender_host_address is listed at $dnslist_domain ($dnslist_value: $dnslist_text)
    log_message = $sender_host_address is listed at $dnslist_domain ($dnslist_value: $dnslist_text)
  .endif


  # Check against DNSBLs which list sender domains, with an option to locally
  # whitelist certain domains that might be blacklisted.
  #
  # Note: If you define CHECK_RCPT_DOMAIN_DNSBLS, you must append
  # "/$sender_address_domain" after each domain.  For example:
  # CHECK_RCPT_DOMAIN_DNSBLS = rhsbl.foo.org/$sender_address_domain \
  #                            : rhsbl.bar.org/$sender_address_domain
  .ifdef CHECK_RCPT_DOMAIN_DNSBLS
  warn
    !senders = ${if exists{CONFDIR/local_domain_dnsbl_whitelist}\
                    {CONFDIR/local_domain_dnsbl_whitelist}\
                    {}}
    dnslists = CHECK_RCPT_DOMAIN_DNSBLS
    add_header = X-Warning: $sender_address_domain is listed at $dnslist_domain ($dnslist_value: $dnslist_text)
    log_message = $sender_address_domain is listed at $dnslist_domain ($dnslist_value: $dnslist_text)
  .endif


  # This hook allows you to hook in your own ACLs without having to
  # modify this file. If you do it like we suggest, you'll end up with
  # a small performance penalty since there is an additional file being
  # accessed. This doesn't happen if you leave the macro unset.
  .ifdef CHECK_RCPT_LOCAL_ACL_FILE
  .include CHECK_RCPT_LOCAL_ACL_FILE
  .endif


  #############################################################################
  # This check is commented out because it is recognized that not every
  # sysadmin will want to do it. If you enable it, the check performs
  # Client SMTP Authorization (csa) checks on the sending host. These checks
  # do DNS lookups for SRV records. The CSA proposal is currently (May 2005)
  # an Internet draft. You can, of course, add additional conditions to this
  # ACL statement to restrict the CSA checks to certain hosts only.
  #
  # require verify = csa
  #############################################################################


  # Accept if the address is in a domain for which we are an incoming relay,
  # but again, only if the recipient can be verified.

  accept
    domains = +relay_to_domains
    endpass
    verify = recipient


  # At this point, the address has passed all the checks that have been
  # configured, so we accept it unconditionally.

  accept
#####################################################
### end acl/30_exim4-config_check_rcpt
#####################################################
#####################################################
### acl/40_exim4-config_check_data
#####################################################

### acl/40_exim4-config_check_data
#################################

# This ACL is used after the contents of a message have been received. This
# is the ACL in which you can test a message's headers or body, and in
# particular, this is where you can invoke external virus or spam scanners.

acl_check_data:

  # Deny if the message contains an overlong line.  Per the standards
  # we should never receive one such via SMTP.
  #
  .ifndef IGNORE_SMTP_LINE_LENGTH_LIMIT
  deny
    condition  = ${if > {$max_received_linelength}{998}}
    message    = maximum allowed line length is 998 octets, \
                       got $max_received_linelength
  .endif

  # Deny if the headers contain badly-formed addresses.
  #
  .ifndef NO_CHECK_DATA_VERIFY_HEADER_SYNTAX
  deny
    !acl = acl_local_deny_exceptions
    !verify = header_syntax
    message = header syntax
    log_message = header syntax ($acl_verify_message)
  .endif


  # require that there is a verifiable sender address in at least
  # one of the "Sender:", "Reply-To:", or "From:" header lines.
  .ifdef CHECK_DATA_VERIFY_HEADER_SENDER
  deny
    !acl = acl_local_deny_exceptions
    !verify = header_sender
    message = No verifiable sender address in message headers
  .endif


  # Deny if the message contains malware. Before enabling this check, you
  # must install a virus scanner and set the av_scanner option in the
  # main configuration.
  #
  # exim4-daemon-heavy must be used for this section to work.
  #
  # deny
  #   malware = *
  #   message = This message was detected as possible malware ($malware_name).


  # Add headers to a message if it is judged to be spam. Before enabling this,
  # you must install SpamAssassin. You may also need to set the spamd_address
  # option in the main configuration.
  #
  # exim4-daemon-heavy must be used for this section to work.
  #
  # Please note that this is only suiteable as an example. See
  # /usr/share/doc/exim4-base/README.Debian.gz
  #
  # See the exim docs and the exim wiki for more suitable examples.
  #
  # # Remove internal headers
  # warn
  #   remove_header = X-Spam_score: X-Spam_score_int : X-Spam_bar : \
  #                   X-Spam_report
  #
  # warn
  #   condition = ${if <{$message_size}{120k}{1}{0}}
  #   # ":true" to add headers/acl variables even if not spam
  #   spam = nobody:true
  #   add_header = X-Spam_score: $spam_score
  #   add_header = X-Spam_bar: $spam_bar
  #   # Do not enable this unless you have shorted SpamAssassin's report
  #   #add_header = X-Spam_report: $spam_report
  #
  # Reject spam messages (score >15.0).
  # This breaks mailing list and forward messages.
  # deny
  #   condition = ${if <{$message_size}{120k}{1}{0}}
  #   condition = ${if >{$spam_score_int}{150}{true}{false}}
  #   message = Classified as spam (score $spam_score)


  # This hook allows you to hook in your own ACLs without having to
  # modify this file. If you do it like we suggest, you'll end up with
  # a small performance penalty since there is an additional file being
  # accessed. This doesn't happen if you leave the macro unset.
  .ifdef CHECK_DATA_LOCAL_ACL_FILE
  .include CHECK_DATA_LOCAL_ACL_FILE
  .endif


  # accept otherwise
  accept
#####################################################
### end acl/40_exim4-config_check_data
#####################################################
#####################################################
### router/00_exim4-config_header
#####################################################

######################################################################
#                      ROUTERS CONFIGURATION                         #
#               Specifies how addresses are handled                  #
######################################################################
#     THE ORDER IN WHICH THE ROUTERS ARE DEFINED IS IMPORTANT!       #
# An address is passed to each router in turn until it is accepted.  #
######################################################################

begin routers

#####################################################
### end router/00_exim4-config_header
#####################################################
#####################################################
### router/100_exim4-config_domain_literal
#####################################################

### router/100_exim4-config_domain_literal
#################################

# This router handles e-mail addresses in "domain literal" form like
# <user@[10.11.12.13]>. The RFCs require this facility, but it is disabled
# in the default config since it is rarely used and frequently abused.
# Domain literal support also needs to be enabled in the main config,
# which is automatically done if you use the enable macro
# MAIN_ALLOW_DOMAIN_LITERALS.

.ifdef MAIN_ALLOW_DOMAIN_LITERALS
domain_literal:
  debug_print = "R: domain_literal for $local_part@$domain"
  driver = ipliteral
  domains = ! +local_domains
  transport = remote_smtp
.endif
#####################################################
### end router/100_exim4-config_domain_literal
#####################################################
#####################################################
### router/150_exim4-config_hubbed_hosts
#####################################################

# router/150_exim4-config_hubbed_hosts
#################################

# route specific domains manually.
#
# see exim4-config_files(5) and spec.txt chapter 20.3 through 20.7 for
# more detailed documentation.

hubbed_hosts:
  debug_print = "R: hubbed_hosts for $domain"
  driver = manualroute
  domains = "${if exists{CONFDIR/hubbed_hosts}\
                   {partial-lsearch;CONFDIR/hubbed_hosts}\
              fail}"
  same_domain_copy_routing = yes
  route_data = ${lookup{$domain}partial-lsearch{CONFDIR/hubbed_hosts}}
  transport = remote_smtp
#####################################################
### end router/150_exim4-config_hubbed_hosts
#####################################################
#####################################################
### router/200_exim4-config_primary
#####################################################

### router/200_exim4-config_primary
#################################
# This file holds the primary router, responsible for nonlocal mails

.ifdef DCconfig_internet
# configtype=internet
#
# deliver mail to the recipient if recipient domain is a domain we
# relay for. We do not ignore any target hosts here since delivering to
# a site local or even a link local address might be wanted here, and if
# such an address has found its way into the MX record of such a domain,
# the local admin is probably in a place where that broken MX record
# could be fixed.

dnslookup_relay_to_domains:
  debug_print = "R: dnslookup_relay_to_domains for $local_part@$domain"
  driver = dnslookup
  domains = ! +local_domains : +relay_to_domains
  transport = remote_smtp
  same_domain_copy_routing = yes
  no_more

# ignore private rfc1918, loopback, APIPA/link-local, local broadcast, unspecified, unique local, linked-scoped unicast and discard-Only
.ifndef ROUTER_DNSLOOKUP_IGNORE_TARGET_HOSTS
ROUTER_DNSLOOKUP_IGNORE_TARGET_HOSTS = <; 0.0.0.0 ; 127.0.0.0/8 ; 192.168.0.0/16 ; 172.16.0.0/12 ; 10.0.0.0/8 ; 169.254.0.0/16 ; 255.255.255.255 ; ::/128 ; ::1/128 ; fc00::/7 ; fe80::/10 ; 100::/64
.endif

# deliver mail directly to the recipient. This router is only reached
# for domains that we do not relay for. Since we most probably can't
# have broken MX records pointing to site local or link local IP
# addresses fixed, we ignore target hosts pointing to these addresses.

dnslookup:
  debug_print = "R: dnslookup for $local_part@$domain"
  driver = dnslookup
  domains = ! +local_domains
  transport = remote_smtp
  same_domain_copy_routing = yes
  ignore_target_hosts = ROUTER_DNSLOOKUP_IGNORE_TARGET_HOSTS
  no_more

.endif


.ifdef DCconfig_local
# configtype=local
#
# Stand-alone system, so generate an error for mail to a non-local domain
nonlocal:
  debug_print = "R: nonlocal for $local_part@$domain"
  driver = redirect
  domains = ! +local_domains
  allow_fail
  data = :fail: Mailing to remote domains not supported
  no_more

.endif


.ifdef DCconfig_smarthost DCconfig_satellite
# configtype=smarthost or configtype=satellite
#
# Send all non-local mail to a single other machine (smarthost).
#
# This means _ALL_ non-local mail goes to the smarthost. This will most
# probably not do what you want for domains that are listed in
# relay_domains. The most typical use for relay_domains is to control
# relaying for incoming e-mail on secondary MX hosts. In that case,
# it doesn't make sense to send the mail to the smarthost since the
# smarthost will probably send the message right back here, causing a
# loop.
#
# If you want to use a smarthost while being secondary MX for some
# domains, you'll need to copy the dnslookup_relay_to_domains router
# here so that mail to relay_domains is handled separately.

smarthost:
  debug_print = "R: smarthost for $local_part@$domain"
  driver = manualroute
  domains = ! +local_domains
  transport = remote_smtp_smarthost
  route_list = * DCsmarthost byname
  host_find_failed = ignore
  same_domain_copy_routing = yes
  no_more

.endif


# The "no_more" above means that all later routers are for
# domains in the local_domains list, i.e. just like Exim 3 directors.
#####################################################
### end router/200_exim4-config_primary
#####################################################
#####################################################
### router/300_exim4-config_real_local
#####################################################

### router/300_exim4-config_real_local
#################################

# This router allows reaching a local user while avoiding local
# processing. This can be used to inform a user of a broken .forward
# file, for example. The userforward router does this.

COND_LOCAL_SUBMITTER = "\
               ${if match_ip{$sender_host_address}{:@[]}\
                    {1}{0}\
		}"

real_local:
  debug_print = "R: real_local for $local_part@$domain"
  driver = accept
  domains = +local_domains
  condition = COND_LOCAL_SUBMITTER
  local_part_prefix = real-
  check_local_user
  transport = LOCAL_DELIVERY

#####################################################
### end router/300_exim4-config_real_local
#####################################################
#####################################################
### router/400_exim4-config_system_aliases
#####################################################

### router/400_exim4-config_system_aliases
#################################

# This router handles aliasing using a traditional /etc/aliases file.
#
##### NB  You must ensure that /etc/aliases exists. It used to be the case
##### NB  that every Unix had that file, because it was the Sendmail default.
##### NB  These days, there are systems that don't have it. Your aliases
##### NB  file should at least contain an alias for "postmaster".
#
# This router handles the local part in a case-insensitive way which
# satisfies the RFCs requirement that postmaster be reachable regardless
# of case. If you decide to handle /etc/aliases in a caseful way, you
# need to make arrangements for a caseless postmaster.
#
# Delivery to arbitrary directories, files, and piping to programs in
# /etc/aliases is disabled per default.
# If that is a problem for you, see
#   /usr/share/doc/exim4-base/README.Debian.gz
# for explanation and some workarounds.

system_aliases:
  debug_print = "R: system_aliases for $local_part@$domain"
  driver = redirect
  domains = +local_domains
  allow_fail
  allow_defer
  data = ${lookup{$local_part}lsearch{/etc/aliases}}
  .ifdef SYSTEM_ALIASES_USER
  user = SYSTEM_ALIASES_USER
  .endif
  .ifdef SYSTEM_ALIASES_GROUP
  group = SYSTEM_ALIASES_GROUP
  .endif
  .ifdef SYSTEM_ALIASES_FILE_TRANSPORT
  file_transport = SYSTEM_ALIASES_FILE_TRANSPORT
  .endif
  .ifdef SYSTEM_ALIASES_PIPE_TRANSPORT
  pipe_transport = SYSTEM_ALIASES_PIPE_TRANSPORT
  .endif
  .ifdef SYSTEM_ALIASES_DIRECTORY_TRANSPORT
  directory_transport = SYSTEM_ALIASES_DIRECTORY_TRANSPORT
  .endif
#####################################################
### end router/400_exim4-config_system_aliases
#####################################################
#####################################################
### router/500_exim4-config_hubuser
#####################################################

### router/500_exim4-config_hubuser
#################################

.ifdef DCconfig_satellite
# This router is only used for configtype=satellite.
# It takes care to route all mail targeted to <somelocaluser@this.machine>
# to the host where we read our mail
#
hub_user:
  debug_print = "R: hub_user for $local_part@$domain"
  driver = redirect
  domains = +local_domains
  data = ${local_part}@DCreadhost
  check_local_user

# Grab the redirected mail and deliver it.
# This is a duplicate of the smarthost router, needed because
# DCreadhost might end up as part of +local_domains
hub_user_smarthost:
  debug_print = "R: hub_user_smarthost for $local_part@$domain"
  driver = manualroute
  domains = DCreadhost
  transport = remote_smtp_smarthost
  route_list = * DCsmarthost byname
  host_find_failed = ignore
  same_domain_copy_routing = yes
  check_local_user
.endif


#####################################################
### end router/500_exim4-config_hubuser
#####################################################
#####################################################
### router/600_exim4-config_userforward
#####################################################

### router/600_exim4-config_userforward
#################################

# This router handles forwarding using traditional .forward files in users'
# home directories. It also allows mail filtering with a forward file
# starting with the string "# Exim filter" or "# Sieve filter".
#
# The no_verify setting means that this router is skipped when Exim is
# verifying addresses. Similarly, no_expn means that this router is skipped if
# Exim is processing an EXPN command.
#
# The check_ancestor option means that if the forward file generates an
# address that is an ancestor of the current one, the current one gets
# passed on instead. This covers the case where A is aliased to B and B
# has a .forward file pointing to A.
#
# The four transports specified at the end are those that are used when
# forwarding generates a direct delivery to a directory, or a file, or to a
# pipe, or sets up an auto-reply, respectively.
#
userforward:
  debug_print = "R: userforward for $local_part@$domain"
  driver = redirect
  domains = +local_domains
  check_local_user
  file = $home/.forward
  require_files = $local_part_data:$home/.forward
  no_verify
  no_expn
  check_ancestor
  allow_filter
  forbid_smtp_code = true
  directory_transport = address_directory
  file_transport = address_file
  pipe_transport = address_pipe
  reply_transport = address_reply
  skip_syntax_errors
  syntax_errors_to = real-$local_part@$domain
  syntax_errors_text = \
    This is an automatically generated message. An error has\n\
    been found in your .forward file. Details of the error are\n\
    reported below. While this error persists, you will receive\n\
    a copy of this message for every message that is addressed\n\
    to you. If your .forward file is a filter file, or if it is\n\
    a non-filter file containing no valid forwarding addresses,\n\
    a copy of each incoming message will be put in your normal\n\
    mailbox. If a non-filter file contains at least one valid\n\
    forwarding address, forwarding to the valid addresses will\n\
    happen, and those will be the only deliveries that occur.

#####################################################
### end router/600_exim4-config_userforward
#####################################################
#####################################################
### router/700_exim4-config_procmail
#####################################################

procmail:
  debug_print = "R: procmail for $local_part@$domain"
  driver = accept
  domains = +local_domains
  check_local_user
  transport = procmail_pipe
  # emulate OR with "if exists"-expansion
  require_files = ${local_part_data}:\
                  ${if exists{/etc/procmailrc}\
                    {/etc/procmailrc}{${home}/.procmailrc}}:\
                  +/usr/bin/procmail
  no_verify
  no_expn

#####################################################
### end router/700_exim4-config_procmail
#####################################################
#####################################################
### router/800_exim4-config_maildrop
#####################################################

### router/800_exim4-config_maildrop
#################################

maildrop:
  debug_print = "R: maildrop for $local_part@$domain"
  driver = accept
  domains = +local_domains
  check_local_user
  transport = maildrop_pipe
  require_files = ${local_part_data}:${home}/.mailfilter:+/usr/bin/maildrop
  no_verify
  no_expn

#####################################################
### end router/800_exim4-config_maildrop
#####################################################
#####################################################
### router/850_exim4-config_lowuid
#####################################################

### router/850_exim4-config_lowuid
#################################

.ifndef FIRST_USER_ACCOUNT_UID
FIRST_USER_ACCOUNT_UID = 0
.endif

.ifndef DEFAULT_SYSTEM_ACCOUNT_ALIAS
DEFAULT_SYSTEM_ACCOUNT_ALIAS = :fail: no mail to system accounts
.endif

COND_SYSTEM_USER_AND_REMOTE_SUBMITTER = "\
               ${if and{{! match_ip{$sender_host_address}{:@[]}}\
                        {<{$local_user_uid}{FIRST_USER_ACCOUNT_UID}}}\
                    {1}{0}\
		}"

lowuid_aliases:
  debug_print = "R: lowuid_aliases for $local_part@$domain (UID $local_user_uid)"
  check_local_user
  driver = redirect
  allow_fail
  domains = +local_domains
  condition = COND_SYSTEM_USER_AND_REMOTE_SUBMITTER
  data = ${if exists{CONFDIR/lowuid-aliases}\
              {${lookup{$local_part}lsearch{CONFDIR/lowuid-aliases}\
              {$value}{DEFAULT_SYSTEM_ACCOUNT_ALIAS}}}\
              {DEFAULT_SYSTEM_ACCOUNT_ALIAS}}
#####################################################
### end router/850_exim4-config_lowuid
#####################################################
#####################################################
### router/900_exim4-config_local_user
#####################################################

### router/900_exim4-config_local_user
#################################

# This router matches local user mailboxes. If the router fails, the error
# message is "Unknown user".

local_user:
  debug_print = "R: local_user for $local_part@$domain"
  driver = accept
  domains = +local_domains
  check_local_user
  local_parts = ! root
  transport = LOCAL_DELIVERY
  cannot_route_message = Unknown user
#####################################################
### end router/900_exim4-config_local_user
#####################################################
#####################################################
### router/mmm_mail4root
#####################################################

### router/mmm_mail4root
#################################
# deliver mail addressed to root to /var/mail/mail as user mail:mail
# if it was not redirected in /etc/aliases or by other means
# Exim cannot deliver as root since 4.24 (FIXED_NEVER_USERS)

mail4root:
  debug_print = "R: mail4root for $local_part@$domain"
  driver = redirect
  domains = +local_domains
  data = /var/mail/mail
  file_transport = address_file
  local_parts = root
  user = mail
  group = mail

#####################################################
### end router/mmm_mail4root
#####################################################
#####################################################
### transport/00_exim4-config_header
#####################################################

######################################################################
#                      TRANSPORTS CONFIGURATION                      #
######################################################################
#                       ORDER DOES NOT MATTER                        #
#     Only one appropriate transport is called for each delivery.    #
######################################################################

# A transport is used only when referenced from a router that successfully
# handles an address.

begin transports

#####################################################
### end transport/00_exim4-config_header
#####################################################
#####################################################
### transport/10_exim4-config_transport-macros
#####################################################

### transport/10_exim4-config_transport-macros
#################################

.ifdef HIDE_MAILNAME
REMOTE_SMTP_HEADERS_REWRITE=*@+local_domains $1@DCreadhost frs : *@ETC_MAILNAME $1@DCreadhost frs
REMOTE_SMTP_RETURN_PATH=${if match_domain{$sender_address_domain}{+local_domains}{${sender_address_local_part}@DCreadhost}{${if match_domain{$sender_address_domain}{ETC_MAILNAME}{${sender_address_local_part}@DCreadhost}fail}}}
.endif

.ifdef REMOTE_SMTP_HELO_FROM_DNS
.ifdef REMOTE_SMTP_HELO_DATA
REMOTE_SMTP_HELO_DATA==${lookup dnsdb {ptr=$sending_ip_address}{$value}{$primary_hostname}}
.else
REMOTE_SMTP_HELO_DATA=${lookup dnsdb {ptr=$sending_ip_address}{$value}{$primary_hostname}}
.endif
.endif

.ifndef REMOTE_SMTP_SMARTHOST_TLS_VERIFY_HOSTS
  REMOTE_SMTP_SMARTHOST_TLS_VERIFY_HOSTS = *
.endif
#####################################################
### end transport/10_exim4-config_transport-macros
#####################################################
#####################################################
### transport/30_exim4-config_address_file
#####################################################

# This transport is used for handling deliveries directly to files that are
# generated by aliasing or forwarding.
#
address_file:
  debug_print = "T: address_file for $local_part@$domain"
  driver = appendfile
  delivery_date_add
  envelope_to_add
  return_path_add

#####################################################
### end transport/30_exim4-config_address_file
#####################################################
#####################################################
### transport/30_exim4-config_address_pipe
#####################################################

# This transport is used for handling pipe deliveries generated by
# .forward files. If the commands fails and produces any output on standard
# output or standard error streams, the output is returned to the sender
# of the message as a delivery error.
address_pipe:
  debug_print = "T: address_pipe for $local_part@$domain"
  driver = pipe
  return_fail_output

#####################################################
### end transport/30_exim4-config_address_pipe
#####################################################
#####################################################
### transport/30_exim4-config_address_reply
#####################################################

# This transport is used for handling autoreplies generated by the filtering
# option of the userforward router.
#
address_reply:
  debug_print = "T: autoreply for $local_part@$domain"
  driver = autoreply

#####################################################
### end transport/30_exim4-config_address_reply
#####################################################
#####################################################
### transport/30_exim4-config_mail_spool
#####################################################

### transport/30_exim4-config_mail_spool

# This transport is used for local delivery to user mailboxes in traditional
# BSD mailbox format.
#
mail_spool:
  debug_print = "T: appendfile for $local_part@$domain"
  driver = appendfile
  file = /var/mail/$local_part_data
  delivery_date_add
  envelope_to_add
  return_path_add
  group = mail
  mode = 0660
  mode_fail_narrower = false

#####################################################
### end transport/30_exim4-config_mail_spool
#####################################################
#####################################################
### transport/30_exim4-config_maildir_home
#####################################################

### transport/30_exim4-config_maildir_home
#################################

# Use this instead of mail_spool if you want to to deliver to Maildir in
# home-directory - change the definition of LOCAL_DELIVERY
#
maildir_home:
  debug_print = "T: maildir_home for $local_part@$domain"
  driver = appendfile
  .ifdef MAILDIR_HOME_MAILDIR_LOCATION
  directory = MAILDIR_HOME_MAILDIR_LOCATION
  .else
  directory = $home/Maildir
  .endif
  .ifdef MAILDIR_HOME_CREATE_DIRECTORY
  create_directory
  .endif
  .ifdef MAILDIR_HOME_CREATE_FILE
  create_file = MAILDIR_HOME_CREATE_FILE
  .endif
  delivery_date_add
  envelope_to_add
  return_path_add
  maildir_format
  .ifdef MAILDIR_HOME_DIRECTORY_MODE
  directory_mode = MAILDIR_HOME_DIRECTORY_MODE
  .else
  directory_mode = 0700
  .endif
  .ifdef MAILDIR_HOME_MODE
  mode = MAILDIR_HOME_MODE
  .else
  mode = 0600
  .endif
  mode_fail_narrower = false
  # This transport always chdirs to $home before trying to deliver. If
  # $home is not accessible, this chdir fails and prevents delivery.
  # If you are in a setup where home directories might not be
  # accessible, uncomment the current_directory line below.
  # current_directory = /
#####################################################
### end transport/30_exim4-config_maildir_home
#####################################################
#####################################################
### transport/30_exim4-config_maildrop_pipe
#####################################################

maildrop_pipe:
  debug_print = "T: maildrop_pipe for $local_part@$domain"
  driver = pipe
  path = "/bin:/usr/bin:/usr/local/bin"
  command = "/usr/bin/maildrop"
  message_prefix =
  message_suffix =
  return_path_add
  delivery_date_add
  envelope_to_add

#####################################################
### end transport/30_exim4-config_maildrop_pipe
#####################################################
#####################################################
### transport/30_exim4-config_procmail_pipe
#####################################################

procmail_pipe:
  debug_print = "T: procmail_pipe for $local_part@$domain"
  driver = pipe
  path = "/bin:/usr/bin:/usr/local/bin"
  command = "/usr/bin/procmail"
  return_path_add
  delivery_date_add
  envelope_to_add

#####################################################
### end transport/30_exim4-config_procmail_pipe
#####################################################
#####################################################
### transport/30_exim4-config_remote_smtp
#####################################################

### transport/30_exim4-config_remote_smtp
#################################
# This transport is used for delivering messages over SMTP connections.
# Refuse to send any message with over-long lines, which could have
# been received other than via SMTP. The use of message_size_limit to
# enforce this is a red herring.

remote_smtp:
  debug_print = "T: remote_smtp for $local_part@$domain"
  driver = smtp
.ifndef IGNORE_SMTP_LINE_LENGTH_LIMIT
  message_size_limit = ${if > {$max_received_linelength}{998} {1}{0}}
.endif
.ifdef REMOTE_SMTP_HOSTS_AVOID_TLS
  hosts_avoid_tls = REMOTE_SMTP_HOSTS_AVOID_TLS
.endif
.ifdef REMOTE_SMTP_HEADERS_REWRITE
  headers_rewrite = REMOTE_SMTP_HEADERS_REWRITE
.endif
.ifdef REMOTE_SMTP_RETURN_PATH
  return_path = REMOTE_SMTP_RETURN_PATH
.endif
.ifdef REMOTE_SMTP_HELO_DATA
  helo_data=REMOTE_SMTP_HELO_DATA
.endif
.ifdef REMOTE_SMTP_INTERFACE
  interface = REMOTE_SMTP_INTERFACE
.endif
.ifdef DKIM_DOMAIN
dkim_domain = DKIM_DOMAIN
.endif
.ifdef DKIM_SELECTOR
dkim_selector = DKIM_SELECTOR
.endif
.ifdef DKIM_PRIVATE_KEY
dkim_private_key = DKIM_PRIVATE_KEY
.endif
.ifdef DKIM_CANON
dkim_canon = DKIM_CANON
.endif
.ifdef DKIM_STRICT
dkim_strict = DKIM_STRICT
.endif
.ifdef DKIM_SIGN_HEADERS
dkim_sign_headers = DKIM_SIGN_HEADERS
.endif
.ifdef DKIM_TIMESTAMPS
dkim_timestamps = DKIM_TIMESTAMPS
.endif
.ifdef TLS_DH_MIN_BITS
tls_dh_min_bits = TLS_DH_MIN_BITS
.endif
.ifdef REMOTE_SMTP_TLS_CERTIFICATE
tls_certificate = REMOTE_SMTP_TLS_CERTIFICATE
.endif
.ifdef REMOTE_SMTP_PRIVATEKEY
tls_privatekey = REMOTE_SMTP_PRIVATEKEY
.endif
.ifdef REMOTE_SMTP_HOSTS_REQUIRE_TLS
  hosts_require_tls = REMOTE_SMTP_HOSTS_REQUIRE_TLS
.endif
.ifdef REMOTE_SMTP_TRANSPORTS_HEADERS_REMOVE
  headers_remove = REMOTE_SMTP_TRANSPORTS_HEADERS_REMOVE
.endif
#####################################################
### end transport/30_exim4-config_remote_smtp
#####################################################
#####################################################
### transport/30_exim4-config_remote_smtp_smarthost
#####################################################

### transport/30_exim4-config_remote_smtp_smarthost
#################################

# This transport is used for delivering messages over SMTP connections
# to a smarthost. The local host tries to authenticate.
# This transport is used for smarthost and satellite configurations.
# Refuse to send any messsage with over-long lines, which could have
# been received other than via SMTP. The use of message_size_limit to
# enforce this is a red herring.

remote_smtp_smarthost:
  debug_print = "T: remote_smtp_smarthost for $local_part@$domain"
  driver = smtp
  multi_domain
.ifndef IGNORE_SMTP_LINE_LENGTH_LIMIT
  message_size_limit = ${if > {$max_received_linelength}{998} {1}{0}}
.endif
  hosts_try_auth = <; ${if exists{CONFDIR/passwd.client} \
        {\
        ${lookup{$host}nwildlsearch{CONFDIR/passwd.client}{$host_address}}\
        }\
        {} \
      }
.ifdef REMOTE_SMTP_SMARTHOST_HOSTS_AVOID_TLS
  hosts_avoid_tls = REMOTE_SMTP_SMARTHOST_HOSTS_AVOID_TLS
.endif
.ifdef REMOTE_SMTP_SMARTHOST_HOSTS_REQUIRE_TLS
  hosts_require_tls = REMOTE_SMTP_SMARTHOST_HOSTS_REQUIRE_TLS
.endif
.ifdef REMOTE_SMTP_SMARTHOST_TLS_VERIFY_CERTIFICATES
  tls_verify_certificates = REMOTE_SMTP_SMARTHOST_TLS_VERIFY_CERTIFICATES
.endif
.ifdef REMOTE_SMTP_SMARTHOST_TLS_VERIFY_HOSTS
  tls_verify_hosts = REMOTE_SMTP_SMARTHOST_TLS_VERIFY_HOSTS
.endif
.ifdef REMOTE_SMTP_HEADERS_REWRITE
  headers_rewrite = REMOTE_SMTP_HEADERS_REWRITE
.endif
.ifdef REMOTE_SMTP_RETURN_PATH
  return_path = REMOTE_SMTP_RETURN_PATH
.endif
.ifdef REMOTE_SMTP_HELO_DATA
  helo_data=REMOTE_SMTP_HELO_DATA
.endif
.ifdef TLS_DH_MIN_BITS
tls_dh_min_bits = TLS_DH_MIN_BITS
.endif
.ifdef REMOTE_SMTP_SMARTHOST_TLS_CERTIFICATE
tls_certificate = REMOTE_SMTP_SMARTHOST_TLS_CERTIFICATE
.endif
.ifdef REMOTE_SMTP_SMARTHOST_PRIVATEKEY
tls_privatekey = REMOTE_SMTP_SMARTHOST_PRIVATEKEY
.endif
.ifdef REMOTE_SMTP_TRANSPORTS_HEADERS_REMOVE
  headers_remove = REMOTE_SMTP_TRANSPORTS_HEADERS_REMOVE
.endif
#####################################################
### end transport/30_exim4-config_remote_smtp_smarthost
#####################################################
#####################################################
### transport/35_exim4-config_address_directory
#####################################################
# This transport is used for handling file addresses generated by alias
# or .forward files if the path ends in "/", which causes it to be treated
# as a directory name rather than a file name.

address_directory:
  debug_print = "T: address_directory for $local_part@$domain"
  driver = appendfile
  delivery_date_add
  envelope_to_add
  return_path_add
  check_string = ""
  escape_string = ""
  maildir_format

#####################################################
### end transport/35_exim4-config_address_directory
#####################################################
#####################################################
### retry/00_exim4-config_header
#####################################################

######################################################################
#                      RETRY CONFIGURATION                           #
######################################################################

begin retry

#####################################################
### end retry/00_exim4-config_header
#####################################################
#####################################################
### retry/30_exim4-config
#####################################################

### retry/30_exim4-config
#################################

# This single retry rule applies to all domains and all errors. It specifies
# retries every 15 minutes for 2 hours, then increasing retry intervals,
# starting at 1 hour and increasing each time by a factor of 1.5, up to 16
# hours, then retries every 6 hours until 4 days have passed since the first
# failed delivery.

# Please note that these rules only limit the frequency of retries, the
# effective retry-time depends on the frequency of queue-running, too.
# See QUEUEINTERVAL in /etc/default/exim4.

# Address or Domain    Error       Retries
# -----------------    -----       -------

*                      *           F,2h,15m; G,16h,1h,1.5; F,4d,6h

#####################################################
### end retry/30_exim4-config
#####################################################
#####################################################
### rewrite/00_exim4-config_header
#####################################################

######################################################################
#                      REWRITE CONFIGURATION                         #
######################################################################

begin rewrite

#####################################################
### end rewrite/00_exim4-config_header
#####################################################
#####################################################
### rewrite/31_exim4-config_rewriting
#####################################################

### rewrite/31_exim4-config_rewriting
#################################

# This rewriting rule is particularly useful for dialup users who
# don't have their own domain, but could be useful for anyone.
# It looks up the real address of all local users in a file
.ifndef NO_EAA_REWRITE_REWRITE
*@+local_domains "${lookup{${local_part}}lsearch{/etc/email-addresses}\
                   {$value}fail}" Ffrs
# identical rewriting rule for /etc/mailname
*@ETC_MAILNAME "${lookup{${local_part}}lsearch{/etc/email-addresses}\
                   {$value}fail}" Ffrs
.endif


#####################################################
### end rewrite/31_exim4-config_rewriting
#####################################################
#####################################################
### auth/00_exim4-config_header
#####################################################

######################################################################
#                   AUTHENTICATION CONFIGURATION                     #
######################################################################

begin authenticators


#####################################################
### end auth/00_exim4-config_header
#####################################################
#####################################################
### auth/30_exim4-config_examples
#####################################################

### auth/30_exim4-config_examples
#################################

# The examples below are for server side authentication, when the
# local exim is SMTP server and clients authenticate to the local exim.

# They allow two styles of plain-text authentication against an
# CONFDIR/passwd file whose syntax is described in exim4_passwd(5).

# Hosts that are allowed to use AUTH are defined by the
# auth_advertise_hosts option in the main configuration. The default is
# "*", which allows authentication to all hosts over all kinds of
# connections if there is at least one authenticator defined here.
# Authenticators which rely on unencrypted clear text passwords don't
# advertise on unencrypted connections by default. Thus, it might be
# wise to set up TLS to allow encrypted connections. If TLS cannot be
# used for some reason, you can set AUTH_SERVER_ALLOW_NOTLS_PASSWORDS to
# advertise unencrypted clear text password based authenticators on all
# connections. As this is severely reducing security, using TLS is
# preferred over allowing clear text password based authenticators on
# unencrypted connections.

# PLAIN authentication has no server prompts. The client sends its
# credentials in one lump, containing an authorization ID (which we do not
# use), an authentication ID, and a password. The latter two appear as
# $auth2 and $auth3 in the configuration and should be checked against a
# valid username and password. In a real configuration you would typically
# use $auth2 as a lookup key, and compare $auth3 against the result of the
# lookup, perhaps using the crypteq{}{} condition.

# plain_server:
#   driver = plaintext
#   public_name = PLAIN
#   server_condition = "${if crypteq{$auth3}{${extract{1}{:}{${lookup{$auth2}lsearch{CONFDIR/passwd}{$value}{*:*}}}}}{1}{0}}"
#   server_set_id = $auth2
#   server_prompts = :
#   .ifndef AUTH_SERVER_ALLOW_NOTLS_PASSWORDS
#   server_advertise_condition = ${if eq{$tls_in_cipher}{}{}{*}}
#   .endif

# LOGIN authentication has traditional prompts and responses. There is no
# authorization ID in this mechanism, so unlike PLAIN the username and
# password are $auth1 and $auth2. Apart from that you can use the same
# server_condition setting for both authenticators.

# login_server:
#   driver = plaintext
#   public_name = LOGIN
#   server_prompts = "Username:: : Password::"
#   server_condition = "${if crypteq{$auth2}{${extract{1}{:}{${lookup{$auth1}lsearch{CONFDIR/passwd}{$value}{*:*}}}}}{1}{0}}"
#   server_set_id = $auth1
#   .ifndef AUTH_SERVER_ALLOW_NOTLS_PASSWORDS
#   server_advertise_condition = ${if eq{$tls_in_cipher}{}{}{*}}
#   .endif
#
# cram_md5_server:
#   driver = cram_md5
#   public_name = CRAM-MD5
#   server_secret = ${extract{2}{:}{${lookup{$auth1}lsearch{CONFDIR/passwd}{$value}fail}}}
#   server_set_id = $auth1

# Here is an example of CRAM-MD5 authentication against PostgreSQL:
#
# psqldb_auth_server:
#   driver = cram_md5
#   public_name = CRAM-MD5
#   server_secret = ${lookup pgsql{SELECT pw FROM users WHERE username = '${quote_pgsql:$auth1}'}{$value}fail}
#   server_set_id = $auth1

#Authenticate against local passwords using sasl2-bin
# Requires exim_uid to be a member of sasl group, see README.Debian.gz
 plain_saslauthd_server:
   driver = plaintext
   public_name = PLAIN
   server_condition = ${if saslauthd{{$auth2}{$auth3}}{1}{0}}
   server_set_id = $auth2
   server_prompts = :
   .ifndef AUTH_SERVER_ALLOW_NOTLS_PASSWORDS
   server_advertise_condition = ${if eq{$tls_in_cipher}{}{}{*}}
   .endif

# login_saslauthd_server:
#   driver = plaintext
#   public_name = LOGIN
#   server_prompts = "Username:: : Password::"
#   # don't send system passwords over unencrypted connections
#   server_condition = ${if saslauthd{{$auth1}{$auth2}}{1}{0}}
#   server_set_id = $auth1
#   .ifndef AUTH_SERVER_ALLOW_NOTLS_PASSWORDS
#   server_advertise_condition = ${if eq{$tls_in_cipher}{}{}{*}}
#   .endif
#
# ntlm_sasl_server:
#   driver = cyrus_sasl
#   public_name = NTLM
#   server_realm = <short main hostname>
#   server_set_id = $auth1
#   .ifndef AUTH_SERVER_ALLOW_NOTLS_PASSWORDS
#   server_advertise_condition = ${if eq{$tls_in_cipher}{}{}{*}}
#   .endif
# 
# digest_md5_sasl_server:
#   driver = cyrus_sasl
#   public_name = DIGEST-MD5
#   server_realm = <short main hostname>
#   server_set_id = $auth1
#   .ifndef AUTH_SERVER_ALLOW_NOTLS_PASSWORDS
#   server_advertise_condition = ${if eq{$tls_in_cipher}{}{}{*}}
#   .endif

# Authentcate against cyrus-sasl
# This is mainly untested, please report any problems to
# pkg-exim4-users@lists.alioth.debian.org.
# cram_md5_sasl_server:
#   driver = cyrus_sasl
#   public_name = CRAM-MD5
#   server_realm = <short main hostname>
#   server_set_id = $auth1
#
# plain_sasl_server:
#   driver = cyrus_sasl
#   public_name = PLAIN
#   server_realm = <short main hostname>
#   server_set_id = $auth1
#   .ifndef AUTH_SERVER_ALLOW_NOTLS_PASSWORDS
#   server_advertise_condition = ${if eq{$tls_in_cipher}{}{}{*}}
#   .endif
#
# login_sasl_server:
#   driver = cyrus_sasl
#   public_name = LOGIN
#   server_realm = <short main hostname>
#   server_set_id = $auth1
#   .ifndef AUTH_SERVER_ALLOW_NOTLS_PASSWORDS
#   server_advertise_condition = ${if eq{$tls_in_cipher}{}{}{*}}
#   .endif

# Authenticate against courier authdaemon

# This is now the (working!) example from
# http://www.exim.org/eximwiki/FAQ/Policy_controls/Q0730
# Possible pitfall: access rights on /run/courier/authdaemon/socket.
# plain_courier_authdaemon:
#   driver = plaintext
#   public_name = PLAIN
#   server_condition = \
#     ${extract {ADDRESS} \
#               {${readsocket{/run/courier/authdaemon/socket} \
#               {AUTH ${strlen:exim\nlogin\n$auth2\n$auth3\n}\nexim\nlogin\n$auth2\n$auth3\n} }} \
#               {yes} \
#               fail}
#   server_set_id = $auth2
#   .ifndef AUTH_SERVER_ALLOW_NOTLS_PASSWORDS
#   server_advertise_condition = ${if eq{$tls_in_cipher}{}{}{*}}
#   .endif

# login_courier_authdaemon:
#   driver = plaintext
#   public_name = LOGIN
#   server_prompts = Username:: : Password::
#   server_condition = \
#     ${extract {ADDRESS} \
#               {${readsocket{/run/courier/authdaemon/socket} \
#               {AUTH ${strlen:exim\nlogin\n$auth1\n$auth2\n}\nexim\nlogin\n$auth1\n$auth2\n} }} \
#               {yes} \
#               fail}
#   server_set_id = $auth1
#   .ifndef AUTH_SERVER_ALLOW_NOTLS_PASSWORDS
#   server_advertise_condition = ${if eq{$tls_in_cipher}{}{}{*}}
#   .endif

# This one is a bad hack to support the broken version 4.xx of
# Microsoft Outlook Express which violates the RFCs by demanding
# "250-AUTH=" instead of "250-AUTH ".
# If your list of offered authenticators is other than PLAIN and LOGIN,
# you need to adapt the public_name line manually.
# It has to be the last authenticator to work and has not been tested
# well. Use at your own risk.
# See the thread entry point from
# http://www.exim.org/mail-archives/exim-users/Week-of-Mon-20050214/msg00213.html
# for the related discussion on the exim-users mailing list.
# Thanks to Fred Viles for this great work.

# support_broken_outlook_express_4_server:
#   driver = plaintext
#   public_name = "\r\n250-AUTH=PLAIN LOGIN"
#   server_prompts = User Name : Password
#   server_condition = no
#   .ifndef AUTH_SERVER_ALLOW_NOTLS_PASSWORDS
#   server_advertise_condition = ${if eq{$tls_in_cipher}{}{}{*}}
#   .endif

##############
# See /usr/share/doc/exim4-base/README.Debian.gz
##############

# These examples below are the equivalent for client side authentication.
# They get the passwords from CONFDIR/passwd.client, whose format is
# defined in exim4_passwd_client(5)

# Because AUTH PLAIN and AUTH LOGIN send the password in clear, we
# only allow these mechanisms over encrypted connections by default.
# You can set AUTH_CLIENT_ALLOW_NOTLS_PASSWORDS to allow unencrypted
# clear text password authentication on all connections.

cram_md5:
  driver = cram_md5
  public_name = CRAM-MD5
  client_name = ${extract{1}{:}{${lookup{$host}nwildlsearch{CONFDIR/passwd.client}{$value}fail}}}
  client_secret = ${extract{2}{:}{${lookup{$host}nwildlsearch{CONFDIR/passwd.client}{$value}fail}}}

# this returns the matching line from passwd.client and doubles all ^
PASSWDLINE=${sg{\
                ${lookup{$host}nwildlsearch{CONFDIR/passwd.client}{$value}fail}\
	        }\
	        {\\N[\\^]\\N}\
	        {^^}\
	    }

plain:
  driver = plaintext
  public_name = PLAIN
.ifndef AUTH_CLIENT_ALLOW_NOTLS_PASSWORDS
  client_send = "<; ${if !eq{$tls_out_cipher}{}\
                    {^${extract{1}{:}{PASSWDLINE}}\
		     ^${sg{PASSWDLINE}{\\N([^:]+:)(.*)\\N}{\\$2}}\
		   }fail}"
.else
  client_send = "<; ^${extract{1}{:}{PASSWDLINE}}\
		    ^${sg{PASSWDLINE}{\\N([^:]+:)(.*)\\N}{\\$2}}"
.endif

login:
  driver = plaintext
  public_name = LOGIN
.ifndef AUTH_CLIENT_ALLOW_NOTLS_PASSWORDS
  # Return empty string if not non-TLS AND looking up $host in passwd-file
  # yields a non-empty string; fail otherwise.
  client_send = "<; ${if and{\
                          {!eq{$tls_out_cipher}{}}\
                          {!eq{PASSWDLINE}{}}\
                         }\
                      {}fail}\
                 ; ${extract{1}{::}{PASSWDLINE}}\
		 ; ${sg{PASSWDLINE}{\\N([^:]+:)(.*)\\N}{\\$2}}"
.else
  # Return empty string if looking up $host in passwd-file yields a
  # non-empty string; fail otherwise.
  client_send = "<; ${if !eq{PASSWDLINE}{}\
                      {}fail}\
                 ; ${extract{1}{::}{PASSWDLINE}}\
		 ; ${sg{PASSWDLINE}{\\N([^:]+:)(.*)\\N}{\\$2}}"
.endif
#####################################################
### end auth/30_exim4-config_examples
#####################################################
```
</details>



```
echo "tls_on_connect_ports = 465" > /etc/exim4/exim4.conf.localmacros
```

```
nano /etc/default/exim4
```

```
# /etc/default/exim4
EX4DEF_VERSION=''

# 'combined' -	 one daemon running queue and listening on SMTP port
# 'no'       -	 no daemon running the queue
# 'separate' -	 two separate daemons
# 'ppp'      -   only run queue with /etc/ppp/ip-up.d/exim4.
# 'nodaemon' - no daemon is started at all.
# 'queueonly' - only a queue running daemon is started, no SMTP listener.
# setting this to 'no' will also disable queueruns from /etc/ppp/ip-up.d/exim4
QUEUERUNNER='combined'
# how often should we run the queue
QUEUEINTERVAL='30m'
# options common to quez-runner and listening daemon
COMMONOPTIONS=''
# more options for the daemon/process running the queue (applies to the one
# started in /etc/ppp/ip-up.d/exim4, too.
QUEUERUNNEROPTIONS=''
# special flags given to exim directly after the -q. See exim(8)
QFLAGS=''
# Options for the SMTP listener daemon. By default, it is listening on
# port 25 only. To listen on more ports, it is recommended to use
# -oX 25:587:10025 -oP /run/exim4/exim.pid
SMTPLISTENEROPTIONS='-oX 25:465:587:10025 -oP /run/exim4/exim.pid'
```


```
cd /etc/skel
mkdir Maildir && cd Maildir && maildirmake.dovecot .
systemctl enable saslauthd exim4 dovecot
systemctl restart saslauthd exim4 dovecot
systemctl status saslauthd exim4 dovecot
systemctl status saslauthd exim4 dovecot
```






