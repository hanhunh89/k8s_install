# k8s_study

gcp e2c에서 k8s를 수동으로 설치한다. <br>
참고 : e2c는 debian linux를 사용한다. 

## [master,worker] linux update
```
sudo apt-get update && sudo apt-get upgrade
```

## [master,worker] apt-transport-https, curl 설치
```
sudo apt install -y apt-transport-https curl
```

## [master,worker] docker install 
https://docs.docker.com/engine/install/debian/ 를 참조함
```
#old 버전 삭제. 처음 설치하면 안해도됨. 
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

## [master,worker] docker run
```
sudo systemctl start docker
sudo systemctl enable docker
sudo systemctl status docker # running 확인
```

## [master,worker] swap mem 비활성화
```
sudo swapoff -a # swap 비활성화
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab # /etc/fstab파일의 swap 관련 내용 모두 주
```
## [master,worker] kubelet, kubeadm, kubectl, kubernetes-cni 패키지가 위치한 레포지터리 지정
```
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key --keyring /usr/share/keyrings/kubernetes-archive-keyring.gpg add -
sudo apt-get update
```

## [master,worker] k8s 설치
```
sudo apt install -y kubeadm kubelet kubectl kubernetes-cni
```

## [master] containerd 설정 삭제 
```
sudo rm /etc/containerd/config.toml
sudo service containerd restart
```

## [master] kubeadm으로 클러스터 생성
```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=[master node ip]
#kubeadm init  쿠버네티스 클러스터 생성
#--pod-network-cidr=10.244.0.0/16  pod의 네트워크 대역 설정
#--apiserver-advertise-address=123.123.123.123  API 서버가 사용할 IP 주소를 명시적으로 지정
```

## [master] 설정파일 복사
```
sudo su - 
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## [master] 설정파일 확인
sudo kubectl config view를  했을 때 아래와 유사하게 나와야 한다.
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

아래와 같이 나오면 설정파일이 적용이 안된것이다.
```
apiVersion: v1
clusters: null
contexts: null
current-context: ""
kind: Config
preferences: {}
users: null
```

설정파일은 /etc/kubernetes/admin.conf이다<br>
export $KUBECONFIG=/etc/kubernetes/admin.conf 로 환경변수를 등록하거나,<br>
cp /etc/kubernetes/admin.conf $HOME/.kube/config 로 홈 디렉토리에 환경변수를 넣어주어야 한다. <br>

sudo를 사용하여 쿠버네티스를 구동한다면, root의 홈 디렉토리에 설정파일을 넣어주어야 한다. 


## [master] 네트워크 설정
서로 다른 pods간 통신을 위해서는 CNI 플러그인이 필요하다. 
```
sudo kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```


## [master] worker 노드 join 명령어 확인
아래의 명령어를 쳐서 join 명령어를 확인한다. 

```
kubeadm token create --print-join-command
```
kubeadm join 10.178.0.13:6443 --token b6kdl1.5267eqpt8jd2bi2x --discovery-token-ca-cert-hash sha256:adedc0a64cbacddfe2a86b43ee5465ecb279d67f83206141e8a229c5d72334a2 
이와 유사한 출력이 나올 것이다. 이를 복사해서 worker node에서 실행한다. 

## [worker] master에 접속
```
kubeadm join 10.178.0.13:6443 --token b6kdl1.5267eqpt8jd2bi2x --discovery-token-ca-cert-hash sha256:adedc0a64cbacddfe2a86b43ee5465ecb279d67f83206141e8a229c5d72334a2
```

## [master] worker가 잘 접속되었는지 확인
```
sudo kubectl get nodes
```
아래와 같이 worker1이 추가된 것을 확인할 수 있다. 끗.

NAME         STATUS     ROLES           AGE     VERSION
instance-1   NotReady   control-plane   24m     v1.28.2
worker1      NotReady   <none>          2m36s   v1.28.2



## 쿠버네티스 구성 삭제 후 재설치[필요시 시행]
```
kubeadm reset 
sudo apt-get -y purge kubeadm kubectl kubelet kubernetes-cni
sudo apt-get -y autoremove
sudo rm -rf ~/.kube/*

sudo apt install -y kubeadm kubelet kubectl kubernetes-cni
sudo kubeadm init
```
