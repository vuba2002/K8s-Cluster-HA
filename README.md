# Kubernetes HA Cluster - Triển Khai Cụm 3 Master Node

## 📁 Thông Tin Tài Nguyên Cụm K8s

| Hostname      | OS           | IP              | RAM  | CPU |
| ------------- | ------------ | --------------- | ---- | --- |
| master-node-1 | Ubuntu 22.04 | 192.168.154.130 | 3 GB | 2   |
| master-node-2 | Ubuntu 22.04 | 192.168.154.131 | 3 GB | 2   |
| master-node-3 | Ubuntu 22.04 | 192.168.154.132 | 3 GB | 2   |
| worker-node-1 | Ubuntu 22.04 | 192.168.154.133 | 3 GB | 1   |
| worker-node-2 | Ubuntu 22.04 | 192.168.154.134 | 3 GB | 1   |
| ha-loadbancer | Ubuntu 22.04 | 192.168.154.135 | 3 GB | 1   |

## 🛠️ Các Bước Cài Đặt Chung

### 1. Cập nhật hệ thống

```bash
sudo apt update -y && sudo apt upgrade -y
```

### 2. Cài đặt HAProxy trên `ha-loadbalancer`

```bash
sudo apt-get install -y haproxy
```

**Chỉnh sửa cấu hình HAProxy:**

```bash
sudo nano /etc/haproxy/haproxy.cfg
```

Thêm nội dung sau:

```cfg
frontend kubernetes-frontend
    bind *:6443
    option tcplog
    mode tcp
    default_backend kubernetes-backend

backend kubernetes-backend
    mode tcp
    balance roundrobin
    option tcp-check
    server master-node-1 192.168.154.130:6443 check
    server master-node-2 192.168.154.131:6443 check
    server master-node-3 192.168.154.132:6443 check
```

Khởi động lại HAProxy:

```bash
sudo systemctl restart haproxy
```

### 3. Cập nhật file `/etc/hosts` trên mọi node

```bash
sudo vi /etc/hosts
```

Thêm:

```
192.168.154.130 master-node-1
192.168.154.131 master-node-2
192.168.154.132 master-node-3
192.168.154.133 worker-node-1
192.168.154.134 worker-node-2
192.168.154.135 ha-loadbalancer
```

### 4. Tắt Swap

```bash
sudo swapoff -a
sudo sed -i '/swap.img/s/^/#/' /etc/fstab
```

### 5. Kích hoạt kernel modules

```bash
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

### 6. Cài đặt sysctl

```bash
cat <<EOF | sudo tee /etc/sysctl.d/kubernetes.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

### 7. Cài đặt containerd

```bash
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/docker.gpg
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt update -y
sudo apt install -y containerd.io

containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

### 8. Cài đặt kubeadm, kubelet, kubectl

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update -y
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

## ♻️ Reset Cluster (nếu cần)

```bash
sudo kubeadm reset -f
sudo rm -rf /var/lib/etcd
sudo rm -rf /etc/kubernetes/manifests/*
```

## 🚀 Triển Khai Cluster

### Thực hiện trên `master-node-1`

```bash
sudo kubeadm init --control-plane-endpoint "192.168.154.135:6443" --upload-certs

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
```

### Thực hiện trên `master-node-2` và `master-node-3`

Sao chép lệnh sau (sử dụng token được sinh ra sau khi init):

```bash
sudo kubeadm join 192.168.154.135:6443 \
  --token <your_token> \
  --discovery-token-ca-cert-hash <your_sha> \
  --control-plane \
  --certificate-key <your_cert>

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---
### 9. Cài đặt etcdctl trên các master-node
Install etcdctl using apt:

```bash
sudo apt-get update
sudo apt-get install -y etcd-client
```

Verify Etcd Cluster Health:
```bash
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/peer.crt --key=/etc/kubernetes/pki/etcd/peer.key endpoint health
```
Verify HAProxy Configuration and Functionality:
```bash
listen stats
    bind *:8404
    mode http
    stats enable
    stats uri /
    stats refresh 10s
    stats admin if LOCALHOST
```
