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

실습

```
(eks@myeks:N/A) [root@myeks-bastion-EC2 ~]# kubectl exec -it $PODNAME2 -- ping -c 2 $PODIP3
PING 192.168.2.91 (192.168.2.91) 56(84) bytes of data.
64 bytes from 192.168.2.91: icmp_seq=1 ttl=62 time=0.919 ms
64 bytes from 192.168.2.91: icmp_seq=2 ttl=62 time=0.820 ms

--- 192.168.2.91 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1011ms
rtt min/avg/max/mdev = 0.820/0.869/0.919/0.049 ms
(eks@myeks:N/A) [root@myeks-bastion-EC2 ~]# kubectl exec -it $PODNAME3 -- ping -c 2 $PODIP1
PING 192.168.3.107 (192.168.3.107) 56(84) bytes of data.
64 bytes from 192.168.3.107: icmp_seq=1 ttl=62 time=1.48 ms
64 bytes from 192.168.3.107: icmp_seq=2 ttl=62 time=1.23 ms

--- 192.168.3.107 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 1.231/1.357/1.484/0.126 ms
(eks@myeks:N/A) [root@myeks-bastion-EC2 ~]# kubectl exec -it $PODNAME1 -- ping -c 2 $PODIP2
PING 192.168.1.40 (192.168.1.40) 56(84) bytes of data.
64 bytes from 192.168.1.40: icmp_seq=1 ttl=62 time=1.13 ms
64 bytes from 192.168.1.40: icmp_seq=2 ttl=62 time=2.02 ms
```

![tcpdump](https://github.com/jiwonYun9332/AWES-1/blob/a4e0939c1cc6022ed28b7c94a760ca9946c4317a/Study/images/2_11_tcpdump.png)

파드 간의 통신을 확인할 수 있다.

### 파드 외부 통신

파드에서 외부 www.google.com 으로 ping을 보내 외부 통신이 가능한 지 확인한다.

```
# kubectl exec -it $PODNAME2 -- ping -c 1 www.google.com
```

```
$ sudo tcpdump -i any -nn icmp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on any, link-type LINUX_SLL (Linux cooked), capture size 262144 bytes
19:36:00.841769 IP 192.168.1.40 > 172.217.175.36: ICMP echo request, id 10290, seq 1, length 64
19:36:00.841791 IP 192.168.1.29 > 172.217.175.36: ICMP echo request, id 11847, seq 1, length 64
19:36:00.870177 IP 172.217.175.36 > 192.168.1.29: ICMP echo reply, id 11847, seq 1, length 64
19:36:00.870212 IP 172.217.175.36 > 192.168.1.40: ICMP echo reply, id 10290, seq 1, length 64
```

IPTable을 확인하여, eth0 주소인 "192.168.1.29"으로 SNAT 되는 것을 확인할 수 있다.

```
$ sudo iptables -t nat -S | grep 'A AWS-SNAT-CHAIN'
-A AWS-SNAT-CHAIN-0 ! -d 192.168.0.0/16 -m comment --comment "AWS SNAT CHAIN" -j AWS-SNAT-CHAIN-1
-A AWS-SNAT-CHAIN-1 ! -o vlan+ -m comment --comment "AWS, SNAT" -m addrtype ! --dst-type LOCAL -j SNAT --to-source 192.168.1.29 --random-fully
```

```
$ sudo conntrack -L -n |grep -v '169.254.169'
icmp     1 29 src=192.168.1.40 dst=8.8.8.8 type=8 code=0 id=31305 src=8.8.8.8 dst=192.168.1.29 type=0 code=0 id=7780 mark=128 use=1
tcp      6 431974 ESTABLISHED src=192.168.1.29 dst=15.165.80.13 sport=58860 dport=443 src=15.165.80.13 dst=192.168.1.29 sport=443 dport=54487 [ASSURED] mark=128 use=1
```


### 노드에서 파트 생성 갯수제한


각 인스턴스 유형별 최대 ENI 갯수가 다르다.
awe-node와 kube-proxy의 경우 호스트 IP를 사용하기 때문에 최대 갯수에서는 제외하고 계산한다.

**최대 파드 생성 갯수 공식**
```
(Number of network interfaces for the instance type × 
(the number of IP addressess per network interface - 1)) + 2
```

EKS 구성한 Node의 인스턴스 유형은 모두 t3.medium 이다.

t3 타입의 인스턴스가 파드를 최대 몇 개까지 생성할 수 있는 지 확인하는 AWS CLI 명령어이다.

```
# aws ec2 describe-instance-types --filters Name=instance-type,Values=t3.* \
>  --query "InstanceTypes[].{Type: InstanceType, MaxENI: NetworkInfo.MaximumNetworkInterfaces, IPv4addr: NetworkInfo.Ipv4AddressesPerInterface}" \
>  --output table
--------------------------------------
|        DescribeInstanceTypes       |
+----------+----------+--------------+
| IPv4addr | MaxENI   |    Type      |
+----------+----------+--------------+
|  15      |  4       |  t3.2xlarge  |
|  15      |  4       |  t3.xlarge   |
|  6       |  3       |  t3.medium   |
|  12      |  3       |  t3.large    |
|  2       |  2       |  t3.nano     |
|  2       |  2       |  t3.micro    |
|  4       |  3       |  t3.small    |
+----------+----------+--------------+

```

사용가능한 파드IP
((MaxENI * (IPv4addr-1)) + 2)
t3.medium 경우 : ((3 * (6 - 1) + 2 ) = 17개


```
# kubectl scale deployment nginx-deployment --replicas=17

# kubectl get pod -o=custom-columns=NAME:.metadata.name,IP:.status.podIP
NAME                                IP
nginx-deployment-6fb79bc456-44t7d   192.168.3.222
nginx-deployment-6fb79bc456-5k6m6   192.168.2.12
nginx-deployment-6fb79bc456-6lc5f   192.168.2.51
nginx-deployment-6fb79bc456-bf25x   192.168.1.149
nginx-deployment-6fb79bc456-cpzzt   192.168.2.91
nginx-deployment-6fb79bc456-cxc2h   192.168.3.107
nginx-deployment-6fb79bc456-g22x4   192.168.2.33
nginx-deployment-6fb79bc456-k5r4p   192.168.1.116
nginx-deployment-6fb79bc456-lj7hs   192.168.1.221
nginx-deployment-6fb79bc456-n2lhq   192.168.3.241
nginx-deployment-6fb79bc456-ns949   192.168.2.104
nginx-deployment-6fb79bc456-q2ql2   192.168.1.40
nginx-deployment-6fb79bc456-q7lmr   192.168.3.19
nginx-deployment-6fb79bc456-rz49w   192.168.3.232
nginx-deployment-6fb79bc456-swt6m   192.168.3.131
nginx-deployment-6fb79bc456-x6wdt   192.168.1.46
nginx-deployment-6fb79bc456-xq98c   192.168.1.133

```


최대 갯수인 17개의 pod를 생성하였을 때는 모두 ip가 할당되었다. 그러면 30으로 하면 어떻게 될까?

```
# kubectl scale deployment nginx-deployment --replicas=30

kubectl get pod -o=custom-columns=NAME:.metadata.name,IP:.status.pod
NAME                                IP
nginx-deployment-6fb79bc456-44t7d   <none>
nginx-deployment-6fb79bc456-4pwxv   <none>
nginx-deployment-6fb79bc456-5k6m6   <none>
nginx-deployment-6fb79bc456-6lc5f   <none>
nginx-deployment-6fb79bc456-77n9z   <none>
nginx-deployment-6fb79bc456-8xxcb   <none>
nginx-deployment-6fb79bc456-b4tl2   <none>
nginx-deployment-6fb79bc456-bf25x   <none>
nginx-deployment-6fb79bc456-cjrfl   <none>
nginx-deployment-6fb79bc456-cpzzt   <none>
nginx-deployment-6fb79bc456-cxc2h   <none>
nginx-deployment-6fb79bc456-d47jn   <none>
nginx-deployment-6fb79bc456-g22x4   <none>
nginx-deployment-6fb79bc456-k5r4p   <none>
nginx-deployment-6fb79bc456-krtqt   <none>
nginx-deployment-6fb79bc456-lhn96   <none>
nginx-deployment-6fb79bc456-lj7hs   <none>
nginx-deployment-6fb79bc456-n2lhq   <none>
nginx-deployment-6fb79bc456-ns949   <none>
nginx-deployment-6fb79bc456-q2ql2   <none>
nginx-deployment-6fb79bc456-q7lmr   <none>
nginx-deployment-6fb79bc456-r294p   <none>
nginx-deployment-6fb79bc456-rqqjk   <none>
nginx-deployment-6fb79bc456-rz49w   <none>
nginx-deployment-6fb79bc456-swt6m   <none>
nginx-deployment-6fb79bc456-wk7ht   <none>
nginx-deployment-6fb79bc456-wvs9l   <none>
nginx-deployment-6fb79bc456-x6wdt   <none>
nginx-deployment-6fb79bc456-xq98c   <none>
nginx-deployment-6fb79bc456-zjcmm   <none>
```

기존에 IP가 할당되어 있던 pod 모두가 <none> 으로 표시된다. 다만 해당 문제는
새로운 터미널을 열고 다시 같은 명령어를 입력해보았을 때는 다시 정상적으로 표기가 된다.
갑자기 pod를 많이 생성했을 때 발생하는 자그만한 표기 오류인 것 같다.


```
kubectl get pod -o=custom-columns=NAME:.metadata.name,IP:.status.podIP
NAME                                IP
nginx-deployment-6fb79bc456-44t7d   192.168.3.222
nginx-deployment-6fb79bc456-4pwxv   192.168.1.70
nginx-deployment-6fb79bc456-5k6m6   192.168.2.12
nginx-deployment-6fb79bc456-6lc5f   192.168.2.51
nginx-deployment-6fb79bc456-77n9z   192.168.3.198
nginx-deployment-6fb79bc456-8xxcb   192.168.2.24
nginx-deployment-6fb79bc456-b4tl2   192.168.2.120
nginx-deployment-6fb79bc456-bf25x   192.168.1.149
nginx-deployment-6fb79bc456-cjrfl   192.168.3.39
nginx-deployment-6fb79bc456-cpzzt   192.168.2.91
nginx-deployment-6fb79bc456-cxc2h   192.168.3.107
nginx-deployment-6fb79bc456-d47jn   192.168.1.186
nginx-deployment-6fb79bc456-g22x4   192.168.2.33
nginx-deployment-6fb79bc456-k5r4p   192.168.1.116
nginx-deployment-6fb79bc456-krtqt   192.168.3.151
nginx-deployment-6fb79bc456-lhn96   192.168.2.101
nginx-deployment-6fb79bc456-lj7hs   192.168.1.221
nginx-deployment-6fb79bc456-n2lhq   192.168.3.241
nginx-deployment-6fb79bc456-ns949   192.168.2.104
nginx-deployment-6fb79bc456-q2ql2   192.168.1.40
nginx-deployment-6fb79bc456-q7lmr   192.168.3.19
nginx-deployment-6fb79bc456-r294p   192.168.3.226
nginx-deployment-6fb79bc456-rqqjk   192.168.2.231
nginx-deployment-6fb79bc456-rz49w   192.168.3.232
nginx-deployment-6fb79bc456-swt6m   192.168.3.131
nginx-deployment-6fb79bc456-wk7ht   192.168.2.77
nginx-deployment-6fb79bc456-wvs9l   192.168.1.147
nginx-deployment-6fb79bc456-x6wdt   192.168.1.46
nginx-deployment-6fb79bc456-xq98c   192.168.1.133
nginx-deployment-6fb79bc456-zjcmm   192.168.1.105
```

이번에는 pod를 70개 생성해보겠다.

```
# kubectl scale deployment nginx-deployment --replicas=70
# kubectl get pod -o=custom-columns=NAME:.metadata.name,IP:.status.podIP
NAME                                IP
nginx-deployment-6fb79bc456-2lrnf   192.168.1.155
nginx-deployment-6fb79bc456-44t7d   192.168.3.222
nginx-deployment-6fb79bc456-4pwxv   192.168.1.70
nginx-deployment-6fb79bc456-4xvd9   <none>
nginx-deployment-6fb79bc456-5k6m6   192.168.2.12
nginx-deployment-6fb79bc456-5rl7t   <none>
nginx-deployment-6fb79bc456-64d2k   192.168.2.117
nginx-deployment-6fb79bc456-6jffc   <none>
nginx-deployment-6fb79bc456-6k657   <none>
nginx-deployment-6fb79bc456-6lc5f   192.168.2.51
nginx-deployment-6fb79bc456-77n9z   192.168.3.198
nginx-deployment-6fb79bc456-85rrf   <none>
nginx-deployment-6fb79bc456-86vq2   <none>
nginx-deployment-6fb79bc456-8xxcb   192.168.2.24
nginx-deployment-6fb79bc456-9445c   <none>
nginx-deployment-6fb79bc456-b4tl2   192.168.2.120
nginx-deployment-6fb79bc456-b8jjh   <none>
nginx-deployment-6fb79bc456-b9kx9   <none>
nginx-deployment-6fb79bc456-bf25x   192.168.1.149
nginx-deployment-6fb79bc456-cjrfl   192.168.3.39
nginx-deployment-6fb79bc456-cpzzt   192.168.2.91
nginx-deployment-6fb79bc456-cxc2h   192.168.3.107
nginx-deployment-6fb79bc456-d47jn   192.168.1.186
nginx-deployment-6fb79bc456-dz8hs   <none>
nginx-deployment-6fb79bc456-fk4n5   192.168.1.172
nginx-deployment-6fb79bc456-g22x4   192.168.2.33
nginx-deployment-6fb79bc456-g257p   <none>
nginx-deployment-6fb79bc456-g7mx8   <none>
nginx-deployment-6fb79bc456-hr9pl   <none>
nginx-deployment-6fb79bc456-jqtxh   <none>
nginx-deployment-6fb79bc456-jwxt4   <none>
nginx-deployment-6fb79bc456-k5r4p   192.168.1.116
nginx-deployment-6fb79bc456-kdlgs   <none>
nginx-deployment-6fb79bc456-kgswv   192.168.2.54
nginx-deployment-6fb79bc456-krtqt   192.168.3.151
nginx-deployment-6fb79bc456-l2q9k   192.168.1.52
nginx-deployment-6fb79bc456-l69mg   <none>
nginx-deployment-6fb79bc456-lhn96   192.168.2.101
nginx-deployment-6fb79bc456-lj7hs   192.168.1.221
nginx-deployment-6fb79bc456-mj87r   192.168.3.149
nginx-deployment-6fb79bc456-mskkz   <none>
nginx-deployment-6fb79bc456-n2lhq   192.168.3.241
nginx-deployment-6fb79bc456-n6czb   192.168.2.126
nginx-deployment-6fb79bc456-nbnts   <none>
nginx-deployment-6fb79bc456-nhd2p   <none>
nginx-deployment-6fb79bc456-ns949   192.168.2.104
nginx-deployment-6fb79bc456-nzlzx   <none>
nginx-deployment-6fb79bc456-q2ql2   192.168.1.40
nginx-deployment-6fb79bc456-q5zjx   192.168.3.95
nginx-deployment-6fb79bc456-q7lmr   192.168.3.19
nginx-deployment-6fb79bc456-qxzg2   192.168.2.81
nginx-deployment-6fb79bc456-r294p   192.168.3.226
nginx-deployment-6fb79bc456-rqqjk   192.168.2.231
nginx-deployment-6fb79bc456-rt9dz   <none>
nginx-deployment-6fb79bc456-rz49w   192.168.3.232
nginx-deployment-6fb79bc456-swt6m   192.168.3.131
nginx-deployment-6fb79bc456-thvfh   <none>
nginx-deployment-6fb79bc456-vhdgs   <none>
nginx-deployment-6fb79bc456-vrdf8   192.168.1.174
nginx-deployment-6fb79bc456-vz492   192.168.1.67
nginx-deployment-6fb79bc456-wk7ht   192.168.2.77
nginx-deployment-6fb79bc456-wvs9l   192.168.1.147
nginx-deployment-6fb79bc456-x5p98   <none>
nginx-deployment-6fb79bc456-x6wdt   192.168.1.46
nginx-deployment-6fb79bc456-xg6x5   <none>
nginx-deployment-6fb79bc456-xq98c   192.168.1.133
nginx-deployment-6fb79bc456-z27gq   <none>
nginx-deployment-6fb79bc456-z6gwk   192.168.3.32
nginx-deployment-6fb79bc456-zbmp9   192.168.3.21
nginx-deployment-6fb79bc456-zjcmm   192.168.1.105
```

최대로 할당할 수 있는 IP를 넘어서게 pod가 생성되면 pod에 ip가 할당되지 않고 생성되는 것을 볼 수 있다.


```
kubectl get pods | grep Pending
nginx-deployment-6fb79bc456-4xvd9   0/1     Pending   0          93s
nginx-deployment-6fb79bc456-5rl7t   0/1     Pending   0          93s
nginx-deployment-6fb79bc456-6jffc   0/1     Pending   0          93s
nginx-deployment-6fb79bc456-6k657   0/1     Pending   0          93s
nginx-deployment-6fb79bc456-85rrf   0/1     Pending   0          93s
nginx-deployment-6fb79bc456-86vq2   0/1     Pending   0          93s
nginx-deployment-6fb79bc456-9445c   0/1     Pending   0          93s
nginx-deployment-6fb79bc456-b8jjh   0/1     Pending   0          93s
nginx-deployment-6fb79bc456-b9kx9   0/1     Pending   0          94s
nginx-deployment-6fb79bc456-dz8hs   0/1     Pending   0          93s
nginx-deployment-6fb79bc456-g257p   0/1     Pending   0          93s
nginx-deployment-6fb79bc456-g7mx8   0/1     Pending   0          94s
nginx-deployment-6fb79bc456-hr9pl   0/1     Pending   0          93s
nginx-deployment-6fb79bc456-jqtxh   0/1     Pending   0          93s
nginx-deployment-6fb79bc456-jwxt4   0/1     Pending   0          93s
nginx-deployment-6fb79bc456-kdlgs   0/1     Pending   0          93s
nginx-deployment-6fb79bc456-l69mg   0/1     Pending   0          93s
nginx-deployment-6fb79bc456-mskkz   0/1     Pending   0          93s
nginx-deployment-6fb79bc456-nbnts   0/1     Pending   0          93s
nginx-deployment-6fb79bc456-nhd2p   0/1     Pending   0          94s
nginx-deployment-6fb79bc456-nzlzx   0/1     Pending   0          93s
nginx-deployment-6fb79bc456-rt9dz   0/1     Pending   0          93s
nginx-deployment-6fb79bc456-thvfh   0/1     Pending   0          93s
nginx-deployment-6fb79bc456-vhdgs   0/1     Pending   0          93s
nginx-deployment-6fb79bc456-x5p98   0/1     Pending   0          93s
nginx-deployment-6fb79bc456-xg6x5   0/1     Pending   0          93s
nginx-deployment-6fb79bc456-z27gq   0/1     
```

### Service & AWS LoadBalancer Controller
 

![LB](https://github.com/jiwonYun9332/AWES-1/blob/f496d03123f67e66b8e1365212bb496541cccffb/Study/images/2_13_lb.png)
 
![LB](https://github.com/jiwonYun9332/AWES-1/blob/f496d03123f67e66b8e1365212bb496541cccffb/Study/images/2_14_lb.png)

- externalTrafficPolicy : ClusterIP ⇒ 2번 분산 및 SNAT으로 Client IP 확인 불가능 ← LoadBalancer 타입 (기본 모드) 동작
- externalTrafficPolicy : Local ⇒ 1번 분산 및 ClientIP 유지, 워커 노드의 iptables 사용함

인스턴스 유형

**통신 흐름**

![LB](https://github.com/jiwonYun9332/AWES-1/blob/f496d03123f67e66b8e1365212bb496541cccffb/Study/images/2_15_lb.png)
 
- 노드는 외부에 공개되지 않고 로드밸런서만 외부에 공개되어, 외부 클라이언트는 로드밸랜서에 접속을 할 뿐 내부 노드의 정보를 알 수 없다
- 로드밸런서가 부하분산하여 파드가 존재하는 노드들에게 전달한다, iptables 룰에서는 자신의 노드에 있는 파드만 연결한다 (externalTrafficPolicy: local)
- DNAT 2번 동작 : 첫번째(로드밸런서 접속 후 빠져 나갈때), 두번째(노드의 iptables 룰에서 파드IP 전달 시)
- 외부 클라이언트 IP 보존(유지) : AWS NLB 는 타켓이 인스턴스일 경우 클라이언트 IP를 유지, iptables 룰 경우도 externalTrafficPolicy 로 클라이언트 IP를 보존
 
![LB](https://github.com/jiwonYun9332/AWES-1/blob/6d761d661a23f5e39d09c1e967ade4673cc4f9b4/Study/images/2_16_lb.png)
 
 
IP 유형 ⇒ 반드시 AWS LoadBalancer 컨트롤러 파드 및 정책 설정이 필요
 
Proxy Protocol v2 비활성화 -> NLB에서 바로 파드로 인입, 단 ClientIP가 NLB로 SNAT 되어 Client IP 확인 불가능
Proxy Protocol v2 활성화 -> NLB에서 바로 파드로 인입 및 ClientIP 확인 가능(→ 단 PPv2 를 애플리케이션이 인지할 수 있게 설정 필요)
 
> 실습
 
 ```
 # OIDC 확인
aws eks describe-cluster --name $CLUSTER_NAME --query "cluster.identity.oidc.issuer" --output text
aws iam list-open-id-connect-providers | jq

# IAM Policy (AWSLoadBalancerControllerIAMPolicy) 생성
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.7/docs/install/iam_policy.json
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json

# 혹시 이미 IAM 정책이 있지만 예전 정책일 경우 아래 처럼 최신 업데이트 할 것
# aws iam update-policy ~~~

# 생성된 IAM Policy Arn 확인
aws iam list-policies --scope Local
aws iam get-policy --policy-arn arn:aws:iam::$ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy
aws iam get-policy --policy-arn arn:aws:iam::$ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy --query 'Policy.Arn'

# AWS Load Balancer Controller를 위한 ServiceAccount를 생성 >> 자동으로 매칭되는 IAM Role 을 CloudFormation 으로 생성됨!
# IAM 역할 생성. AWS Load Balancer Controller의 kube-system 네임스페이스에 aws-load-balancer-controller라는 Kubernetes 서비스 계정을 생성하고 IAM 역할의 이름으로 Kubernetes 서비스 계정에 주석을 답니다
eksctl create iamserviceaccount --cluster=$CLUSTER_NAME --namespace=kube-system --name=aws-load-balancer-controller \
--attach-policy-arn=arn:aws:iam::$ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy --override-existing-serviceaccounts --approve

## IRSA 정보 확인
eksctl get iamserviceaccount --cluster $CLUSTER_NAME

## 서비스 어카운트 확인
kubectl get serviceaccounts -n kube-system aws-load-balancer-controller -o yaml | yh

# Helm Chart 설치
helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=$CLUSTER_NAME \
  --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller

## 설치 확인
kubectl get crd
kubectl get deployment -n kube-system aws-load-balancer-controller
kubectl describe deploy -n kube-system aws-load-balancer-controller
kubectl describe deploy -n kube-system aws-load-balancer-controller | grep 'Service Account'
  Service Account:  aws-load-balancer-controller
```
 
**클러스터롤, 롤 확인**

![clusterCheck](https://github.com/jiwonYun9332/AWES-1/blob/40e8cae0d82aeed02f1aef1efe6ec7c0992d49ec/Study/images/2_17_clstuerCheck.png)
 

**생성된 IAM Role 신뢰 관계 확인**

![roleCheck](https://github.com/jiwonYun9332/AWES-1/blob/40e8cae0d82aeed02f1aef1efe6ec7c0992d49ec/Study/images/2_18_roleCheck.png)
 
***작업용 EC2 - 디플로이먼트 & 서비스 생성***

 ```
curl -s -O https://raw.githubusercontent.com/gasida/PKOS/main/2/echo-service-nlb.yaml
cat echo-service-nlb.yaml | yh
kubectl apply -f echo-service-nlb.yaml
```

**AWS ELB(NLB) 정보 확인**
 
![elbCheck](https://github.com/jiwonYun9332/AWES-1/blob/d15a289bcd470c12af2475298601380042a778f1/Study/images/2_19_elbCheck.png)

**웹 접속 주소 확인**
 
```
# kubectl get svc svc-nlb-ip-type -o jsonpath={.status.loadBalancer.ingress[0].hostname} | awk '{ print "Pod Web URL = http://"$1 }'
Pod Web URL = http://k8s-default-svcnlbip-a8f1be3313-6bf005ccda39eac4.elb.ap-northeast-2.amazonaws.com
```

**분산 접속 확인**

![elbcheck](https://github.com/jiwonYun9332/AWES-1/blob/d15a289bcd470c12af2475298601380042a778f1/Study/images/2_20_elbCheck.png)

 
 
 
 
 
 
