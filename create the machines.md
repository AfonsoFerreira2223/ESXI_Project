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

### Criar switches virtuais para a dmz e para o inside, e criar prot groups para ambos

## ----------------------------

### Diferença entre thing provision e thick provision:


HIN disk provision- Cria um disco com mais memoria do que e possivel. Por exemplo, cada pessoa na sala pode usar um disco THIN de 1 terra. No total dá 10 terras. O disco só tem 5 terras. Se cada um nao usar de mais, consegue.

THICK- lazy zeroed nao apaga a data que ja esta no disco, o eagered apaga tudo de uma vez.


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




