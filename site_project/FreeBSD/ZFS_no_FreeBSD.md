# ZFS no FreeBSD

## Preparação dos discos

Alinhar setores de 512 para 4096 bytes.

$ sysctl vfs.zfs.min_auto_ashift=12

Pode ser adicionado ao /etc/sysctl.conf

## Criar partições GPT nos discos.

Criar tabela GPT do disco da0. Adaptar para cada disco, da1, da2, etc…

$ gpart create -s gpt da0

Se precisar de disco de swap ZFS. sw0 é o label do disco e o -t indica o tipo de função como swap.

$ gpart add -a 1m -s1g -l sw0 -t freebsd-swap da0

Criar discos de dados normais. O zfs0 é o label do disco e deve ser explicativo, geralmente indicando posição física ou serial number ou função.

$ gpart add -a 1m -l zfs0 -t freebsd-zfs da0

Verificar discos e tabelas, por exemplo discos da0

$ gpart show -l da0


## Criação de pools e vdevs

Checar os pools com comando zpool status. Ele mostrará estado de cada zpool se o nome não especificado.

Para criar pools usa-se o comando zpool create seguido do nome do pool e se necessário o(s) tipo(s) de vdevs e lista de discos. Palavras chaves para os tipos de vdevs são:

•	Em branco: cria up stripped-array sem redundância. Usado para operações temporárias, scratch disk.
•	mirror: espelho com dois ou mais discos similar ao RAID1
•	raidz1 a raidz3: vdevs com discos redundantes similares ao RAID5 ou RAID6 com um, dois ou três discos de paridade respectivamente.
•	log: vdevs para ZFS intent logs or ZILs, em geral discos mais rápidos para gravação. Podem ser combinados com palavras-chave de redundância como mirror ou RAIDZ1. Exemplo mais comum é usar discos SSD para aumentar velocidade de gravação para arrays grandes de discos magnéticos comuns.
•	cache: usado para armazenagem de dados mais acessados como cache. Em geral disco SSD (ou na RAM?)

As palavras chaves podem ser combinadas para criar um vdev de cache com mirror ou invés de stripped para mais segurança na gravação dos dados. Exemplos.

$ zpool create BIGARRAY raidz1 gpt/zfs0 gpt/zfs1 gpt/zfs2 log mirror gpt/zil0 gpt/zil1 cache gpt/cache0

Este comando criará um zpool com um VDEV de 3 discos (ou storage providers) em um array tipo RAID 5, um VDEV de log com array espelhado tipo RAID 1 com dois discos (mas poderia haver mais) e um VDEV de cache com um array stripped de um disco só.

Para rever informações do zpool, use o comando zpool status seguido do nome do pool. Por exemplo um pool com 5 discos num VDEV RAIDZ3 (contém 3 discos de paridade) e dois de log e um de cache seria:

```
$ zpool status BIGARRAY

  pool: BIGARRAY
 state: ONLINE
  scan: none requested
config:

NAME          STATE     READ WRITE CKSUM
BIGARRAY      ONLINE       0     0     0
  raidz3-0    ONLINE       0     0     0
gpt/zfs1  ONLINE       0     0     0
gpt/zfs2  ONLINE       0     0     0
gpt/zfs3  ONLINE       0     0     0
gpt/zfs4  ONLINE       0     0     0
gpt/zfs5  ONLINE       0     0     0
logs
  mirror-1    ONLINE       0     0     0
gpt/zil1  ONLINE       0     0     0
gpt/zil2  ONLINE       0     0     0
cache
  gpt/cache1  ONLINE       0     0     0

errors: No known data errors
$
```


 Você pode adicionar mais de um VDEV em cada pool para armazenagem, mas nunca retirar um deles do pool. Para adicionar VDEV use o comando zpool add.

$ zpool add BIGARRAY RAIDZ3 gpt/zfs11 gpt/zfs12 gpt/zfs13 gpt/zfs14 gpt/zfs15

Este comando adicionará um VDEV tipo RAIDZ3, array com 3 storage providers de redundância, ao pool BIGARRAY. Note que o ZFS impede que diferentes tipos de VDEV sejam adicionados por engano, como por exemplo, um tipo mirror num pool com outro tipo de VDEV. Isto serve para evitar enganos que tornariam o pool desbalanceado com tipos diferentes de VDEV já que os dados são distribuídos entre eles. Entretanto é possível forçar a criação usando-se a  chave ‘-f’. Em produção isto não deve ser usado.

Para ver mais detalhes do pool:

```bash
$ zpool get all BIGARRAY
NAME      PROPERTY                       VALUE                          SOURCE
BIGARRAY  size                           99.5G                          -
BIGARRAY  capacity                       0%                             -
BIGARRAY  altroot                        -                              default
BIGARRAY  health                         ONLINE                         -
BIGARRAY  guid                           14649615821998128421           default
BIGARRAY  version                        -                              default
BIGARRAY  bootfs                         -                              default
BIGARRAY  delegation                     on                             default
BIGARRAY  autoreplace                    off                            default
BIGARRAY  cachefile                      -                              default
BIGARRAY  failmode                       wait                           default
BIGARRAY  listsnapshots                  off                            default
BIGARRAY  autoexpand                     off                            default
BIGARRAY  dedupditto                     0                              default
BIGARRAY  dedupratio                     1.00x                          -
BIGARRAY  free                           99.5G                          -
BIGARRAY  allocated                      1.69M                          -
BIGARRAY  readonly                       off                            -
BIGARRAY  comment                        -                              default
BIGARRAY  expandsize                     -                              -
BIGARRAY  freeing                        0                              default
BIGARRAY  fragmentation                  0%                             -
BIGARRAY  leaked                         0                              default
BIGARRAY  feature@async_destroy          enabled                        local
BIGARRAY  feature@empty_bpobj            enabled                        local
BIGARRAY  feature@lz4_compress           active                         local
BIGARRAY  feature@multi_vdev_crash_dump  enabled                        local
BIGARRAY  feature@spacemap_histogram     active                         local
BIGARRAY  feature@enabled_txg            active                         local
BIGARRAY  feature@hole_birth             active                         local
BIGARRAY  feature@extensible_dataset     enabled                        local
BIGARRAY  feature@embedded_data          active                         local
BIGARRAY  feature@bookmarks              enabled                        local
BIGARRAY  feature@filesystem_limits      enabled                        local
BIGARRAY  feature@large_blocks           enabled                        local
BIGARRAY  feature@sha512                 enabled                        local
BIGARRAY  feature@skein                  enabled                        local
$
```
[^1]: Escrito originalmente em 25/7/2017.


