# ESXi:


### Criar 3 máquinas, um linux que se vai ligar à net e a uma dmz, e um cliente windows

### ----------------------------------

### Criar switches virtuais para a dmz e para o inside, e criar prot groups para ambos

### -------------------------

#### THIN disk provision- Cria um disco com mais memoria do que e possivel. Por exemplo, cada pessoa na sala pode usar um disco THIN de 1 terra. No total dá 10 terras. O disco só tem 5 terras. Se cada um nao usar de mais, consegue.

#### THICK- lazy zeroed nao apaga a data que ja esta no disco, o eagered apaga tudo de uma vez.


## -----------------------------

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

