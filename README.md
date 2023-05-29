# Install Kubernetes Cluster on CentOS 7 with kubeadm





## Step 1: Prepare Kubernetes Servers



The minimal server requirements for the servers used in the cluster are:

- **2 GiB** or more of RAM per machine–any less leaves little room for your apps.
- At least **2 CPUs** on the machine that you use as a **control-plane** node.
- Full network connectivity among all machines in the cluster – Can be private or public



Login to server and update the OS.



```
sudo yum -y update && sudo systemctl reboot
```



## Install required packages

```
sudo yum -y install epel-release vim git curl wget lsof htop
```





## Step 2: Install kubelet, kubeadm and kubectl



```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
```

```
yum install -y kubelet-1.21.2 kubeadm-1.21.2 kubectl-1.21.2
```

```
systemctl enable kubelet
systemctl start kubelet
```





## Step 3: Prepare Hostname, Firewall and SELinux



```
hostnamectl set-hostname master-node
```

##### Disable **SElinux** and update your firewall rules

```
sudo setenforce 0
sudo sed -i 's/^SELINUX=.*/SELINUX=permissive/g' /etc/selinux/config
```



Turn off swap.

```
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
sudo swapoff -a
```



Configure sysctl

```
sudo modprobe overlay
sudo modprobe br_netfilter

sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```



##### Install docker

```
# Install packages
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install docker-ce docker-ce-cli containerd.io
# Create required directories
sudo mkdir /etc/docker
sudo mkdir -p /etc/systemd/system/docker.service.d

# Create daemon json config file
sudo tee /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF

# Start and enable Services
sudo systemctl daemon-reload 
sudo systemctl restart docker
sudo systemctl enable docker
```



##### Firewall

I recommend you disable firewalld on your nodes:

```
sudo systemctl disable --now firewalld
```

If you have an active firewalld service there are a number of ports to be enabled.

**Master Server ports:**

```
sudo firewall-cmd --add-port={6443,2379-2380,10250,10251,10252,5473,179,5473}/tcp --permanent
sudo firewall-cmd --add-port={4789,8285,8472}/udp --permanent
sudo firewall-cmd --reload
```

**Worker Node ports:**

```
sudo firewall-cmd --add-port={10250,30000-32767,5473,179,5473}/tcp --permanent
sudo firewall-cmd --add-port={4789,8285,8472}/udp --permanent
sudo firewall-cmd --reload
```



## Step 4: Initialize Kubernetes Master and Setup Default User

```
sudo kubeadm init --apiserver-advertise-address=your_server_public_ip --pod-network-cidr=10.244.0.0/16
```

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```



Check cluster status:

```
$ kubectl cluster-info
Kubernetes master is running at https://k8s-cluster.computingforgeeks.com:6443
KubeDNS is running at https://k8s-cluster.computingforgeeks.com:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```



Additional Master nodes can be added using the command in installation output:

```
kubeadm join 185.226.117.54:6443 --token rqdjgq.0cgtc2kjfz4tgxen .bash--discovery-token-ca-cert-hash sha256:ce4da17a853b0eaac4deda13489a2d66994a06be09caa06b68b403b0c08c3f09
```



## Step 5: Install network plugin



In this guide we’ll use [Flannel](https://github.com/flannel-io/flannel). You can choose any other [supported network plugins](https://kubernetes.io/docs/concepts/cluster-administration/addons/).



```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

Confirm that all of the pods are running:

```
kubectl get pods  --all-namespaces
NAMESPACE     NAME                                                       READY   STATUS    RESTARTS   AGE
default       dnsutils                                                   1/1     Running   20         20h
default       dnsutils-master                                            1/1     Running   20         20h
kube-system   coredns-558bd4d5db-2w9k9                                   1/1     Running   0          20h
kube-system   coredns-558bd4d5db-4chcn                                   1/1     Running   0          20h
kube-system   etcd-centos-g1-medium1-fr-1.novalocal                      1/1     Running   0          20h
kube-system   kube-apiserver-centos-g1-medium1-fr-1.novalocal            1/1     Running   0          20h
kube-system   kube-controller-manager-centos-g1-medium1-fr-1.novalocal   1/1     Running   0          20h
kube-system   kube-flannel-ds-h5gl5                                      1/1     Running   0          20h
kube-system   kube-flannel-ds-hp2rx                                      1/1     Running   0          20h
kube-system   kube-proxy-2g7nx                                           1/1     Running   0          20h
kube-system   kube-proxy-6cctr                                           1/1     Running   0          20h
kube-system   kube-scheduler-centos-g1-medium1-fr-1.novalocal            1/1     Running   0          20h

```



## Step 5: Join the Worker Node to the Kubernetes Cluster

Get join command:

```
kubeadm token create --print-join-command


kubeadm join your_server_public_ip:6443 --token rqdjgq.0cgtc2kjfz4tgxen .bash--discovery-token-ca-cert-hash sha256:ce4da17a853b0eaac4deda13489a2d66994a06be09caa06b68b403b0c08c3f09
```



Set role worker to worker node:

```
kubectl label node mlops-test-2.novalocal node-role.kubernetes.io/worker=worker
```



## Optional Installation



### Local Path Provisioner

Dynamic volume provisioning allows storage volumes to be created  on-demand. Without dynamic provisioning, cluster administrators have to  manually make calls to their cloud or storage provider to create new  storage volumes, and then create PersistentVolume objects to represent  them in Kubernetes. The dynamic provisioning feature eliminates the need for cluster administrators to pre-provision storage. Instead, it  automatically provisions storage when it is requested by users.

- [Dynamic Volume Provisioning](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/)
- [Change the default StorageClass](https://kubernetes.io/docs/tasks/administer-cluster/change-default-storage-class/)

The provisioner will be installed in `local-path-storage` namespace by default

```
> kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml

namespace/local-path-storage created
serviceaccount/local-path-provisioner-service-account created
clusterrole.rbac.authorization.k8s.io/local-path-provisioner-role created
clusterrolebinding.rbac.authorization.k8s.io/local-path-provisioner-bind created
deployment.apps/local-path-provisioner created
storageclass.storage.k8s.io/local-path created
configmap/local-path-config created
```

Get storage classes

```
> kubectl get storageclass

NAME         PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  4h10m
```

Set default storage class

```
> kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

NAME                   PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  4h11m
```
