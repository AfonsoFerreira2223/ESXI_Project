## Just like in the router, run the commands

```
apt-cdrom add
apt update
apt upgrade
```


## First of all, make sure that the DMZ can request a DHCP IP address from your router

```
dhclient -r
dhclient -v
ip a
```

## Create the web page

```
apt install apache2
```

After installing apache, you can check what your IP address is with

```
hostname -I
```

