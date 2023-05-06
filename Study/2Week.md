# 2Week

![eks logo](https://github.com/jiwonYun9332/AWES-1/blob/988ee97c30ec07bc1a3360fef09b0c944f5f0530/Study/images/1_logo.png)


## 2주차 Study(Networking)

2주차 스터디 내용은 EKS Network 환경 내용으로 진행되었다.


### AWS VPC CNI

첫 번째 네트워크 키워드는 "AWS VPC CNI"이다. 해당 내용에 들어가기 앞서 사전지식을 학습한다.

### CNI
: CNCF(Cloud Native Computing Foundation)의 프로젝트 중 하나인 CNI는 컨테이너 간의 네트워킹을 제어할 수 있는 플러그인을 만들기 위한 표준이다.
다양한 컨테이너 런타임과 오케스트레이터 사이의 네트워크 계층을 구현하는 방식이 각자의 방식으로 발전하게 되는 것을 방지하고 공통된 인터페이스를 제공하기 위해 만들어졌다.

### K8S CNI(Container Network Interface)
: 쿠버네티스에서 Pod 간 통신을 위해 CNI를 사용한다.

### AWS VPC CNI
:

AWS에서 쿠버네티스를 설치할 때 네트워크 설정, 동작은 AWS VPC 환경 위에 설치하게 된다.
따라서 AWS와 쿠버네티스 사이 중계 역할을 하는 모듈이 필요하며. 이때 K8S CNI 플러그인 중 하나인 AWS VPC CNI으로 VPC와 통합이 된다.


"AWS VPC CNI Plugin for Kubernetes"는 Amazon EKS 클러스터의 각 Amazon EC2 노드에 배포된다.
배포된 플러그인은 인스턴스의 ENI를 생성하여 Amazon EC2 노드에 연결되고, IPv4 or IPv6 주소를 VPC에서 각 pod 와 서비스에 할당한다.







AWS VPC CNI Version 확인

```Bash
# kubectl describe daemonset aws-node --namespace kube-system | grep Image | cut -d "/" -f 2
amazon-k8s-cni-init:v1.12.6-eksbuild.1
amazon-k8s-cni:v1.12.6-eksbuild.1
```

``` Bash
# kubectl describe daemonset aws-node --namespace kube-system | grep amazon-k8s-cni: | cut -d : -f 3
v1.12.6-eksbuild.1
```





