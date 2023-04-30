# 1Week

![eks logo](https://github.com/jiwonYun9332/AWES-1/blob/988ee97c30ec07bc1a3360fef09b0c944f5f0530/Study/images/1_logo.png)

## 참여 계기

지난 2년 간 인프라 업무를 하면서 주로 레거시 환경에서의 인프라 환경을 운영해왔습니다, 그러던 중 Docker, Kubernetes, AWS, TAS, Ansible과 같이
가상화, 컨테이너, 오케스트레이션, 클라우드의 기술을 점점 접하게 되면서 엔지니어로서의 업무 역량을 향상하기 위해 해당 스터디에 참여하게 되었습니다.

IT 동향을 파악하기 위해 가입한 오픈채팅방에서 가시다님의 EKS 스터디를 접하게 되면서 저에게
클라우드 및 쿠버네티스에 진입점이 되리라 저는 보고 있습니다.

## 스터디 참여 전 (준비 단계)

AWES 1회 스터디 전, 스터디를 진행하기 위해 알아야하는 지식을 미리 슬랙과 노션을 통해 공유 받았습니다.
기존에 관련 지식이 없더라도 해당 내용을 숙지한다면 스터디를 따라 갈 수 있었습니다.

## AWS EKS(Amazon Elastic Kubernetes Service)

쿠버네티스로 컨테이너 배포 프로세스를 간소화할 수 있지만 쿠버네티스 그 자체를 배포하고 운영하기가 복잡할 수 있다.
실제로 기업이 쿠버네티스를 채택하는 데 가장 큰 걸림돌은 경험과 전문성이 부족하기 때문이다. 

따라서 컨트롤 플레인과 노드를 설치, 운영, 유지 관리가 필요없는 쿠버네티스 런타임 서비스인 Amazon EKS가 도움이 된다.

> EKS 장점

- EKS는 사용자를 위해 여러 AWS 가용 영역 전체에서 Kuberbetes 관리 인프라를 운영하여 단일 장애 발샌 지점을 제거한다. 

- Amazon EKS는 공인 Kubernetes 준수 서비스로 기존 도구 및 플러그인을 사용할 수 있다. 

- 표준 Kubernetes 환경에서 실행되는 애플리케이션은 완벽하게 호환되며 Amazon EKS로 마이그레이션 할 수 있다.

> AWS에서의 EKS 구축 예시

![eks](https://github.com/jiwonYun9332/AWES-1/blob/240facf869b14e8897e0a389cfbe513f590a1284/Study/images/2_eks.png)

**사용 Tool**: draw.io

[https://aws.amazon.com/ko/solutions/implementations/amazon-eks/]


**구성**
- 가용 영역 4개에 걸쳐 있는 고가용성 아키텍처
- Public Subnet
  - 프라이빗 서브넷의 리소스에 대한 아웃바운드 인터넷 액세스를 허용하기 위한 관리형 NAT 게이트웨이
  - 프라이빗 서브넷에서 Amazon Elastic Compute Cloud(Amazon EC2) 인스턴스에 대한 인바운드 Secure Shell(SSH) 액세스를 허용하기 위한 오토 스케일링 그룹의 
  Linux Bastion 호스트, Bastion 호스트는 Kubernetes 클러스터를 관리하기 위한 Kubernetes kubectl 명령줄 인터페이스로도 구성된다.
- Private Subnet
: Kubernetes 노드 그룹
- Kubernetes 컨트롤 플레인을 제공하는 Amazon EKS 클러스터

Amazon EKS에서는 Amazon VPC 통합 네트워킹을 지원하고 있어 파드에서 VPC 내부 주소 
대역을 사용할 수 있으며 클러스터 외부와의 통신이 가능하다.


## EKS Architecture (+추가)

![eks](https://github.com/jiwonYun9332/AWES-1/blob/d97f916697eda0194fdd843330dd7bebc04c4d20/Study/images/5_eks.png)

**사용 Tool**: draw.io

위 그림에서 AWS VPC는 컨트롤 플레인이 설치되며,

Customer VPC는 고객이 관리하는 Worker Node를 호스팅합니다.

[출처]
https://medium.com/beck-et-al/private-kubernetes-cluster-on-aws-using-elastic-kubernetes-service-and-its-challenges-89730e097867

**위 내용을 이해하기 위해 상세적으로 알아보겠습니다**

> k8s architecture

![k8s](https://github.com/jiwonYun9332/AWES-1/blob/dee88e9dfed270f7478fcd2c68405b7ed286e2b4/Study/images/6_k8s.png)

**사용 Tool**: draw.io

마스터 노드는 Kubernetes 클러스터의 전반적인 관리를 담당하며, 통신, 스케줄링 및 컨트롤러를 처리하는 세 가지 컴포넌트인
Kube API 서버, 스케줄러, 컨트롤러 관리자로 구성된다.

Kube API 서버를 사용하여 Kubernetes API와 상호 작용할 수 있다.

스케줄러는 생성된 Pods를 감시하며 아직 Node 설계가 되지 않은 Pod를 생성하고 특정 Node에서 실행되도록 Pod를 설계한다.

컨트롤러 관리자는 worker의 상태를 담당하는 노드 컨트롤러, 복제된 컨트롤러에 대해 올바른 수의 Pods를 유지 관리하는 책임을 지는 리플리케이션 컨트롤러, 서비스와 포드를 함께 연결하는
엔드포인트 컨트롤러가 포함된다. 그리고 액세스 관리를 처리하는 서비스 계정 및 토큰 컨트롤러 마지막으로, 저장된 간단한 분산 키 값인 CD가 있다.

Kubernetes용 CLI인 kubectl application을 사용하여 마스터 노드와 상호 작용하게되며,
kubectl에는 kubeconfig이라는 설정 파일에는 서버 정보와 API 서버에 접근하기 위한 인증 정보가 있습니다.

워커 노드는 우리가 설치한 애플리케이션이 작동하는 노드로 pod가 노드에 맞게 설계되었는지 확인하는 API 서버와 통신하는 에이전트입니다.
워커 노드에서는 containerd, docker와 같은 컨테이너 런타임을 통해 pod 컨테이너를 실행합니다. 노드에 탑재된 포드의 상태를 인지하고 master에게 관련 상태 정보를 제공합니다. 

kube-proxy는 단일 작업자 노드에서 서비스에 대한 네트워크 프록시이면서 로드 밸런서입니다.
TCP 및 UDP 패킷에 대한 네트워크 라우팅을 처리하고 연결 전달을 수행합니다.

출처
[https://www.argopacific.io/blog/post/architecture-of-kubernetes-cluster]

**etcd**는 한 줄로 요약하면 분산시스템에서 중요한 데이터를 저장할 때 사용할 수 있는 key-value 분산 저장소입니다.

```
A distributed, reliable key-value store for the most critical data of a distributed system.
```

k8s에서는 etcd에 cluster의 모든 정보를 저장하고 있는 데이터베이스이며
etcd를 통해 cluster를 백업/복구할 수 있습니다.

etcd는 k8s에서 중요한 역할을 하고 있어 분산된 환경에 구성하는 것이 좋습니다.

분산된 환경에서 구성된 etcd는 **replicated state machine** 방법을 통해 다른 가용 영역의 etcd가 
죽더라도 서비스를 정상적으로 제공합니다.

RSM(Replicated state machine)
: rsm은 command가 들어있는 log 단위로 데이터를 처리하며 데이터의 write를 log append라고 부릅니다.

[출처]
https://velog.io/@squarebird/Kubernetes-etcd

> eks architecture

![controlplane](https://github.com/jiwonYun9332/AWES-1/blob/f3482f051fbf0f9dc6f48409128b129a2d5be3df/Study/images/7_amazoneks.png)

AWS EKS 서비스에서는 AWS Managed VPC에서 컨트롤 플레인을 제공해줍니다.

api서버와 etcd는 각 3개의 가용 영역에 분산되어 있으며, 로드밸런서를 통해 부하 분산됩니다.

![dateplane](https://github.com/jiwonYun9332/AWES-1/blob/9ca3296196ebed894d6bf409a12c8c9bec5e227e/Study/images/11_dateplane.png)

Customer VPC는 고객의 애플리케이션이 올라가는 워커 노드가 올라가게 됩니다.

클러스터를 퍼블릭 또는 프라이빗하게 구성할 지에 따라 아래 구성과 같이 설정이 달라집니다.

**pulibc**

![public](https://github.com/jiwonYun9332/AWES-1/blob/9ca3296196ebed894d6bf409a12c8c9bec5e227e/Study/images/8_public.png)

퍼블릭 엔드포인트만 해당
: 이 경우 노드에는 컨트롤 플레인에 연결하기 위한 퍼블릭 IP 주소가 있어야 한다. NAT 게이트웨이의 퍼블릭 IP 주소를 사용할 수 있는 
인터넷 게이트웨이 또는 NAT 게이트웨이에 대한 경로도 있어야 한다.

**publicprivate**

![publicprivate](https://github.com/jiwonYun9332/AWES-1/blob/9ca3296196ebed894d6bf409a12c8c9bec5e227e/Study/images/9_publicprivate.png)

퍼블릭 및 프라이빗 엔드포인트
: 이 모드에서 작업자 노드 VPC 내에서 제어 플레인으로의 Kubernetes API 요청은 작업 노드 VPC 내의 EKS 관리 ENI를 통과한다.

**private**

![private](https://github.com/jiwonYun9332/AWES-1/blob/9ca3296196ebed894d6bf409a12c8c9bec5e227e/Study/images/10_private.png)

프라이빗 엔드포인트 전용
: 인터넷에서 API 서버에 대한 퍼블릭 액세스가 닫힌다. 모든 kubectl 명령은 VPC or AWS VPC or AWS DirectConnect와 같은 연결된 네트워크 내에서
VPC로 시작되는 경우에만 작동한다.

출처
https://awskoreamarketingasset.s3.amazonaws.com/2022%20Summit/pdf/T14S4_Amazon%20EKS%20%EB%A7%88%EC%9D%B4%EA%B7%B8%EB%A0%88%EC%9D%B4%EC%85%98%20%EC%9A%94%EC%A0%90%20%EC%A0%95%EB%A6%AC.pdf

## 스터디 배포 실습

![architecture](https://github.com/jiwonYun9332/AWES-1/blob/05df934c1a1dd2b4ab4c1aa225d34ac5d089213f/Study/images/12_aws.png)

1. 사전 준비 작업

1.1 Key-Pair 생성

AWS WEB Console에서 키페어 생성

2. 설치

Study에서 제공해주신 템플릿으로 cloudformation 활용하여 

```
AWSTemplateFormatVersion: '2010-09-09'

Metadata:
  AWS::CloudFormation::Interface:
    ParameterGroups:
      - Label:
          default: "<<<<< EKSCTL MY EC2 >>>>>"
        Parameters:
          - ClusterBaseName
          - KeyName
#          - SgIngressSshCidr
          - MyInstanceType
          - LatestAmiId
      - Label:
          default: "<<<<< Region AZ >>>>>"
        Parameters:
          - TargetRegion
          - AvailabilityZone1
          - AvailabilityZone2
      - Label:
          default: "<<<<< VPC Subnet >>>>>"
        Parameters:
          - VpcBlock
          - PublicSubnet1Block
          - PublicSubnet2Block
          - PrivateSubnet1Block
          - PrivateSubnet2Block


Parameters:
  ClusterBaseName:
    Type: String
    Default: myeks
    AllowedPattern: "[a-zA-Z][-a-zA-Z0-9]*"
    Description: must be a valid Allowed Pattern '[a-zA-Z][-a-zA-Z0-9]*'
    ConstraintDescription: ClusterBaseName - must be a valid Allowed Pattern

  KeyName:
    Description: Name of an existing EC2 KeyPair to enable SSH access to the instances. Linked to AWS Parameter
    Type: AWS::EC2::KeyPair::KeyName
    ConstraintDescription: must be the name of an existing EC2 KeyPair.

 # SgIngressSshCidr:
 #   Description: The IP address range that can be used to communicate to the EC2 instances
 #   Type: String
 #    MinLength: '9'
 #   MaxLength: '18'
 #   Default: 0.0.0.0/0
 #   AllowedPattern: (\d{1,3})\.(\d{1,3})\.(\d{1,3})\.(\d{1,3})/(\d{1,2})
 #   ConstraintDescription: must be a valid IP CIDR range of the form x.x.x.x/x.

  MyInstanceType:
    Description: Enter t2.micro, t2.small, t2.medium, t3.micro, t3.small, t3.medium. Default is t2.micro.
    Type: String
    Default: t3.medium
    AllowedValues:
      - t2.micro
      - t2.small
      - t2.medium
      - t3.micro
      - t3.small
      - t3.medium

  LatestAmiId:
    Description: (DO NOT CHANGE)
    Type: 'AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>'
    Default: '/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2'
    AllowedValues:
      - /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2

  TargetRegion:
    Type: String
    Default: ap-northeast-2

  AvailabilityZone1:
    Type: String
    Default: ap-northeast-2a

  AvailabilityZone2:
    Type: String
    Default: ap-northeast-2c

  VpcBlock:
    Type: String
    Default: 192.168.0.0/16

  PublicSubnet1Block:
    Type: String
    Default: 192.168.1.0/24

  PublicSubnet2Block:
    Type: String
    Default: 192.168.2.0/24

  PrivateSubnet1Block:
    Type: String
    Default: 192.168.3.0/24

  PrivateSubnet2Block:
    Type: String
    Default: 192.168.4.0/24

Resources:
# VPC
  EksVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcBlock
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: !Sub ${ClusterBaseName}-VPC

# PublicSubnets
  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      AvailabilityZone: !Ref AvailabilityZone1
      CidrBlock: !Ref PublicSubnet1Block
      VpcId: !Ref EksVPC
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub ${ClusterBaseName}-PublicSubnet1
        - Key: kubernetes.io/role/elb
          Value: 1

  PublicSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      AvailabilityZone: !Ref AvailabilityZone2
      CidrBlock: !Ref PublicSubnet2Block
      VpcId: !Ref EksVPC
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub ${ClusterBaseName}-PublicSubnet2
        - Key: kubernetes.io/role/elb
          Value: 1

  InternetGateway:
    Type: AWS::EC2::InternetGateway

  VPCGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      InternetGatewayId: !Ref InternetGateway
      VpcId: !Ref EksVPC

  PublicSubnetRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref EksVPC
      Tags:
        - Key: Name
          Value: !Sub ${ClusterBaseName}-PublicSubnetRouteTable

  PublicSubnetRoute:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PublicSubnetRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

  PublicSubnet1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet1
      RouteTableId: !Ref PublicSubnetRouteTable

  PublicSubnet2RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet2
      RouteTableId: !Ref PublicSubnetRouteTable

# PrivateSubnets
  PrivateSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      AvailabilityZone: !Ref AvailabilityZone1
      CidrBlock: !Ref PrivateSubnet1Block
      VpcId: !Ref EksVPC
      Tags:
        - Key: Name
          Value: !Sub ${ClusterBaseName}-PrivateSubnet1
        - Key: kubernetes.io/role/internal-elb
          Value: 1

  PrivateSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      AvailabilityZone: !Ref AvailabilityZone2
      CidrBlock: !Ref PrivateSubnet2Block
      VpcId: !Ref EksVPC
      Tags:
        - Key: Name
          Value: !Sub ${ClusterBaseName}-PrivateSubnet2
        - Key: kubernetes.io/role/internal-elb
          Value: 1

  PrivateSubnetRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref EksVPC
      Tags:
        - Key: Name
          Value: !Sub ${ClusterBaseName}-PrivateSubnetRouteTable

  PrivateSubnet1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PrivateSubnet1
      RouteTableId: !Ref PrivateSubnetRouteTable

  PrivateSubnet2RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PrivateSubnet2
      RouteTableId: !Ref PrivateSubnetRouteTable

# EKSCTL-Host
  EKSEC2SG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: eksctl-host Security Group
      VpcId: !Ref EksVPC
      Tags:
        - Key: Name
          Value: !Sub ${ClusterBaseName}-HOST-SG
      SecurityGroupIngress:
      #- IpProtocol: '-1'
        #FromPort: '22'
        #ToPort: '22'
        #CidrIp: !Ref SgIngressSshCidr
      - IpProtocol: tcp
        FromPort: 22
        ToPort: 22
        CidrIp: 0.0.0.0/0

  EKSEC2:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !Ref MyInstanceType
      ImageId: !Ref LatestAmiId
      KeyName: !Ref KeyName
      Tags:
        - Key: Name
          Value: !Sub ${ClusterBaseName}-host
      NetworkInterfaces:
        - DeviceIndex: 0
          SubnetId: !Ref PublicSubnet1
          GroupSet:
          - !Ref EKSEC2SG
          AssociatePublicIpAddress: true
          PrivateIpAddress: 192.168.1.100
      BlockDeviceMappings:
        - DeviceName: /dev/xvda
          Ebs:
            VolumeType: gp3
            VolumeSize: 20
            DeleteOnTermination: true
      UserData:
        Fn::Base64:
          !Sub |
            #!/bin/bash
            hostnamectl --static set-hostname "${ClusterBaseName}-host"

            # Config convenience
            echo 'alias vi=vim' >> /etc/profile
            echo "sudo su -" >> /home/ec2-user/.bashrc

            # Change Timezone
            sed -i "s/UTC/Asia\/Seoul/g" /etc/sysconfig/clock
            ln -sf /usr/share/zoneinfo/Asia/Seoul /etc/localtime

            # Install Packages
            cd /root
            yum -y install tree jq git htop lynx

            # Install kubectl & helm
            #curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.26.2/2023-03-17/bin/linux/amd64/kubectl
            curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.25.7/2023-03-17/bin/linux/amd64/kubectl
            install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
            curl -s https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash

            # Install eksctl
            curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
            mv /tmp/eksctl /usr/local/bin

            # Install aws cli v2
            curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
            unzip awscliv2.zip >/dev/null 2>&1
            sudo ./aws/install
            complete -C '/usr/local/bin/aws_completer' aws
            echo 'export AWS_PAGER=""' >>/etc/profile
            export AWS_DEFAULT_REGION=${AWS::Region}
            echo "export AWS_DEFAULT_REGION=$AWS_DEFAULT_REGION" >> /etc/profile

            # Install YAML Highlighter
            wget https://github.com/andreazorzetto/yh/releases/download/v0.4.0/yh-linux-amd64.zip
            unzip yh-linux-amd64.zip
            mv yh /usr/local/bin/

            # Install krew
            curl -LO https://github.com/kubernetes-sigs/krew/releases/download/v0.4.3/krew-linux_amd64.tar.gz
            tar zxvf krew-linux_amd64.tar.gz
            ./krew-linux_amd64 install krew
            export PATH="$PATH:/root/.krew/bin"
            echo 'export PATH="$PATH:/root/.krew/bin"' >> /etc/profile

            # Install kube-ps1
            echo 'source <(kubectl completion bash)' >> /etc/profile
            echo 'alias k=kubectl' >> /etc/profile
            echo 'complete -F __start_kubectl k' >> /etc/profile

            git clone https://github.com/jonmosco/kube-ps1.git /root/kube-ps1
            cat <<"EOT" >> /root/.bash_profile
            source /root/kube-ps1/kube-ps1.sh
            KUBE_PS1_SYMBOL_ENABLE=false
            function get_cluster_short() {
              echo "$1" | cut -d . -f1
            }
            KUBE_PS1_CLUSTER_FUNCTION=get_cluster_short
            KUBE_PS1_SUFFIX=') '
            PS1='$(kube_ps1)'$PS1
            EOT

            # Install krew plugin
            kubectl krew install ctx ns get-all  # ktop df-pv mtail tree

            # Install Docker
            amazon-linux-extras install docker -y
            systemctl start docker && systemctl enable docker

            # CLUSTER_NAME
            export CLUSTER_NAME=${ClusterBaseName}
            echo "export CLUSTER_NAME=$CLUSTER_NAME" >> /etc/profile

            # Create SSH Keypair
            ssh-keygen -t rsa -N "" -f /root/.ssh/id_rsa

Outputs:
  eksctlhost:
    Value: !GetAtt EKSEC2.PublicIp

```

3. 생성화면

![cloudformation](https://github.com/jiwonYun9332/AWES-1/blob/6c6c6f437472ac5583416498626dd4733d116dfd/Study/images/3_cloudformation.png)


4. 생성된 인스턴스 접속 및 administratorAccess 권한 IAM 유저 생성

4.1 EKS Cluster 생성

```
eksctl create cluster --name $CLUSTER_NAME --region=$AWS_DEFAULT_REGION --nodegroup-name=$CLUSTER_NAME-nodeg24 --ssh-access --external-dns-access --verbose 4
```

5. 

EKS 생성완료

![createEKS](https://github.com/jiwonYun9332/AWES-1/blob/1306070741b1b460c26be5b2f3ba146fc30e49dc/Study/images/4_createEKS.png)


## EKS 1회차 스터디 진행 후 개선사항

- 습득한 지식을 다른 사람에게 공유할 수 있는 글 작성에 어려움을 느낌

원인: 지금까지는 나만 이해할 수 있는 정리를 해왔음

- 어떻게 개선할 것인가? 

: 글 작성 후 주변 지인에게 공유 및 피드백을 받고 한 번 정리한 내용을
다시 리메이크하여, 올려 EKS를 사람들이 접하게 쉽게 할 것

## EKS 1회차 스터디를 통해 얻은 점

: AWS를 실무에 적용하는 것에 대해 막연하게 느꼈던 부분을 직접적으로 EKS를 배포하고, EKS Architecture 를 draw.io 사이트를 통해
아키텍처를 작성해보면서, IAM 권한을 할당하고, 보안적으로 우수한 아키텍처에 대한 이해, cloudformation 배포 등 실습을 통해
aws 활용 실무에서 운영하는 로직에 대한 그림이 조금 뚜렷해졌다. 




