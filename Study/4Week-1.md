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











