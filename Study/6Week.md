
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















