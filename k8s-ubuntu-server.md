## Preparando o ambiente:
```
sudo apt update -y
sudo apt upgrade -y
sudo apt install -y curl git fish jq fonts-powerline
```

## Desabilitar a memória SWAP, comente a linha referente a memoria swap no arquivo /etc/fstab
```
sudo sed -i '/swap/ s/^\(.*\)$/#\1/g' /etc/fstab
sudo swapoff -a

# antes
/swap.img	none	swap	sw	0	0
# depois
#/swap.img	none	swap	sw	0	0
```
## Configuração da rede
## Habilitar o ip forward e bridged
```
echo -e "net.bridge.bridge-nf-call-iptables=1\nnet.bridge.bridge-nf-call-ip6tables=1\nnet.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/k8s.conf
echo -e "overlay\nbr_netfilter" | sudo tee /etc/modules-load.d/k8s.conf
sudo modprobe overlay
sudo modprobe br_netfilter
sudo sysctl --system
```
## Instalando o containerd (opções extra: cri-o)
```
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```
### Usando FISH
```
arch=(dpkg --print-architecture) code=(lsb_release -c -s) echo "deb [arch=$arch signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $code stable" | sudo tee /etc/apt/sources.list.d/docker.list
```

### Usando BASH/ZSH
```
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(lsb_release -c -s) stable" | sudo tee /etc/apt/sources.list.d/docker.list
```

```
sudo apt update -y
sudo apt install -y containerd.io
```
## Configurando o containerd, habilitar o cgroup e alterar a versão da imagem de sandbox
```
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
sudo sed -i 's/sandbox_image = ".*"/sandbox_image = "registry.k8s.io\/pause:3.10"/g' /etc/containerd/config.toml
sudo systemctl restart containerd
```
## Instalando kubernetes
```
sudo apt install -y apt-transport-https gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update -y
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```

## K8S Control Plane (Master) os comandos a seguir deve ser executado apenas no master
### usando fish
```
addr=(ip --json a s enp1s0 | jq -r '.[0].addr_info[0].local') sudo sed -i "s/KUBELET_EXTRA_ARGS=/KUBELET_EXTRA_ARGS=--node-ip=$addr /g" /etc/default/kubelet
addr=(ip --json a s enp1s0 | jq -r '.[0].addr_info[0].local') sudo kubeadm init --apiserver-advertise-address=$addr --pod-network-cidr=10.244.0.0/16 --apiserver-cert-extra-sans=$addr --node-name=$(hostname -s)
```
### Usando bash/zsh | Muita atenção onde tem e onde não tem sudo pois as permissões de arquivos são importantes
```
sudo sudo sed -i "s/KUBELET_EXTRA_ARGS=/KUBELET_EXTRA_ARGS=--node-ip=$(ip --json a s enp1s0 | jq -r '.[0].addr_info[0].local') /g" /etc/default/kubelet
sudo kubeadm init --apiserver-advertise-address=$(ip --json a s enp1s0 | jq -r '.[0].addr_info[0].local') --pod-network-cidr=10.244.0.0/16 --apiserver-cert-extra-sans=$(ip --json a s enp1s0 | jq -r '.[0].addr_info[0].local') --node-name=$(hostname -s)
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl get nodes
```
### Se aparecer a seguinte resposta quer dizer q vc fez tudo certo:
```
NAME         STATUS     ROLES           AGE   VERSION
k8s-master   NotReady   control-plane   86s   v1.32.2
```
## Adicionando um worker
### No master execute o comando:
```
kubeadm token create --print-join-command # copie o código gerado e execute como sudo no worker
```
### Exemplo: sudo kubeadm join 192.168.122.195:6443 --token 584bb5.qnksexedu08pjrtx --discovery-token-ca-cert-hash sha256:155c585eea84d7e23850e5f758a86acd7f470b427ab774f9535327763669b887
### No master execute:
```
kubectl get nodes
```
### Se aparecer esse resultado quer dizer q vc fez tudo certo
```
NAME         STATUS     ROLES           AGE   VERSION
k8s-master   NotReady   control-plane   19m   v1.32.2
k8s-worker   NotReady   <none>          84s   v1.32.2
```

## Agora basta repetir o processo para quantos worker vc queira

## Configurando a rede isolado do kubernetes (nesse caso vou usar o flannel porém existem inumeras opções cada caso é um caso, mas flannel para um ambiente de testes dá e sobra)
### Os comandos asseguir deverão ser executados no master ou em qualquer maquina q tenha acesso administrativo ao kubernetes
```
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

### execute:
```
watch kubectl get po --all-namespaces
```
### Se em algum momento mostrar o seguinte resultado, quer dizer q vc fez tudo certo:
```
NAMESPACE      NAME                                 READY   STATUS    RESTARTS   AGE
kube-flannel   kube-flannel-ds-cw6s8                1/1     Running   0          102s
kube-flannel   kube-flannel-ds-m64xh                1/1     Running   0          102s
kube-system    coredns-668d6bf9bc-hbr49             1/1     Running   0          21m
kube-system    coredns-668d6bf9bc-tmhxr             1/1     Running   0          21m
kube-system    etcd-k8s-master                      1/1     Running   0          21m
kube-system    kube-apiserver-k8s-master            1/1     Running   0          21m
kube-system    kube-controller-manager-k8s-master   1/1     Running   0          21m
kube-system    kube-proxy-njbsz                     1/1     Running   0          21m
kube-system    kube-proxy-v9vdr                     1/1     Running   0          3m56s
kube-system    kube-scheduler-k8s-master            1/1     Running   0          21m
```

### Agora vamos novamente ver o status dos nodes:
```
kubectl get nodes
```
### Se vc fez tudo certo o resultado será esse:
```
NAME         STATUS   ROLES           AGE     VERSION
k8s-master   Ready    control-plane   23m     v1.32.2
k8s-worker   Ready    <none>          5m55s   v1.32.2
```
