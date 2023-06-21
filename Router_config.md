# Router configuration:


> su -

Go to root as most commands here require high-level permissions

## -----------------------------

> apt update
> apt install resolvconf

If it asks to insert the disk:

Go to esxi- edit- then pick the connect box on the datasotre iso section (this is where the package is included)


![image](https://github.com/AfonsoFerreira2223/ESXI_Project/assets/114146560/1a186dad-89c3-46d4-a66b-09bd58600d5a)


## -----------------------------


Configuring the interfaces on router:


> nano /etc/network/interfaces

CTRL K all

copy paste this, after altering the Ip address to suit your defined ranges:





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


> ip a

THe result should still be empty

> systemctl restart networking

> reboot

Wait a little and reconnect. All interfaces should have their assigned IPs now.
