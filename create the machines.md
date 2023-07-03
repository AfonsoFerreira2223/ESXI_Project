# ESXi:

## This project aims to create a virtual network with working DNS, DHCP, and Email servers. NFTABLES/IPTABLES will be used to filter traffic

> If you encounter an error during configurations, consider disabling the firewalls for the time being (definetly not best practice)

### Criar 4 máquinas:

> 1 Linux that will act as the router
> 
> 1 Linux that will act as a server on a DMZ
> 
> 1 windows that will be an inside machine
> 
> 1 more Linux router that will act as a failover for DHCP and DNS services

## ----------------------------

### Create the port groups on VMware ESXi according to what your network will look like

#### > Windows port group (inside topology) :

![image](https://github.com/AfonsoFerreira2223/ESXI_Project/assets/114146560/e4a4f292-504b-412e-9662-f563233ab30d)



#### > DMZ port group (dmz topology) :

![image](https://github.com/AfonsoFerreira2223/ESXI_Project/assets/114146560/7f0665a6-f4c4-4b73-9871-4306d4fe9f22)



#### > Router port group (outside topology with internet access) :

![image](https://github.com/AfonsoFerreira2223/ESXI_Project/assets/114146560/ba7a9c91-ffaa-455a-8398-649dce7177e6)


### Create two virtual switches as well, and assign them to port groups >inside< and >dmz<


### >outside< is connected to switchv0, which is the one with internet access


## ----------------------------


> <details>
>   <summary>RTR_1 Configuration</summary>
>   
>   - **Storage**: 4 GB of storage
>   - **Memory**: 768 MB
>   - **Provisioning**: Thin provisioned
>   - **Operating System**: Debian 10 (SSD image)
>   
>   **Network Interfaces:**
>   
>   - Interface 1 (facing outward):
>     - IP: 192.168.15.0/24
>     
>   - Interface 2 (inside - Windows network):
>     - IP: 192.168.31.0/24
>     
>   - Interface 3 (DMZ):
>     - IP: 172.31.0.0/24
>     
>   **Disk Partitioning:**
>   
>   During installation, the Debian disk was partitioned with Logical Volume (LV).
>   
> </details>

## ----------------------------

> <details>
>   <summary>DMZ Configuration</summary>
>   
>   - **Storage**: 4 GB of storage
>   - **Memory**: 768 MB
>   - **Provisioning**: Thin provisioned
>   - **Operating System**: Debian 10 (SSD image)
>   
>   **Networking Interface:**
>   
>   - Interface: 256
>   
>   **Disk Partitioning:**
>   
>   Encrypted LV partitioning was used.
>   
> </details>


## ----------------------------


> <details>
>   <summary>Windows Configuration</summary>
>   
>   - **Storage**: 30 GB of storage
>   - **Memory**: 3 GB
>   - **Provisioning**: Thin provisioned
>   - **Operating System**: Windows 10 64-bit Professional
>   
>   **Networking Interface:**
>   
>   - Interface: 224
>   
> </details>



## ----------------------------


> Once the congiruation for the router as ended, it's possible to access its terminal by using some ssh software such as Termius. Don't forget to add port 22 to the firewall rules once they are created.
> 
> Just put the interface IP facing the internet, and username and password.


