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


![CNI](https://github.com/jiwonYun9332/AWES-1/blob/bbe2a949bc1d7b37ffaa4978ab2efd3790269ea7/Study/images/2_1_cni.png)


Amazon VPC CNI에는 두 가지 구성 요소가 있다.

**CNI 바이너리**
: 포드 간 통신을 활성화하도록 포드 네트워크를 설정한다, CNI 바이너리는 노드 루트 파일 시스템에서 실행되며 새 Pod가 추가되거나 기존 Pod가 노드에서 제거될 때 kubelet에 의해 호출된다.

**ipamd**
 - 노드에서 ENI 관리
 - 사용 가능한 IP 주소 또는 접두사의 웜 풀 유지

인스턴스가 생성되면 EC2는 기본 서브넷과 연결된 기본 ENI를 생성하고 연결한다. hostNetwork 모드에서 실행되는 포드는 노드 기본 ENI에 할당된
기본 IP 주소를 사용하고 호스트와 동일한 네트워크 네임스페이스를 공유한다.

CNI 플러그인은 노드가 프로비저닝되면 CNI 플러그인은 노드의 서브넷에서 기본 ENI로 슬롯 풀을 자동으로 할당한다.
ENI의 슬롯이 할당되면 CNI는 웜 슬롯 풀이 있는 추가 ENI를 노드에 연결할 수 있다. 이러한 추가 ENI를 보조 ENI이라고 한다.

CNI는 포드 수에 해당하는 필요한 슬롯 수에 따라 더 많은 ENI를 인스턴스에 연결한다. 이 프로세스는 노드가 더 이상의 추가 ENI를 지원할 수 없을 때까지
계속된다. 

제약조건
 - 인스턴스 유형
 : 각 인스턴스 유형에 따라 연결할 수 있는 최대 ENI 수가 있다.
 - 컴퓨팅 리소스
 - 포드 밀도(노드당 포드 수)

아래 스크립트를 사용하여, 인스턴스 유형에 따른 최대 포드 사용 갯수를 확인할 수 있다.

https://github.com/awslabs/amazon-eks-ami/blob/master/files/max-pods-calculator.sh 


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

``` Bash
# bash max-pods-calculator.sh --instance-type t2.micro --cni-version 1.12.6-eksbuild.1 --cni-prefix-delegation-enabled
4
```

t2.micro 인스턴스 유형은 최대 4개의 포드까지 사용할 수 있다. (Warm-pool 크기 4)


Worker node에 aws-node라는 Daemonset으로 배포된다.

```
k get all --all-namespaces
```

![awsnode](https://github.com/jiwonYun9332/AWES-1/blob/798792a9154409de9fef20aab75cf225a161c956/Study/images/2_2_awsnode.png)

현재, 3개의 노드와 3개의 kube-proxy, 2개의 coreDNS로 클러스터가 구성되어 있는 것을 확인할 수 있다.
여기서 잠깐 CoreDNS에 대해 알아본다.

### CoreDNS

쿠버네티스 클러스터 내 POD에서 어떤 도메인을 찾고자 할 때 kube-system 네임스페이스에 실행되고 있는 CoreDNS가 네임서버로 사용된다.
기존에는 Kube-DNS가 해당 역할을 하였는데, 1.12 버전부터 CoreDNS가 표준으로 채택되었다.

설치 확인

```
# kubectl get po -n kube-system -l k8s-app=kube-dns
NAME                       READY   STATUS    RESTARTS   AGE
coredns-6777fcd775-j9fmp   1/1     Running   0          178m
coredns-6777fcd775-qk6hc   1/1     Running   0          178m
```

```
# kubectl get svc -n kube-system -l k8s-app=kube-dns
NAME       TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)         AGE
kube-dns   ClusterIP   10.100.0.10   <none>        53/UDP,53/TCP   3h8m
```

Name이 kube-dns으로 되어 있다. 이는 해당 이름으로 사용하던 기존 리소스와의 호환성을 위해 그대로 사용한다고 한다.

/etc/resolv.conf 설정 값을 확인해보면 아래와 같이 설정되어 있는 것을 확인할 수 있다.

```
search ap-northeast-2.compute.internal
options timeout:2 attempts:5
nameserver 192.168.0.2
```

- search ap-northeast-2.compute.internal
  : DNS에 질의할 부분 도메인 주소 경로들을 표시
- options
  - timeouts: timeout 설정
  - attempts: 재시도 횟수
- nameserver: DNS 쿼리를 보낼 곳으로 현재 실습환경에서는 CoreDNS Service 오브젝트이 IP 주소가 설정되어 있다.

※ 

/etc/resolv 의 nameserver 의 주소가 192.168.0.2 로 설정되어 있다.

0.2 의 주소의 경우 AWS에서 기본적으로 DNS 주소로 사용하는 주소이다.

현재 배포받은 환경에서는 CoreDNS로 설정되어 있지 않은 것을 확인할 수 있다. 특별한 이유가 있는 것인지에 대해 확인을 해봐야할 것 같다.

※


다시 CNI으로 돌아와서, ENI가 생성되고 IP가 할당되는 동작과정을 따라간다. 먼저 coredns 파드의 IP를 확인한 후,
해당 노드에 ENI와 비교해본다.

**coredns 파드 IP 정보 확인**

```
# kubectl get pod -n kube-system -l k8s-app=kube-dns -owide
NAME                       READY   STATUS    RESTARTS   AGE     IP              NODE                                               NOMINATED NODE   READINESS GATES
coredns-6777fcd775-j9fmp   1/1     Running   0          5h26m   192.168.2.156   ip-192-168-2-58.ap-northeast-2.compute.internal    <none>           <none>
coredns-6777fcd775-qk6hc   1/1     Running   0          5h26m   192.168.3.99    ip-192-168-3-243.ap-northeast-2.compute.internal   <none>           <none>
```

**노드의 라우팅 정보 확인 >> EC2 네트워크 정보의 '보조 프라이빗 IPv4 주소'와 비교**

```
# ssh ec2-user@$N1 sudo ip -c route
default via 192.168.1.1 dev eth0
169.254.169.254 dev eth0
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.29
```

```
# ssh ec2-user@$N2 sudo ip -c route
default via 192.168.2.1 dev eth0
169.254.169.254 dev eth0
192.168.2.0/24 dev eth0 proto kernel scope link src 192.168.2.58
192.168.2.156 dev enicd25d3bb248 scope link
```

```
# ssh ec2-user@$N3 sudo ip -c route
default via 192.168.3.1 dev eth0
169.254.169.254 dev eth0
192.168.3.0/24 dev eth0 proto kernel scope link src 192.168.3.243
192.168.3.99 dev eni30e3e9d2eea scope link
```

![secondary_ip](https://github.com/jiwonYun9332/AWES-1/blob/1bc0ace9d0b9e0af8a2e3cfc2ac2f4896e9bd11f/Study/images/2_3_ip.png)

coreDNS IP가 192.168.3.99 인 파드로 Node3에서 라우팅 설정되어 있는 것을 볼 수 있다.

3번 노드에 새로운 파드 "netshoot-pod"를 생성한 후 다시 한 번 확인해보자

> 테스트용 파드 생성

```
cat <<EOF | kubectl create -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: netshoot-pod
spec:
  replicas: 3
  selector:
    matchLabels:
      app: netshoot-pod
  template:
    metadata:
      labels:
        app: netshoot-pod
    spec:
      containers:
      - name: netshoot-pod
        image: nicolaka/netshoot
        command: ["tail"]
        args: ["-f", "/dev/null"]
      terminationGracePeriodSeconds: 0
EOF
```

> 생성된 파드 확인

```
# kubectl get pod -o=custom-columns=NAME:.metadata.name,IP:.status.podIP
NAME                            IP
netshoot-pod-7757d5dd99-2t7qd   192.168.3.107
netshoot-pod-7757d5dd99-sd6lm   192.168.1.40
netshoot-pod-7757d5dd99-vlq2z   192.168.2.91
```

위에 보조IP 중 하나였던 192.168.3.107 IP가 할당된 것을 볼 수 있다.

Node3 의 라우팅 정보를 다시 확인해보면 아래와 같이 생성한 파드에 라우팅 정보가 추가된 것을 볼 수 있다.

```
# ssh ec2-user@$N3 sudo ip -c route
default via 192.168.3.1 dev eth0
169.254.169.254 dev eth0
192.168.3.0/24 dev eth0 proto kernel scope link src 192.168.3.243
192.168.3.99 dev eni30e3e9d2eea scope link
192.168.3.107 dev eni849a9f7053a scope link
```

![podIp](https://github.com/jiwonYun9332/AWES-1/blob/01f72661e4bad43174d873e9516d9f26f5517d68/Study/images/2_4_podIp.png)

**위 실습을 통해 세컨더리 IP가 파드에 할당되는 것을 알 수 있다.**

그러면 방금 생성한 파드 192.168.3.107은 3번 노드의 어떤 ENI를 통해 통신이 되는 지 확인해보자

## TCPDUMP Test

![3node](https://github.com/jiwonYun9332/AWES-1/blob/4e7b8c8a4a7bb18478dd8652227c8fe711eb91fa/Study/images/2_5_node3.png)

![eth0](https://github.com/jiwonYun9332/AWES-1/blob/8f1f9ea4842a06e2dade1a91dbca8b90b0da2866/Study/images/2_6_eth0.png)

![eth1](https://github.com/jiwonYun9332/AWES-1/blob/8f1f9ea4842a06e2dade1a91dbca8b90b0da2866/Study/images/2_6_eth1.png)

3번 노드의 2개의 ENI 중 192.168.3.107은 eth0의 보조 세컨더리 IP이다.

![pingTest](https://github.com/jiwonYun9332/AWES-1/blob/e5635d26df791c12a42a8483a17031a2fc80c0c4/Study/images/2_7_podNetwork.png)

테스트 파드에서 192.168.1.1으로 ping을 보내 eth0과 eth1 중 어떤 ENI를 통해 통신하고 있는 지 확인할 수 있다.

![eth1](https://github.com/jiwonYun9332/AWES-1/blob/e5635d26df791c12a42a8483a17031a2fc80c0c4/Study/images/2_8_podNetwork.png)

![eth0](https://github.com/jiwonYun9332/AWES-1/blob/e5635d26df791c12a42a8483a17031a2fc80c0c4/Study/images/2_8_podNetwork2.png)

위 결과에서도 확인할 수 있듯, eth0을 통해 네트워크 통신을 하고 있음을 알 수 있다.

### 파드 간 네트워크 통신, calico vs ENI

calico 와 aws cni 의 파드 간의 통신 간 차이점에 대해 알아본다.

![calico](https://github.com/jiwonYun9332/AWES-1/blob/89719821dd0bdefb18cae84c5e1960408e2a565f/Study/images/2_10_calico.png)

칼리코는 외부와의 통신 시 iptables 을 통해 NAT 되어 파드 IP가 아닌 노드 IP의 eth0 인터페이스 IP로 변환되어 통신하게 된다.

```
root@ubun20-81:~# calicoctl get ippool -o wide
NAME           CIDR             NAT    IPIPMODE   VXLANMODE   DISABLED   DISABLEBGPEXPORT   SELECTOR
default-pool   10.233.64.0/18   true   Always     Never       false      false              all()
```

```
# iptables -n -t nat --list cali-nat-outgoing
Chain cali-nat-outgoing (1 references)
target     prot opt source               destination
MASQUERADE  all  --  0.0.0.0/0            0.0.0.0/0            /* cali:flqWnvo8yq4ULQLa */ match-set cali40masq-ipam-pools src ! match-set cali40all-ipam-pools dst random-fully
```

위 정책을 확인하면 알 수 있듯이 모든 0.0.0.0 도착지에 대해 (cali-nat-outgoing) MASQUERADE 설정이 되어 있다.

반면에 AWS ENI을 이용한 파드 간의 통신의 경우

![podNetworking](https://github.com/jiwonYun9332/AWES-1/blob/bdbc8c1e93a45ad243be199ecbe16628e88db1c3/Study/images/2_9_podNetwork.png)

Overlay 기술 없이 파드 간의 통신이 가능하다.








