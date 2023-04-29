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

> EKS 구축 예시

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


