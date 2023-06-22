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

Once you are here, copy paste this (alter the Ip addresses to suit your defined ranges):



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


> systemctl restart networking
> 
> reboot
>ip a

Your interfaces should now have the respective IP add assigned to them

![image](https://mail.google.com/mail/u/0?ui=2&ik=c3415e6fdd&attid=0.1&permmsgid=msg-a:r-1599343331034313875&th=188e28f4a2556c86&view=fimg&fur=ip&sz=s0-l75-ft&attbid=ANGjdJ_zx_ShBmvvrA40T7rj3MnuZ9uBqKvPEX8sbeVEWxFPSysDbMAqDuNKkbYoFO4CnHcvZnnfeEtEEXvvleqP935zQRt0cKlVWgjAKAPzU_mkPxnK4qfSQnNkjsc&disp=emb&realattid=ii_lj6z6zcm0)
