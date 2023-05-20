# 4Week-1

## 4주차 실습

### 실습 환경 배포 확인

```
# kubectl get node --label-columns=node.kubernetes.io/instance-type,eks.amazonaws.com/capacityType,topology.kubernetes.io/zone
NAME                                               STATUS   ROLES    AGE     VERSION                INSTANCE-TYPE   CAPACITYTYPE   ZONE
ip-192-168-1-159.ap-northeast-2.compute.internal   Ready    <none>   7m23s   v1.24.13-eks-0a21954   t3.xlarge       ON_DEMAND      ap-northeast-2a
ip-192-168-2-90.ap-northeast-2.compute.internal    Ready    <none>   7m24s   v1.24.13-eks-0a21954   t3.xlarge       ON_DEMAND      ap-northeast-2b
ip-192-168-3-18.ap-northeast-2.compute.internal    Ready    <none>   7m22s   v1.24.13-eks-0a21954   t3.xlarge       ON_DEMAND      ap-northeast-2c
```

**for 문 사용 노드 SSH 접속 hostname 확인**

```
# for node in $N1 $N2 $N3; do ssh ec2-user@$node hostname; done
ip-192-168-1-159.ap-northeast-2.compute.internal
ip-192-168-2-90.ap-northeast-2.compute.internal
ip-192-168-3-18.ap-northeast-2.compute.internal
```

**AWS LB/ExternalDNS/EBS/EFS, kube-ops-view 설치**

```
MyDomain=senidin.com
echo "export MyDomain=senidin.com" >> /etc/profile
MyDnzHostedZoneId=$(aws route53 list-hosted-zones-by-name --dns-name "${MyDomain}." --query "HostedZones[0].Id" --output text)
echo $MyDomain, $MyDnzHostedZoneId
curl -s -O https://raw.githubusercontent.com/gasida/PKOS/main/aews/externaldns.yaml
MyDomain=$MyDomain MyDnzHostedZoneId=$MyDnzHostedZoneId envsubst < externaldns.yaml | kubectl apply -f -

# kube-ops-view
helm repo add geek-cookbook https://geek-cookbook.github.io/charts/
helm install kube-ops-view geek-cookbook/kube-ops-view --version 1.2.2 --set env.TZ="Asia/Seoul" --namespace kube-system
kubectl patch svc -n kube-system kube-ops-view -p '{"spec":{"type":"LoadBalancer"}}'
kubectl annotate service kube-ops-view -n kube-system "external-dns.alpha.kubernetes.io/hostname=kubeopsview.$MyDomain"
echo -e "Kube Ops View URL = http://kubeopsview.$MyDomain:8080/#scale=1.5"

Kube Ops View URL = http://kubeopsview.senidin.com:8080/#scale=1.5
```

URL 접속 : **Kube Ops View URL = http://kubeopsview.senidin.com:8080/#scale=1.5**

![image](https://github.com/jiwonYun9332/AWES-1/blob/59e11cc0f979f778a114a7cce1378ddf5f6923ab/Study/images/48_images.jpg)

**EBS csi driver 설치 확인**

```
eksctl get addon --cluster ${CLUSTER_NAME}
kubectl get pod -n kube-system -l 'app in (ebs-csi-controller,ebs-csi-node)'
kubectl get csinodes
```

**gp3 스토리지 클래스 생성**

```
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
EOT
kubectl apply -f gp3-sc.yaml
kubectl get sc
```

**EFS csi driver 설치**

```
helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/
helm repo update
helm upgrade -i aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver \
    --namespace kube-system \
    --set image.repository=602401143452.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/eks/aws-efs-csi-driver \
    --set controller.serviceAccount.create=false \
    --set controller.serviceAccount.name=efs-csi-controller-sa
```

**EFS 스토리지클래스 생성 및 확인**

```
curl -s -O https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/examples/kubernetes/dynamic_provisioning/specs/storageclass.yaml
sed -i "s/fs-92107410/$EfsFsId/g" storageclass.yaml
kubectl apply -f storageclass.yaml
kubectl get sc efs-sc
```

**이미지 정보 확인**

```
kubectl get pods --all-namespaces -o jsonpath="{.items[*].spec.containers[*].image}" | tr -s '[[:space:]]' '\n' | sort | uniq -c
```

**eksctl 설치/업데이트 addon 확인**

```
eksctl get addon --cluster $CLUSTER_NAME
```

**IRSA 확인**

```
eksctl get iamserviceaccount --cluster $CLUSTER_NAME
```
![image](https://github.com/jiwonYun9332/AWES-1/blob/e0771045e61bbefdc267b66dd3d08834df4e4d09/Study/images/51_images.jpg)

### RBAC, IRSA 이란?

**RBAC (role based access control)**

: K&8에서 정의할 수 있는 리소스 객체들을 이용하여 접근 제어를 하는 개념

- Role
- RoleBinding
- ServiceAccount

Role Binding된 ServiceAccount를 Pod에 할당하여, Pod는 지정된 Role을 사용할 수 있다.

![iamge](https://github.com/jiwonYun9332/AWES-1/blob/78c62db11d219a6aa4d75690b248b321156104d0/Study/images/49_images.jpg)

**IRSA (IAM Role Service Account)**

: ServiceAccount와 IAM Role을 매핑하기 위해 AWS에서 제공하는 서비스

K&8의 Pod가 AWS 특정 리소스에 접근하고자 할 때, AWS에서 IAM policy 연결된 Role을 부여함으로서 접근을 허용한다.

![image](https://github.com/jiwonYun9332/AWES-1/blob/78c62db11d219a6aa4d75690b248b321156104d0/Study/images/50_images.jpg)


### Logging in EKS


**api,audit,authenticator,controllerManager,scheduler 로그 활성화**

```
aws eks update-cluster-config --region $AWS_DEFAULT_REGION --name $CLUSTER_NAME \
>     --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'
{
    "update": {
        "id": "a12dfe39-b460-4d89-befa-2e7f1ce37986",
        "status": "InProgress",
        "type": "LoggingUpdate",
        "params": [
            {
                "type": "ClusterLogging",
                "value": "{\"clusterLogging\":[{\"types\":[\"api\",\"audit\",\"authenticator\",\"controllerManager\",\"scheduler\"],\"enabled\":true}]}"
            }
        ],
        "createdAt": "2023-05-21T02:37:13.689000+09:00",
        "errors": []
    }
}
```

![images](https://github.com/jiwonYun9332/AWES-1/blob/ba2cafc4b66c76f1c82a4432bd336058d08a3021/Study/images/52_images.jpg)

**로그 그룹 확인**

```
aws logs describe-log-groups | jq
{
  "logGroups": [
    {
      "logGroupName": "/aws/eks/myeks/cluster",
      "creationTime": 1684604251559,
      "metricFilterCount": 0,
      "arn": "arn:aws:logs:ap-northeast-2:485702506058:log-group:/aws/eks/myeks/cluster:*",
      "storedBytes": 0
    }
  ]
}
```

**로그 확인**

```
# 로그 tail 확인 : aws logs tail help
aws logs tail /aws/eks/$CLUSTER_NAME/cluster | more

# 신규 로그를 바로 출력
aws logs tail /aws/eks/$CLUSTER_NAME/cluster --follow

# 필터 패턴
aws logs tail /aws/eks/$CLUSTER_NAME/cluster --filter-pattern <필터 패턴>

# 로그 스트림이름
aws logs tail /aws/eks/$CLUSTER_NAME/cluster --log-stream-name-prefix <로그 스트림 prefix> --follow
aws logs tail /aws/eks/$CLUSTER_NAME/cluster --log-stream-name-prefix kube-controller-manager --follow
kubectl scale deployment -n kube-system coredns --replicas=1
kubectl scale deployment -n kube-system coredns --replicas=2
```

![image](https://github.com/jiwonYun9332/AWES-1/blob/7e088b622fe8025d88f44f45e4efd9db564fe6dd/Study/images/53_images.jpg)

```
# 시간 지정: 1초(s) 1분(m) 1시간(h) 하루(d) 한주(w)
aws logs tail /aws/eks/$CLUSTER_NAME/cluster --since 1h30m

# 짧게 출력
aws logs tail /aws/eks/$CLUSTER_NAME/cluster --since 1h30m --format short
```

**로그 검색**

```
# EC2 Instance가 NodeNotReady 상태인 로그 검색
fields @timestamp, @message
| filter @message like /NodeNotReady/
| sort @timestamp desc

# kube-apiserver-audit 로그에서 userAgent 정렬해서 아래 4개 필드 정보 검색
fields userAgent, requestURI, @timestamp, @message
| filter @logStream ~= "kube-apiserver-audit"
| stats count(userAgent) as count by userAgent
| sort count desc

#
fields @timestamp, @message
| filter @logStream ~= "kube-scheduler"
| sort @timestamp desc

#
fields @timestamp, @message
| filter @logStream ~= "authenticator"
| sort @timestamp desc

#
fields @timestamp, @message
| filter @logStream ~= "kube-controller-manager"
| sort @timestamp desc
```

![image](https://github.com/jiwonYun9332/AWES-1/blob/f55bba47a1d9b3a10f479f16ccf42e61a6408618/Study/images/54_images.jpg)

![image](https://github.com/jiwonYun9332/AWES-1/blob/f55bba47a1d9b3a10f479f16ccf42e61a6408618/Study/images/55_images.jpg)

**로깅 비활성화 및 삭제**

```
# EKS Control Plane 로깅(CloudWatch Logs) 비활성화
eksctl utils update-cluster-logging --cluster $CLUSTER_NAME --region $AWS_DEFAULT_REGION --disable-types all --approve

# 로그 그룹 삭제
aws logs delete-log-group --log-group-name /aws/eks/$CLUSTER_NAME/cluster
```

**컨테이너 로그 확인**

```
# NGINX 웹서버 배포
helm repo add bitnami https://charts.bitnami.com/bitnami

# 사용 리전의 인증서 ARN 확인
CERT_ARN=$(aws acm list-certificates --query 'CertificateSummaryList[].CertificateArn[]' --output text)
echo $CERT_ARN

# 도메인 확인
echo $MyDomain

# 파라미터 파일 생성
cat <<EOT > nginx-values.yaml
service:
    type: NodePort

ingress:
  enabled: true
  ingressClassName: alb
  hostname: nginx.$MyDomain
  path: /*
  annotations: 
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}, {"HTTP":80}]'
    alb.ingress.kubernetes.io/certificate-arn: $CERT_ARN
    alb.ingress.kubernetes.io/success-codes: 200-399
    alb.ingress.kubernetes.io/load-balancer-name: $CLUSTER_NAME-ingress-alb
    alb.ingress.kubernetes.io/group.name: study
    alb.ingress.kubernetes.io/ssl-redirect: '443'
EOT
cat nginx-values.yaml | yh

# 배포
helm install nginx bitnami/nginx --version 14.1.0 -f nginx-values.yaml
```

![image](https://github.com/jiwonYun9332/AWES-1/blob/e4f32a15ee2297c51a4c43de2d7b8cbd3a2b6560/Study/images/56_images.jpg)

**삭제**

```
helm uninstall nginx
```

**Container Insights metrics in Amazon CloudWatch & Fluent Bit (Logs)**

**application 로그 소스(All log files in /var/log/containers → 심볼릭 링크 /var/log/pods/<컨테이너>, 각 컨테이너/파드 로그**

```
# 로그 위치 확인

ssh ec2-user@$N1 sudo tree /var/log/containers
ssh ec2-user@$N1 sudo ls -al /var/log/containers
for node in $N1 $N2 $N3; do echo ">>>>> $node <<<<<"; ssh ec2-user@$node sudo tree /var/log/containers; echo; done
for node in $N1 $N2 $N3; do echo ">>>>> $node <<<<<"; ssh ec2-user@$node sudo ls -al /var/log/containers; echo; done

# 개별 파드 로그 확인
ssh ec2-user@$N1 sudo tail -f /var/log/pods/kube-system_aws-node-7fwbm_a618a16d-1fcf-4111-b58d-6e3f94e8d81e/aws-node/0.log

2023-05-20T16:57:52.340311538Z stdout F Installed /host/opt/cni/bin/aws-cni
2023-05-20T16:57:52.34181981Z stdout F Installed /host/opt/cni/bin/egress-v4-cni
2023-05-20T16:57:52.341863161Z stderr F time="2023-05-20T16:57:52Z" level=info msg="Starting IPAM daemon... "
2023-05-20T16:57:52.34627694Z stderr F time="2023-05-20T16:57:52Z" level=info msg="Checking for IPAM connectivity... "
2023-05-20T16:57:53.352287529Z stderr F time="2023-05-20T16:57:53Z" level=info msg="Copying config file... "
2023-05-20T16:57:53.352612604Z stderr F time="2023-05-20T16:57:53Z" level=info msg="Successfully copied CNI plugin binary and config file."
```

**host 로그 소스(Logs from /var/log/dmesg, /var/log/secure, and /var/log/messages), 노드(호스트) 로그**

```
# 로그 위치 확인
#ssh ec2-user@$N1 sudo tree /var/log/ -L 1
#ssh ec2-user@$N1 sudo ls -la /var/log/
for node in $N1 $N2 $N3; do echo ">>>>> $node <<<<<"; ssh ec2-user@$node sudo tree /var/log/ -L 1; echo; done
for node in $N1 $N2 $N3; do echo ">>>>> $node <<<<<"; ssh ec2-user@$node sudo ls -la /var/log/; echo; done

# 호스트 로그 확인
#ssh ec2-user@$N1 sudo tail /var/log/dmesg
#ssh ec2-user@$N1 sudo tail /var/log/secure
#ssh ec2-user@$N1 sudo tail /var/log/messages
for log in dmesg secure messages; do echo ">>>>> Node1: /var/log/$log <<<<<"; ssh ec2-user@$N1 sudo tail /var/log/$log; echo; done
for log in dmesg secure messages; do echo ">>>>> Node2: /var/log/$log <<<<<"; ssh ec2-user@$N2 sudo tail /var/log/$log; echo; done
for log in dmesg secure messages; do echo ">>>>> Node3: /var/log/$log <<<<<"; ssh ec2-user@$N3 sudo tail /var/log/$log; echo; done
```

**dataplane 로그 소스(/var/log/journal for kubelet.service, kubeproxy.service, and docker.service), 쿠버네티스 데이터플레인 로그**

```
# 로그 위치 확인
#ssh ec2-user@$N1 sudo tree /var/log/journal -L 1
#ssh ec2-user@$N1 sudo ls -la /var/log/journal
for node in $N1 $N2 $N3; do echo ">>>>> $node <<<<<"; ssh ec2-user@$node sudo tree /var/log/journal -L 1; echo; done

# 저널 로그 확인 - 링크
ssh ec2-user@$N3 sudo journalctl -x -n 200
ssh ec2-user@$N3 sudo journalctl -f
```

**CloudWatch Container Insight 설치**

```
# 설치
FluentBitHttpServer='On'
FluentBitHttpPort='2020'
FluentBitReadFromHead='Off'
FluentBitReadFromTail='On'
curl -s https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluent-bit-quickstart.yaml | sed 's/{{cluster_name}}/'${CLUSTER_NAME}'/;s/{{region_name}}/'${AWS_DEFAULT_REGION}'/;s/{{http_server_toggle}}/"'${FluentBitHttpServer}'"/;s/{{http_server_port}}/"'${FluentBitHttpPort}'"/;s/{{read_from_head}}/"'${FluentBitReadFromHead}'"/;s/{{read_from_tail}}/"'${FluentBitReadFromTail}'"/' | kubectl apply -f -

# 설치 확인
kubectl get-all -n amazon-cloudwatch
kubectl get ds,pod,cm,sa -n amazon-cloudwatch
kubectl describe clusterrole cloudwatch-agent-role fluent-bit-role                          # 클러스터롤 확인
kubectl describe clusterrolebindings cloudwatch-agent-role-binding fluent-bit-role-binding  # 클러스터롤 바인딩 확인
kubectl -n amazon-cloudwatch logs -l name=cloudwatch-agent -f # 파드 로그 확인
kubectl -n amazon-cloudwatch logs -l k8s-app=fluent-bit -f    # 파드 로그 확인
for node in $N1 $N2 $N3; do echo ">>>>> $node <<<<<"; ssh ec2-user@$node sudo ss -tnlp | grep fluent-bit; echo; done

# cloudwatch-agent 설정 확인
kubectl describe cm cwagentconfig -n amazon-cloudwatch
```

**CW 파드가 수집하는 방법**

![image](https://github.com/jiwonYun9332/AWES-1/blob/56d54961982fdd00a805051cc7fc995271234db9/Study/images/57_images.jpg)

**Fluent Bit Cluster Info 확인**

```
# kubectl get cm -n amazon-cloudwatch fluent-bit-cluster-info -o yaml | yh
apiVersion: v1
data:
  cluster.name: myeks
  http.port: "2020"
  http.server: "On"
  logs.region: ap-northeast-2
  read.head: "Off"
  read.tail: "On"
kind: ConfigMap
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"cluster.name":"myeks","http.port":"2020","http.server":"On","logs.region":"ap-northeast-2","read.head":"Off","read.tail":"On"},"kind":"ConfigMap","metadata":{"annotations":{},"name":"fluent-bit-cluster-info","namespace":"amazon-cloudwatch"}}
  creationTimestamp: "2023-05-20T18:28:10Z"
  name: fluent-bit-cluster-info
  namespace: amazon-cloudwatch
  resourceVersion: "24585"
  uid: 22e01683-2a8c-4ea2-82f9-5c71337afdfe
```

**Fluent Bit 로그 INPUT/FILTER/OUTPUT 설정 확인**

![image](https://github.com/jiwonYun9332/AWES-1/blob/cecf7a87a31b4573e146d5bb9d9ee41bf861f11e/Study/images/58_images.jpg)

```
# 삭제
curl -s https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluent-bit-quickstart.yaml | sed 's/{{cluster_name}}/'${CLUSTER_NAME}'/;s/{{region_name}}/'${AWS_DEFAULT_REGION}'/;s/{{http_server_toggle}}/"'${FluentBitHttpServer}'"/;s/{{http_server_port}}/"'${FluentBitHttpPort}'"/;s/{{read_from_head}}/"'${FluentBitReadFromHead}'"/;s/{{read_from_tail}}/"'${FluentBitReadFromTail}'"/' | kubectl delete -f -
```

### Metrics-server & kwatch & botkube

Metrics-server 확인 : kubelet으로부터 수집한 리소스 메트릭을 수집 및 집계하는 클러스터 애드온 구성 요소

- cAdvisor : kubelet에 포함된 컨테이너 메트릭을 수집, 집계, 노출하는 데몬

![image](https://github.com/jiwonYun9332/AWES-1/blob/66151b83acd0280a6a8cef8838cec92ed3d240b4/Study/images/59_images.jpg)

```
# 배포
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# 메트릭 서버 확인 : 메트릭은 15초 간격으로 cAdvisor를 통하여 가져옴
kubectl get pod -n kube-system -l k8s-app=metrics-server
kubectl api-resources | grep metrics
kubectl get apiservices |egrep '(AVAILABLE|metrics)'

# 노드 메트릭 확인
kubectl top node

# 파드 메트릭 확인
kubectl top pod -A
kubectl top pod -n kube-system --sort-by='cpu'
kubectl top pod -n kube-system --sort-by='memory'
```

![image](https://github.com/jiwonYun9332/AWES-1/blob/1c1d45910278d4ac007bc75b8d2131a0379c378c/Study/images/60_images.jpg)

**kwatch 설치/사용**

```
# configmap 생성
cat <<EOT > ~/kwatch-config.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: kwatch
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: kwatch
  namespace: kwatch
data:
  config.yaml: |
    alert:
      slack:
        webhook: '웹 훅 URL'
        title: $NICK-EKS
        #text:
    pvcMonitor:
      enabled: true
      interval: 5
      threshold: 70
EOT
kubectl apply -f kwatch-config.yaml

# 배포
kubectl apply -f https://raw.githubusercontent.com/abahmed/kwatch/v0.8.3/deploy/deploy.yaml
```

```
# 터미널1
watch kubectl get pod

# 잘못된 이미지 정보의 파드 배포
kubectl apply -f https://raw.githubusercontent.com/junghoon2/kube-books/main/ch05/nginx-error-pod.yml
kubectl get events -w
```

![image](https://github.com/jiwonYun9332/AWES-1/blob/7fb1051843d4946c8b887132dee089724f2a7e7d/Study/images/61_images.jpg)

![image](https://github.com/jiwonYun9332/AWES-1/blob/7fb1051843d4946c8b887132dee089724f2a7e7d/Study/images/62_images.jpg)

```
# 이미지 업데이트 방안2 : set 사용 - iamge 등 일부 리소스 값을 변경 가능!
kubectl set 
kubectl set image pod nginx-19 nginx-pod=nginx:1.19

# 삭제
kubectl delete pod nginx-19

# kwatch 삭제
kubectl delete -f https://raw.githubusercontent.com/abahmed/kwatch/v0.8.3/deploy/deploy.yaml
```

### Prometheus

: 목표대상의 상태값을 수집하고 관리하는 시스템, 상태값은 자원사용률(cpu, memory 등) 또는 애플리케이션 수치(API호출 횟수 등)를 의미한다.

쿠버네티스에서 프로메테우스를 사용한다면, 프로메테우스 오퍼레이터를 사용하는 것이 관리가 편하다.

**프로메테우스 오퍼레이터**

: 프로메테우스를 쿠버네티스 오퍼레이터패턴으로 관리한다, 프로메테우스 설치부터 설정관리까지 쿠버네티스 CRD로 관리한다.

![image](https://github.com/jiwonYun9332/AWES-1/blob/20c7b0a1b63fcca5bc03432d9a9ba6f4c95a7651/Study/images/63_images.jpg)

- Prometheus Server
  : 메트릭 수집, 저장, 처리 및 쿼리 기능을 수행한다. 메트릭 수집 방식으로 Pull 방식을 기본적으로 사용한다. 해당 서버가 대상 서비스로부터 메트릭을 주기적으로 수집하고, 시계열 데이터베이스(TSDB, HDD/SDD)에 저장한다.
- Pushgateway : Pushgateway
  : Push 방식을 사용하는 일부 유형의 메트릭을 Prometheus에서 수집하기 위한 중간 서버
- Alertmanager
  : Prometheus 서버에서 발생한 경고를 관리하고, 사용자에게 알림을 전달하는 컴포넌트
- Prometheus UI
  : 내장된 웹 인터페이스로, 사용자가 Prometheus 서버에서 메트릭을 쿼리하고, 시각화된 그래프를 확인할 수 있다.

**확장성, 고가용성 문제**

프로메테우스는 단일 노드 시스템으로 설계되어 있어 클러스터링 구조를 직접 지원하지 않는다. 이로인해 확장성과 고가용성에 일부 보완이 필요하다.

- 확장성 문제
  - 단일 노드에서 모든 메트릭을 처리하려 할 때 노드의 자원이 고갈되어 성능 저하를 초래할 수 있다.
  - 대규모 인프라에서 많은 수의 메트릭을 수집하고 처리하는 데 있어 성능 저하와 저장소 부족 문제가 발생할 수 있다. 외부 스토리지 연결이 필요하다.

- 고가용성 문제
  - 단일 노드에서 발생하는 장애나 다운타임이 생겨 프로메테우스 서버가 내려가면 그 시간 동안에는 메트릭을 수집할 수 없다.
  - 볼륨이 AWS EBS 를 사용해도 단일 노드에서만 연결이 가능하다. 연결 노드에 다운 타임이 발생하면 메트릭을 가져올 수 없다.

해결방안으로 Thanos를 사용한다.

```

**프로메테우스-스택 설치**

# 모니터링
kubectl create ns monitoring
watch kubectl get pod,pvc,svc,ingress -n monitoring

# 사용 리전의 인증서 ARN 확인
CERT_ARN=`aws acm list-certificates --query 'CertificateSummaryList[].CertificateArn[]' --output text`
echo $CERT_ARN

# repo 추가
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# 파라미터 파일 생성
cat <<EOT > monitor-values.yaml
prometheus:
  prometheusSpec:
    podMonitorSelectorNilUsesHelmValues: false
    serviceMonitorSelectorNilUsesHelmValues: false
    retention: 5d
    retentionSize: "10GiB"

  ingress:
    enabled: true
    ingressClassName: alb
    hosts: 
      - prometheus.$MyDomain
    paths: 
      - /*
    annotations:
      alb.ingress.kubernetes.io/scheme: internet-facing
      alb.ingress.kubernetes.io/target-type: ip
      alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}, {"HTTP":80}]'
      alb.ingress.kubernetes.io/certificate-arn: $CERT_ARN
      alb.ingress.kubernetes.io/success-codes: 200-399
      alb.ingress.kubernetes.io/load-balancer-name: myeks-ingress-alb
      alb.ingress.kubernetes.io/group.name: study
      alb.ingress.kubernetes.io/ssl-redirect: '443'

grafana:
  defaultDashboardsTimezone: Asia/Seoul
  adminPassword: prom-operator

  ingress:
    enabled: true
    ingressClassName: alb
    hosts: 
      - grafana.$MyDomain
    paths: 
      - /*
    annotations:
      alb.ingress.kubernetes.io/scheme: internet-facing
      alb.ingress.kubernetes.io/target-type: ip
      alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}, {"HTTP":80}]'
      alb.ingress.kubernetes.io/certificate-arn: $CERT_ARN
      alb.ingress.kubernetes.io/success-codes: 200-399
      alb.ingress.kubernetes.io/load-balancer-name: myeks-ingress-alb
      alb.ingress.kubernetes.io/group.name: study
      alb.ingress.kubernetes.io/ssl-redirect: '443'

defaultRules:
  create: false
kubeControllerManager:
  enabled: false
kubeEtcd:
  enabled: false
kubeScheduler:
  enabled: false
alertmanager:
  enabled: false

# alertmanager:
#   ingress:
#     enabled: true
#     ingressClassName: alb
#     hosts: 
#       - alertmanager.$MyDomain
#     paths: 
#       - /*
#     annotations:
#       alb.ingress.kubernetes.io/scheme: internet-facing
#       alb.ingress.kubernetes.io/target-type: ip
#       alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}, {"HTTP":80}]'
#       alb.ingress.kubernetes.io/certificate-arn: $CERT_ARN
#       alb.ingress.kubernetes.io/success-codes: 200-399
#       alb.ingress.kubernetes.io/load-balancer-name: myeks-ingress-alb
#       alb.ingress.kubernetes.io/group.name: study
#       alb.ingress.kubernetes.io/ssl-redirect: '443'
EOT
cat monitor-values.yaml | yh

# 배포
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack --version 45.27.2 \
--set prometheus.prometheusSpec.scrapeInterval='15s' --set prometheus.prometheusSpec.evaluationInterval='15s' \
-f monitor-values.yaml --namespace monitoring

# 확인
## alertmanager-0 : 사전에 정의한 정책 기반(예: 노드 다운, 파드 Pending 등)으로 시스템 경고 메시지를 생성 후 경보 채널(슬랙 등)로 전송
## grafana : 프로메테우스는 메트릭 정보를 저장하는 용도로 사용하며, 그라파나로 시각화 처리
## prometheus-0 : 모니터링 대상이 되는 파드는 ‘exporter’라는 별도의 사이드카 형식의 파드에서 모니터링 메트릭을 노출, pull 방식으로 가져와 내부의 시계열 데이터베이스에 저장
## node-exporter : 노드익스포터는 물리 노드에 대한 자원 사용량(네트워크, 스토리지 등 전체) 정보를 메트릭 형태로 변경하여 노출
## operator : 시스템 경고 메시지 정책(prometheus rule), 애플리케이션 모니터링 대상 추가 등의 작업을 편리하게 할수 있게 CRD 지원
## kube-state-metrics : 쿠버네티스의 클러스터의 상태(kube-state)를 메트릭으로 변환하는 파드
helm list -n monitoring
kubectl get pod,svc,ingress -n monitoring
kubectl get-all -n monitoring
kubectl get prometheus,servicemonitors -n monitoring
kubectl get crd | grep monitoring
```

![image](https://github.com/jiwonYun9332/AWES-1/blob/b12caebc38088ac449b2293d622a11f1809cfe1e/Study/images/64_images.jpg)

**웹 페이지 접속**

```
echo -e "Prometheus Web URL = https://prometheus.$MyDomain"
```

![image](https://github.com/jiwonYun9332/AWES-1/blob/27ece8151238f0e533fc02e188877f05b6110015/Study/images/65_images.jpg)

**삭제**

```
# helm 삭제
helm uninstall -n monitoring kube-prometheus-stack

# crd 삭제
kubectl delete crd alertmanagerconfigs.monitoring.coreos.com
kubectl delete crd alertmanagers.monitoring.coreos.com
kubectl delete crd podmonitors.monitoring.coreos.com
kubectl delete crd probes.monitoring.coreos.com
kubectl delete crd prometheuses.monitoring.coreos.com
kubectl delete crd prometheusrules.monitoring.coreos.com
kubectl delete crd servicemonitors.monitoring.coreos.com
kubectl delete crd thanosrulers.monitoring.coreos.com
```

### 그라파나

TSDB 데이터를 시각화

```
- **[Grafana open source software](https://grafana.com/oss/)** enables you to query, visualize, alert on, and explore your metrics, logs, and traces wherever they are stored.
    - Grafana OSS provides you with tools to turn your time-series database (TSDB) data into insightful graphs and visualizations.
```

```
# 그라파나 버전 확인
kubectl exec -it -n monitoring deploy/kube-prometheus-stack-grafana -- grafana-cli --version
grafana cli version 9.5.1

# ingress 확인
kubectl get ingress -n monitoring kube-prometheus-stack-grafana
kubectl describe ingress -n monitoring kube-prometheus-stack-grafana

# ingress 도메인으로 웹 접속 : 기본 계정 - admin / prom-operator
echo -e "Grafana Web URL = https://grafana.$MyDomain"
```

![image](https://github.com/jiwonYun9332/AWES-1/blob/660dac94fcc0b0899f408fe36d1d9908f45573bc/Study/images/66_images.jpg)

![image](https://github.com/jiwonYun9332/AWES-1/blob/660dac94fcc0b0899f408fe36d1d9908f45573bc/Study/images/67_images.jpg)

```
# 서비스 주소 확인
kubectl get svc,ep -n monitoring kube-prometheus-stack-prometheus
NAME                                       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/kube-prometheus-stack-prometheus   ClusterIP   10.100.253.162   <none>        9090/TCP   7m3s

NAME                                         ENDPOINTS            AGE
endpoints/kube-prometheus-stack-prometheus   192.168.2.123:9090   7m3s
```

```
# 테스트용 파드 배포
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: netshoot-pod
spec:
  containers:
  - name: netshoot-pod
    image: nicolaka/netshoot
    command: ["tail"]
    args: ["-f", "/dev/null"]
  terminationGracePeriodSeconds: 0
EOF
kubectl get pod netshoot-pod

# 접속 확인
kubectl exec -it netshoot-pod -- nslookup kube-prometheus-stack-prometheus.monitoring
kubectl exec -it netshoot-pod -- curl -s kube-prometheus-stack-prometheus.monitoring:9090/graph -v ; echo

# 삭제
kubectl delete pod netshoot-pod
```

![image](https://github.com/jiwonYun9332/AWES-1/blob/5869da8f979ce602b0dfc1da88569e7ec527a36b/Study/images/68_images.jpg)












