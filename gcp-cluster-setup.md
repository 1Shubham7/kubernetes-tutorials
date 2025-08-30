# Kubernetes Master and Worker Node Setup Guide

## Master Node Setup

### 1. SSH into the Master EC2 Server
Start by accessing your Master EC2 instance via SSH.

### 2. Disable Swap
Run the following commands to disable swap:
```bash
swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### 3. Configure IPv4 Forwarding and iptables for Bridged Traffic
Load necessary modules and apply system parameters:
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

Set up sysctl parameters:
```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

Verify the modules and parameters:
```bash
lsmod | grep br_netfilter
lsmod | grep overlay
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

### 4. Install Container Runtime
Download and install containerd:
```bash
curl -LO https://github.com/containerd/containerd/releases/download/v1.7.14/containerd-1.7.14-linux-amd64.tar.gz
sudo tar Cxzvf /usr/local containerd-1.7.14-linux-amd64.tar.gz
curl -LO https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
sudo mkdir -p /usr/local/lib/systemd/system/
sudo mv containerd.service /usr/local/lib/systemd/system/
```

Set up the default containerd configuration:
```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
```
Enable and start the containerd service:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now containerd
systemctl status containerd
```

### 5. Install runc
```bash
curl -LO https://github.com/opencontainers/runc/releases/download/v1.1.12/runc.amd64
sudo install -m 755 runc.amd64 /usr/local/sbin/runc
```

### 6. Install CNI Plugin
```bash
curl -LO https://github.com/containernetworking/plugins/releases/download/v1.5.0/cni-plugins-linux-amd64-v1.5.0.tgz
sudo mkdir -p /opt/cni/bin
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.5.0.tgz
```

### 7. Install kubeadm, kubelet, and kubectl
Update packages and install Kubernetes tools:
```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet=1.29.6-1.1 kubeadm=1.29.6-1.1 kubectl=1.29.6-1.1 --allow-downgrades --allow-change-held-packages
sudo apt-mark hold kubelet kubeadm kubectl
```

Verify the installation:
```bash
kubeadm version
kubelet --version
kubectl version --client
```
> **Note:** Version 1.29 is installed to facilitate a later upgrade to version 1.30.

### 8. Configure crictl for containerd
```bash
sudo crictl config runtime-endpoint unix:///var/run/containerd/containerd.sock
```

### 9. Initialize the Control Plane
Run the following command to initialize the Kubernetes control plane:
```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=172.31.89.68 --node-name master
```
> **Note:** Save the generated join command for later use.

### 10. Configure kubeconfig
Prepare kubeconfig for kubectl:
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 11. Install Calico Network Plugin
```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml

curl https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml -O
kubectl apply -f custom-resources.yaml
```

## Worker Node Setup

### Steps for Worker Nodes
1. Repeat steps 1-8 from the Master node setup on each worker node.
2. Use the saved join command from step 9 to join the worker nodes to the cluster:
   ```bash
   sudo kubeadm join 172.31.71.210:6443 --token xxxxx --discovery-token-ca-cert-hash sha256:xxx
   ```
   > **Note:** If the join command is not available, regenerate it on the master node:
   ```bash
   kubeadm token create --print-join-command
   ```
