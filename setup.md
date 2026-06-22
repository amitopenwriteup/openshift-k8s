Kubernetes Cluster Setup on Rocky Linux 9 using kubeadm + CRI-O + Podman
Architecture
Kubernetes v1.30
CRI-O as Container Runtime
Podman installed on host
Flannel CNI
Rocky Linux 9
---
Prerequisites
Master Node
2 CPU
4 GB RAM
20 GB Disk
Worker Node
2 CPU
2 GB RAM
20 GB Disk
Hostname Configuration
Master:
```bash
hostnamectl set-hostname k8s-master
```
Worker:
```bash
hostnamectl set-hostname k8s-worker1
```
Update hosts file on all nodes:
```bash
cat >> /etc/hosts <<EOF
192.168.1.10 k8s-master
192.168.1.11 k8s-worker1
EOF
```
---
Step 1: Disable Swap
```bash
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab
```
Verify:
```bash
free -h
```
---
Step 2: Configure Kernel Modules
```bash
cat <<EOF > /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter
```
Verify:
```bash
lsmod | grep overlay
lsmod | grep br_netfilter
```
---
Step 3: Configure Sysctl
```bash
cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
EOF
```
Apply:
```bash
sysctl --system
```
Verify:
```bash
sysctl net.ipv4.ip_forward
```
---
Step 4: Configure SELinux
```bash
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```
Verify:
```bash
getenforce
```
---
Step 5: Install Podman
```bash
dnf install -y podman
```
Verify:
```bash
podman version
```
---
Step 6: Install CRI-O
```bash
cat <<EOF > /etc/yum.repos.d/cri-o.repo
[cri-o]
name=CRI-O
baseurl=https://pkgs.k8s.io/addons:/cri-o:/stable:/v1.30/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/addons:/cri-o:/stable:/v1.30/rpm/repodata/repomd.xml.key
EOF
```
Install:
```bash
dnf install -y cri-o
```
Enable:
```bash
systemctl enable --now crio
```
Verify:
```bash
systemctl status crio
```
---
Step 7: Install Kubernetes Packages
```bash
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/repodata/repomd.xml.key
EOF
```
Install:
```bash
dnf install -y kubelet kubeadm kubectl
```
Enable kubelet:
```bash
systemctl enable kubelet
```
---
Step 8: Firewall Configuration
For lab:
```bash
systemctl disable --now firewalld
```
OR open required ports manually.
---
Step 9: Initialize Kubernetes Master
Run only on Master:
```bash
kubeadm init \
--cri-socket=unix:///var/run/crio/crio.sock \
--pod-network-cidr=10.244.0.0/16
```
Save the generated kubeadm join command.
---
Step 10: Configure kubectl
```bash
mkdir -p $HOME/.kube

cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

chown $(id -u):$(id -g) $HOME/.kube/config
```
Verify:
```bash
kubectl get nodes
```
---
Step 11: Install Flannel CNI
```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```
Check:
```bash
kubectl get pods -A
```
Wait until all pods are Running.
---
Step 12: Verify Master Node
```bash
kubectl get nodes
```
Expected:
```text
NAME         STATUS   ROLES           AGE
k8s-master   Ready    control-plane   5m
```
---
Step 13: Join Worker Node
Run on worker:
```bash
modprobe br_netfilter

echo 1 > /proc/sys/net/ipv4/ip_forward
```
Run the join command generated earlier:
```bash
kubeadm join <MASTER-IP>:6443 \
--token <TOKEN> \
--discovery-token-ca-cert-hash sha256:<HASH> \
--cri-socket=unix:///var/run/crio/crio.sock
```
---
Step 14: Verify Cluster
On master:
```bash
kubectl get nodes -o wide
```
Expected:
```text
NAME          STATUS   ROLES           VERSION
k8s-master    Ready    control-plane   v1.30.x
k8s-worker1   Ready    <none>          v1.30.x
```
---
Useful Commands
Get cluster info:
```bash
kubectl cluster-info
```
List pods:
```bash
kubectl get pods -A
```
Get join command again:
```bash
kubeadm token create --print-join-command
```
Restart CRI-O:
```bash
systemctl restart crio
```
Restart kubelet:
```bash
systemctl restart kubelet
```
Check node status:
```bash
kubectl get nodes
```
---
Cluster Successfully Installed
