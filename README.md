# k8s_study

## [master,worker] docker install 
```
sudo apt-get install -y docker.io
```

## [master,worker] docker run
```
sudo systemctl start docker
sudo systemctl enable docker
```

## [master,worker] swap mem 비활성화
```
sudo swapoff -a # swap 비활성화
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab # /etc/fstab파일의 swap 관련 내용 모두 주
```

## [master,worker] k8s 설치
```
sudo apt-get update && sudo apt-get install -y apt-transport-https curl  # 필요 패키지 설치
sudo curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -  #쿠버네티스 apt 저장소를 신뢰할 수 있는 저장소로 저
echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list #쿠버네티스 apt 저장소 추
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl kubernetes-cni
```

## [master] kubeadm으로 클러스터 생성
```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=172.30.1.26
#kubeadm init  쿠버네티스 클러스터 생성
#--pod-network-cidr=10.244.0.0/16  pod의 네트워크 대역 설정
#--apiserver-advertise-address=172.30.1.26  API 서버가 사용할 IP 주소를 명시적으로 지정
```
