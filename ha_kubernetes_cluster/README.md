# Creating High Availability Kubernets Cluster v1.15 on Azure VMs

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
  apt update && apt install -y docker-ce=5:18.09.8~3-0~ubuntu-bionic containerd.io
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
apt update && apt install -y kubeadm=1.15.12-00 kubelet=1.15.12-00 kubectl=1.15.12-00
```

## On any one of the Kubernetes master node
#### Initialize Kubernetes Cluster
```
mkdir /etc/kubernetes/kubeadm
```
create a configuration file to initialize the cluster
```
# /etc/kubernetes/kubeadm/kubeadm-config.yaml

apiVersion: kubeadm.k8s.io/v1beta1
kind: ClusterConfiguration
kubernetesVersion: stable
controlPlaneEndpoint: "10.0.1.4:6443"
networking:
  podSubnet: 192.168.0.0/16
```

initialize kubernetes cluster
```
kubeadm init --config=/etc/kubernetes/kubeadm/kubeadm-config.yaml --upload-certs
```
<!-- ```
kubeadm init --control-plane-endpoint="10.0.1.40:6443" --upload-certs --apiserver-advertise-address=10.0.1.10 --pod-network-cidr=192.168.0.0/16

``` -->

#### Deploy Calico network
Install Calico Network
```
kubectl apply -f https://docs.projectcalico.org/archive/v3.12/manifests/calico.yaml
```
<!-- Install the Tigera Calico operator and custom resource definitions.
```
kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml

```

Install Calico by creating the necessary custom resource. For more information on configuration options available in this manifest. https://docs.projectcalico.org/reference/installation/api
```
kubectl create -f https://docs.projectcalico.org/manifests/custom-resources.yaml
```

Confirm that all of the pods are running with the following command.
```
watch kubectl get pods -n calico-system
``` -->

## Join other nodes to the cluster 


# Deploy webserver to kubernetes cluster
#### create webserver.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: webserver
  name: webserver
spec:
  selector:
    matchLabels:
      app: webserver
  replicas: 2 
  template:
    metadata:
      labels:
        app: webserver
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

#### create deployment with this command
```
kubectl create -f webserver.yaml
```

#### create kubernetes services for webserver (webserver-svc.yaml)
```
apiVersion: v1
kind: Service
metadata:
  name: web-service
  labels:
    run: web-service
spec:
  type: NodePort
  ports:
  - port: 80:32200
    protocol: TCP
  selector:
    app: webserver 
```

#### deploy service wiht this command
```
kubectl create -f webserver-svc.yaml
```

## Load balancing the webserver
#### add this to haproxy configurations file
```
frontend weserver-frontend
    bind 10.0.1.40:[weserver NodePort]
    mode tcp
    option tcplog
    default_backend weserver-backend

backend weserver-backend
    mode tcp
    option tcp-check
    balance roundrobin
    server masterNodes10 10.0.1.10:[weserver NodePort] check fall 3 rise 2
    server masterNOdes20 10.0.1.20:[weserver NodePort] check fall 3 rise 2
```

# Upgrade Kubernetes Cluster to v1.19
to upgrade from v1.15 to v1.19 the cluster must be upgraded gradually from v1.15 to v1.16, from v1.16 to v1.17 and so on
#### Check for latest kubernetes release version
```
sudo apt-cache madison kubeadm
```

## Upgrade Master Nodes
### Upgrade kubeadm
```

# Unholds to upgrade kubeadm package

sudo apt-mark unhold kubeadm


# Install the new version of kubeadm

sudo apt update && sudo apt install -y kubeadm=1.xx.xx-00

 
# Hold the package to prevent accidential upgrade

sudo apt-mark hold kubeadm

 
# Fetches the control plane component versions to which we can update to.

sudo kubeadm upgrade plan

 
# Upgrade kubeadm

sudo kubeadm upgrade apply v1.xx.x

```

### Upgrade kubectl and kubelet
```

# Drain the control plane node

kubectl drain [master_nodes] --ignore-daemonsets

 

# Unholds to upgrade for kubelet and kubectl

sudo apt-mark unhold kubelet kubectl

 

# Install the new version of kubelet and kubectl

sudo apt-get update && sudo apt-get install -y kubelet=1.xx.xx-00 kubectl=1.xx.xx-00

 

# Hold the package to prevent accidential upgrade

sudo apt-mark hold kubelet kubectl

 

# Will reloads the systemd manager configuration

sudo systemctl daemon-reload

 

# Will restart the kubelet service

sudo systemctl restart kubelet

 

# Check the status of the service

sudo systemctl status kubelet

 

# Uncordon the cntrol plane node.

kubectl uncordon [master_nodes]
```

## Upgrade worker nodes
Note: worker nodes should not be upgraded simultaneously

### Upgrade Kubeadm
#### On worker nodes
```

# Update the repositiry

sudo apt update

 

# Unholds to upgrade kubeadm

sudo apt-mark unhold kubeadm

 

# Install the new version of kubeadm

sudo apt update && sudo apt install -y kubeadm=1.xx.xx-00

 

# Hold the package to prevent upgrade

sudo apt-mark hold kubeadm

 

# Upgrade the local configuration

sudo kubeadm upgrade node
```

### Upgrade kubectl and kubelet
#### On control plane node
```
# Drain the worker_node node

kubectl drain [worker_nodes] --ignore-daemonsets --delete-emptydir-data
```

#### On worker nodes
```

# Unholds to upgrade kubelet and kubectl

sudo apt-mark unhold kubelet kubectl

 

# Install the new version of kubelet and kubectl

sudo apt-get update && sudo apt-get install -y kubelet=1.xx.xx-00 kubectl=1.xx.xx-00

 

# Hold the package to prevent upgrade

sudo apt-mark hold kubelet kubectl

 

# Will reload the systemd manager configuration

sudo systemctl daemon-reload

 

# Will restart the kubelet service

sudo systemctl restart kubelet

```

#### on control plane node
```
# Uncordon the worker node to bring it online and run the below command on control plane node.

kubectl uncordon [worker_nodes]

```