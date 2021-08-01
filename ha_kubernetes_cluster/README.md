# Creating High Availability Kubernets Cluster on Azure VMs

## Azure Environment

|Role             | hostname       | IP         | OS            | RAM  | CPU  |
|-----------------|----------------|------------|---------------|------|------|
| Load Balancer   | lbHaProxy      | 10.0.1.40  | Ubuntu 18.04  | 1GB  | 1    |
| Master          | masterNodes10  | 10.0.1.10  | Ubuntu 18.04  | 2GB  | 2    |
| Master          | masterNodes20  | 10.0.1.20  | Ubuntu 18.04  | 2GB  | 2    |
| Worker          | workerNodes    | 10.0.1.30  | Ubuntu 18.04  | 1GB  | 1    |

## Pre-requisites
- Provisioning Azure Vms by running ansible-playbook
- Need to create NAT Gateway manualy after Vms provisioned

## Set up load balancer node

#### Install Haproxy
```
apt update && apt install -y haproxy
```

#### Configure haproxy
Append the below lines to /etc/haproxy/haproxy.cfg
```
frontend kubernetes-frontend
    bind 10.0.1.40:6443
    mode tcp
    option tcplog
    default_backend kubernetes-backend

backend kubernetes-backend
    mode tcp
    option tcp-check
    balance roundrobin
    server masterNodes10 10.0.1.10:6443 check fall 3 rise 2
    server masterNOdes20 10.0.1.20:6443 check fall 3 rise 2
```

#### Restart haproxy service
```
systemctl restart haproxy
```

## On all kubernetes nodes
#### Disable Firewall
```
ufw disable
```

#### Disable swap
```
swapoff -a; sed -i '/swap/d' /etc/fstab
```

#### Update sysctl settings for Kubernetes networking
```
cat >>/etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

#### Install docker engine
```
{
  apt install -y apt-transport-https ca-certificates curl gnupg-agent software-properties-common
  curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
  add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
  apt update && apt install -y docker-ce=5:19.03.10~3-0~ubuntu-focal containerd.io
}
```

### Kubernetes Setup
#### Add Apt repository
```
{
  curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
  echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list
}
```

#### Install Kubernetes components
```
apt update && apt install -y kubeadm=1.19.2-00 kubelet=1.19.2-00 kubectl=1.19.2-00
```

## On any one of the Kubernetes master node
#### Initialize Kubernetes Cluster
```
kubeadm init --control-plane-endpoint="10.0.1.40:6443" --upload-certs --apiserver-advertise-address=10.0.1.10 --pod-network-cidr=192.168.0.0/16

```

#### Deploy Calico network
```
kubectl --kubeconfig=/etc/kubernetes/admin.conf create -f https://docs.projectcalico.org/v3.15/manifests/calico.yaml

```