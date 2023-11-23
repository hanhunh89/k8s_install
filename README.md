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
