# ESXi:


### Criar 3 máquinas, um linux que se vai ligar à net e a uma dmz, e um cliente windows


### Criar switches virtuais para a dmz e para o inside, e criar prot groups para ambos


#### THIN disk provision- Cria um disco com mais memoria do que e possivel. Por exemplo, cada pessoa na sala pode usar um disco THIN de 1 terra. No total dá 10 terras. O disco só tem 5 terras. Se cada um nao usar de mais, consegue.


#### THICK- lazy zeroed nao apaga a data que ja esta no disco, o eagered apaga tudo de uma vez.



<RTR_1:

4 GB de armazenamento

768 de memoria

thin provisioned 

SSD image of debian 10

tres network interfaces, 192 facing outwards com IP 192.168.15.0/24

224 para inside windows com IP 192.168.31.0/24

256 para dmz com IP 172.31.0.0/24


During installation, debian disk was partitioned with LV>

