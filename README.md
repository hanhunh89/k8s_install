# k8s install


date : 2023.11.30.<br>
kubectl version : 1.28.2

install k8s in gcp e2c<br>
e2c use debian

[master] indicates performing tasks on the master server.<br>
[worker] indicates performing tasks on the worker server.<br>
[master, worker] means both.

## [master,worker] linux update
```
sudo apt-get update && sudo apt-get upgrade
```

## [master,worker] apt-transport-https, curl install
```
sudo apt install -y apt-transport-https curl
```

## [master,worker] docker install 
you can find more detail https://docs.docker.com/engine/install/debian/
```
#[optional] delete old version
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove $pkg; done   
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add the repository to Apt sources:
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

#install latest version
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

#check running
sudo docker run hello-world
```

## [master,worker] swap mem off
```
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab # commenting out swap option in /etc/fstab
```

## [master,worker] repository for kubelet, kubeadm, kubectl, kubernetes-cni 
```
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key --keyring /usr/share/keyrings/kubernetes-archive-keyring.gpg add -
sudo apt-get update
```

## [master,worker] install k8s
```
sudo apt install -y kubeadm kubelet kubectl kubernetes-cni
```

## [master] change containerd config 
you can find detail in https://kubernetes.io/docs/setup/production-environment/container-runtimes/#docker
```
# change config.toml file to default
sudo sh -c 'containerd config default > /etc/containerd/config.toml'
# change sandbox_image version
sudo sed -i 's|sandbox_image = "registry\.k8s\.io/pause:3\.6"|sandbox_image = "registry\.k8s\.io/pause:3.9"|g' /etc/containerd/config.toml
# change systemd option
sudo sed -i 's|SystemdCgroup = false|SystemdCgroup = true|g' /etc/containerd/config.toml

sudo service containerd restart
```
Why make default config file?<br>
In k8s document, you can make config.toml with small config.
But... I got errorr... I don't know why.<br>
But, if you write empty config, containerd read default config.<br>
So, i just make config file with default config, and change it.<br>
I don't think it is perfect why, but it works!<br>

And, why i use sandbox_image version 3.9? <br>
When i use default version, 3.6, i get error in "kubeadm init" <br>
error said, i have to use version 3.9. So i changed it.<br>
<p>$\bf{\large{\color{#DD6565}If \ you \ use \ not \ good \ sandbox \ version, \ your \ pod \ go \ to \ CrashLoopBackOff, \ restart \ again \ again \ again... }}$</p>


## [master] create cluster with kubeadm
```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=[master node ip]

#kubeadm init   create k8s cluster
#--pod-network-cidr=10.244.0.0/16  set pod network range
#--apiserver-advertise-address=123.123.123.123  set API ip. default is the server that you create cluster
```

## [master] copy config file to your $home
```
sudo su - 
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## [master] check your config file
sudo kubectl config view -> you have to get result like that.
```
root@master:~# kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://10.178.0.11:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: DATA+OMITTED
    client-key-data: DATA+OMITTED
```

if you see result like that, you don't have config file in $home
```
apiVersion: v1
clusters: null
contexts: null
current-context: ""
kind: Config
preferences: {}
users: null
```
if you run in root, path is "/root/.kube/config"


## [master] confirm running k8s
all pod is peding or running.<br>
if you got crashloopbackoff, .... cry.... and go to "change containerd config"
```
$ kubectl get pods --all-namespaces
NAMESPACE     NAME                                  READY   STATUS    RESTARTS   AGE
kube-system   coredns-5dd5756b68-c9jrf              0/1     Pending   0          3m21s
kube-system   coredns-5dd5756b68-wf8bb              0/1     Pending   0          3m21s
kube-system   etcd-master-node                      1/1     Running   0          3m35s
kube-system   kube-apiserver-master-node            1/1     Running   0          3m35s
kube-system   kube-controller-manager-master-node   1/1     Running   0          3m35s
kube-system   kube-proxy-42dsg                      1/1     Running   0          3m21s
kube-system   kube-scheduler-master-node            1/1     Running   0          3m35s```
```


## [master] install CNI flugin, flannel
for networking within pod in different node, you have to install CNI flugin<br>
in this project, we install flannel 
```
mamdjango2@master-node:~$ sudo kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
namespace/kube-flannel created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds created

mamdjango2@master-node:~$ sudo kubectl get pods --all-namespaces

NAMESPACE      NAME                                  READY   STATUS    RESTARTS   AGE
kube-flannel   kube-flannel-ds-gtvqz                 1/1     Running   0          16s
kube-system    coredns-5dd5756b68-c9jrf              1/1     Running   0          41m
kube-system    coredns-5dd5756b68-wf8bb              1/1     Running   0          41m
kube-system    etcd-master-node                      1/1     Running   0          42m
kube-system    kube-apiserver-master-node            1/1     Running   0          42m
kube-system    kube-controller-manager-master-node   1/1     Running   0          42m
kube-system    kube-proxy-42dsg                      1/1     Running   0          41m
kube-system    kube-scheduler-master-node            1/1     Running   0          42m
```
cordns and flannel have to be Running.

## [master] find join command 

```
$sudo kubeadm token create --print-join-command
kubeadm join 10.178.0.13:6443 --token b6kdl1.5267eqpt8jd2bi2x --discovery-token-ca-cert-hash sha256:adedc0a64cbacddfe2a86b43ee5465ecb279d67f83206141e8a229c5d72334a2 
```
copy "kubeadm join ~ " and paste on workter node.

## [worker] rm containerd config
```
sudo rm /etc/containerd/config.toml
sudo service containerd restart
```

## [worker] join to master
```
sudo kubeadm join 10.178.0.13:6443 --token b6kdl1.5267eqpt8jd2bi2x --discovery-token-ca-cert-hash sha256:adedc0a64cbacddfe2a86b43ee5465ecb279d67f83206141e8a229c5d72334a2
```

## [master] Check if the worker has successfully joined.

```
$ sudo kubectl get nodes
NAME          STATUS   ROLES           AGE    VERSION
master-node   Ready    control-plane   60m    v1.28.2
worker1       Ready    <none>          117s   v1.28.2
```
you can find worker1. 



# utils

 
## remove k8s and install again
```
kubeadm reset 
sudo apt-get -y purge kubeadm kubectl kubelet kubernetes-cni
sudo apt-get -y autoremove
sudo rm -rf ~/.kube/*

sudo apt install -y kubeadm kubelet kubectl kubernetes-cni
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```
