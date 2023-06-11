
# 7Week - 7주차 실습

![image](https://github.com/jiwonYun9332/AWES-1/blob/ba891951a2e3d48a6534bc1ce66dd04c3a0a98ee/Study/images/118_image.jpg)

### 1. AWS Controller for Kubernetes (ACK)

**AWS Controller for Kubernetes이란?**

```
The idea behind AWS Controllers for Kubernetes (ACK) is to enable Kubernetes users to describe the desired state of AWS resources using the Kubernetes API and configuration language.
```

![image](https://github.com/jiwonYun9332/AWES-1/blob/2f7701c25a215f9476dd2ee8714fea259f59b801/Study/images/119_image.jpg)

AWS에는 익숙하지 않은 엔지니어들을 위해 Kubernetes 안에서 AWS 리소스들을 관리할 수 있는 Controller이다.

aws 서비스 리소스를 k8s에서 직접 정의하고 사용 할 수 있다.

1. kube-apiserver -> ack-s3-controller에 요청 전달
사용자는 kubectl를 사용하여 aws s3 버킷을 생성 할 수 있다.

2. ack-s3-controller가 IRSA 권한이 있는 경우, AWS S3 API를 통해 버킷 생성
쿠버네티스 api는 ack-s3-controller에 요청을 전달하고, ack-s3-controller(IRSA)이 aws s3 api를 통해 버킷을 생성하게 된다.

`ACK 가 지원하는 AWS 서비스` : (‘23.5.29일) 현재 **17개** 서비스는 **GA** General Availability (정식 출시) 되었고, **10개**는 **Preview** (평가판) 상태입니다 - [링크](https://aws-controllers-k8s.github.io/community/docs/community/services/)

- ****Maintenance Phases**** 관리 단계 : **PREVIEW** (테스트 단계, 상용 서비스 비권장) , **GENERAL AVAILABILITY** (상용 서비스 권장), DEPRECATED, NOT SUPPORTED
- **GA** 서비스 : ApiGatewayV2, CloudTrail, **DynamoDB**, **EC2**, ECR, EKS, IAM, KMS, **Lambda**, MemoryDB, **RDS**, **S3**, SageMaker…
- **Preview** 서비스 : ACM, ElastiCache, EventBridge, MQ, Route 53, SNS, SQS…

Permissions : k8s api 와 aws api 의 2개의 RBAC 시스템 확인, 각 서비스 컨트롤러 파드는 AWS 서비스 권한 필요 ← IRSA role for ACK Service Controller

**ACK S3 Controller 설치 with Helm**

```
# 서비스명 변수 지정
export SERVICE=s3

# helm 차트 다운로드
#aws ecr-public get-login-password --region us-east-1 | helm registry login --username AWS --password-stdin public.ecr.aws
export RELEASE_VERSION=$(curl -sL https://api.github.com/repos/aws-controllers-k8s/$SERVICE-controller/releases/latest | grep '"tag_name":' | cut -d'"' -f4 | cut -c 2-)
helm pull oci://public.ecr.aws/aws-controllers-k8s/$SERVICE-chart --version=$RELEASE_VERSION
tar xzvf $SERVICE-chart-$RELEASE_VERSION.tgz

# helm chart 확인
tree ~/$SERVICE-chart

# ACK S3 Controller 설치
export ACK_SYSTEM_NAMESPACE=ack-system
export AWS_REGION=ap-northeast-2
helm install --create-namespace -n $ACK_SYSTEM_NAMESPACE ack-$SERVICE-controller --set aws.region="$AWS_REGION" ~/$SERVICE-chart

# 설치 확인
helm list --namespace $ACK_SYSTEM_NAMESPACE
NAME                    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
ack-s3-controller       ack-system      1               2023-06-11 17:24:32.768580184 +0900 KST deployed        s3-chart-1.0.4  1.0.4
kubectl -n ack-system get pods
kubectl get crd | grep $SERVICE
buckets.s3.services.k8s.aws                  2023-06-11T08:24:32Z
kubectl get all -n ack-system
kubectl get-all -n ack-system
kubectl describe sa -n ack-system ack-s3-controller
```

**IRSA 설정 : 권장 정책 AmazonS3FullAccess**

```
# Create an iamserviceaccount - AWS IAM role bound to a Kubernetes service account
eksctl create iamserviceaccount \
  --name ack-$SERVICE-controller \
  --namespace ack-system \
  --cluster $CLUSTER_NAME \
  --attach-policy-arn $(aws iam list-policies --query 'Policies[?PolicyName==`AmazonS3FullAccess`].Arn' --output text) \
  --override-existing-serviceaccounts --approve

# 확인 >> 웹 관리 콘솔에서 CloudFormation Stack >> IAM Role 확인
eksctl get iamserviceaccount --cluster $CLUSTER_NAME

# Inspecting the newly created Kubernetes Service Account, we can see the role we want it to assume in our pod.
kubectl get sa -n ack-system
kubectl describe sa ack-$SERVICE-controller -n ack-system

# Restart ACK service controller deployment using the following commands.
kubectl -n ack-system rollout restart deploy ack-$SERVICE-controller-$SERVICE-chart

# IRSA 적용으로 Env, Volume 추가 확인
kubectl describe pod -n ack-system -l k8s-app=$SERVICE-chart
```

**S3 버킷 생성, 업데이트, 삭제**

```
# [터미널1] 모니터링
watch -d aws s3 ls

# S3 버킷 생성을 위한 설정 파일 생성
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text)
export BUCKET_NAME=my-ack-s3-bucket-$AWS_ACCOUNT_ID

read -r -d '' BUCKET_MANIFEST <<EOF
apiVersion: s3.services.k8s.aws/v1alpha1
kind: Bucket
metadata:
  name: $BUCKET_NAME
spec:
  name: $BUCKET_NAME
EOF

echo "${BUCKET_MANIFEST}" > bucket.yaml
cat bucket.yaml | yh

# S3 버킷 생성
aws s3 ls
kubectl create -f bucket.yaml
bucket.s3.services.k8s.aws/my-ack-s3-bucket-<my account id> created

# S3 버킷 확인
aws s3 ls
kubectl get buckets
kubectl describe bucket/$BUCKET_NAME | head -6
Name:         my-ack-s3-bucket-485702506058
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  s3.services.k8s.aws/v1alpha1
Kind:         Bucket

aws s3 ls | grep $BUCKET_NAME
2023-06-11 19:53:29 my-ack-s3-bucket-485702506058

# S3 버킷 업데이트 : 태그 정보 입력
read -r -d '' BUCKET_MANIFEST <<EOF
apiVersion: s3.services.k8s.aws/v1alpha1
kind: Bucket
metadata:
  name: $BUCKET_NAME
spec:
  name: $BUCKET_NAME
  tagging:
    tagSet:
    - key: myTagKey
      value: myTagValue
EOF

echo "${BUCKET_MANIFEST}" > bucket.yaml

# S3 버킷 설정 업데이트 실행 : 필요 주석 자동 업뎃 내용이니 무시해도됨!
kubectl apply -f bucket.yaml

# S3 버킷 업데이트 확인 
kubectl describe bucket/$BUCKET_NAME | grep Spec: -A5
Spec:
  Name:  my-ack-s3-bucket-485702506058
  Tagging:
    Tag Set:
      Key:    myTagKey
      Value:  myTagValue

# S3 버킷 삭제
kubectl delete -f bucket.yaml

# verify the bucket no longer exists
kubectl get bucket/$BUCKET_NAME
aws s3 ls | grep $BUCKET_NAME
```

**ACK S3 Controller 삭제**

```
# helm uninstall
export SERVICE=s3
helm uninstall -n $ACK_SYSTEM_NAMESPACE ack-$SERVICE-controller

# ACK S3 Controller 관련 crd 삭제
kubectl delete -f ~/$SERVICE-chart/crds

# IRSA 삭제
eksctl delete iamserviceaccount --cluster myeks --name ack-$SERVICE-controller --namespace ack-system

# namespace 삭제 >> ACK 모든 실습 후 삭제
kubectl delete namespace $ACK_K8S_NAMESPACE
```

**ACK EC2-Controller 설치 with Helm**

```
# 서비스명 변수 지정 및 helm 차트 다운로드
export SERVICE=ec2
export RELEASE_VERSION=$(curl -sL https://api.github.com/repos/aws-controllers-k8s/$SERVICE-controller/releases/latest | grep '"tag_name":' | cut -d'"' -f4 | cut -c 2-)
helm pull oci://public.ecr.aws/aws-controllers-k8s/$SERVICE-chart --version=$RELEASE_VERSION
tar xzvf $SERVICE-chart-$RELEASE_VERSION.tgz

# helm chart 확인
tree ~/$SERVICE-chart

# ACK EC2-Controller 설치
export ACK_SYSTEM_NAMESPACE=ack-system
export AWS_REGION=ap-northeast-2
helm install -n $ACK_SYSTEM_NAMESPACE ack-$SERVICE-controller --set aws.region="$AWS_REGION" ~/$SERVICE-chart

# 설치 확인
helm list --namespace $ACK_SYSTEM_NAMESPACE
kubectl -n $ACK_SYSTEM_NAMESPACE get pods -l "app.kubernetes.io/instance=ack-$SERVICE-controller"
kubectl get crd | grep $SERVICE
dhcpoptions.ec2.services.k8s.aws             2023-06-11T10:59:14Z
elasticipaddresses.ec2.services.k8s.aws      2023-06-11T10:59:14Z
instances.ec2.services.k8s.aws               2023-06-11T10:59:14Z
internetgateways.ec2.services.k8s.aws        2023-06-11T10:59:14Z
natgateways.ec2.services.k8s.aws             2023-06-11T10:59:14Z
routetables.ec2.services.k8s.aws             2023-06-11T10:59:15Z
securitygroups.ec2.services.k8s.aws          2023-06-11T10:59:15Z
subnets.ec2.services.k8s.aws                 2023-06-11T10:59:15Z
transitgateways.ec2.services.k8s.aws         2023-06-11T10:59:15Z
vpcendpoints.ec2.services.k8s.aws            2023-06-11T10:59:15Z
vpcs.ec2.services.k8s.aws                    2023-06-11T10:59:15Z
```

**IRSA 설정 : 권장 정책 AmazonEC2FullAccess**

```
# Create an iamserviceaccount - AWS IAM role bound to a Kubernetes service account
eksctl create iamserviceaccount \
  --name ack-$SERVICE-controller \
  --namespace $ACK_SYSTEM_NAMESPACE \
  --cluster $CLUSTER_NAME \
  --attach-policy-arn $(aws iam list-policies --query 'Policies[?PolicyName==`AmazonEC2FullAccess`].Arn' --output text) \
  --override-existing-serviceaccounts --approve

# 확인 >> 웹 관리 콘솔에서 CloudFormation Stack >> IAM Role 확인
eksctl get iamserviceaccount --cluster $CLUSTER_NAME

# Inspecting the newly created Kubernetes Service Account, we can see the role we want it to assume in our pod.
kubectl get sa -n $ACK_SYSTEM_NAMESPACE
kubectl describe sa ack-$SERVICE-controller -n $ACK_SYSTEM_NAMESPACE

# Restart ACK service controller deployment using the following commands.
kubectl -n $ACK_SYSTEM_NAMESPACE rollout restart deploy ack-$SERVICE-controller-$SERVICE-chart

# IRSA 적용으로 Env, Volume 추가 확인
kubectl describe pod -n $ACK_SYSTEM_NAMESPACE -l k8s-app=$SERVICE-chart
```

**VPC, Subnet 생성 및 삭제**

```
# [터미널1] 모니터링
while true; do aws ec2 describe-vpcs --query 'Vpcs[*].{VPCId:VpcId, CidrBlock:CidrBlock}' --output text; echo "-----"; sleep 1; done

# VPC 생성
cat <<EOF > vpc.yaml
apiVersion: ec2.services.k8s.aws/v1alpha1
kind: VPC
metadata:
  name: vpc-tutorial-test
spec:
  cidrBlocks: 
  - 10.0.0.0/16
  enableDNSSupport: true
  enableDNSHostnames: true
EOF
 
kubectl apply -f vpc.yaml
vpc.ec2.services.k8s.aws/vpc-tutorial-test created

# VPC 생성 확인
kubectl get vpcs
kubectl describe vpcs
aws ec2 describe-vpcs --query 'Vpcs[*].{VPCId:VpcId, CidrBlock:CidrBlock}' --output text


# [터미널1] 모니터링
VPCID=$(kubectl get vpcs vpc-tutorial-test -o jsonpath={.status.vpcID})
while true; do aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPCID" --query 'Subnets[*].{SubnetId:SubnetId, CidrBlock:CidrBlock}' --output text; echo "-----"; sleep 1 ; done

# 서브넷 생성
VPCID=$(kubectl get vpcs vpc-tutorial-test -o jsonpath={.status.vpcID})

cat <<EOF > subnet.yaml
apiVersion: ec2.services.k8s.aws/v1alpha1
kind: Subnet
metadata:
  name: subnet-tutorial-test
spec:
  cidrBlock: 10.0.0.0/20
  vpcID: $VPCID
EOF
kubectl apply -f subnet.yaml
subnet.ec2.services.k8s.aws/subnet-tutorial-test created

# 서브넷 생성 확인
kubectl get subnets
kubectl describe subnets
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPCID" --query 'Subnets[*].{SubnetId:SubnetId, CidrBlock:CidrBlock}' --output text
```

**Create a VPC Workflow : VPC, Subnet, SG, RT, EIP, IGW, NATGW, Instance 생성**

![image](https://github.com/jiwonYun9332/AWES-1/blob/3ec2e102537ed760f90d4aa8d9ffd7d443ec5368/Study/images/124_image.jpg)

**vpc-workflow.yaml 파일 생성**

```
cat <<EOF > vpc-workflow.yaml
apiVersion: ec2.services.k8s.aws/v1alpha1
kind: VPC
metadata:
  name: tutorial-vpc
spec:
  cidrBlocks: 
  - 10.0.0.0/16
  enableDNSSupport: true
  enableDNSHostnames: true
  tags:
    - key: name
      value: vpc-tutorial
---
apiVersion: ec2.services.k8s.aws/v1alpha1
kind: InternetGateway
metadata:
  name: tutorial-igw
spec:
  vpcRef:
    from:
      name: tutorial-vpc
---
apiVersion: ec2.services.k8s.aws/v1alpha1
kind: NATGateway
metadata:
  name: tutorial-natgateway1
spec:
  subnetRef:
    from:
      name: tutorial-public-subnet1
  allocationRef:
    from:
      name: tutorial-eip1
---
apiVersion: ec2.services.k8s.aws/v1alpha1
kind: ElasticIPAddress
metadata:
  name: tutorial-eip1
spec:
  tags:
    - key: name
      value: eip-tutorial
---
apiVersion: ec2.services.k8s.aws/v1alpha1
kind: RouteTable
metadata:
  name: tutorial-public-route-table
spec:
  vpcRef:
    from:
      name: tutorial-vpc
  routes:
  - destinationCIDRBlock: 0.0.0.0/0
    gatewayRef:
      from:
        name: tutorial-igw
---
apiVersion: ec2.services.k8s.aws/v1alpha1
kind: RouteTable
metadata:
  name: tutorial-private-route-table-az1
spec:
  vpcRef:
    from:
      name: tutorial-vpc
  routes:
  - destinationCIDRBlock: 0.0.0.0/0
    natGatewayRef:
      from:
        name: tutorial-natgateway1
---
apiVersion: ec2.services.k8s.aws/v1alpha1
kind: Subnet
metadata:
  name: tutorial-public-subnet1
spec:
  availabilityZone: ap-northeast-2a
  cidrBlock: 10.0.0.0/20
  mapPublicIPOnLaunch: true
  vpcRef:
    from:
      name: tutorial-vpc
  routeTableRefs:
  - from:
      name: tutorial-public-route-table
---
apiVersion: ec2.services.k8s.aws/v1alpha1
kind: Subnet
metadata:
  name: tutorial-private-subnet1
spec:
  availabilityZone: ap-northeast-2a
  cidrBlock: 10.0.128.0/20
  vpcRef:
    from:
      name: tutorial-vpc
  routeTableRefs:
  - from:
      name: tutorial-private-route-table-az1
---
apiVersion: ec2.services.k8s.aws/v1alpha1
kind: SecurityGroup
metadata:
  name: tutorial-security-group
spec:
  description: "ack security group"
  name: tutorial-sg
  vpcRef:
     from:
       name: tutorial-vpc
  ingressRules:
    - ipProtocol: tcp
      fromPort: 22
      toPort: 22
      ipRanges:
        - cidrIP: "0.0.0.0/0"
          description: "ingress"
EOF
```

```
# VPC 환경 생성
kubectl apply -f vpc-workflow.yaml

# [터미널1] NATGW 생성 완료 후 tutorial-private-route-table-az1 라우팅 테이블 ID가 확인되고 그후 tutorial-private-subnet1 서브넷ID가 확인됨 > 5분 정도 시간 소요
watch -d kubectl get routetables,subnet

# VPC 환경 생성 확인
kubectl describe vpcs
kubectl describe internetgateways
kubectl describe routetables
kubectl describe natgateways
kubectl describe elasticipaddresses
kubectl describe securitygroups

# 배포 도중 2개의 서브넷 상태 정보 비교 해보자
kubectl describe subnets
Name:         tutorial-private-subnet1
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  ec2.services.k8s.aws/v1alpha1
Kind:         Subnet
Metadata:
  Creation Timestamp:  2023-06-11T11:07:42Z
  Generation:          1
  Resource Version:    51760
  UID:                 e66a692e-9466-43e8-81de-fd9c7581e543
Spec:
  Availability Zone:  ap-northeast-2a
  Cidr Block:         10.0.128.0/20
  Route Table Refs:
    From:
      Name:  tutorial-private-route-table-az1
  Vpc Ref:
    From:
      Name:  tutorial-vpc
Status:
  Conditions:
    Last Transition Time:  2023-06-11T11:08:04Z
    Message:               Reference resolution failed
    Reason:                the referenced resource is not synced yet. resource:RouteTable, namespace:default, name:tutorial-private-route-table-az1
    Status:                Unknown
    Type:                  ACK.ReferencesResolved
Events:                    <none>


Name:         tutorial-public-subnet1
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  ec2.services.k8s.aws/v1alpha1
Kind:         Subnet
Metadata:
  Creation Timestamp:  2023-06-11T11:07:42Z
  Finalizers:
    finalizers.ec2.services.k8s.aws/Subnet
  Generation:        2
  Resource Version:  51667
  UID:               3a83bc6c-ae75-48a7-b790-3346fe1dbb60
Spec:
  assignIPv6AddressOnCreation:  false
  Availability Zone:            ap-northeast-2a
  Availability Zone ID:         apne2-az1
  Cidr Block:                   10.0.0.0/20
  enableDNS64:                  false
  ipv6Native:                   false
  Map Public IP On Launch:      true
  Route Table Refs:
    From:
      Name:  tutorial-public-route-table
  Vpc Ref:
    From:
      Name:  tutorial-vpc
Status:
  Ack Resource Metadata:
    Arn:                       arn:aws:ec2:ap-northeast-2:485702506058:subnet/subnet-05233ba5b9ff93158
    Owner Account ID:          485702506058
    Region:                    ap-northeast-2
  Available IP Address Count:  4091
  Conditions:
    Last Transition Time:           2023-06-11T11:07:45Z
    Status:                         True
    Type:                           ACK.ReferencesResolved
    Last Transition Time:           2023-06-11T11:07:45Z
    Message:                        Resource synced successfully
    Reason:
    Status:                         True
    Type:                           ACK.ResourceSynced
  Default For AZ:                   false
  Map Customer Owned IP On Launch:  false
  Owner ID:                         485702506058
  Private DNS Name Options On Launch:
  State:      available
  Subnet ID:  subnet-05233ba5b9ff93158
Events:       <none>
```

![image](https://github.com/jiwonYun9332/AWES-1/blob/5c08c699a57892b18d73b147b3fdb6d26dc90dbc/Study/images/125_image.jpg)

**퍼블릭 서브넷에 인스턴스 생성**

```
# public 서브넷 ID 확인
PUBSUB1=$(kubectl get subnets tutorial-public-subnet1 -o jsonpath={.status.subnetID})
echo $PUBSUB1

# 보안그룹 ID 확인
TSG=$(kubectl get securitygroups tutorial-security-group -o jsonpath={.status.id})
echo $TSG

# Amazon Linux 2 최신 AMI ID 확인
AL2AMI=$(aws ec2 describe-images --owners amazon --filters "Name=name,Values=amzn2-ami-hvm-2.0.*-x86_64-gp2" --query 'Images[0].ImageId' --output text)
echo $AL2AMI

# 각자 자신의 SSH 키페어 이름 변수 지정
MYKEYPAIR=<각자 자신의 SSH 키페어 이름>
MYKEYPAIR=kp-gasida

# 변수 확인 > 특히 서브넷 ID가 확인되었는지 꼭 확인하자!
echo $PUBSUB1 , $TSG , $AL2AMI , $MYKEYPAIR


# [터미널1] 모니터링
while true; do aws ec2 describe-instances --query "Reservations[*].Instances[*].{PublicIPAdd:PublicIpAddress,PrivateIPAdd:PrivateIpAddress,InstanceName:Tags[?Key=='Name']|[0].Value,Status:State.Name}" --filters Name=instance-state-name,Values=running --output table; date ; sleep 1 ; done

# public 서브넷에 인스턴스 생성
cat <<EOF > tutorial-bastion-host.yaml
apiVersion: ec2.services.k8s.aws/v1alpha1
kind: Instance
metadata:
  name: tutorial-bastion-host
spec:
  imageID: $AL2AMI # AL2 AMI ID - ap-northeast-2
  instanceType: t3.medium
  subnetID: $PUBSUB1
  securityGroupIDs:
  - $TSG
  keyName: $MYKEYPAIR
  tags:
    - key: producer
      value: ack
EOF
kubectl apply -f tutorial-bastion-host.yaml

# 인스턴스 생성 확인
kubectl get instance
kubectl describe instance
aws ec2 describe-instances --query "Reservations[*].Instances[*].{PublicIPAdd:PublicIpAddress,PrivateIPAdd:PrivateIpAddress,InstanceName:Tags[?Key=='Name']|[0].Value,Status:State.Name}" --filters Name=instance-state-name,Values=running --output table
------------------------------------------------------------------
|                        DescribeInstances                       |
+----------------+-----------------+------------------+----------+
|  InstanceName  |  PrivateIPAdd   |   PublicIPAdd    | Status   |
+----------------+-----------------+------------------+----------+
|  myeks-ng1-Node|  192.168.3.61   |  13.209.15.136   |  running |
|  myeks-ng1-Node|  192.168.2.130  |  15.164.174.34   |  running |
|  myeks-bastion |  192.168.1.100  |  3.36.69.66      |  running |
|  myeks-ng1-Node|  192.168.1.97   |  54.180.137.188  |  running |
|  None          |  10.0.12.205    |  43.202.49.136   |  running |
+----------------+-----------------+------------------+----------+
```

**public 서브넷에 인스턴스 접속**

```
ssh -i <자신의 키페어파일> ec2-user@<public 서브넷에 인스턴스 퍼블릭IP>
------
# public 서브넷에 인스턴스 접속 후 외부 인터넷 통신 여부 확인 
ping -c 2 8.8.8.8
```

**보안 그룹 정책 수정 : egress 규칙 추가**

```
cat <<EOF > modify-sg.yaml
apiVersion: ec2.services.k8s.aws/v1alpha1
kind: SecurityGroup
metadata:
  name: tutorial-security-group
spec:
  description: "ack security group"
  name: tutorial-sg
  vpcRef:
     from:
       name: tutorial-vpc
  ingressRules:
    - ipProtocol: tcp
      fromPort: 22
      toPort: 22
      ipRanges:
        - cidrIP: "0.0.0.0/0"
          description: "ingress"
  egressRules:
    - ipProtocol: '-1'
      ipRanges:
        - cidrIP: "0.0.0.0/0"
          description: "egress"
EOF
kubectl apply -f modify-sg.yaml

# 변경 확인 >> 보안그룹에 아웃바운드 규칙 확인
kubectl logs -n $ACK_SYSTEM_NAMESPACE -l k8s-app=ec2-chart -f
```

**public 서브넷에 인스턴스 접속 후 외부 인터넷 통신 확인**

```
ssh -i <자신의 키페어파일> ec2-user@<public 서브넷에 인스턴스 퍼블릭IP>
------
# public 서브넷에 인스턴스 접속 후 외부 인터넷 통신 여부 확인 
ping -c 10 8.8.8.8
curl ipinfo.io/ip ; echo # 출력되는 공인IP는 무엇인가?
exit
------
```

**프라이빗 서브넷에 인스턴스 생성**

```
# private 서브넷 ID 확인 >> NATGW 생성 완료 후 RT/SubnetID가 확인되어 다소 시간 필요함
PRISUB1=$(kubectl get subnets tutorial-private-subnet1 -o jsonpath={.status.subnetID})
echo $PRISUB1

# 변수 확인 > 특히 private 서브넷 ID가 확인되었는지 꼭 확인하자!
echo $PRISUB1 , $TSG , $AL2AMI , $MYKEYPAIR

# private 서브넷에 인스턴스 생성
cat <<EOF > tutorial-instance-private.yaml
apiVersion: ec2.services.k8s.aws/v1alpha1
kind: Instance
metadata:
  name: tutorial-instance-private
spec:
  imageID: $AL2AMI # AL2 AMI ID - ap-northeast-2
  instanceType: t3.medium
  subnetID: $PRISUB1
  securityGroupIDs:
  - $TSG
  keyName: $MYKEYPAIR
  tags:
    - key: producer
      value: ack
EOF
kubectl apply -f tutorial-instance-private.yaml

# 인스턴스 생성 확인
kubectl get instance
kubectl describe instance
aws ec2 describe-instances --query "Reservations[*].Instances[*].{PublicIPAdd:PublicIpAddress,PrivateIPAdd:PrivateIpAddress,InstanceName:Tags[?Key=='Name']|[0].Value,Status:State.Name}" --filters Name=instance-state-name,Values=running --output table
```

**실습 후 리소스 삭제**

```
kubectl delete -f tutorial-bastion-host.yaml && kubectl delete -f tutorial-instance-private.yaml
kubectl delete -f vpc-workflow.yaml  # vpc 관련 모든 리소스들 삭제에는 다소 시간이 소요됨
```

### 2. Flux

**flux란?**

![image](https://github.com/jiwonYun9332/AWES-1/blob/cc7a4c70cddfbf8ccb26bc7d3d7975c53f876fbe/Study/images/120_image.jpg)

flux는 쿠버네티스를 위한 gitops 도구이다. flux는 git에 있는 쿠버네티스를 manifest를 읽고, 쿠버네티스에 manifest를 배포한다.

**flux와 argocd 비교**

flux는 argocd와 같은 역할을 하고 있어 비교대상으로 자주 언급된다.

argocd는 kustomize를 사용할 때 디폴트 설정이 부족하기 때문에, argocd configmap에서 kustomize옵션을 한땀한땀 설정해야 한다.
반면에  flux는 바로 kustomize를 사용하도록 설정이 되어 있다.

flux는 쿠버네티스 operator패턴을 사용한다. 사용자가 쿠버네티스에 배포할 내용을 flux CRD로 정의하면, flux controller가 CRD를 읽어 리소스를 쿠버네티스에 배포한다.

![image](https://github.com/jiwonYun9332/AWES-1/blob/cc7a4c70cddfbf8ccb26bc7d3d7975c53f876fbe/Study/images/121_image.jpg)

핵심 CRD는 소스(source)와 애플리케이션(application)입니다. 소스는 git, helm registry 등 manifest 주소를 설정합니다. 
애플리케이션은 helm 또는 kustomize를 사용하여 manifest를 쿠버네티스에 배포한다.

**Flux CLI 설치 및 Bootstrap**

Github Token 발급

![image](https://github.com/jiwonYun9332/AWES-1/blob/6870a6ce454de27974b6bbbcbcb269a923c4e75a/Study/images/126_image.jpg)




3. ArgoCD

**GitOps란?**

GitOps는 DevOps의 실천 방법 중 하나로,
애플리케이션의 배포와 운영에 관련된 모든 요소들을 Git에서 관리한다는 뜻이다.

GitOps는 Git Pull 요청을 사용하여 인프라 프로비저닝 및 배포를 자동으로 관리한다.

Git 레포지토리에는 전체 시스템 상태가 포함되어 있어 시스템 상태의 변화 추이를 확인, 감사할 수 있다.

![image](https://github.com/jiwonYun9332/AWES-1/blob/385ae106cde740cb70bdec952d2fea69dca5a043/Study/images/122_image.jpg)

GitOps의 원칙

- 모든 시스템은 선언적으로 선언되어야 한다.
- 시스템의 상태는 Git의 버전을 따라간다.
- 승인된 변화는 자동으로 시스템에 대응한다.
- 배포에 실패하면 이를 사용자에게 경고해야 한다.

ArgoCD란?

ArgoCD는 GitOps 방식으로 관리되는 Manifest(yaml) 파일의 변경사항을 감시하며, 현재 배포된 환경의 상태와 Github Manifest 파일에 정의된 상태를 동일하게 유지하는 역할을 수행한다.

![image](https://github.com/jiwonYun9332/AWES-1/blob/385ae106cde740cb70bdec952d2fea69dca5a043/Study/images/123_image.jpg)

```
"ArgoCD is a declarative, GitOps continuous delivery tool for Kubernetes."
```

ArgoCD는 쿠버네티스를 위한 CD(Continuous Delivery) 툴이라고 할 수 있다.

쿠버네티스의 구성 요소들을 관리 및 배포하기 위해서는 Manifest 파일을 구성하여 실행해야 하며,
이러한 파일들은 계속해서 변경되기 때문에 지속적인 관리가 필요하다.
이를 간편하게 Git으로 관리하는 방식이 GitOps이다, GitOps를 실현시키며 쿠버네티스에 배포까지 해주는 툴이 ArgoCD이다.

4. Crossplane

5. eksdemo

