Follow Below steps to implement MultiMaster 
=============================================

<h3>Installing cfssl</h3>

<b>1- Download the binaries.</b>

$ wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64

$ wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64

<b>2- Add the execution permission to the binaries.</b>

$ chmod +x cfssl*

<b>3- Move the binaries to /usr/local/bin.</b>

$ sudo mv cfssl_linux-amd64 /usr/local/bin/cfssl

$ sudo mv cfssljson_linux-amd64 /usr/local/bin/cfssljson

<b>4- Verify the installation.</b>

$ cfssl version

<h3>Installing kubectl</h3>

<b>1- Download the binary.</b>

$ wget https://storage.googleapis.com/kubernetes-release/release/v1.12.1/bin/linux/amd64/kubectl

<b>2- Add the execution permission to the binary.</b>

$ chmod +x kubectl

<b>3- Move the binary to /usr/local/bin.</b>

$ sudo mv kubectl /usr/local/bin

<b>4- Verify the installation.</b>

$ kubectl version

<h3>Installing the HAProxy load balancer</h3>

As we will deploy three Kubernetes master nodes, we need to deploy an HAPRoxy load balancer in front of them to distribute the traffic.

<b>1- SSH to the 10.10.40.93 Ubuntu machine.</b>

<b>2- Update the machine.<b>

$ sudo apt-get update

$ sudo apt-get upgrade

<b>3- Install HAProxy.</b>

$ sudo apt-get install haproxy

<b>4- Configure HAProxy to load balance the traffic between the three Kubernetes master nodes.</b>
<pre>
$ sudo vim /etc/haproxy/haproxy.cfg
global
...
default
...
frontend kubernetes
bind 10.10.40.93:6443
option tcplog
mode tcp
default_backend kubernetes-master-nodes



backend kubernetes-master-nodes
mode tcp
balance roundrobin
option tcp-check
server k8s-master-0 10.10.40.90:6443 check fall 3 rise 2
server k8s-master-1 10.10.40.91:6443 check fall 3 rise 2
server k8s-master-2 10.10.40.92:6443 check fall 3 rise 2
</pre>
<b>5- Restart HAProxy.</b>

$ sudo systemctl restart haproxy

<h3>Generating the TLS certificates</h3>

These steps can be done on your Linux desktop if you have one or on the HAProxy machine depending on where you installed the cfssl tool.

<h3>Creating a certificate authority</h3>

<b>1- Create the certificate authority configuration file.</b>
<pre>
$ vim ca-config.json
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
</pre>
<b>2- Create the certificate authority signing request configuration file.</b>
<pre>
$ vim ca-csr.json
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
  {
    "C": "IE",
    "L": "Cork",
    "O": "Kubernetes",
    "OU": "CA",
    "ST": "Cork Co."
  }
 ]
}
</pre>
<b>3- Generate the certificate authority certificate and private key.</b>

$ cfssl gencert -initca ca-csr.json | cfssljson -bare ca

<b>4- Verify that the ca-key.pem and the ca.pem were generated.</b>

$ ls -la

<h3>Creating the certificate for the Etcd cluster</h3>

<b>1- Create the certificate signing request configuration file.</b>
<pre>
$ vim kubernetes-csr.json
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
  {
    "C": "IE",
    "L": "Cork",
    "O": "Kubernetes",
    "OU": "Kubernetes",
    "ST": "Cork Co."
  }
 ]
}
</pre>
<b>2- Generate the certificate and private key.</b>
<pre>
$ cfssl gencert \
-ca=ca.pem \
-ca-key=ca-key.pem \
-config=ca-config.json \
-hostname=10.10.40.90,10.10.40.91,10.10.40.92,10.10.40.93,127.0.0.1,kubernetes.default \
-profile=kubernetes kubernetes-csr.json | \
cfssljson -bare kubernetes
</pre>
<b>3- Verify that the kubernetes-key.pem and the kubernetes.pem file were generated.</b>

$ ls -la

<b>4- Copy the certificate to each nodes.</b>
<pre>
$ scp ca.pem kubernetes.pem kubernetes-key.pem sguyennet@10.10.40.90:~
$ scp ca.pem kubernetes.pem kubernetes-key.pem sguyennet@10.10.40.91:~
$ scp ca.pem kubernetes.pem kubernetes-key.pem sguyennet@10.10.40.92:~
$ scp ca.pem kubernetes.pem kubernetes-key.pem sguyennet@10.10.40.100:~
$ scp ca.pem kubernetes.pem kubernetes-key.pem sguyennet@10.10.40.101:~
$ scp ca.pem kubernetes.pem kubernetes-key.pem sguyennet@10.10.40.102:~
</pre>
<b>Preparing the nodes for kubeadm</b>
<b>Preparing the 10.10.40.90 machine</b>
<b>Installing Docker</b>

<b>1- SSH to the 10.10.40.90 machine.</b>

$ ssh sguyennet@10.10.40.90
Install Docker 
============== 
sudo apt-get update && sudo apt-get upgrade -y

sudo apt-get install -y apt-transport-https ca-certificates curl gnupg-agent software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

sudo apt-get update && sudo apt-get install docker-ce=18.06.3~ce~3-0~ubuntu

sudo systemctl start docker
sudo systemctl enable docker


Install Kubernetes
===================

sudo curl -s http://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

sudo vi /etc/apt/sources.list.d/kubernetes.list

sudo nano /etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main

sudo sh -c 'echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" >> /etc/apt/sources.list.d/kubernetes.list'

sudo apt-get update && sudo apt-get install -y kubelet kubeadm kubectl kubernetes-cni

sudo systemctl daemon-reload 

sudo systemctl restart kubelet

sudo iptables -F

sudo swapoff -a

sudo nano /etc/fstab [comment line with swap]


<b>Preparing the 10.10.40.91 machine</b>
<b>Installing Docker</b>

<b>1- SSH to the 10.10.40.91 machine.</b>

$ ssh sguyennet@10.10.40.91
Install Docker 
============== 
sudo apt-get update && sudo apt-get upgrade -y

sudo apt-get install -y apt-transport-https ca-certificates curl gnupg-agent software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

sudo apt-get update && sudo apt-get install docker-ce=18.06.3~ce~3-0~ubuntu

sudo systemctl start docker
sudo systemctl enable docker


Install Kubernetes
===================

sudo curl -s http://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

sudo vi /etc/apt/sources.list.d/kubernetes.list

sudo nano /etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main

sudo sh -c 'echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" >> /etc/apt/sources.list.d/kubernetes.list'

sudo apt-get update && sudo apt-get install -y kubelet kubeadm kubectl kubernetes-cni

sudo systemctl daemon-reload 

sudo systemctl restart kubelet

sudo iptables -F

sudo swapoff -a

sudo nano /etc/fstab [comment line with swap]



<b>Preparing the 10.10.40.92 machine</b>
<b>Installing Docker</b>

<b>1- SSH to the 10.10.40.92 machine.</b>

$ ssh sguyennet@10.10.40.92
Install Docker 
============== 
sudo apt-get update && sudo apt-get upgrade -y

sudo apt-get install -y apt-transport-https ca-certificates curl gnupg-agent software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

sudo apt-get update && sudo apt-get install docker-ce=18.06.3~ce~3-0~ubuntu

sudo systemctl start docker
sudo systemctl enable docker


Install Kubernetes
===================

sudo curl -s http://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

sudo vi /etc/apt/sources.list.d/kubernetes.list

sudo nano /etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main

sudo sh -c 'echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" >> /etc/apt/sources.list.d/kubernetes.list'

sudo apt-get update && sudo apt-get install -y kubelet kubeadm kubectl kubernetes-cni

sudo systemctl daemon-reload 

sudo systemctl restart kubelet

sudo iptables -F

sudo swapoff -a

sudo nano /etc/fstab [comment line with swap]




<b>Preparing the 10.10.40.102 machine</b>
<b>Installing Docker</b>

<b>1- SSH to the 10.10.40.102 machine.</b>

Install Docker 
============== 
sudo apt-get update && sudo apt-get upgrade -y

sudo apt-get install -y apt-transport-https ca-certificates curl gnupg-agent software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

sudo apt-get update && sudo apt-get install docker-ce=18.06.3~ce~3-0~ubuntu

sudo systemctl start docker
sudo systemctl enable docker


Install Kubernetes
===================

sudo curl -s http://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

sudo vi /etc/apt/sources.list.d/kubernetes.list

sudo nano /etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main

sudo sh -c 'echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" >> /etc/apt/sources.list.d/kubernetes.list'

sudo apt-get update && sudo apt-get install -y kubelet kubeadm kubectl kubernetes-cni

sudo systemctl daemon-reload 

sudo systemctl restart kubelet

sudo iptables -F

sudo swapoff -a

sudo nano /etc/fstab [comment line with swap]



<h3>Installing and configuring Etcd</h3>
<b>Installing and configuring Etcd on the 10.10.40.90 machine</b>

<b>1- SSH to the 10.10.40.90 machine.</b>

$ ssh sguyennet@10.10.40.90

<b>2- Create a configuration directory for Etcd.</b>

$ sudo mkdir /etc/etcd /var/lib/etcd

<b>3- Move the certificates to the configuration directory.</b>

$ sudo mv ~/ca.pem ~/kubernetes.pem ~/kubernetes-key.pem /etc/etcd

<b>4- Download the etcd binaries.</b>

$ wget https://github.com/coreos/etcd/releases/download/v3.3.9/etcd-v3.3.9-linux-amd64.tar.gz

<b>5- Extract the etcd archive.</b>

$ tar xvzf etcd-v3.3.9-linux-amd64.tar.gz

<b>6- Move the etcd binaries to /usr/local/bin.</b>

$ sudo mv etcd-v3.3.9-linux-amd64/etcd* /usr/local/bin/

<b>7- Create an etcd systemd unit file.</b>
<pre>
$ sudo vim /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos



[Service]
ExecStart=/usr/local/bin/etcd \
  --name 10.10.40.90 \
  --cert-file=/etc/etcd/kubernetes.pem \
  --key-file=/etc/etcd/kubernetes-key.pem \
  --peer-cert-file=/etc/etcd/kubernetes.pem \
  --peer-key-file=/etc/etcd/kubernetes-key.pem \
  --trusted-ca-file=/etc/etcd/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ca.pem \
  --peer-client-cert-auth \
  --client-cert-auth \
  --initial-advertise-peer-urls https://10.10.40.90:2380 \
  --listen-peer-urls https://10.10.40.90:2380 \
  --listen-client-urls https://10.10.40.90:2379,http://127.0.0.1:2379 \
  --advertise-client-urls https://10.10.40.90:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster 10.10.40.90=https://10.10.40.90:2380,10.10.40.91=https://10.10.40.91:2380,10.10.40.92=https://10.10.40.92:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5


[Install]
WantedBy=multi-user.target
</pre>
<b>8- Reload the daemon configuration.</b>

$ sudo systemctl daemon-reload

<b>9- Enable etcd to start at boot time.</b>

$ sudo systemctl enable etcd

<b>10- Start etcd.</b>

$ sudo systemctl start etcd

<b>Installing and configuring Etcd on the 10.10.40.91 machine</b>

<b>1- SSH to the 10.10.40.91 machine.</b>

$ ssh sguyennet@10.10.40.91

<b>2- Create a configuration directory for Etcd.</b>

$ sudo mkdir /etc/etcd /var/lib/etcd

<b>3- Move the certificates to the configuration directory.</b>

$ sudo mv ~/ca.pem ~/kubernetes.pem ~/kubernetes-key.pem /etc/etcd

<b>4- Download the etcd binaries.</b>

$ wget https://github.com/coreos/etcd/releases/download/v3.3.9/etcd-v3.3.9-linux-amd64.tar.gz

<b>5- Extract the etcd archive.</b>

$ tar xvzf etcd-v3.3.9-linux-amd64.tar.gz

<b>6- Move the etcd binaries to /usr/local/bin.</b>

$ sudo mv etcd-v3.3.9-linux-amd64/etcd* /usr/local/bin/

<b>7- Create an etcd systemd unit file.</b>
<pre>
$ sudo vim /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos



[Service]
ExecStart=/usr/local/bin/etcd \
  --name 10.10.40.91 \
  --cert-file=/etc/etcd/kubernetes.pem \
  --key-file=/etc/etcd/kubernetes-key.pem \
  --peer-cert-file=/etc/etcd/kubernetes.pem \
  --peer-key-file=/etc/etcd/kubernetes-key.pem \
  --trusted-ca-file=/etc/etcd/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ca.pem \
  --peer-client-cert-auth \
  --client-cert-auth \
  --initial-advertise-peer-urls https://10.10.40.91:2380 \
  --listen-peer-urls https://10.10.40.91:2380 \
  --listen-client-urls https://10.10.40.91:2379,http://127.0.0.1:2379 \
  --advertise-client-urls https://10.10.40.91:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster 10.10.40.90=https://10.10.40.90:2380,10.10.40.91=https://10.10.40.91:2380,10.10.40.92=https://10.10.40.92:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5


[Install]
WantedBy=multi-user.target
</pre>
<b>8- Reload the daemon configuration.</b>

$ sudo systemctl daemon-reload

<b>9- Enable etcd to start at boot time.</b>

$ sudo systemctl enable etcd

<b>10- Start etcd.</b>

$ sudo systemctl start etcd

<b>Installing and configuring Etcd on the 10.10.40.92 machine</b>

<b>1- SSH to the 10.10.40.92 machine.</b>

$ ssh sguyennet@10.10.40.92

<b>2- Create a configuration directory for Etcd.</b>

$ sudo mkdir /etc/etcd /var/lib/etcd

<b>3- Move the certificates to the configuration directory.</b>

$ sudo mv ~/ca.pem ~/kubernetes.pem ~/kubernetes-key.pem /etc/etcd

<b>4- Download the etcd binaries.</b>

$ wget https://github.com/coreos/etcd/releases/download/v3.3.9/etcd-v3.3.9-linux-amd64.tar.gz

<b>5- Extract the etcd archive.</b>

$ tar xvzf etcd-v3.3.9-linux-amd64.tar.gz

<b>6- Move the etcd binaries to /usr/local/bin.</b>

$ sudo mv etcd-v3.3.9-linux-amd64/etcd* /usr/local/bin/

<b>7- Create an etcd systemd unit file.</b>
<pre>
$ sudo vim /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos



[Service]
ExecStart=/usr/local/bin/etcd \
  --name 10.10.40.92 \
  --cert-file=/etc/etcd/kubernetes.pem \
  --key-file=/etc/etcd/kubernetes-key.pem \
  --peer-cert-file=/etc/etcd/kubernetes.pem \
  --peer-key-file=/etc/etcd/kubernetes-key.pem \
  --trusted-ca-file=/etc/etcd/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ca.pem \
  --peer-client-cert-auth \
  --client-cert-auth \
  --initial-advertise-peer-urls https://10.10.40.92:2380 \
  --listen-peer-urls https://10.10.40.92:2380 \
  --listen-client-urls https://10.10.40.92:2379,http://127.0.0.1:2379 \
  --advertise-client-urls https://10.10.40.92:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster 10.10.40.90=https://10.10.40.90:2380,10.10.40.91=https://10.10.40.91:2380,10.10.40.92=https://10.10.40.92:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5


[Install]
WantedBy=multi-user.target
</pre>
<b>8- Reload the daemon configuration.</b>

$ sudo systemctl daemon-reload

<b>9- Enable etcd to start at boot time.</b>

$ sudo systemctl enable etcd

<b>10- Start etcd.</b>

$ sudo systemctl start etcd

<b>11- Verify that the cluster is up and running.</b>

$ ETCDCTL_API=3 etcdctl member list

<b>Initializing the master nodes</br>
Initializing the 10.10.40.90 master node</b>

<b>1- SSH to the 10.10.40.90 machine.</b>

$ ssh sguyennet@10.10.40.90

<b>2- Create the configuration file for kubeadm.</b>
<pre>
$ vim config.yaml
apiVersion: kubeadm.k8s.io/v1alpha3
kind: ClusterConfiguration
kubernetesVersion: stable
apiServerCertSANs:
- 10.10.40.93
controlPlaneEndpoint: "10.10.40.93:6443"
etcd:
  external:
    endpoints:
    - https://10.10.40.90:2379
    - https://10.10.40.91:2379
    - https://10.10.40.92:2379
    caFile: /etc/etcd/ca.pem
    certFile: /etc/etcd/kubernetes.pem
    keyFile: /etc/etcd/kubernetes-key.pem
networking:
  podSubnet: 10.30.0.0/24
apiServerExtraArgs:
  apiserver-count: "3"
</pre>
<b>3- Initialize the machine as a master node.</b>

$ sudo kubeadm init --config=config.yaml

4- Copy the certificates to the two other masters.

$ sudo scp -r /etc/kubernetes/pki sguyennet@10.10.40.91:~

$ sudo scp -r /etc/kubernetes/pki sguyennet@10.10.40.92:~

<h3>Initializing the 10.10.40.91 master node </h3>

<b>1- SSH to the 10.10.40.91 machine.</b>

$ ssh sguyennet@10.10.40.91

<b>2- Remove the apiserver.crt and apiserver.key.</b>

$ rm ~/pki/apiserver.*

<b>3- Move the certificates to the /etc/kubernetes directory.</b>

$ sudo mv ~/pki /etc/kubernetes/

<b>4 - Create the configuration file for kubeadm.</b>
<pre>
$ vim config.yaml
apiVersion: kubeadm.k8s.io/v1alpha3
kind: ClusterConfiguration
kubernetesVersion: stable
apiServerCertSANs:
- 10.10.40.93
controlPlaneEndpoint: "10.10.40.93:6443"
etcd:
  external:
    endpoints:
    - https://10.10.40.90:2379
    - https://10.10.40.91:2379
    - https://10.10.40.92:2379
    caFile: /etc/etcd/ca.pem
    certFile: /etc/etcd/kubernetes.pem
    keyFile: /etc/etcd/kubernetes-key.pem
networking:
  podSubnet: 10.30.0.0/24
apiServerExtraArgs:
  apiserver-count: "3"
</pre>
<b>5- Initialize the machine as a master node.</b>

$ sudo kubeadm init --config=config.yaml

<h3>Initializing the 10.10.40.92 master node</h3>

1- SSH to the 10.10.40.92 machine.

$ ssh sguyennet@10.10.40.92

<b>2- Remove the apiserver.crt and apiserver.key.</b>

$ rm ~/pki/apiserver.*

<b>3- Move the certificates to the /etc/kubernetes directory. </b>

$ sudo mv ~/pki /etc/kubernetes/

<b>4 - Create the configuration file for kubeadm.</b>
<pre>
$ vim config.yaml
apiVersion: kubeadm.k8s.io/v1alpha3
kind: ClusterConfiguration
kubernetesVersion: stable
apiServerCertSANs:
- 10.10.40.93
controlPlaneEndpoint: "10.10.40.93:6443"
etcd:
  external:
    endpoints:
    - https://10.10.40.90:2379
    - https://10.10.40.91:2379
    - https://10.10.40.92:2379
    caFile: /etc/etcd/ca.pem
    certFile: /etc/etcd/kubernetes.pem
    keyFile: /etc/etcd/kubernetes-key.pem
networking:
  podSubnet: 10.30.0.0/24
apiServerExtraArgs:
  apiserver-count: "3"
</pre>
<b>5- Initialize the machine as a master node. </b>

$ sudo kubeadm init --config=config.yaml

<b> 6- Copy the "kubeadm join" command line printed as the result of the previous command.</b>
<h3>Initializing the worker nodes</h3>
<h3>Initializing the 10.10.40.100 worker node</h3>

<b>1- SSH to the 10.10.40.100 machine.</b>

$ ssh sguyennet@10.10.40.100

<b>2- Execute the "kubeadm join" command that you copied from the last step of the initialization of the masters.</b>

$ sudo kubeadm join 10.10.40.93:6443 --token [your_token] --discovery-token-ca-cert-hash sha256:[your_token_ca_cert_hash]

<b>Initializing the 10.10.40.101 worker node </b>

1- SSH to the 10.10.40.101 machine.

$ ssh sguyennet@10.10.40.101

2- Execute the "kubeadm join" command that you copied from the last step of the masters initialization.

$ sudo kubeadm join 10.10.40.93:6443 --token [your_token] --discovery-token-ca-cert-hash sha256:[your_token_ca_cert_hash]

<h3>Initializing the 10.10.40.102 worker node</h3>

<b>1- SSH to the 10.10.40.102 machine. </b>

$ ssh sguyennet@10.10.40.102

<b>2- Execute the "kubeadm join" command that you copied from the last step of the masters initialization. </b>

$ sudo kubeadm join 10.10.40.93:6443 --token [your_token] --discovery-token-ca-cert-hash sha256:[your_token_ca_cert_hash]

<h3>Verifying that the workers joined the cluster</h3>

<b>1- SSH to one of the master node.</b>

$ ssh sguyennet@10.10.40.90

<b>2- Get the list of the nodes.</b>

$ sudo kubectl --kubeconfig /etc/kubernetes/admin.conf get nodes
NAME STATUS ROLES AGE VERSION
k8s-kubeadm-master-0 NotReady master 1h v1.12.1
k8s-kubeadm-master-1 NotReady master 1h v1.12.1
k8s-kubeadm-master-2 NotReady master 1h v1.12.1
k8s-kubeadm-worker-0 NotReady  2m v1.12.1
k8s-kubeadm-worker-1 NotReady  1m v1.12.1
k8s-kubeadm-worker-2 NotReady  1m v1.12.1

The status of the nodes is NotReady as we haven't configured the networking overlay yet.
Configuring kubectl on the client machine

<b>1- SSH to one of the master node.</b>

$ ssh sguyennet@10.10.40.90

<b>2- Add permissions to the admin.conf file. </b>

$ sudo chmod +r /etc/kubernetes/admin.conf

<b> 3- From the client machine, copy the configuration file.</b>

$ scp sguyennet@10.10.40.90:/etc/kubernetes/admin.conf .

<b>4- Create the kubectl configuration directory.</b>

$ mkdir ~/.kube

<b>5- Move the configuration file to the configuration directory.</b>

$ mv admin.conf ~/.kube/config

<b>6- Modify the permissions of the configuration file.</b>

$ chmod 600 ~/.kube/config

<b> 7- Go back to the SSH session on the master and change back the permissions of the configuration file.</b>

$ sudo chmod 600 /etc/kubernetes/admin.conf

<b>8- check that you can access the Kubernetes API from the client machine.</b>

$ kubectl get nodes

<b> Deploying the overlay network </b>

We are going to use Weavenet as the overlay network. You can also use static route or another overlay network tool like Calico or Flannel.

<b>1- Deploy the overlay network pods from the client machine.</b>

$ kubectl apply -f https://git.io/weave-kube-1.6

<b>2- Check that the pods are deployed properly.</b>

$ kubectl get pods -n kube-system

<b>3- Check that the nodes are in Ready state.</b>
<pre>
$ kubectl get nodes
NAME STATUS ROLES AGE VERSION
k8s-kubeadm-master-0 Ready master 18h v1.12.1
k8s-kubeadm-master-1 Ready master 18h v1.12.1
k8s-kubeadm-master-2 Ready master 18h v1.12.1
k8s-kubeadm-worker-0 Ready  16h v1.12.1
k8s-kubeadm-worker-1 Ready  16h v1.12.1
k8s-kubeadm-worker-2 Ready  16h v1.12.1
</pre>
<h3> Installing Kubernetes add-ons<h3>

We will deploy two Kubernetes add-ons on our new cluster: the dashboard add-on to have a graphical view of the cluster, and the Heapster add-on to monitor our workload.

<h4>Installing the Kubernetes dashboard </h4>

<b> 1- Create the Kubernetes dashboard manifest.</b>
<pre>
$ vim kubernetes-dashboard.yaml
# Copyright 2017 The Kubernetes Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.



# Configuration to deploy release version of the Dashboard UI compatible with
# Kubernetes 1.8.
#
# Example usage: kubectl create -f 
# ------------------- Dashboard Secret ------------------- #
apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-certs
  namespace: kube-system
type: Opaque
---
# ------------------- Dashboard Service Account ------------------- #
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
---
# ------------------- Dashboard Role & Role Binding ------------------- #
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: kubernetes-dashboard-minimal
  namespace: kube-system
rules:
  # Allow Dashboard to create 'kubernetes-dashboard-key-holder' secret.
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["create"]
  # Allow Dashboard to create 'kubernetes-dashboard-settings' config map.
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["create"]
  # Allow Dashboard to get, update and delete Dashboard exclusive secrets.
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["kubernetes-dashboard-key-holder", "kubernetes-dashboard-certs"]
  verbs: ["get", "update", "delete"]
  # Allow Dashboard to get and update 'kubernetes-dashboard-settings' config map.
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["kubernetes-dashboard-settings"]
  verbs: ["get", "update"]
  # Allow Dashboard to get metrics from heapster.
- apiGroups: [""]
  resources: ["services"]
  resourceNames: ["heapster"]
  verbs: ["proxy"]
- apiGroups: [""]
  resources: ["services/proxy"]
  resourceNames: ["heapster", "http:heapster:", "https:heapster:"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: kubernetes-dashboard-minimal
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: kubernetes-dashboard-minimal
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system
---
# ------------------- Dashboard Deployment ------------------- #
kind: Deployment
apiVersion: apps/v1beta2
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: kubernetes-dashboard
  template:
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
    spec:
      containers:
      - name: kubernetes-dashboard
        image: k8s.gcr.io/kubernetes-dashboard-amd64:v1.8.3
        ports:
        - containerPort: 8443
          protocol: TCP
        args:
          - --auto-generate-certificates
          # Uncomment the following line to manually specify Kubernetes API server Host
          # If not specified, Dashboard will attempt to auto discover the API server and connect
          # to it. Uncomment only if the default does not work.
          # - --apiserver-host=http://my-address:port
        volumeMounts:
        - name: kubernetes-dashboard-certs
          mountPath: /certs
          # Create on-disk volume to store exec logs
        - mountPath: /tmp
          name: tmp-volume
        livenessProbe:
          httpGet:
            scheme: HTTPS
            path: /
            port: 8443
          initialDelaySeconds: 30
          timeoutSeconds: 30
      volumes:
      - name: kubernetes-dashboard-certs
        secret:
          secretName: kubernetes-dashboard-certs
      - name: tmp-volume
        emptyDir: {}
      serviceAccountName: kubernetes-dashboard
      # Comment the following tolerations if Dashboard must not be deployed on master
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
---
# ------------------- Dashboard Service ------------------- #
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  ports:
    - port: 443
      targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
</pre>
<b> 2- Deploy the dashboard.</b>

$ kubectl create -f kubernetes-dashboard.yaml

<h3> Installing Heapster</h3>

<b> 1- Create a manifest for Heapster. </b>
<pre>
$ vim heapster.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: heapster
  namespace: kube-system
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: heapster
  namespace: kube-system
spec:
  replicas: 1
  template:
    metadata:
      labels:
        task: monitoring
        k8s-app: heapster
    spec:
      serviceAccountName: heapster
      containers:
      - name: heapster
        image: gcr.io/google_containers/heapster-amd64:v1.4.2
        imagePullPolicy: IfNotPresent
        command:
        - /heapster
        - --source=kubernetes.summary_api:''?useServiceAccount=true&kubeletHttps=true&kubeletPort=10250&insecure=true
---
apiVersion: v1
kind: Service
metadata:
  labels:
    task: monitoring
    # For use as a Cluster add-on (https://github.com/kubernetes/kubernetes/tree/master/cluster/addons)
    # If you are NOT using this as an addon, you should comment out this line.
    kubernetes.io/cluster-service: 'true'
    kubernetes.io/name: Heapster
  name: heapster
  namespace: kube-system
spec:
  ports:
  - port: 80
    targetPort: 8082
  selector:
    k8s-app: heapster
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: heapster
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:heapster
subjects:
- kind: ServiceAccount
  name: heapster
  namespace: kube-system
</pre>
<b> 2- Deploy Heapster. </b>

$ kubectl create -f heapster.yaml

<b> 3- Edit the Heapster RBAC role and add the get permission on the nodes statistic at the end. </b>

$ kubectl edit clusterrole system:heapster
<pre>...
- apiGroups:
  - ""
  resources:
  - nodes/stats
  verbs:
  - get
</pre>
<h3> Accessing the Kubernetes dashboard </h3>

<b> 1- Create an admin user manifest.</b>
<pre>
$ vim kubernetes-dashboard-admin.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system </pre>

<b> 2- Create the admin user. </b>

$ kubectl create -f kubernetes-dashboard-admin.yaml

<b> 3- Get the admin user token. </b>

$ kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')

<b> 4- Copy the token. </b>

<b> 5- Start the proxy to access the dashboard.</b>

$ kubectl proxy




