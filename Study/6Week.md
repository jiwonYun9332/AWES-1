
# 6Week - 6주차 실습

![image](https://github.com/jiwonYun9332/AWES-1/blob/dc733821bb4a531f61099b2797ece54ee18700bb/Study/images/95_image.jpg)


### K8S 인증/인가

**Admission Controller Concept**

Admission Controller는 쿠버네티스의 API를 호출했을 때, 해당 요청의 내용을 변형하거나 검증하는 플러그인의 집합이다.
어떤 사용자가 쿠버네티스 API를 통해 포드의 생성을 요청했을 떄 해당 포드의 설정이 적절한지 검증함으로써 포드 템플릿을 변경할 수도 있고, 부적절하다고 판단되면
아예 포드 생성 요청을 거부할 수 있다.

- LimitRanger, ResourceQuota: 포드의 리소스 request, limit를 자동으로 설정해준다.
- Istio의 Envoy Proxy Injection: injection=true인 네임스페이스의 모든 포드는 sidecar가 자동으로 추가된다.
- ServiceAccount도 Admission Controller 플러그인의 한 종류이다.

이러한 작업을 수행하기 위해 Admission Controller는 두 가지 단계를 거치도록 구현되어 있다.

- Mutate
: 사용자가 요청한 API의 매니페스트를 검사해 적절한 값으로 수정하는 변형 작업을 수행한다.
- Validate
: 매니페스트가 유효한지를 검사함으로써 요청을 거절 할 것인지를 결정한다.

![image](https://github.com/jiwonYun9332/AWES-1/blob/5d1eae78f4cf86cf124c95e7b30615720e8fec42/Study/images/96_image.jpg)

API 서버로 들어오는 모든 요청은 최종적으로 etcd에 저장되는데, 그 전에 당연히 Auth를 거치게 된다. JWT 또는 인증서 등을 통해 클라이언트를 인증한 뒤,
클라이언트의 API 요청이 RBAC 권한과 매칭되는지를 검사한다. 클라이언트가 해당 요청을 수행할 수 있는 적절한 권한이 있다고 판단되면, API 서버는
Admission Controller의 Mutate, Validate 과정을 거쳐 etcd에 요청 데이터를 저장한다. 그 뒤에는 컨트롤러나 스케줄러 등이 etcd의 데이터를 감지해 적절한 작업을 수행하는 식이다.

요점은 'API 서버는 Mutate와 Validate 과정을 거친 뒤에야 비로소 etcd에 요청 데이터를 저장한다'

또한, 컨테이너의 root privileged 권한을 부여하는 것은 보안 취약점을 가져오기 때문에 PodSecurityPolicy 라는 별도의 Admission Controller가 쿠버네티스에
이미 내장되어 있다. 

**x.509 Client Certs**

 x.509
: ITU-T가 만든 PKI(Public Key Infrastructure, 공개키 기반 구조)의 표준이다.

**RBAC (Role-Based Access Control)**

- 액세스, 무엇에 접근할 수 있는가?
- 운영, 무엇을 읽고,  무엇을 작성할 수 있는가? 파일을 생성 또는 삭제할 수 있는가?
- 세션, 시스템에 얼마나 오래 머무를 수 있는 가? 로그인은 언제 작동하고, 언제 만료되는가?

**ABAC(Attribute-Based Access Control)**

- 사용자, 사람의 직급과 일반적인 작업 또는 연공서열에 따라 수행 가능한 작업이 결정될 수 있다.
- 리소스 속성, 파일 유형 또는 만든 사람, 문서의 중요도에 따라 액세스 권한이 결정될 수 있다.
- 환경, 파일에 액세스하고 있는 위치, 시간대 또는 날짜에 따라 액세스 권한이 결정될 수 있다.

**K8S(API 접근) 인증/인가**

API 서버 접근 과정 : 인증 → 인가 → Admission Control(API 요청 검증, 필요 시 변형 - 예. ResourceQuota, LimitRange)

![image](https://github.com/jiwonYun9332/AWES-1/blob/6ab46ef6bbfc18953d764d543f4b31e3f0ab1d88/Study/images/97_image.jpg)

**Authentication**

- X.509 Client Certs
: kubeconfig에 CA crt, Client crt(클라이언트 인증서), Client key(클라이언트 개인키)를 통해 인증

- kubectl
: 여러 클러스터(kubeconfig)를 관리 가능, contexts에 클러스터와 유저 및 인증서/키 참고

- ServiceAccount
: 기본 서비스 어카운트(default) - 시크릿(CA crt 와 token)

![image](https://github.com/jiwonYun9332/AWES-1/blob/b7561f44d0afec6a6a8544a813fd9ece9a4660c9/Study/images/98_image.jpg)

**Authorization**

인가 방식: RBAC(Role, RoleBinding), ABAC, Webhook, Node Authorization
RBAC: 역할 기반의 권한 관리, 사용자와 역할을 별개로 선언 후 두 가지를 조합(binding)해서 사용자에게 권한을 부여하여 kubectl or API로 관리 가능하다.

- Namespace/Cluster - Role/ClusterRole, RoleBinding/ClusterRoleBinding, Service Account
- Role(롤) - (RoleBinding 롤 바인딩) - Service Account(서비스 어카운트)
: 롤 바인딩은 롤과 서비스 어카운트를 연결
- Role(네임스페이스 내 자원의 권한) vs ClusterRole(클러스터 수준의 자원의 권한)

![image](https://github.com/jiwonYun9332/AWES-1/blob/21381c74cd10cad63d7909009c1b2154f6fe91b4/Study/images/99_image.jpg)

**.kube/config 파일 내용**

- clusters: kubectl 이 사용할 쿠버네티스 API 서버의 접속 정보 목록. 원격의 쿠버네티스 API 서버의 주소를 추가해 사용 가능
- users: 쿠버네티스의 API 서버에 접속하기 위한 사용자 인증 정보 목록. (서비스 어카운트의 토큰, 혹은 인증서의 데이터 등)
- contexts: cluster 항목과 users 항목에 정의된 값을 조합해 최종적으로 사용할 쿠버네티스 클러스터의 정보(컨텍스트)를 설정.
  - 예를 들어 cluster 항목에 클러스터 A, B가 정의돼 있고, users 항목에 사용자 a, b가 정의돼 있다면 cluster A + user a 를 조합해,
  'cluster A 에 user a 로 인증해 쿠버네티스를 사용한다' 라는 새로운 컨텍스트를 정의할 수 있다.
  - kubectl 을 사용하려면 여러 개의 컨텍스트 중 하나를 선택

```
cat .kube/config
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: { 값 }
    server: https://192.168.100.10:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    namespace: default
    user: kubernetes-admin
  name: admin@k8s
current-context: admin@k8s
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: { 값 }
    client-key-data: { 값 }
```

**실습**

![image](https://github.com/jiwonYun9332/AWES-1/blob/81cb13cb1476f2fd5e5762bff5daeecbca302110/Study/images/100_image.jpg)

- 쿠버네티스에 사용자를 위한 서비스 어카운트(Service Account, SA)를 생성
: dev-k8s, infra-k8s
- 사용자는 각기 다른 궎한(Role, 인가)을 가짐
: dev-k8s(dev-team 네임스페이스 내 모든 동작), infra-k8s(dev-team 네임스페이스 내 모든 동작)
- 각각 별도의 kubectl 파드를 생성하고, 해당 파드에 SA를 지정하여 권한에 대한 테스트를 진행















