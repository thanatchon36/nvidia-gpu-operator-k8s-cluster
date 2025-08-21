# NVIDIA GPU Operator บน AWS Kubernetes Cluster - Complete Guide

## 🏗️ Phase 1: เตรียม Environment (ทุก nodes)

### 1.1 อัพเดทระบบและติดตั้ง Dependencies
```bash
# รันบนทุก nodes (control plane + workers)
sudo apt update && sudo apt install -y apt-transport-https ca-certificates curl gpg
```

### 1.2 ติดตั้ง Container Runtime (containerd)
```bash
# รันบนทุก nodes
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd && sudo systemctl enable containerd
```

### 1.3 Configure Network และ Kernel Modules
```bash
# รันบนทุก nodes
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay && sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system
```

---

## ⚙️ Phase 2: ติดตั้ง Kubernetes

### 2.1 ติดตั้ง kubeadm, kubelet, kubectl
```bash
# รันบนทุก nodes
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update && sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### 2.2 Initialize Control Plane (รันเฉพาะ Control Plane Node)
```bash
# รันบน control plane node เท่านั้น
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --control-plane-endpoint=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)

# Setup kubectl สำหรับ user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 2.3 ติดตั้ง CNI Plugin (Flannel)
```bash
# รันบน control plane node
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

### 2.4 Join Worker Nodes
```bash
# รันบน control plane เพื่อดู join command
kubeadm token create --print-join-command

# คัดลอก output แล้วรันบน worker nodes
# ตอว่างจะเป็น: sudo kubeadm join <CONTROL_PLANE_IP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```

### 2.5 ตรวจสอบ Cluster Status
```bash
# รันบน control plane
kubectl get nodes -o wide
kubectl get pods -n kube-system
```

---

## 🚀 Phase 3: ติดตั้ง NVIDIA GPU Operator

### 3.1 ติดตั้ง Helm
```bash
# รันบน control plane
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### 3.2 Add NVIDIA Helm Repository
```bash
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia && helm repo update
```

### 3.3 ติดตั้ง GPU Operator
```bash
helm install --wait gpu-operator nvidia/gpu-operator \
  --namespace gpu-operator-resources \
  --create-namespace \
  --set driver.enabled=true \
  --set toolkit.enabled=true \
  --set devicePlugin.enabled=true \
  --set nodeStatusExporter.enabled=true \
  --set gfd.enabled=true \
  --set migManager.enabled=false \
  --set operator.cleanupCRD=true
```

### 3.4 ตรวจสอบการติดตั้ง GPU Operator
```bash
# ตรวจสอบ pods
kubectl get pods -n gpu-operator-resources

# ตรวจสอบ GPU nodes
kubectl get nodes -l nvidia.com/gpu.present=true

# ตรวจสอบ GPU resources
kubectl describe nodes | grep -A 10 "Allocatable:" | grep nvidia.com/gpu
```
