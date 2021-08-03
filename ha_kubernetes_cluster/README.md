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
Install the Tigera Calico operator and custom resource definitions.
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
```

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