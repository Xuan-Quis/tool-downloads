[Cài đặt thủ công](#cài-đặt-thủ-công)

- [Hướng dẫn cài đặt Kubernetes với CRI-O](#hướng-dẫn-cài-đặt-kubernetes-với-cri-o)
- [Hướng dẫn cài đặt với containerd](#hướng-dẫn-cài-đặt-với-containerd)

[Cài đặt bằng ansible](#cài-đặt-tự-động-ansible)

# Cài đặt thủ công 

## Hướng dẫn cài đặt Kubernetes với CRI-O

### Bước 1: Cập nhật hệ thống
```bash
# Cập nhật và nâng cấp các gói
sudo apt update && sudo apt -y upgrade

# Khởi động lại nếu cần thiết
[ -f /var/run/reboot-required ] && sudo reboot -f

# Cập nhật lại sau khi khởi động
sudo apt update -y
```

#### Cài đặt các gói cần thiết
```bash
# Cài đặt curl và apt-transport-https
sudo apt -y install curl apt-transport-https
```

### Bước 2: Cài đặt Kubernetes

#### Thêm repository Kubernetes
```bash
# Thêm GPG key của Google
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

# Thêm repository Kubernetes
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Cập nhật package list
sudo apt update
```

#### Cài đặt các thành phần Kubernetes
```bash
# Cài đặt kubelet, kubeadm, kubectl và các công cụ khác
sudo apt -y install vim git curl wget kubelet kubeadm kubectl

# Giữ phiên bản các gói Kubernetes (tránh tự động cập nhật)
sudo apt-mark hold kubelet kubeadm kubectl

# Kiểm tra phiên bản
kubectl version --client && kubeadm version
```

### Bước 3: Cấu hình hệ thống

####  Vô hiệu hóa swap
```bash
# Comment out swap trong fstab
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Kiểm tra file fstab
sudo vim /etc/fstab

# Tắt swap hiện tại
sudo swapoff -a

# Mount lại các filesystem
sudo mount -a

# Kiểm tra swap đã được tắt
free -h
```

#### Cấu hình kernel modules
```bash
# Load các module cần thiết
sudo modprobe overlay
sudo modprobe br_netfilter
```

####  Cấu hình sysctl
```bash
# Tạo file cấu hình sysctl cho Kubernetes
sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

# Áp dụng cấu hình
sudo sysctl --system
```

### Bước 4: Cài đặt CRI-O Container Runtime

#### Thêm repository CRI-O
```bash
# Chuyển sang root user
sudo su -

# Thiết lập biến môi trường
OS="xUbuntu_22.04"
VERSION=1.33

# Thêm repository CRI-O
echo "deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
echo "deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/ /" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.list

# Thêm GPG keys
curl -L https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$VERSION/$OS/Release.key | apt-key add -
curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/Release.key | apt-key add -

# Quay lại user ubuntu
su - ubuntu
```

#### Cài đặt CRI-O
```bash
# Cài đặt CRI-O
sudo apt install cri-o cri-o-runc -y

# Cấu hình network CIDR (thay đổi theo môi trường của bạn)
sudo sed -i 's/10.85.0.0/172.24.0.0/g' /etc/cni/net.d/100-crio-bridge.conflist

# Reload và khởi động CRI-O
sudo systemctl daemon-reload
sudo systemctl restart crio
sudo systemctl enable crio

# Kiểm tra trạng thái CRI-O
sudo systemctl status crio
```

### Bước 5: Cấu hình Kubernetes

#### Kích hoạt kubelet
```bash
# Kích hoạt kubelet service
sudo systemctl enable kubelet
```

####  Pull Kubernetes images
```bash
# Pull các container images cần thiết
sudo kubeadm config images pull --cri-socket unix:///var/run/crio/crio.sock
```

####  Áp dụng cấu hình sysctl
```bash
# Áp dụng lại cấu hình sysctl
sudo sysctl -p
```

## Hướng dẫn cài đặt với containerd

### Bước 1: Cập nhật hệ thống
```bash
apt update && apt upgrade -y
# Cài đặt các gói cần thiết
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
```
### Bước 2: Cấu hình hệ thống
#### Tắt swap
```bash
sudo swapoff -a
# Tắt hoàn toàn swap(not recommend)
sudo sed -i '/swap.img/s/^/#/' /etc/fstab
```
#### Cấu hình kernel
```bash
echo "overlay" | sudo tee -a /etc/modules-load.d/containerd.conf

echo "br_netfilter" | sudo tee -a /etc/modules-load.d/containerd.conf 
EOF
# Cài đặt kernel
sudo modprobe overlay
sudo modprobe br_netfilter
```
#### Cấu hình mạng
```bash
echo "net.bridge.bridge-nf-call-ip6tables = 1" | sudo tee -a /etc/sysctl.d/kubernetes.conf
echo "net.bridge.bridge-nf-call-iptables = 1" | sudo tee -a /etc/sysctl.d/kubernetes.conf
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.d/kubernetes.conf
sudo sysctl --system
```
### Cài đặt container runtime interface: containerd
#### Thêm các gói
```bash
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/docker.gpg
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```
#### cài đặt containerd
```bash
sudo apt update -y
sudo apt install -y containerd.io
```
###$ Cấu hình containerd
```bash
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list containerd

# Khởi động
sudo systemctl restart containerd
sudo systemctl enable containerd
```
### Cài đặt k8s

```bash
# thêm gói
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
# cài đặt
sudo apt update -y
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

# Cài đặt tự động bằng Ansible

## Bước 1: Cài đặt Ansible trên máy điều khiển

```bash
# Cập nhật package list
sudo apt update

# Cài đặt Ansible và các công cụ cần thiết
sudo apt install -y ansible sshpass python3-pip

# Kiểm tra phiên bản Ansible
ansible --version
```

## Bước 2: Tạo SSH key và cấu hình kết nối

```bash
# Tạo SSH key pair (nếu chưa có)
ssh-keygen -t rsa -b 4096 -C "ansible" -f ~/.ssh/id_rsa -N ""

# Sao chép public key sang các node
ssh-copy-id <user>@<IP_Master>
ssh-copy-id <user>@<IP_Node1>
ssh-copy-id <user>@<IP_Node2>

# Kiểm tra kết nối SSH
ssh <user>@<IP_Master> "echo 'SSH OK'"
ssh <user>@<IP_Node1> "echo 'SSH OK'"
ssh <user>@<IP_Node2> "echo 'SSH OK'"
```

## Bước 3: Cấu hình inventory file

Chỉnh sửa file `k8s/install k8s with ansible/hosts`:

```ini
[control_plane]
control1 ansible_ssh_host=<IP_Master>

[workers]
worker1 ansible_ssh_host=<IP_Node1>
worker2 ansible_ssh_host=<IP_Node2>

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

## Bước 4: Di chuyển vào thư mục playbook

```bash
cd "k8s/install k8s with ansible"
```

## Bước 5: Kiểm tra kết nối Ansible

```bash
# Test ping tới tất cả node
ansible -i hosts all -m ping
```

## Bước 6: Chạy playbook initial.yml

```bash
# Tạo user ubuntu, cấp quyền sudo, thêm SSH key
ansible-playbook -i hosts initial.yml
```

Playbook này sẽ:
- Tạo user `ubuntu` trên tất cả node
- Cấp quyền sudo không mật khẩu cho user `ubuntu`
- Thêm SSH public key vào `authorized_keys`

## Bước 7: Chạy playbook kube-dependencies.yml

```bash
# Cài đặt Docker và các phụ thuộc Kubernetes
ansible-playbook -i hosts kube-dependencies.yml
```

Playbook này sẽ:
- Cài đặt Docker với cgroup driver systemd
- Thêm repository Kubernetes
- Cài đặt kubelet, kubeadm (tất cả node)
- Cài đặt kubectl (chỉ control plane)

## Bước 8: Chạy playbook control-plane.yml

```bash
# Khởi tạo control plane và cài mạng pod
ansible-playbook -i hosts control-plane.yml
```

Playbook này sẽ:
- Chạy `kubeadm init --pod-network-cidr=10.244.0.0/16`
- Tạo thư mục `.kube` cho user ubuntu
- Copy file cấu hình admin.conf
- Cài đặt Flannel network plugin

## Bước 9: Chạy playbook workers.yml

```bash
# Join các worker node vào cluster
ansible-playbook -i hosts workers.yml
```

Playbook này sẽ:
- Lấy lệnh join từ control plane
- Thực thi lệnh join trên các worker node

## Bước 10: Kiểm tra cluster

```bash
# SSH vào control plane và kiểm tra nodes
ssh ubuntu@<IP_Master> "kubectl get nodes -o wide"

# Kiểm tra pods trong tất cả namespace
ssh ubuntu@<IP_Master> "kubectl get pods -A"

# Kiểm tra trạng thái cluster
ssh ubuntu@<IP_Master> "kubectl cluster-info"
```

## Bước 11: Kiểm tra mạng pod

```bash
# Kiểm tra Flannel pods
ssh ubuntu@<IP_Master> "kubectl get pods -n kube-flannel"

# Kiểm tra coredns pods
ssh ubuntu@<IP_Master> "kubectl get pods -n kube-system"
```

## Xử lý lỗi thường gặp

### Lỗi 1: Swap chưa tắt
```bash
# Trên từng node, chạy:
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### Lỗi 2: Docker cgroup driver
```bash
# Kiểm tra file /etc/docker/daemon.json
cat /etc/docker/daemon.json

# Nếu chưa có, tạo với nội dung:
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}

# Khởi động lại Docker
sudo systemctl restart docker
```

### Lỗi 3: Mạng pod không hoạt động
```bash
# Kiểm tra Flannel pods
kubectl get pods -n kube-flannel

# Nếu có lỗi, xóa và tạo lại:
kubectl delete -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

### Lỗi 4: Version mismatch
Nếu muốn dùng phiên bản Kubernetes khác, chỉnh sửa file `kube-dependencies.yml`:

```yaml
# Thay đổi các dòng này:
- name: install kubelet
  apt:
    name: kubelet=1.30.0-00  # Thay đổi version

- name: install kubeadm  
  apt:
    name: kubeadm=1.30.0-00  # Thay đổi version

- name: install kubectl
  apt:
    name: kubectl=1.30.0-00  # Thay đổi version
```

## Lưu ý quan trọng

1. **Phiên bản cố định**: Playbook hiện dùng Kubernetes 1.22.4
2. **Container Runtime**: Sử dụng Docker thay vì containerd
3. **Mạng pod**: Flannel với CIDR 10.244.0.0/16
4. **User**: Tạo user `ubuntu` với quyền sudo không mật khẩu
5. **SSH**: Cần cấu hình SSH key authentication

## Kiểm tra cuối cùng

Sau khi hoàn tất, cluster sẽ có:
- 1 control plane node (Ready)
- 2 worker nodes (Ready)  
- Tất cả system pods đang chạy
- Mạng pod hoạt động bình thường