# 3Week

![logo](https://github.com/jiwonYun9332/AWES-1/blob/914b358b03c0fe27f42a00e3e5f176700dd61213/Study/images/21_logo.jpg)

### 실습환경 준비
Cloudformation Deploy

```
curl -O https://s3.ap-northeast-2.amazonaws.com/cloudformation.cloudneta.net/K8S/eks-oneclick2.yaml
```

```
# Default 네임스페이스 적용
kubectl ns default

# (옵션) contect 이름 변경
NICK=jiwon
kubectl ctx
```

### EFS 마운트 포인트 확인

**EFS(Elastic File System)**

![efs](https://github.com/jiwonYun9332/AWES-1/blob/97234e719ab25910ec8b3a323a2984218b0f109a/Study/images/23_efs.png)

![path](https://github.com/jiwonYun9332/AWES-1/blob/97234e719ab25910ec8b3a323a2984218b0f109a/Study/images/24_path.png)

**Mount**

```
# mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport fs-008165f9e0d190c32.efs.ap-northeast-2.amazonaws.com:/ /mnt/myefs
# df -hT --type nfs4
# mount | grep nfs4
# echo "efs file test" > /mnt/myefs/memo.txt
# cat /mnt/myefs/memo.txt
```

![mount](https://github.com/jiwonYun9332/AWES-1/blob/10305687614f18890747521bb512a1b30bcbe2ad/Study/images/25_mount.png)

**스토리지클래스 및 CSI 노드 확인**

![csinode](https://github.com/jiwonYun9332/AWES-1/blob/28f280b487336576f0bc49bde4a056a4f0b09b2a/Study/images/26_csinode.png)

**노드 정보 확인**

![nodeinformation](https://github.com/jiwonYun9332/AWES-1/blob/28f280b487336576f0bc49bde4a056a4f0b09b2a/Study/images/27_nodeinformation.png)

**노드 IP 확인 및 PrivateIP 변수 지정**

```
N1=$(kubectl get node --label-columns=topology.kubernetes.io/zone --selector=topology.kubernetes.io/zone=ap-northeast-2a -o jsonpath={.items[0].status.addresses[0].address})
N2=$(kubectl get node --label-columns=topology.kubernetes.io/zone --selector=topology.kubernetes.io/zone=ap-northeast-2b -o jsonpath={.items[0].status.addresses[0].address})
N3=$(kubectl get node --label-columns=topology.kubernetes.io/zone --selector=topology.kubernetes.io/zone=ap-northeast-2c -o jsonpath={.items[0].status.addresses[0].address})
echo "export N1=$N1" >> /etc/profile
echo "export N2=$N2" >> /etc/profile
echo "export N3=$N3" >> /etc/profile
echo $N1, $N2, $N3
```

**노드 보안그룹 ID 확인**

```
NGSGID=$(aws ec2 describe-security-groups --filters Name=group-name,Values=*ng1* --query "SecurityGroups[*].[GroupId]" --output text)
aws ec2 authorize-security-group-ingress --group-id $NGSGID --protocol '-1' --cidr 192.168.1.100/32
```

**워커노드 SSH 접속**

```
ssh ec2-user@$N1 hostname
ssh ec2-user@$N2 hostname
ssh ec2-user@$N3 hostname
```

**노드에 툴 설치**

```
ssh ec2-user@$N1 sudo yum install links tree jq tcpdump sysstat -y
ssh ec2-user@$N2 sudo yum install links tree jq tcpdump sysstat -y
ssh ec2-user@$N3 sudo yum install links tree jq tcpdump sysstat -y
```

### AWS LB/ExternalDNS, kube-ops-view 설치

**AWS LB Controller**

```
helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=$CLUSTER_NAME \
  --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller
```

**ExternalDNS**

```
MyDomain=senidin.com
MyDnzHostedZoneId=$(aws route53 list-hosted-zones-by-name --dns-name "${MyDomain}." --query "HostedZones[0].Id" --output text)
echo $MyDomain, $MyDnzHostedZoneId
curl -s -O https://raw.githubusercontent.com/gasida/PKOS/main/aews/externaldns.yaml
MyDomain=$MyDomain MyDnzHostedZoneId=$MyDnzHostedZoneId envsubst < externaldns.yaml | kubectl apply -f -
```

**kube-ops-view**

```
helm repo add geek-cookbook https://geek-cookbook.github.io/charts/
helm install kube-ops-view geek-cookbook/kube-ops-view --version 1.2.2 --set env.TZ="Asia/Seoul" --namespace kube-system
kubectl patch svc -n kube-system kube-ops-view -p '{"spec":{"type":"LoadBalancer"}}'
kubectl annotate service kube-ops-view -n kube-system "external-dns.alpha.kubernetes.io/hostname=kubeopsview.$MyDomain"
echo -e "Kube Ops View URL = http://kubeopsview.$MyDomain:8080/#scale=1.5"
```

### kubeopsview URL 접속
http://kubeopsview.senidin.com:8080/#scale=1.5

![kubeopsview](https://github.com/jiwonYun9332/AWES-1/blob/06f6c1b1749e4644d1a2de0ab4019ec94e0727e3/Study/images/28_kubeopsview.png)


```
# 이미지 정보 확인
kubectl get pods --all-namespaces -o jsonpath="{.items[*].spec.containers[*].image}" | tr -s '[[:space:]]' '\n' | sort | uniq -c

# eksctl 설치/업데이트 addon 확인
eksctl get addon --cluster $CLUSTER_NAME

# IRSA 확인
eksctl get iamserviceaccount --cluster $CLUSTER_NAME
```

### 1. 스토리지 이해

**Kubernetes의 지속성**
: 상태 저장 애플리케이션을 실행할 때, 영구 스토리지가 없으면 데이터는 포드 또는 컨테이너의 수명 주기에 연결된다. 포드가 충돌하거나 종료되면 데이터는 손실된다.

이러한 데이터 손실을 방지하고 Kubernetes에서 상태 저장 애플리케이션을 실행하려면 세 가지 간단한 스토리지 요구사항을 준수해야 한다.
1. 스토리지는 포드의 수명 주기에 의존해서는 안된다.
2. 스토리지는 Kubernetes 클러스터의 모든 포드 및 노드에서 사용할 수 있어야 한다.
3. 스토리지는 충돌이나 애플리케이션 오류에 관계없이 가용성이 높아야 한다.

Kubernetes Volume
: Kubernetes에는 여러가지 유형의 스토리지 옵션이 있으며, 모두 영구적인 것은 아니다.

Ephemeral Storage
: 컨테이너는 temporary filesystem(tmpfs)를 사용하여 파일을 읽고 쓸 수 있다. 그러나 임시 스토리지는 세 가지의 저장소 요구사항을 충족하지 않는다.
컨테이너가 충돌하는 경우 temporary filesystem은 손실되고, 컨테이너는 깨끗한 상태로 다시 시작된다. 또한 여러 컨테이너가 temporary filesystem을 공유할 수 없다.

임시 Kubernetes Volume은 임시 스토리지가 직면한 두 가지 문제를 모두 해결한다. 임시 Volume의 수명은 Pod와 연결된다.
이를 통해 컨테이너를 안전하게 재시작하고 Pod 내의 컨테이너간의 데이터를 공유할 수 있다. 그러나 Pod가 삭제되는 즉시 Volume도 삭제가 되므로,
이는 여전히 세 가지 요구사항을 충족하지 못한다.

스토리지와 포드 분리, Persistent Volumes
: Kubernetes는 Persistent Volumes도 지원한다. Persistent Volumes을 사용하면 애플리케이션, 컨테이너, 포드, 노드 또는 클러스터 자체의 수명 주기와 관계없이 데이터가 지속된다.
Persistent Volumes는 앞에서 설명한 세 가지 요구사항을 충족한다.

Persistent Volumes(PV) 객체는 애플리케이션 데이터를 유지하는 데 사용되는 스토리지 볼륨을 나타낸다. PV는 Kubernetes Pods 수명 주기와 별개로 자체의 수명 주기를 가진다.

PV는 기본적으로 두 가지로 구성된다.
- Persistent Volume 이라고 불리는 백엔드 기술
- Kubernetes에 볼륨을 마운트하는 방법을 알려주는 접근 모드

백엔드 기술(Backend technology)

PV는 추상적인 구성 요소이며, 실제 물리적 스토리지는 어딘가에서 가져와야 한다. 다음은 몇 가지 예이다.
csi: Container Storage Interface(CSI) -> Amazon EBS, Amazon FSx 등
iscsi: iSCSI(SCSI over IP) 스토리지
local: 노드에 마운트된 로컬 저장 장치
nfs: 네트워크 파일 시스템(NFS) 스토리지

Kubernetes는 다목적이며 다양한 유형의 PV를 지원한다. Kubernetes는 연결되는 스토리지 내부에는 신경을 쓰지 않는다.
PV 구성 요소를 실제 저장소에 대한 인터페이스로 제공할 뿐이다.

PV는 다음과 같은 세 가지의 주요 이점이 있다.
- PV는 Pod의 수명 주기에 구속되지 않는다.
: PV 객체에 연결된 포드를 삭제해도 해당 PV는 유지된다.

앞의 설명은 포드 충돌 시에도 유효하다.

접근 모드(Access mode)
: 접근 모드는 PV가 생성될 때 설정되며 Kubernetes 볼륨을 마운트하는 방법을 알려준다. Persistent Volumes은 세 가지 접근 모드 지원
- ReadWriteOnce: 볼륨은 동시에 하나의 노드에서만 읽기/쓰기를 허용한다.
- ReadOnlyMany: 볼륨은 동시에 여러 노드에서 읽기 전용 모드를 허용한다
- ReadWriteMany: 볼륨은 동시에 여러 노드에서 읽기/쓰기를 허용한다.

모든 Persistent Volume 유형이 모든 액세스 모드를 지원하는 것이 아니다.

**영구 볼륨 클레임(Persistent Volume Claim)**

Persistent Volume(PV)은 실제 스토리지 볼륨을 나타낸다. Kubernetes는 PV를 포드에 연결하는데 필요한 추가 추상화 계층인
Persistent Volume Claim(PVC)을 가지고 있다.

PV는 실제 스토리지 볼륨을 나타내며, PVC는 Pod가 실제 스토리지를 얻기 위해 수행하는 스토리지 요청을 나타낸다.

PV와 PVC의 구분은 Kubernetes 환경에서 두 가지 유형의 사용자가 있다는 개념과 관련이 있다.

- Kubernetes 관리자: 이 사용자는 클러스터를 유지 관리하고 운영하며 영구 스토리지와 같은 계산 리소스를 추가한다.
- Kubernetes 애플리케이션 개발자: 이 사용자는 애플리케이션을 개발하고 배포한다.
- 
간단히 말해서, 개발자는 관리자가 제공하는 계산 리소스를 사용한다. Kubernetes는 PV객체는 클러스터 관리자 범위에 속하고, 반면에 PVC 객체는
애플리케이션 개발자 범위에 속해야 한다는 개념으로 구축 되었습니다.

기본적으로 PV 객체를 직접 Pod에 마운트할 수 없다. 이것은 명시적으로 요청되어져야 한다. 그리고 그 요청은 PVC 객체를 생성하고 Pod에 연결함으로써 달성된다.
이것이 이 추가 추상화 계층이 존재하는 유일한 이유이다. PVC와 PV는 1:1 관계가 있다. PV는 하나의 PVC에만 연결될 수 있음

**CSI(Container Storage Interface) 드라이버**

CSI는 Kubernetes에서 다양한 스토리지 솔루션을 쉽게 사용할 수 있도록 설계된 추상화이다. 다양한 스토리지 공급업체는 CSI 표준을 구현하는 자체 드라이버를 개발하여
스토리지 솔루션이 Kubernetes와 함께 작동하도록 할 수 있다.

**정적 프로비저닝(Static provisioning)**

"영구 볼륨 클레임" 섹션에서 설명한 바와 같이, 먼저 관리자가 하나 이상의 PV를 생성하고 애플리케이션 개발자는 PVC를 생성한다.
이를 정적 프로비저닝이라고 한다. Kubernetes에서 PV 및 PVC를 수동으로 만들어야 하므로 정적이다.
대규모 환경에서는 관리하기가 점점 더 어려워질 수 있으며, 특히 수백 개의 PV와 PVC를 관리하는 경우에도 더욱 그러하다.

Amazon EFS 파일 시스템을 생성하여 PV 객체로 마운트하기 위해, 정적 프로비저닝을 사용하려고 한다고 가정해 본다.

**Kubernetes 관리자의 작업**

- Amazon EFS 파일 시스템 볼륨을 생성한다.
- 파일 시스템 ID를 복사하여 PV YAML 정의 파일에 붙여 넣는다.
- YAML 파일을 사용하여 PV를 생성한다.

**Kubernetes 애플리케이션 개발자의 작업**

- 이 PV를 요청하기 위해 PVC를 만든다.
- Pod YAML 정의 파일의 Pod 객체에 PVC를 마운트한다.

**동적 프로비저닝(Dynamic Provisioning)

동적 프로비저닝을 사용하면 PV 객체를 생성할 필요가 없다. 대신에 PVC를 생성할 때 내부적으로 자동으로 생성된다. Kubernetes는 Storage Class라는 다른 객체를 사용하여 이를 수행한다.

Storage Class는 컨테이너 애플리케이션에 사용되는 백엔드 영구 스토리지(Amazon EFS 파일 스토리지, Amazon EBS 블록 스토리지 등)의 클래스를 정의하는 추상화이다.

Storage Class 기본적으로 두 가지를 포함한다.
- 이름: 스토리지 클래스 객체를 고유하게 식별하는 이름이다.
- 제공자(Provisioner): 연결되는 스토리지 기술을 정의한다. 예를 들어 Provisioner는 Amazon EFS용
efs.csi.aws.com 또는 Amazon EBS용 ebs.csi.aws.com이 될 수 있다.

Storage Class 객체는 Kubernetes가 매우 다양한 스토리지 기술을 처리할 수 있는 이유이다. Pod 관점에서 볼 때, EFS 볼륨, EBS 볼륨, NFS 드라이브 또는 기타 어떤 것이든, 그 Pod는
PVC 객체만 보게 된다. 실제 스토리지 기술을 다루는 모든 내부적인 논리는 Storage Class 객체가 사용하는 프로비저너에 의해 구분된다.

### 실습 1.1 기본 컨테이너 환경의 임시 파일시스템 사용

**파드 배포**
**date 명령어로 현재 시간을 10초 간격으로 /home/pod-out.txt 파일에 저장**

```
curl -s -O https://raw.githubusercontent.com/gasida/PKOS/main/3/date-busybox-pod.yaml
cat date-busybox-pod.yaml | yh
kubectl apply -f date-busybox-pod.yaml
```

**파일 확인**

```
kubectl get pod
kubectl exec busybox -- tail -f /home/pod-out.txt
```

**파드 삭제 후 다시 생성 후 파일 정보 확인**

![pod](https://github.com/jiwonYun9332/AWES-1/blob/a7c90f088806f826927d2679a829f4a8e7be3396/Study/images/29_test.png)

**전에 있던 데이터느 삭제됨**


### 실습 1.2 호스트 Path 를 사용하는 PV/PVC: local-path-provisioner 스트리지 클래스 배포

```
# 배포
curl -s -O https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
kubectl apply -f local-path-storage.yaml

# 확인
kubectl get-all -n local-path-storage
kubectl get pod -n local-path-storage -owide
kubectl describe cm -n local-path-storage local-path-config
kubectl get sc
kubectl get sc local-path
```

![deploy](https://github.com/jiwonYun9332/AWES-1/blob/1b6b5b60ca305118ca2ee06e698b922c6b4baf16/Study/images/30_local-path.png)

**PV/PVC를 사용하는 파드 생성**

```
# PVC 생성
curl -s -O https://raw.githubusercontent.com/gasida/PKOS/main/3/localpath1.yaml
cat localpath1.yaml | yh
kubectl apply -f localpath1.yaml

# PVC 확인
kubectl get pvc
kubectl describe pvc

# 파드 생성
curl -s -O https://raw.githubusercontent.com/gasida/PKOS/main/3/localpath2.yaml
cat localpath2.yaml | yh
kubectl apply -f localpath2.yaml

# 파드 확인
kubectl get pod,pv,pvc
kubectl describe pv    # Node Affinity 확인
kubectl exec -it app -- tail -f /data/out.txt

# 워커노드 out.txt 파일 존재 확인
ssh ec2-user@$N1 tree /opt/local-path-provisioner
/opt/local-path-provisioner
└── pvc-57ef0ebf-bc71-46e4-a097-552f88b026ef_default_localpath-claim
    └── out.txt
```

**파드 삭제 후 파드 재생성해서 데이터 유지 되는지 확인**

```
# 파드 삭제 후 PV/PVC 확인
kubectl delete pod app
kubectl get pod,pv,pvc
ssh ec2-user@$N3 tree /opt/local-path-provisioner

# 파드 다시 실행
kubectl apply -f localpath2.yaml
 
# 확인
kubectl exec -it app -- head /data/out.txt
kubectl exec -it app -- tail -f /data/out.txt
```

![check](https://github.com/jiwonYun9332/AWES-1/blob/f165daad5e7c54847858e2a81e842e28f32e8bd3/Study/images/31_check.png)

**다음 실습을 위해서 파드와 PVC 삭제**

```
# 파드와 PVC 삭제 
kubectl delete pod app
kubectl get pv,pvc
kubectl delete pvc localpath-claim

# 확인
kubectl get pv
ssh ec2-user@$N3 tree /opt/local-path-provisioner
```

### AWS EBS Controller

![ebs_controller](https://github.com/jiwonYun9332/AWES-1/blob/cb5b035bcf053d6488e710f999d767a36b76b7df/Study/images/32_ebs%20controller.png)

![ebs_controller2](https://github.com/jiwonYun9332/AWES-1/blob/cb5b035bcf053d6488e710f999d767a36b76b7df/Study/images/33_deployeks.png)

**설치**

```
# 아래는 aws-ebs-csi-driver 전체 버전 정보와 기본 설치 버전(True) 정보 확인
aws eks describe-addon-versions \
    --addon-name aws-ebs-csi-driver \
    --kubernetes-version 1.24 \
    --query "addons[].addonVersions[].[addonVersion, compatibilities[].defaultVersion]" \
    --output text

# ISRA 설정 : AWS관리형 정책 AmazonEBSCSIDriverPolicy 사용
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster ${CLUSTER_NAME} \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve \
  --role-only \
  --role-name AmazonEKS_EBS_CSI_DriverRole

# ISRA 확인
kubectl get sa -n kube-system ebs-csi-controller-sa -o yaml | head -5
eksctl get iamserviceaccount --cluster myeks

# Amazon EBS CSI driver addon 추가
eksctl create addon --name aws-ebs-csi-driver --cluster ${CLUSTER_NAME} --service-account-role-arn arn:aws:iam::${ACCOUNT_ID}:role/AmazonEKS_EBS_CSI_DriverRole --force

# 확인
eksctl get addon --cluster ${CLUSTER_NAME}
kubectl get deploy,ds -l=app.kubernetes.io/name=aws-ebs-csi-driver -n kube-system
kubectl get pod -n kube-system -l 'app in (ebs-csi-controller,ebs-csi-node)'
kubectl get pod -n kube-system -l app.kubernetes.io/component=csi-driver

# ebs-csi-controller 파드에 6개 컨테이너 확인
kubectl get pod -n kube-system -l app=ebs-csi-controller -o jsonpath='{.items[0].spec.containers[*].name}' ; echo
ebs-plugin csi-provisioner csi-attacher csi-snapshotter csi-resizer liveness-probe

# csinodes 확인
kubectl get csinodes

# gp3 스토리지 클래스 생성
kubectl get sc
cat <<EOT > gp3-sc.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: gp3
allowVolumeExpansion: true
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
  allowAutoIOPSPerGBIncrease: 'true'
  encrypted: 'true'
  #fsType: ext4
EOT
kubectl apply -f gp3-sc.yaml
kubectl get sc
kubectl describe sc gp3 | grep Parameters
```

![install](https://github.com/jiwonYun9332/AWES-1/blob/7fa02f1051834b6017634a50c99f005a720ed124/Study/images/34_install.png)

### PVC/PV 파드 테스트

```
# 워커노드의 EBS 볼륨 확인 : tag(키/값) 필터링 - 링크
aws ec2 describe-volumes --filters Name=tag:Name,Values=$CLUSTER_NAME-ng1-Node --output table
aws ec2 describe-volumes --filters Name=tag:Name,Values=$CLUSTER_NAME-ng1-Node --query "Volumes[*].Attachments" | jq
aws ec2 describe-volumes --filters Name=tag:Name,Values=$CLUSTER_NAME-ng1-Node --query "Volumes[*].{ID:VolumeId,Tag:Tags}" | jq
aws ec2 describe-volumes --filters Name=tag:Name,Values=$CLUSTER_NAME-ng1-Node --query "Volumes[].[VolumeId, VolumeType, Attachments[].[InstanceId, State][]][]" | jq
aws ec2 describe-volumes --filters Name=tag:Name,Values=$CLUSTER_NAME-ng1-Node --query "Volumes[].{VolumeId: VolumeId, VolumeType: VolumeType, InstanceId: Attachments[0].InstanceId, State: Attachments[0].State}" | jq

# 워커노드에서 파드에 추가한 EBS 볼륨 확인
aws ec2 describe-volumes --filters Name=tag:ebs.csi.aws.com/cluster,Values=true --output table
aws ec2 describe-volumes --filters Name=tag:ebs.csi.aws.com/cluster,Values=true --query "Volumes[*].{ID:VolumeId,Tag:Tags}" | jq
aws ec2 describe-volumes --filters Name=tag:ebs.csi.aws.com/cluster,Values=true --query "Volumes[].{VolumeId: VolumeId, VolumeType: VolumeType, InstanceId: Attachments[0].InstanceId, State: Attachments[0].State}" | jq

# 워커노드에서 파드에 추가한 EBS 볼륨 모니터링

# PVC 생성
cat <<EOT > awsebs-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 4Gi
  storageClassName: gp3
EOT

kubectl apply -f awsebs-pvc.yaml
kubectl get pvc,pv

# 파드 생성
cat <<EOT > awsebs-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  terminationGracePeriodSeconds: 3
  containers:
  - name: app
    image: centos
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo \$(date -u) >> /data/out.txt; sleep 5; done"]
    volumeMounts:
    - name: persistent-storage
      mountPath: /data
  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: ebs-claim
EOT
kubectl apply -f awsebs-pod.yaml

# PVC, 파드 확인
kubectl get pvc,pv,pod
kubectl get VolumeAttachment

# 추가된 EBS 볼륨 상세 정보 확인 
aws ec2 describe-volumes --volume-ids $(kubectl get pv -o jsonpath="{.items[0].spec.csi.volumeHandle}") | jq

# PV 상세 확인 : nodeAffinity 
kubectl get pv -o yaml | yh

kubectl get node --label-columns=topology.ebs.csi.aws.com/zone,topology.kubernetes.io/zone
kubectl describe node | more

# 파일 내용 추가 저장 확인
kubectl exec app -- tail -f /data/out.txt

# 아래 명령어는 확인까지 다소 시간이 소요됨
kubectl df-pv

## 파드 내에서 볼륨 정보 확인
kubectl exec -it app -- sh -c 'df -hT --type=overlay'
kubectl exec -it app -- sh -c 'df -hT --type=ext4
```
![volume](https://github.com/jiwonYun9332/AWES-1/blob/d3ce74aea701825cbc1bbf96a2bdcc27b9f7d499/Study/images/35_volume.png)


**볼륨 증가**

```
# 현재 pv 의 이름을 기준하여 4G > 10G 로 증가 : .spec.resources.requests.storage의 4Gi 를 10Gi로 변경
kubectl get pvc ebs-claim -o jsonpath={.spec.resources.requests.storage} ; echo
kubectl get pvc ebs-claim -o jsonpath={.status.capacity.storage} ; echo
kubectl patch pvc ebs-claim -p '{"spec":{"resources":{"requests":{"storage":"10Gi"}}}}'

# 확인 : 볼륨 용량 수정 반영이 되어야 되니, 수치 반영이 조금 느릴수 있다
kubectl exec -it app -- sh -c 'df -hT --type=ext4'
kubectl df-pv
aws ec2 describe-volumes --volume-ids $(kubectl get pv -o jsonpath="{.items[0].spec.csi.volumeHandle}") | jq

# 삭제
kubectl delete pod app & kubectl delete pvc ebs-claim
```

![volumeadd](https://github.com/jiwonYun9332/AWES-1/blob/7f46f63507bf124b35b461cffc8ef1f50fea4e5f/Study/images/36_volumeadd.png)

### 3. AWS Volume SnapShots Controller

**Volume snapshots 컨트롤러 설치**

```
# (참고) EBS CSI Driver에 snapshots 기능 포함 될 것으로 보임
kubectl describe pod -n kube-system -l app=ebs-csi-controller

# Install Snapshot CRDs
curl -s -O https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshots.yaml
curl -s -O https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshotclasses.yaml
curl -s -O https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshotcontents.yaml
kubectl apply -f snapshot.storage.k8s.io_volumesnapshots.yaml,snapshot.storage.k8s.io_volumesnapshotclasses.yaml,snapshot.storage.k8s.io_volumesnapshotcontents.yaml
kubectl get crd | grep snapshot
kubectl api-resources  | grep snapshot

# Install Common Snapshot Controller
curl -s -O https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/deploy/kubernetes/snapshot-controller/rbac-snapshot-controller.yaml
curl -s -O https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/deploy/kubernetes/snapshot-controller/setup-snapshot-controller.yaml
kubectl apply -f rbac-snapshot-controller.yaml,setup-snapshot-controller.yaml
kubectl get deploy -n kube-system snapshot-controller
kubectl get pod -n kube-system -l app=snapshot-controller

# Install Snapshotclass
curl -s -O https://raw.githubusercontent.com/kubernetes-sigs/aws-ebs-csi-driver/master/examples/kubernetes/snapshot/manifests/classes/snapshotclass.yaml
kubectl apply -f snapshotclass.yaml
kubectl get vsclass # 혹은 volumesnapshotclasses
```

**사용**

- 테스트 PVC/파드 생성

```
# PVC 생성
kubectl apply -f awsebs-pvc.yaml

# 파드 생성
kubectl apply -f awsebs-pod.yaml

# 파일 내용 추가 저장 확인
kubectl exec app -- tail -f /data/out.txt

# VolumeSnapshot 생성 : Create a VolumeSnapshot referencing the PersistentVolumeClaim name >> EBS 스냅샷 확인
curl -s -O https://raw.githubusercontent.com/gasida/PKOS/main/3/ebs-volume-snapshot.yaml
cat ebs-volume-snapshot.yaml | yh
kubectl apply -f ebs-volume-snapshot.yaml

# VolumeSnapshot 확인
kubectl get volumesnapshot
kubectl get volumesnapshot ebs-volume-snapshot -o jsonpath={.status.boundVolumeSnapshotContentName} ; echo
kubectl describe volumesnapshot.snapshot.storage.k8s.io ebs-volume-snapshot
kubectl get volumesnapshotcontents
```

![check](https://github.com/jiwonYun9332/AWES-1/blob/11d3c424b2947fa9bc9e069a894171ccd23ece8f/Study/images/37_check.png)

```
# VolumeSnapshot ID 확인 
kubectl get volumesnapshotcontents -o jsonpath='{.items[*].status.snapshotHandle}' ; echo

# AWS EBS 스냅샷 확인
aws ec2 describe-snapshots --owner-ids self | jq
aws ec2 describe-snapshots --owner-ids self --query 'Snapshots[]' --output table

# app & pvc 제거 : 강제로 장애 재현
kubectl delete pod app && kubectl delete pvc ebs-claim
```

![error](https://github.com/jiwonYun9332/AWES-1/blob/5ae8540578cf407da0424199a740e8fb49730ddb/Study/images/38_error.png)

**스냅샷으로 복원**

```
# 스냅샷에서 PVC 로 복원
kubectl get pvc,pv
cat <<EOT > ebs-snapshot-restored-claim.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-snapshot-restored-claim
spec:
  storageClassName: gp3
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 4Gi
  dataSource:
    name: ebs-volume-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
EOT
cat ebs-snapshot-restored-claim.yaml | yh
kubectl apply -f ebs-snapshot-restored-claim.yaml

# 확인
kubectl get pvc,pv

# 파드 생성
curl -s -O https://raw.githubusercontent.com/gasida/PKOS/main/3/ebs-snapshot-restored-pod.yaml
cat ebs-snapshot-restored-pod.yaml | yh
kubectl apply -f ebs-snapshot-restored-pod.yaml

# 파일 내용 저장 확인 : 파드 삭제 전까지의 저장 기록이 남아 있다. 이후 파드 재생성 후 기록도 잘 저장되고 있다
kubectl exec app -- cat /data/out.txt
```

![ok](https://github.com/jiwonYun9332/AWES-1/blob/486278c6da14aa436e4023120998884336a6ba1b/Study/images/39_ok.png)

```
# 삭제
kubectl delete pod app && kubectl delete pvc ebs-snapshot-restored-claim && kubectl delete volumesnapshots ebs-volume-snapshot
```

### AWS EFS Controller 

**구성 아키텍처**

![architecture](https://github.com/jiwonYun9332/AWES-1/blob/485da568e3de3abb4ab85e103bde205ea46f59f4/Study/images/40_architecture.png)

![architecture](https://github.com/jiwonYun9332/AWES-1/blob/485da568e3de3abb4ab85e103bde205ea46f59f4/Study/images/41_architecture.png)


```
# EFS 정보 확인 
aws efs describe-file-systems --query "FileSystems[*].FileSystemId" --output text

# IAM 정책 생성
curl -s -O https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/docs/**iam-policy-example.json**
aws iam create-policy --policy-name **AmazonEKS_EFS_CSI_Driver_Policy** --policy-document file://iam-policy-example.json

# ISRA 설정 : 고객관리형 정책 AmazonEKS_EFS_CSI_Driver_Policy 사용
eksctl create **iamserviceaccount** \
  --name **efs-csi-controller-sa** \
  --namespace kube-system \
  --cluster ${CLUSTER_NAME} \
  --attach-policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/AmazonEKS_EFS_CSI_Driver_Policy \
  --approve

****# ISRA 확인
kubectl get sa -n kube-system efs-csi-controller-sa -o yaml | head -5
****eksctl get iamserviceaccount --cluster myeks

# EFS Controller 설치
helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/
helm repo update
helm upgrade -i aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver \
    --namespace kube-system \
    --set image.repository=602401143452.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/eks/aws-efs-csi-driver \
    --set controller.serviceAccount.create=false \
    --set controller.serviceAccount.name=efs-csi-controller-sa

# 확인
helm list -n kube-system
kubectl get pod -n kube-system -l "app.kubernetes.io/name=aws-efs-csi-driver,app.kubernetes.io/instance=aws-efs-csi-driver"

```

![check](https://github.com/jiwonYun9332/AWES-1/blob/6fffc7c0d02f0631c088db8b4561f0b1d66a634c/Study/images/42_check.png)


**EFS 파일시스템을 다수의 파드가 사용하게 설정**

```
# 모니터링
watch 'kubectl get sc efs-sc; echo; kubectl get pv,pvc,pod'

# 실습 코드 clone
git clone https://github.com/kubernetes-sigs/aws-efs-csi-driver.git /root/efs-csi
cd /root/efs-csi/examples/kubernetes/multiple_pods/specs && tree

# EFS 스토리지클래스 생성 및 확인
cat storageclass.yaml | yh
kubectl apply -f storageclass.yaml
kubectl get sc efs-sc

# PV 생성 및 확인 : volumeHandle을 자신의 EFS 파일시스템ID로 변경
EfsFsId=$(aws efs describe-file-systems --query "FileSystems[*].FileSystemId" --output text)
sed -i "s/fs-4af69aab/$EfsFsId/g" pv.yaml

cat pv.yaml | yh
kubectl apply -f pv.yaml
kubectl get pv; kubectl describe pv

# PVC 생성 및 확인
cat claim.yaml | yh
kubectl apply -f claim.yaml
kubectl get pvc

# 파드 생성 및 연동 : 파드 내에 /data 데이터는 EFS를 사용
cat pod1.yaml pod2.yaml | yh
kubectl apply -f pod1.yaml,pod2.yaml
kubectl df - pv

# 파드 정보 확인 : PV에 5Gi 와 파드 내에서 확인한 NFS4 볼륨 크리 8.0E의 차이는 무엇? 파드에 6Gi 이상 저장 가능한가?
kubectl get pods
kubectl exec -ti app1 -- sh -c "df -hT -t nfs4"
kubectl exec -ti app2 -- sh -c "df -hT -t nfs4"
Filesystem           Type            Size      Used Available Use% Mounted on
127.0.0.1:/          nfs4            8.0E         0      8.0E   0% /data

# 공유 저장소 저장 동작 확인
tree /mnt/myefs              # 작업용EC2에서 확인
tail -f /mnt/myefs/out1.txt  # 작업용EC2에서 확인
kubectl exec -ti app1 -- tail -f /data/out1.txt
kubectl exec -ti app2 -- tail -f /data/out2.txt
```

![check](https://github.com/jiwonYun9332/AWES-1/blob/d020531c8948412bb8c7e970b3d9f2dbaaf70cbc/Study/images/43_check.png)

**쿠버네티스 리소스 삭제**

```
kubectl delete pod app1 app2 
kubectl delete pvc efs-claim && kubectl delete pv efs-pv && kubectl delete sc efs-sc
```

### EKS Persistent Volumes for Instance Store & Add NodeGroup

```
aws ec2 describe-instance-types \
 --filters "Name=instance-type,Values=c5*" "Name=instance-storage-supported,Values=true" \
 --query "InstanceTypes[].[InstanceType, InstanceStorageInfo.TotalSizeInGB]" \
 --output table
```

![images](https://github.com/jiwonYun9332/AWES-1/blob/995999a40af64c064a1a3f0ccee76463ad6ea7d4/Study/images/44_images.png)

```
# 신규 노드 그룹 생성
eksctl create nodegroup -c $CLUSTER_NAME -r $AWS_DEFAULT_REGION --subnet-ids "$PubSubnet1","$PubSubnet2","$PubSubnet3" --ssh-access \
  -n ng2 -t c5d.large -N 1 -m 1 -M 1 --node-volume-size=30 --node-labels disk=nvme --max-pods-per-node 100 --dry-run > myng2.yaml
  
cat <<EOT > nvme.yaml
  preBootstrapCommands:
    - |
      # Install Tools
      yum install nvme-cli links tree jq tcpdump sysstat -y

      # Filesystem & Mount
      mkfs -t xfs /dev/nvme1n1
      mkdir /data
      mount /dev/nvme1n1 /data

      # Get disk UUID
      uuid=\$(blkid -o value -s UUID mount /dev/nvme1n1 /data) 

      # Mount the disk during a reboot
      echo /dev/nvme1n1 /data xfs defaults,noatime 0 2 >> /etc/fstab
EOT

sed -i -n -e '/volumeType/r nvme.yaml' -e '1,$p' myng2.yaml
eksctl create nodegroup -f myng2.yaml

# 노드 보안그룹 ID 확인
NG2SGID=$(aws ec2 describe-security-groups --filters Name=group-name,Values=*ng2* --query "SecurityGroups[*].[GroupId]" --output text)
aws ec2 authorize-security-group-ingress --group-id $NG2SGID --protocol '-1' --cidr 192.168.1.100/32

# 워커 노드 SSH 접속
N4=192.168.1.100
ssh ec2-user@$N4 hostname

# 확인
ssh ec2-user@$N4 sudo nvme list
ssh ec2-user@$N4 sudo lsblk -e 7 -d
ssh ec2-user@$N4 sudo df -hT -t xfs
ssh ec2-user@$N4 sudo tree /data
ssh ec2-user@$N4 sudo cat /etc/fstab

# (옵션) max-pod 확인
kubectl describe node -l disk=nvme | grep Allocatable: -A7

# (옵션) kubelet 데몬 파라미터 확인 : --max-pods=29 --max-pods=100
ssh ec2-user@$N4 sudo ps -ef | grep kubelet
```

![images](https://github.com/jiwonYun9332/AWES-1/blob/a5b8085e901b10d8f44ca0c6f81182008f5830f7/Study/images/45_images.png)

**삭제**

```
# 
kubectl delete -f local-path-storage.yaml

# 노드그룹 삭제
eksctl delete nodegroup -c $CLUSTER_NAME -n ng2
```

**실습 완료 후 삭제**

```
# IRSA 스택 삭제
helm uninstall -n kube-system kube-ops-view
aws cloudformation delete-stack --stack-name eksctl-$CLUSTER_NAME-addon-iamserviceaccount-kube-system-efs-csi-controller-sa
aws cloudformation delete-stack --stack-name eksctl-$CLUSTER_NAME-addon-iamserviceaccount-kube-system-ebs-csi-controller-sa
aws cloudformation delete-stack --stack-name eksctl-$CLUSTER_NAME-addon-iamserviceaccount-kube-system-aws-load-balancer-controller

# 삭제
eksctl delete cluster --name $CLUSTER_NAME && aws cloudformation delete-stack --stack-name $CLUSTER_NAME
```










