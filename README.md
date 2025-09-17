#### Kubernetes 1.32.1 Setup with Docker and Calico CNI

Complete setup guide for Kubernetes cluster with Docker runtime and Calico network plugin.

#### Prerequisites

- Ubuntu 22.04 LTS on all nodes
- Master Node: 2 CPUs, 2GB RAM minimum
- Worker Node: 1 CPU, 1GB RAM minimum
- Root or sudo access

### Phase 1: Prepare All Nodes (Master + Workers)

Run these steps on **ALL nodes**:

### 1.1 Update System and Configure Hostnames
```bash
sudo apt update && sudo apt upgrade -y

### Set hostname (customize for each node)
sudo hostnamectl set-hostname k8s-master     # On master node
sudo hostnamectl set-hostname k8s-worker1    # On worker node

### Configure hosts file
```
sudo vim /etc/hosts
 
 192.168.x.x k8s-master
 192.168.x.x k8s-worker1
```

### 1.2 Disable Swap
```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/###\1/g' /etc/fstab
sudo sed -i '/swap/s/^/###/' /etc/fstab
free -h  ### Verify swap is 0B
```

### 1.3 Load Kernel Modules
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

### 1.4 Configure Kernel Parameters
```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

### 1.5 Install Docker
```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg lsb-release

sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
```

### 1.6 Configure Docker
```bash
sudo mkdir -p /etc/docker
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

sudo systemctl daemon-reload
sudo systemctl restart docker
sudo systemctl enable docker
```

### 1.7 Install cri-dockerd
```bash
VER=$(curl -s https://api.github.com/repos/Mirantis/cri-dockerd/releases/latest|grep tag_name | cut -d '"' -f 4|sed 's/v//g')
wget https://github.com/Mirantis/cri-dockerd/releases/download/v${VER}/cri-dockerd-${VER}.amd64.tgz
tar xvf cri-dockerd-${VER}.amd64.tgz
sudo mv cri-dockerd/cri-dockerd /usr/local/bin/

sudo wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.service
sudo wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.socket
sudo mv cri-docker.socket cri-docker.service /etc/systemd/system/
sudo sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service

sudo systemctl daemon-reload
sudo systemctl enable cri-docker.service
sudo systemctl enable --now cri-docker.socket
sudo systemctl start cri-docker.service

rm -rf cri-dockerd*
```

### 1.8 Install Kubernetes 1.32.1
```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet=1.32.1-1.1 kubeadm=1.32.1-1.1 kubectl=1.32.1-1.1
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable kubelet
```

### Phase 2: Master Node Setup

Run these steps **ONLY on master node**:

### 2.1 Initialize Kubernetes Cluster
```bash
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=$(hostname -I | awk '{print $1}') \
  --cri-socket unix:///var/run/cri-dockerd.sock

#### SAVE THE JOIN COMMAND from the output!
```

### 2.2 Configure kubectl
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 2.3 Install Calico CNI
```bash
### Install Calico operator
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml

#### Configure Calico
cat <<EOF | kubectl apply -f -
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
    - blockSize: 26
      cidr: 10.244.0.0/16
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()
EOF
```

### 2.4 Verify Master Setup
```bash
#### Wait for Calico pods (2-3 minutes)
kubectl get pods -n calico-system

#### Check master node is Ready
kubectl get nodes

#### Should show: k8s-master Ready control-plane
```

#### Phase 3: Worker Node Setup

Run these steps **on each worker node**:

### 3.1 Get Join Command
On master node, generate join command:
```bash
kubeadm token create --print-join-command
```

### 3.2 Join Worker to Cluster
On worker node, use the join command with cri-socket:
```bash
# Example (use your actual command from step 3.1):
sudo kubeadm join 192.168.x.x:6443 \
  --token abc123.xyz789 \
  --discovery-token-ca-cert-hash sha256:1234567890abcdef... \
  --cri-socket unix:///var/run/cri-dockerd.sock
```

### 3.3 Verify Worker Joined
On master node:
```bash
### Check all nodes
``
kubectl get nodes
```
### Should show both master and worker as Ready

### Add worker role label (optional)
```
kubectl label node k8s-worker1 node-role.kubernetes.io/worker=worker
```

## Phase 4: Final Verification

### 4.1 Test Cluster

### depley test application 
```
kubectl create deployment nginx-test --image=nginx --replicas=2
```
check pod distribution
```
kubectl get pods -o wide
```
Verify pods are running on different nodes
```
kubectl get pods -o wide | grep nginx-test
```
clean up test
```
kubectl delete deployment nginx-test
```

### 4.2 Test Calico Network Policies

Create test namespace
```
kubectl create namespace policy-test
```
# Deploy test pods
```
kubectl run web --image=nginx --namespace=policy-test --labels="app=web"
kubectl run db --image=nginx --namespace=policy-test --labels="app=db"
```
### Test connectivity (should work)
```
WEB_POD=$(kubectl get pod -n policy-test -l app=web -o jsonpath='{.items[0].metadata.name}')
DB_IP=$(kubectl get pod -n policy-test -l app=db -o jsonpath='{.items[0].status.podIP}')
kubectl exec -n policy-test $WEB_POD -- ping -c 2 $DB_IP
```
### Apply network policy to block traffic
```
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: policy-test
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF
```
### Test connectivity again (should fail)
```
kubectl exec -n policy-test $WEB_POD -- timeout 5 ping -c 2 $DB_IP
```
### Clean up test
```
kubectl delete namespace policy-test
```

### 4.3 Final Status Check
```
Check cluster status
kubectl get nodes
kubectl get pods -A
kubectl cluster-info
```
Verify versions
```
kubectl version
kubeadm version
kubelet --version
```
All should show v1.32.1

## Quick Reference Commands

### Cluster Status
```
kubectl get nodes
kubectl get pods -A
kubectl cluster-info
```

### Add More Worker Nodes
1. Follow Phase 1 on new node
2. Get join command: `kubeadm token create --print-join-command`
3. Run join command with `--cri-socket unix:///var/run/cri-dockerd.sock`
4. Label node: `kubectl label node <node-name> node-role.kubernetes.io/worker=worker`

### Generate New Join Token
```
kubeadm token create --print-join-command
```

Now, Kubernetes 1.32.1 cluster with Docker runtime and Calico CNI is ready now to play. -vijai