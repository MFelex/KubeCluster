# KubeCluster
Create kubernetes cluster
3 Master nodes
3 etcd nodes
3 Wroker nodes

Network:
Subnet: 192.168.167.0
GW: 192.168.167.2

Master nodes:
1 Core CPU | 1 GB RAM | 10 GB Nvme
master1: 192.168.167.11
master2: 192.168.167.12
master3: 192.168.167.13

etcd nodes:
2 Core CPU | 2 GB RAM | 30 GB Nvme
master1: 192.168.167.31
master2: 192.168.167.32
master3: 192.168.167.33

Wroker nodes:
1 Core CPU | 1 GB RAM | 7 GB Nvme
master1: 192.168.167.21
master2: 192.168.167.22
master3: 192.168.167.23

--------------------------------------------------------------
Verify the MAC address and product_uuid are unique for every node:
- Check Mac Address: ip link
- Check product_uuid: cat /sys/class/dmi/id/product_uuid  (or cat /etc/machine-id)

Check required ports:
- nc 127.0.0.1 6443 (or lsof -i:6443)

Change DNS to shecan DNS:
vim /etc/netplan/00-installer-config.yaml
add add these to addresses:
addresses:
        - 178.22.122.100
        - 185.51.200.2
netplan apply

Install containerd as Container Runtime:
wget https://github.com/containerd/containerd/releases/download/v1.6.8/containerd-1.6.8-linux-amd64.tar.gz
tar Cxzvf /usr/local containerd-1.6.8-linux-amd64.tar.gz
wget https://github.com/opencontainers/runc/releases/download/v1.1.3/runc.amd64
install -m 755 runc.amd64 /usr/local/sbin/runc
wget https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz
mkdir -p /opt/cni/bin
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.1.1.tgz
mkdir /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
curl -L https://raw.githubusercontent.com/containerd/containerd/main/containerd.service -o /etc/systemd/system/containerd.service
systemctl daemon-reload
systemctl enable --now containerd
systemctl status containerd

sudo -i
apt update
swappoff -a (if you restart machine even after install kubeadm, you need to turn it off again)

Update the apt package index and install packages needed to use the Kubernetes apt repository:
apt install -y apt-transport-https ca-certificates curl

Download the Google Cloud public signing key:
curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

Add the Kubernetes apt repository:
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

Update apt package index, install kubelet, kubeadm and kubectl, and pin their version:
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

modprobe br_netfilter
echo '1' > /proc/sys/net/ipv4/ip_forward
kubeadm reset

add Master1 IP into /etc/hossts (Later you can modify cluster-endpoint to point to the address of your load-balancer in an high availability scenario):
192.168.167.11 cluster-endpoint

kubeadm init --control-plane-endpoint="cluster-endpoint" --upload-certs --pod-network-cidr=172.16.0.0/16 --ignore-preflight-errors=NumCPU --ignore-preflight-errors=Mem

mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

Kubeadm token to get token & discovery-token-ca-cert-hash:
kubeadm token create `kubeadm token generate` --print-join-command --ttl=0

kubeadm certificate key:
kubeadm init phase upload-certs --upload-certs

You can reach it from:
kubeadm token list

