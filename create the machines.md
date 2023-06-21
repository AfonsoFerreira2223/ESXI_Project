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

### Create the Switches and port groups on VMware ESXi according to what your network will look like

#### > Router:

![Alt Text](https://lh3.googleusercontent.com/pw/AJFCJaUe2bbpxypdEKJm4LjCNIfU5vouqyE99oHAglvrP02uhxjB6dVOl5BDt7fXHoYzA5WzCzi_GMCivrkeMklyhzVT8IiAm6FAri7DRvXy3dY9gaIc4l-bFJiNKSV6XO94S-yI1J-zTc-0y1PJaRf3zn_L4kvBSb5_gk1zXaanqCed9OpVIVPutim9atJ0qIFssree4Np6oaSQz3FHHM9xcIH4OKAdUgrcwg-nC_3OqHjToDAT-XywdKRFQzf94Q3xqM_604zUpJa2I8IHC6NgsbrODTZXJj47o5vwlo2_LwVEb0cjDqiQAJ15geiOTNy14PHgUPlxiuK6iEc2VXxD4tFjYWx3hzg5bdAxdI1qHZcF1BikxIhe_ZgOfDOJR9VHd5ZM5EU8UhOIr1s7WGB8SKPEhvNVHmAG3qHSJZnoOtb5kGLHq62ySVnj-vhlTIfCc410hBHytwOYN4L9mod_fEyvqRCP2YiR454Va3sJ0aSj9JP9LoNQxGYCVSEYQO8gBv5bR84g6OB1bVxTnN_nPx7T1j7DhhzIKsx8yK66uvT1KJ_tMbSZFh1_BTNuwjY4RR68H0Oy71FRdqtEAgCgWvvorHWPEwQpuEgCn7xvtyVDpDLOoXlEg4zWjCncG8uu8TSJGyfUJFQTA_UK_qnW9TVvhBWgb7KLrjj7zzCoL1xt21CwT-28V5bZCv9DJlj6BiJfDpxJm_jsjsdVIA3RmozGed9rl3V9APfJEDIgh7gzO7Y4u_DRF6-m4GiSkMIfShyCXOmE0tSt8ih7uaZ91vOsJYhUR3N9jcXzpAUbfuqrBI9fYxpWn_UksiKfRkoOfLB_CIqoeJUpoGarDhJMRdqCtjJIH-Y5JcTFXf5GkmIV_u0_JhVpK-QvJ8YO00aY3H1B9-CXMJX7H-nTNDtgaSlh=w1542-h624-s-no?authuser=0)




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




