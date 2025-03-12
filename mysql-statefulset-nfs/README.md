## Configurando MySQL/MariaDB em um StatefulSet com os dados persistidos em um storage NFS
- Passo 1 - Configurando e instalando o servidor NFS
```
sudo apt update -y
sudo apt upgrade -y
sudo apt install -y nfs-kernel-server
```
<i>Não esquecer de instalar as dependencias para conexão ao nfs em seus workers com:</i>
```
sudo apt install -y nfs-common
```
<b>Configurando o servidor NFS, arquivo /etc/exports</b>
<br>
O ip do meu servidor é: 192.168.122.195 e minha subrede é 192.168.122.0/24, ajuste de acordo com os dados da sua rede
<br>
<i>Na minha configuração existem pontos de montagem tanto para o MySQL quanto para o MariaDB faça de acordo com suas necessidades</i>
```
/srv/nfs            192.168.122.0/24(rw,sync,no_subtree_check,fsid=0)
/srv/nfs/mysql      192.168.122.0/24(rw,sync,no_subtree_check,no_root_squash)
/srv/nfs/mariadb    192.168.122.0/24(rw,sync,no_subtree_check,no_root_squash)
/srv/nfs/postgres   192.168.122.0/24(rw,sync,no_subtree_check,no_root_squash)
/srv/nfs/storage    192.168.122.0/24(rw,sync,no_subtree_check)
```
<i>Nesse caso existe 2 informações que são importantes freezar</i>
<br>
A primeira é o uso do fsid=0, que serve para definir o nosso root ou raiz do compartilhamento, então quando for montar vc não precisa informar todo o caminho para o ponto de montagem apenas /nome_do_ponto_de_montagem, por exemplo se formos montar a pasta storage:</i>
<br>
<b>Com fsid=0</b>
```
mount -t nfs 192.168.122.195:/storage /mnt/storage
```
<b>Sem o fsid=0</b>
```
mount -t nfs 192.168.122.195:/srv/nfs/storage /mnt/storage
```
A segunda informação é o uso do no_root_squash, por padrão o nfs vai gravar todos os dados como sendo o dono: nobody e o grupo: nogroup e quem está acessando a pasta não tem a permissão de alterar isso, porém com essa opção o MySQL vai poder gravar a pasta /var/lib/mysql como sendo o dono o usuário mysql e o grupo mysql, oq é um exigência das configurações do MySQL, a não ser que você crie sua prórpia imagem mundando o UID e GID do usuário mysql</i>
<br>
Agora vamos criar as pastas e fazer o bind
```
sudo mkdir -p /srv/nfs/mysql /data/mysql
sudo mkdir -p /srv/nfs/mariadb /data/mariadb
sudo mkdir -p /srv/nfs/postgres /data/postgres
sudo mkdir -p /srv/nfs/storage /data/storage
sudo chown nobody:nogroup /data/storage
```
```
sudo mount --bind /data/mysql /srv/nfs/mysql
sudo mount --bind /data/mariadb /srv/nfs/mariadb
sudo mount --bind /data/postgres /srv/nfs/postgres
sudo mount --bind /data/storage /srv/nfs/storage
```
<i>Para deixar esse processo automatico não se esqueça de adicionar os bind no seu fstab</i>
```
/data/mysql     /srv/nfs/mysql      none    bind    0   0
/data/mariadb   /srv/nfs/mariadb    none    bind    0   0
/data/postgres  /srv/nfs/postgres   none    bind    0   0
/data/storage   /srv/nfs/storage    none    bind    0   0
```
Agora basta carregar as configurações do nfs e reiniciar o servidor
```
sudo exportfs -arv
sudo systemctl restart nfs-kernel-server
```
- Passo 02 - Criando os PersistentVolume para os nossos compartilhamentos.
<br>
<i>Eu irei criar o apenas um PV porém basta repitir o processo para cada um dos compartilhamentos que você tenha.</i>

```
# pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-nfs
spec:
  capacity:
    storage: 2Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Delete
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /mysql
    server: 192.168.122.195
```

<i>Eu irei usar um PersistentVolumeClaim para esse exemplo, porém nada te impede de usar StorageClass</i>
```
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  resources:
    requests:
      storage: 2Gi
  volumeMode: Filesystem
  volumeName: mysql-nfs
  accessModes:
    - ReadWriteMany
```

- Passo 03 - Criado um StatefulSet
```
# mysql.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  serviceName: mysql
  replicas: 2
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:latest
        ports:
        - containerPort: 3306
          name: mysql
        envFrom:
          - secretRef:
              name: mysql-secret
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: mysql-pvc
```
