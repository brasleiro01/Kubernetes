# Configuração do Kubernetes com containerd v2 e Cilium CNI
Este guia fornece um procedimento passo a passo para configurar o Kubernetes 1.31 com containerd v2 e Cilium CNI em um sistema Ubuntu 22.04 LTS ou superior.

**Pré-requisitos**

Antes de começarmos, verifique se você possui os seguintes pré-requisitos:

* Um sistema rodando Ubuntu 22.04 LTS ou superior, 64 bits x86 e kernel 6.8 ou superior.
* Acesso root ou sudo ao sistema.
* Conhecimentos básicos de Kubernetes, containerd e Cilium.
Guia Passo a Passo

**1. Instale as ferramentas necessárias**

Primeiro, precisamos garantir que o figlet e o toilet estejam instalados para imprimir mensagens coloridas.

```
if ! command -v figlet &> /dev/null || ! command -v toilet &> /dev/null; then
    echo -e "\033[0;33mFiglet or Toilet not found, installing..."
    sudo apt-get update && sudo apt-get install -y figlet toilet
fi
```
#
**2. Imprima o título**

Use figlet para imprimir o título do script.
```
figlet -f smblock "Setup Kubernetes 1.31 with containerd v2 and Cilium"
```
#
**3. Desabilitar Swap**

Desabilitar swap é crucial para que o Kubernetes funcione corretamente. O Kubernetes requer que o swap seja desabilitado para garantir desempenho e estabilidade ideais.
```
swapoff -a
sed -i '/swap/d' /etc/fstab
Note: Disabling swap ensures Kubernetes can manage resources more effectively, preventing potential issues with resource allocation.
```
#
**4. Carregue os módulos do kernel necessários**

Carregue os módulos do kernel necessários para o containerd.
```
cat >>/etc/modules-load.d/containerd.conf<<EOF
overlay
br_netfilter
EOF
modprobe overlay
modprobe br_netfilter
```
#
**5. Configurar sysctl para Kubernetes**

Configure os parâmetros sysctl exigidos pelo Kubernetes.
```
cat >>/etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
EOF
sysctl --system
```
#
**6. Desabilite o UFW**

Desative o Uncomplicated Firewall (UFW) para evitar problemas de rede.
```
ufw disable
```
#
**7. Remova pacotes desnecessários**

Remova pacotes desnecessários para evitar conflitos.
```
sudo apt-get remove containernetworking-plugins -y && sudo apt-get remove conmon -y
```
#
**8. Crie um diretório de chaveiros**

Crie o diretório keyrings para armazenar chaves GPG.
```
mkdir -p /etc/apt/keyrings/
```
#
**9. Instale o containerd**

Baixe e instale o containerd.
```
wget https://github.com/containerd/containerd/releases/download/v2.0.0/containerd-2.0.0-linux-amd64.tar.gz
tar -C /usr/local -xzvf containerd-2.0.0-linux-amd64.tar.gz
```
#
**10. Configurar containerd**

Crie um arquivo de serviço systemd para o containerd e inicie o serviço.
```
mkdir -p /usr/local/lib/systemd/system/
cat <<EOF > /usr/local/lib/systemd/system/containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target dbus.service

[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd

Type=notify
Delegate=yes
KillMode=process
Restart=always
RestartSec=5

LimitNPROC=infinity
LimitCORE=infinity
TasksMax=infinity
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now containerd
```
#
**11. Instale o runc**

Baixe e instale o runc.
```
wget https://github.com/opencontainers/runc/releases/download/v1.2.1/runc.amd64
install -m 755 runc.amd64 /usr/local/sbin/runc
```
#
**12. Instalar plugins CNI**

Baixe e instale os plugins CNI.
```
wget https://github.com/containernetworking/plugins/releases/download/v1.6.0/cni-plugins-linux-amd64-v1.6.0.tgz
mkdir -p /opt/cni/bin
tar -C /opt/cni/bin -xzvf cni-plugins-linux-amd64-v1.6.0.tgz
systemctl restart containerd
```
#
**13. Verifique a instalação do containerd**

Verifique a instalação do containerd.
```
containerd -v
```
#
**14. Adicione o repositório Kubernetes e instale os componentes**

Adicione o repositório do Kubernetes e instale os componentes necessários.
```
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo apt-get update
sudo apt-get install kubelet kubeadm kubectl -y
sudo apt-mark hold kubelet kubeadm kubectl
```
#
**15. Versões de saída dos componentes do Kubernetes**

Produza as versões dos componentes do Kubernetes instalados.
```
kubeadm version
kubelet --version
kubectl version --client
```
#
**16. Habilitar e iniciar o kubelet**

Habilite e inicie o serviço kubelet.
```
sudo systemctl enable --now kubelet
```
#
**17. Inicialize o cluster Kubernetes com kubeadm**

Inicialize o cluster Kubernetes com o CIDR de rede de pod especificado usando o kubeadm.
```
kubeadm init --pod-network-cidr=192.168.0.0/16 --cri-socket unix:///run/containerd/containerd.sock --ignore-preflight-errors=NumCPU
```
#
**18. Configure o kubectl para o usuário atual**

Configure o kubectl para o usuário atual.
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
export KUBECONFIG=/etc/kubernetes/admin.conf
kubectl cluster-info dump
```
#
**19. Instale o Cilium CNI**

Cilium é uma solução de rede, observabilidade e segurança baseada em **eBPF** . Pode ser um substituto para kube-proxy e fornece recursos aprimorados de observabilidade e segurança.

Instale o Cilium CNI para rede
```
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}

cilium install --version 1.16.3
```
#
