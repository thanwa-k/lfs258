```
hostnamectl set-hostname name
```
# on load balancer
```
sudo yum install haproxy -y
```
```
sudo vim /etc/haproxy/haproxy.cfg
```

```
frontend kubernetes-frontend
    bind 192.168.0.116:6443
    mode tcp
    option tcplog
    default_backend kubernetes-backend

backend kubernetes-backend
    mode tcp
    option tcp-check
    balance roundrobin
    server master-node1 192.168.0.112:6443 check fall 3 rise 2
    server master-node2 192.168.0.113:6443 check fall 3 rise 2
```
```
setsebool -P haproxy_connect_any on
```
```
systemctl restart haproxy
```

# on other nodes
Login to all servers and update the OS.
```
sudo yum -y update && sudo systemctl reboot
```
Once the servers are rebooted, add Kubernetes repository for CentOS 7 to all the servers.
```
sudo tee /etc/yum.repos.d/kubernetes.repo<<EOF
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
```
Then install required packages.
```
sudo yum -y install epel-release vim git curl wget kubelet kubeadm kubectl --disableexcludes=kubernetes
```
Confirm installation by checking the version of kubectl.
```
$ kubectl version --client
```
```
Client Version: version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.3", GitCommit:"2e7996e3e2712684bc73f0dec0200d64eec7fe40", GitTreeState:"clean", BuildDate:"2020-05-20T12:52:00Z", GoVersion:"go1.13.9", Compiler:"gc", Platform:"linux/amd64"}
```

Disable SELinux and Swap
```
sudo setenforce 0
sudo sed -i 's/^SELINUX=.*/SELINUX=permissive/g' /etc/selinux/config
```
```
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
sudo swapoff -a
```
Configure sysctl.
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
Install Container runtime
```
# Install packages
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum update -y && sudo yum install -y containerd.io-1.2.13 docker-ce-19.03.8 docker-ce-cli-19.03.8

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

Configure Firewalld
Master Server ports:
```
sudo firewall-cmd --add-port={6443,2379-2380,10250,10251,10252,5473,179,5473}/tcp --permanent
sudo firewall-cmd --add-port={4789,8285,8472}/udp --permanent
sudo firewall-cmd --reload
```
Worker Node ports:
```
sudo firewall-cmd --add-port={10250,30000-32767,5473,179,5473}/tcp --permanent
sudo firewall-cmd --add-port={4789,8285,8472}/udp --permanent
sudo firewall-cmd --reload
```

Initialize your control-plane node
```
$ lsmod | grep br_netfilter
```
```
br_netfilter           22256  0 
bridge                151336  2 br_netfilter,ebtable_broute
```

Enable kubelet service.
```
sudo systemctl enable kubelet
```

Pull container images:
```
$ sudo kubeadm config images pull
```
```
[config/images] Pulled k8s.gcr.io/kube-apiserver:v1.18.3
[config/images] Pulled k8s.gcr.io/kube-controller-manager:v1.18.3
[config/images] Pulled k8s.gcr.io/kube-scheduler:v1.18.3
[config/images] Pulled k8s.gcr.io/kube-proxy:v1.18.3
[config/images] Pulled k8s.gcr.io/pause:3.2
[config/images] Pulled k8s.gcr.io/etcd:3.4.3-0
[config/images] Pulled k8s.gcr.io/coredns:1.6.7
```
Create cluster:

```
sudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --control-plane-endpoint=192.168.0.116
```
  
  
  
```
 mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

# on master-node2
```
kubeadm join 192.168.0.116:6443 --token 4idwrf.0to6tfzjwryuc2f7 \
        --discovery-token-ca-cert-hash sha256:ffc4ce3d7685d31c31a2647b1026d504495888407b9c044182cfdf78c3ec5a76 \
        --control-plane --certificate-key 1c59863444279beaaa751f3fec41a4a9807470ed61e46d413f9fa9b48e9e5a72 --apiserver-advertise-address 192.168.0.113
```

# on worker-node1
```
kubeadm join 192.168.0.116:6443 --token 4idwrf.0to6tfzjwryuc2f7 \
        --discovery-token-ca-cert-hash sha256:ffc4ce3d7685d31c31a2647b1026d504495888407b9c044182cfdf78c3ec5a76
```

# to master-node1 again
```
kubectl --kubeconfig=/etc/kubernetes/admin.conf create -f https://docs.projectcalico.org/v3.15/manifests/calico.yaml
```

ref: https://www.youtube.com/watch?v=c1SCdv2hYDc

ref: https://github.com/justmeandopensource/kubernetes/tree/master/kubeadm-ha-multi-master

ref: https://computingforgeeks.com/install-kubernetes-cluster-on-centos-with-kubeadm/
