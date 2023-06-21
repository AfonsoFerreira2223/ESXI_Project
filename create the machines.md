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

![Alt Text](https://lh3.googleusercontent.com/pw/AJFCJaWHFSZ1SdLR9CnxOCZDpUnc9Mix6LEust3vyG9b6mGUrTTLV3mbgmyHwLissxOfp9z4k09amwbjK-QTn04rOCR7dFUSG4ubpxsBhJbXQUEmoBcXmL6F0eKI0Zo62OCA_BfaayEFY2PELp9mcoKPQX-yviiTadYlKDx5Ip6bQT6HmDYG58WPbiSZ0Fr2n3MzN1B2DCMu8KjxNfR76ZmrIKGSfuig00mptSfprI2YXZH5xnjHDNxO2-g0vgKg28gtJu5ve5xtKVzBc-Ab_r6igf796xSfjhAozIuROlo2w-MZbXEpG1teSAfehsC3-0j-PhHA0XPkTRrUBVN2vnHz9sCnT2TYKuFBFdCQZnRbovX-aMT7WXX3rMvFHdlWdNEv9Zjj8uOQ2pDFv23lg_VmWBmnIDu2PDDPm8acwH13dYBr0yyinwwSz-AbKAzKucO7OgE02ZA2nZBNGjr4o_SpMoIuWcRvuQcXS3RjeeYHVNlSyS88a-nhAu-M5HguOXfbJIdp8HLeHcvlqx3GQhj08i18eUKCE7o5V5ETSLjn5lyBpohF4QOMEAo3CgmrNdZ2xVh3LGflzio9IcPQoGDWWkbGBy1g2ZgYWXKjF0RW9T353Y-0n_8ZIyKFkY-O6zq2riKZr0A74u_u1b2Wfb5haAaGQDsGUICSf7H8H4C4Gn8pabneyBTej0hUEdcuLLTDhxnpicBA12qx2Jwv_iJp_mRitbR6AV0OFGYYjJP_vyiqAxrk2XGLGEpqyohQghoZorQkmyCLU7fhrerNKIHf0eIa-BX5We8nSmZ1yoZTdPMGNLz3szavNr7li4qErRB-QVB3YAMPffjyyTa4II9-VzzKQ_ip3fMlIB4ekwEYAaTkkp_04REHAYxjRbL3eo6ddNESQb8HdBjXsw8cS88Osh0i=w1440-h583-no?authuser=0)



#### > DMZ port group (dmz topology) :

![Alt Text](https://lh3.googleusercontent.com/pw/AJFCJaXL_9901NA9kN6z8QJbh4xQ7MBS6V8EA6XFQ-J7WdJuI2MN6t82XJneadGHRKcGeJ_H-eZ8m2EwrvC1H8ZMZWjBapUDrrKr5BQcaEHXr7KsxJsVm9QC3Tq6yrKgFzR4xjWPKhlAbCG68XW32blnNFJTowIRyiqTQEgDPp1VagXGGEYteg3op-Nvud1b285aQ4FRBPlLOzWUrbfN1I0nS9wUebTMxsp0TZlHDDAVB8Jewoc5cy486knKwivLhMoA_zgputB6_b22x7Z6WANKRWi7atojQwlhq6zIe8RI6uFy6IGGgknNTQDg_dNMTeBeh_t9fQCFw_vqL0In_yQc3YLMab7jQfxQ9B84L-fwXPp7ea-kX4L5_43u6SVaGa4m74thXDaFwbGWSODByCJ3UIp97Ynvow4aY16tRVTTFRNgn7Hy3h_MMnp_qMSHGyWMANRtuOXOjzzuziJKXT3QIhrMxxy_QACggp45nM-w8ahNVJgiY4GEHZqNloL9zbdR52d68i2BCFqdFee8FAtNyvM31yLHzdhd52yflGcSk1SwlE0uvrwRrQGddkJ14KJac3AVYq0ubQ3WX4GbplXJpJUVuMj99G1EFRZjSZrff4CuybNbKmaEP2OjPQ7X_Emj42uZqwiDkyDNS6aXSKjhHGKaCR4MdcHxU3XDfY07WCv9rWwIaA3rENC0u1PMUt2psfMo3SQiXtDiX26HkiDDOXsYTVZS3pp6kWcVE9-cWvRCAX1snyfZSxuKhbxm6zZWIiiL55VaLEXvuIaNuguVzY3fFIMebPRuF1WwuhkTdXLRhNPNKXfYyypbuyfz0r5V5HDf1a3yG2RN_dHeQtw6gqPSyTKhKeIqZ9kM54vApMG6E9qBQCduPhPWllnOcN8ipxSdo86IxoDvDzlOdbykCmvO=w1542-h624-s-no?authuser=0)



#### > Router port group (outside topology with internet access) :

![Alt Text](https://lh3.googleusercontent.com/pw/AJFCJaVt9_dtaVenLFN5oWTMGFQMlg-bkyKt93sS5bNIevCFe0OzKHF5t47IVGHB0tXJxzRA96EE-wE9Z5loZANkbVvMNNeYb-38_7p7aLIUbZAoJUSEvid4CHkxaTvQZHgM6OfrcsLbKYP8rn6eMnN1C8ODVrKW_LSsJ_n0Gf068vQ7e1LgDpwEavVFd3O9YDClFgsxTZzjk5__k2J7caWSzDwpvEBYk5m4DL7kq6t_QoEBLVoHDGeTRa_anFaLSP5ZV7dVFg6b_QKzv4Wrgcc7WmhSbN0-PmZydfW3m7b-OQD2EL1JkMCXTUbPg7_PBuTvGcBVhcLOoglBVd_LB3luMkiRosZqqOD7YNtxrUOxNRGJ129RRj1H5ug50FFhHqvOrJ7dleI0AUN3K1xSbS9t8jNuvRHxiM_0PbrXxT0Pw_uft01hQ_jDbgbql9AMRIECaPX1ElxkvRfDDQ37nmyLNnKKRU48_tS5ImTrOwbS7fxK5bTyiFQ1sEaOYl5ULTRzeF8k9v10lUFR0iU4qEhKtj2VYxGVx5xIk4wkjHcRQAhPQyPZYaYi8z9dh1k8uL1BkVyNkSus1jOGbocWe-MSEXoZgC9_pFj6lIpOBojc-w4MS7qaZLvWg_1_h7Sn6w23MfZsdDbVXWU0WyjnUQ19qTpz1M8XMf6YsgJ-7YB4CibewatocHXJ1h5rXmng576fah5YaeD6nt1ma61zSsIk0AnotWOE4ryKlRtDfiutgYL11SrvDTfaNDfLHaYhIcGsM_C6a9XRQEAB6gQV2g0JbN4fcJRMHOMfnL0y1aY-AUsf0pHbe87ou9sy4MPZ-D6TqlkbqH51tZC5A1OtDZ3SYJiCSdw7Jf6F20eHERcK_e0vrv5DVgzMqbaWvVMiy_xhLv3taGc3gtu3b3FBuhYEIOBz=w1542-h624-s-no?authuser=0)


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


