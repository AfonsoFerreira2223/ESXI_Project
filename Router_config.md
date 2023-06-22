# Router configuration:


You can go to root as most commands here require high-level permissions

```
su -
```

## -----------------------------

```
apt update
apt install resolvconf
```

It may ask you to insert the CD-ROM in order to download the packages

Go to esxi- edit- then pick the connect box on the datasotre iso section (this is where the package is included)


![image](https://github.com/AfonsoFerreira2223/ESXI_Project/assets/114146560/1a186dad-89c3-46d4-a66b-09bd58600d5a)


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
option domain-name-servers `YOUR IP addresses`;
```

Add the HMAC-MD5 key for secure DHCP updates by inserting the following lines:
```
key DHCP_UPDATER {
    algorithm HMAC-MD5.SIG-ALG.REG.INT;
    secret "pRP5FapFoJ95JEL06sv4PQ==";
};
```

11. Configure DHCP for each subnet by adding the appropriate subnet configuration. Here's an example for `192.168.31.0/24`:
```
subnet 192.168.31.0 netmask 255.255.255.0 {
    range 192.168.31.128 192.168.31.191;
    option routers 192.168.31.1;
    option broadcast-address 192.168.31.255;
}
```
Repeat this step for the `172.31.0.0/24` subnet or any other subnets you need.

12. If you want to assign specific IP addresses to certain hosts, you can add the following lines to the configuration file. Adjust the values accordingly:
```
host 172.31.0.100 {
    hardware ethernet 00:50:56:86:7c:cf;
    fixed-address 172.31.0.100;
    option routers 172.31.0.1;
    option broadcast-address 172.31.0.255;
}
```

13. Open the `/etc/default/isc-dhcp-server` file and add the interfaces you want the DHCP server to listen on. Modify the following line:
```
INTERFACESv4="ens192 ens224 ens256"
```
Include the names of your desired interfaces


