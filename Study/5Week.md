# 5Week - 5주차 실습

![image](https://github.com/jiwonYun9332/AWES-1/blob/c520ba96f63ae4030ef34f0e5ea5faed858c8811/Study/images/71_images.jpg)

### EKS Node Viewer 설치 

```
# go 설치
yum install -y go

# EKS Node Viewer 설치 : 현재 ec2 spec에서는 설치에 다소 시간이 소요됨 = 2분 이상
go install github.com/awslabs/eks-node-viewer/cmd/eks-node-viewer@latest

# bin 확인 및 사용 
tree ~/go/bin
cd ~/go/bin
./eks-node-viewer

tree ~/go/bin
/root/go/bin
└── eks-node-viewer

cd ~/go/bin

0 directories, 1 file
3 nodes (875m/5790m) 15.1% cpu ██████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ $0.156/hour | $1
20 pods (0 pending 20 running 20 bound)

ip-192-168-2-70.ap-northeast-2.compute.internal  cpu ██████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
ip-192-168-1-166.ap-northeast-2.compute.internal cpu ████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
ip-192-168-3-26.ap-northeast-2.compute.internal  cpu ██████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
Press any key to quit

./eks-node-viewer --resources cpu,memory
3 nodes (875m/5790m)     15.1% cpu    ██████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ $0.156/ho
        390Mi/10165092Ki 3.9%  memory ██░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
20 pods (0 pending 20 running 20 bound)

ip-192-168-2-70.ap-northeast-2.compute.internal  cpu    ██████░░░░░░░░░░░░░░░░░░░░░░░░░░
                                                 memory █░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
ip-192-168-1-166.ap-northeast-2.compute.internal cpu    ████░░░░░░░░░░░░░░░░░░░░░░░░░░░░
                                                 memory █░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
ip-192-168-3-26.ap-northeast-2.compute.internal  cpu    ██████░░░░░░░░░░░░░░░░░░░░░░░░░░
                                                 memory ███░░░░░░░░░░░░░░░░░░░░░░░░░░░░░                                      

# Karenter nodes only
./eks-node-viewer --node-selector "karpenter.sh/provisioner-name"

# Display extra labels, i.e. AZ
./eks-node-viewer --extra-labels topology.kubernetes.io/zone

# Specify a particular AWS profile and region
AWS_PROFILE=myprofile AWS_REGION=us-west-2

기본 옵션
# select only Karpenter managed nodes
node-selector=karpenter.sh/provisioner-name

# display both CPU and memory
resources=cpu,memory
```

![image](https://github.com/jiwonYun9332/AWES-1/blob/2db1cea451b645ca3603c44c535d13fb450bbedc/Study/images/72_images.jpg)

![image](https://github.com/jiwonYun9332/AWES-1/blob/2db1cea451b645ca3603c44c535d13fb450bbedc/Study/images/73_images.jpg)

![image](https://github.com/jiwonYun9332/AWES-1/blob/2db1cea451b645ca3603c44c535d13fb450bbedc/Study/images/74_images.jpg)

![image](https://github.com/jiwonYun9332/AWES-1/blob/2db1cea451b645ca3603c44c535d13fb450bbedc/Study/images/75_images.jpg)

![image](https://github.com/jiwonYun9332/AWES-1/blob/2db1cea451b645ca3603c44c535d13fb450bbedc/Study/images/76_images.jpg)

![image](https://github.com/jiwonYun9332/AWES-1/blob/2db1cea451b645ca3603c44c535d13fb450bbedc/Study/images/77_images.jpg)

![image](https://github.com/jiwonYun9332/AWES-1/blob/2db1cea451b645ca3603c44c535d13fb450bbedc/Study/images/78_images.jpg)

![image](https://github.com/jiwonYun9332/AWES-1/blob/2db1cea451b645ca3603c44c535d13fb450bbedc/Study/images/79_images.jpg)

![image](https://github.com/jiwonYun9332/AWES-1/blob/2db1cea451b645ca3603c44c535d13fb450bbedc/Study/images/80_images.jpg)

### HPA - Horizontal Pod Autoscaler

```
# Run and expose php-apache server
curl -s -O https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/application/php-apache.yaml
cat php-apache.yaml | yh
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  selector:
    matchLabels:
      run: php-apache
  template:
    metadata:
      labels:
        run: php-apache
    spec:
      containers:
      - name: php-apache
        image: registry.k8s.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 500m
          requests:
            cpu: 200m
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
  labels:
    run: php-apache
spec:
  ports:
  - port: 80
  selector:
    run: php-apache

kubectl apply -f php-apache.yaml

# 확인
kubectl exec -it deploy/php-apache -- cat /var/www/html/index.php

# 모니터링 : 터미널2개 사용
watch -d 'kubectl get hpa,pod;echo;kubectl top pod;echo;kubectl top node'
kubectl exec -it deploy/php-apache -- top

# 접속
PODIP=$(kubectl get pod -l run=php-apache -o jsonpath={.items[0].status.podIP})
curl -s $PODIP; echo
```

```
# Create the HorizontalPodAutoscaler : requests.cpu=200m - 알고리즘
# Since each pod requests 200 milli-cores by kubectl run, this means an average CPU usage of 100 milli-cores.
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
kubectl describe hpa

# HPA 설정 확인
kubectl krew install neat
kubectl get hpa php-apache -o yaml
kubectl get hpa php-apache -o yaml | kubectl neat | yh
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
  namespace: default
spec:
  maxReplicas: 10
  metrics:
  - resource:
      name: cpu
      target:
        averageUtilization: 50
        type: Utilization
    type: Resource
  minReplicas: 1
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
    
# 반복 접속 1 (파드1 IP로 접속) >> 증가 확인 후 중지
while true;do curl -s $PODIP; sleep 0.5; done

```bash
# Create the HorizontalPodAutoscaler : requests.cpu=200m - [알고리즘](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#algorithm-details)
# Since each pod requests **200 milli-cores** by kubectl run, this means an average CPU usage of **100 milli-cores**.
kubectl autoscale deployment php-apache **--cpu-percent=50** --min=1 --max=10
**kubectl describe hpa**
...
Metrics:                                               ( current / target )
  resource cpu on pods  (as a percentage of request):  0% (1m) / 50%
Min replicas:                                          1
Max replicas:                                          10
Deployment pods:                                       1 current / 1 desired
...

# HPA 설정 확인
kubectl krew install neat
kubectl get hpa php-apache -o yaml
**kubectl get hpa php-apache -o yaml | kubectl neat | yh**
spec: 
  minReplicas: 1               # [4] 또는 최소 1개까지 줄어들 수도 있습니다
  maxReplicas: 10              # [3] 포드를 최대 5개까지 늘립니다
  **scaleTargetRef**: 
    apiVersion: apps/v1
    kind: **Deployment**
    name: **php-apache**           # [1] php-apache 의 자원 사용량에서
  **metrics**: 
  - type: **Resource**
    resource: 
      name: **cpu**
      target: 
        type: **Utilization**
        **averageUtilization**: 50  # [2] CPU 활용률이 50% 이상인 경우

# 반복 접속 1 (파드1 IP로 접속) >> 증가 확인 후 중지
while true;do curl -s $PODIP; sleep 0.5; done

# 반복 접속 2 (서비스명 도메인으로 접속) >> 증가 확인(몇개까지 증가되는가? 그 이유는?) 후 중지 >> **중지 5분 후** 파드 갯수 감소 확인
# Run this in a separate terminal
# so that the load generation continues and you can carry on with the rest of the steps
kubectl run -i --tty load-generator --rm --image=**busybox**:1.28 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"
```

**부하 테스트 전**

![image](https://github.com/jiwonYun9332/AWES-1/blob/962e07082ea2865cc1010028c34bd14e16972d21/Study/images/81_images.jpg)

**부하 테스트 후**

![image](https://github.com/jiwonYun9332/AWES-1/blob/962e07082ea2865cc1010028c34bd14e16972d21/Study/images/82_images.jpg)

**부하 테스트 종료**

![image](https://github.com/jiwonYun9332/AWES-1/blob/962e07082ea2865cc1010028c34bd14e16972d21/Study/images/83_images.jpg)

**7개까지 파드가 증가하다, 종료 후 1개까지 감소되었다.**

![image](https://github.com/jiwonYun9332/AWES-1/blob/182a6b3088db5757958ceb2eab2719835f24b3ee/Study/images/84_images.jpg)

```
오브젝트 삭제: kubectl delete deploy,svc,hpa,pod --all
```

### KEDA - Kubernetes based Event Driven Autoscaler

KEDA AutoScaler

: 기존의 HPA(Horizontal Pod Autoscaler)는 리소스(CPU, Memory) 메트릭을 기반으로 스케일 여부를 결정하게 된다.
반면에 KEDA는 특정 이벤트를 기반으로 스케일 여부를 결정할 수 있다.

![image](https://github.com/jiwonYun9332/AWES-1/blob/e05f68c9731071a9a3c958b5a01802be6d83c006/Study/images/85_images.jpg)

```
# KEDA 설치
cat <<EOT > keda-values.yaml
metricsServer:
  useHostNetwork: true

prometheus:
  metricServer:
    enabled: true
    port: 9022
    portName: metrics
    path: /metrics
    serviceMonitor:
      # Enables ServiceMonitor creation for the Prometheus Operator
      enabled: true
    podMonitor:
      # Enables PodMonitor creation for the Prometheus Operator
      enabled: true
  operator:
    enabled: true
    port: 8080
    serviceMonitor:
      # Enables ServiceMonitor creation for the Prometheus Operator
      enabled: true
    podMonitor:
      # Enables PodMonitor creation for the Prometheus Operator
      enabled: true

  webhooks:
    enabled: true
    port: 8080
    serviceMonitor:
      # Enables ServiceMonitor creation for the Prometheus webhooks
      enabled: true
EOT

kubectl create namespace keda
helm repo add kedacore https://kedacore.github.io/charts
helm install keda kedacore/keda --version 2.10.2 --namespace keda -f keda-values.yaml

# KEDA 설치 확인
kubectl get-all -n keda
kubectl get all -n keda
kubectl get crd | grep keda
clustertriggerauthentications.keda.sh        2023-05-27T21:30:34Z
scaledjobs.keda.sh                           2023-05-27T21:30:34Z
scaledobjects.keda.sh                        2023-05-27T21:30:34Z
triggerauthentications.keda.sh               2023-05-27T21:30:34Z

# keda 네임스페이스에 디플로이먼트 생성
kubectl apply -f php-apache.yaml -n keda
kubectl get pod -n keda

# ScaledObject 정책 생성 : cron
cat <<EOT > keda-cron.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: php-apache-cron-scaled
spec:
  minReplicaCount: 0
  maxReplicaCount: 2
  pollingInterval: 30
  cooldownPeriod: 300
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  triggers:
  - type: cron
    metadata:
      timezone: Asia/Seoul
      start: 00,15,30,45 * * * *
      end: 05,20,35,50 * * * *
      desiredReplicas: "1"
EOT
kubectl apply -f keda-cron.yaml -n keda

# 그라파나 대시보드 추가
# 모니터링
watch -d 'kubectl get ScaledObject,hpa,pod -n keda'
NAME                                          SCALETARGETKIND      SCALETARGETNAME   MIN
   MAX   TRIGGERS   AUTHENTICATION   READY   ACTIVE   FALLBACK   AGE
scaledobject.keda.sh/php-apache-cron-scaled   apps/v1.Deployment   php-apache        0
   2     cron                        True    False    Unknown    32s

NAME                                                                  REFERENCE
      TARGETS             MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/keda-hpa-php-apache-cron-scaled   Deployment/php-apa
che   <unknown>/1 (avg)   1         2         0          32s

NAME                                                   READY   STATUS    RESTARTS
 AGE
pod/keda-admission-webhooks-68cf687cbf-9vmx2           1/1     Running   0
 6m35s
pod/keda-operator-656478d687-s75jp                     1/1     Running   1 (6m24s ago)
 6m35s
pod/keda-operator-metrics-apiserver-7fd585f657-7kqh8   1/1     Running   0
 6m35s

kubectl get ScaledObject -w

# 확인
kubectl get ScaledObject,hpa,pod -n keda
kubectl get hpa -o jsonpath={.items[0].spec} -n keda | jq
...
"metrics": [
    {
      "external": {
        "metric": {
          "name": "s0-cron-Asia-Seoul-00,15,30,45xxxx-05,20,35,50xxxx",
          "selector": {
            "matchLabels": {
              "scaledobject.keda.sh/name": "php-apache-cron-scaled"
            }
          }
        },
        "target": {
          "averageValue": "1",
          "type": "AverageValue"
        }
      },
      "type": "External"
    }

# KEDA 및 deployment 등 삭제
kubectl delete -f keda-cron.yaml -n keda && kubectl delete deploy php-apache -n keda && helm uninstall keda -n keda
kubectl delete namespace keda
```

### VPA - Vertical Pod Autoscaler


```
# 코드 다운로드
git clone https://github.com/kubernetes/autoscaler.git
cd ~/autoscaler/vertical-pod-autoscaler/
tree hack

# openssl 버전 확인
openssl version
OpenSSL 1.0.2k-fips  26 Jan 2017

# openssl 1.1.1 이상 버전 확인
yum install openssl11 -y
openssl11 version
OpenSSL 1.1.1g FIPS  21 Apr 2020

# 스크립트파일내에 openssl11 수정
sed -i 's/openssl/openssl11/g' ~/autoscaler/vertical-pod-autoscaler/pkg/admission-controller/gencerts.sh

# Deploy the Vertical Pod Autoscaler to your cluster with the following command.
watch -d kubectl get pod -n kube-system
cat hack/vpa-up.sh
./hack/vpa-up.sh
kubectl get crd | grep autoscaling
```

**공식 예제 : pod가 실행되면 약 2~3분 뒤에 pod resource.reqeust가 VPA에 의해 수정**

```
# 모니터링
watch -d kubectl top pod

# 공식 예제 배포
cd ~/autoscaler/vertical-pod-autoscaler/
cat examples/hamster.yaml | yh
---
apiVersion: "autoscaling.k8s.io/v1"
kind: VerticalPodAutoscaler
metadata:
  name: hamster-vpa
spec:



  # recommenders
  #   - name 'alternative'
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: hamster
  resourcePolicy:
    containerPolicies:
      - containerName: '*'
        minAllowed:
          cpu: 100m
          memory: 50Mi
        maxAllowed:
          cpu: 1
          memory: 500Mi
        controlledResources: ["cpu", "memory"]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hamster
spec:
  selector:
    matchLabels:
      app: hamster
  replicas: 2
  template:
    metadata:
      labels:
        app: hamster
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534 # nobody
      containers:
        - name: hamster
          image: registry.k8s.io/ubuntu-slim:0.1
          resources:
            requests:
              cpu: 100m
              memory: 50Mi
          command: ["/bin/sh"]
          args:
            - "-c"
            - "while true; do timeout 0.5s yes >/dev/null; sleep 0.5s; done"

kubectl apply -f examples/hamster.yaml && kubectl get vpa -w

# 파드 리소스 Requestes 확인
kubectl describe pod | grep Requests: -A2
    Requests:
      cpu:        100m
      memory:     50Mi
--
    Requests:
      cpu:        100m
      memory:     50Mi
      
# VPA에 의해 기존 파드 삭제되고 신규 파드가 생성됨
kubectl get events --sort-by=".metadata.creationTimestamp" | grep VPA

# 삭제
kubectl delete -f examples/hamster.yaml && cd ~/autoscaler/vertical-pod-autoscaler/ && ./hack/vpa-down.sh
```

### CA - Cluster Autoscaler

설정 전 확인

```
aws ec2 describe-instances  --filters Name=tag:Name,Values=$CLUSTER_NAME-ng1-Node --query "Reservations[*].Instances[*].Tags[*]" --output yaml | yh

- Key: k8s.io/cluster-autoscaler/myeks
      Value: owned
- Key: k8s.io/cluster-autoscaler/enabled
      Value: 'true'

# 현재 autoscaling(ASG) 정보 확인
# aws autoscaling describe-auto-scaling-groups --query "AutoScalingGroups[? Tags[? (Key=='eks:cluster-name') && Value=='클러스터이름']].[AutoScalingGroupName, MinSize, MaxSize,DesiredCapacity]" --output table
aws autoscaling describe-auto-scaling-groups \
    --query "AutoScalingGroups[? Tags[? (Key=='eks:cluster-name') && Value=='myeks']].[AutoScalingGroupName, MinSize, MaxSize,DesiredCapacity]" \
    --output table
-----------------------------------------------------------------
|                   DescribeAutoScalingGroups                   |
+------------------------------------------------+----+----+----+
|  eks-ng1-7ec42f6c-6f37-7553-7a87-2d033db4653d  |  3 |  3 |  3 |
+------------------------------------------------+----+----+----+

# MaxSize 6개로 수정
export ASG_NAME=$(aws autoscaling describe-auto-scaling-groups --query "AutoScalingGroups[? Tags[? (Key=='eks:cluster-name') && Value=='myeks']].AutoScalingGroupName" --output text)
aws autoscaling update-auto-scaling-group --auto-scaling-group-name ${ASG_NAME} --min-size 3 --desired-capacity 3 --max-size 6

# 확인
aws autoscaling describe-auto-scaling-groups --query "AutoScalingGroups[? Tags[? (Key=='eks:cluster-name') && Value=='myeks']].[AutoScalingGroupName, MinSize, MaxSize,DesiredCapacity]" --output table
-----------------------------------------------------------------
|                   DescribeAutoScalingGroups                   |
+------------------------------------------------+----+----+----+
|  eks-ng1-7ec42f6c-6f37-7553-7a87-2d033db4653d  |  3 |  6 |  3 |
+------------------------------------------------+----+----+----+

# 배포 : Deploy the Cluster Autoscaler (CA)
curl -s -O https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml
sed -i "s/<YOUR CLUSTER NAME>/$CLUSTER_NAME/g" cluster-autoscaler-autodiscover.yaml
kubectl apply -f cluster-autoscaler-autodiscover.yaml

# 확인
kubectl get pod -n kube-system | grep cluster-autoscaler
cluster-autoscaler-74785c8d45-pm55l             0/1     ContainerCreating   0          5

Name:                   cluster-autoscaler
Namespace:              kube-system
CreationTimestamp:      Sun, 28 May 2023 06:57:35 +0900
Labels:                 app=cluster-autoscaler
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=cluster-autoscaler
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:           app=cluster-autoscaler
  Annotations:      prometheus.io/port: 8085
                    prometheus.io/scrape: true
  Service Account:  cluster-autoscaler
  Containers:
   cluster-autoscaler:
    Image:      registry.k8s.io/autoscaling/cluster-autoscaler:v1.26.2
    Port:       <none>
    Host Port:  <none>
    Command:
      ./cluster-autoscaler
      --v=4
      --stderrthreshold=info
      --cloud-provider=aws
      --skip-nodes-with-local-storage=false
      --expander=least-waste
      --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/myeks
    Limits:
      cpu:     100m
      memory:  600Mi
    Requests:
      cpu:        100m
      memory:     600Mi
    Environment:  <none>
    Mounts:
      /etc/ssl/certs/ca-certificates.crt from ssl-certs (ro)
  Volumes:
   ssl-certs:
    Type:               HostPath (bare host directory volume)
    Path:               /etc/ssl/certs/ca-bundle.crt
    HostPathType:
  Priority Class Name:  system-cluster-critical
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   cluster-autoscaler-74785c8d45 (1/1 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  24s   deployment-controller  Scaled up replica set cluster-autoscaler-74785c8d45 to 1

# (옵션) cluster-autoscaler 파드가 동작하는 워커 노드가 퇴출(evict) 되지 않게 설정
```

**SCALE A CLUSTER WITH Cluster Autoscaler(CA)**

```
# 모니터링 
kubectl get nodes -w

while true; do kubectl get node; echo "------------------------------" ; date ; sleep 1; done
Sun May 28 06:59:52 KST 2023
NAME                                               STATUS   ROLES    AGE    VERSION
ip-192-168-1-166.ap-northeast-2.compute.internal   Ready    <none>   104m   v1.24.13-eks-0a21954
ip-192-168-2-70.ap-northeast-2.compute.internal    Ready    <none>   104m   v1.24.13-eks-0a21954
ip-192-168-3-26.ap-northeast-2.compute.internal    Ready    <none>   104m   v1.24.13-eks-0a21954

while true; do aws ec2 describe-instances --query "Reservations[*].Instances[*].{PrivateIPAdd:PrivateIpAddress,InstanceName:Tags[?Key=='Name']|[0].Value,Status:State.Name}" --filters Name=instance-state-name,Values=running --output text ; echo "------------------------------"; date; sleep 1; done
Sun May 28 07:00:15 KST 2023
myeks-ng1-Node  192.168.3.26    running
myeks-ng1-Node  192.168.2.70    running
myeks-bastion-EC2       192.168.1.100   running
myeks-ng1-Node  192.168.1.166   running

# Deploy a Sample App
# We will deploy an sample nginx application as a ReplicaSet of 1 Pod
cat <<EoF> nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-to-scaleout
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        service: nginx
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx-to-scaleout
        resources:
          limits:
            cpu: 500m
            memory: 512Mi
          requests:
            cpu: 500m
            memory: 512Mi
EoF

kubectl apply -f nginx.yaml
kubectl get deployment/nginx-to-scaleout

# Scale our ReplicaSet
# Let’s scale out the replicaset to 15
kubectl scale --replicas=15 deployment/nginx-to-scaleout && date

# 확인
kubectl get pods -l app=nginx -o wide --watch
kubectl -n kube-system logs -f deployment/cluster-autoscaler

# 노드 자동 증가 확인
kubectl get nodes
aws autoscaling describe-auto-scaling-groups \
    --query "AutoScalingGroups[? Tags[? (Key=='eks:cluster-name') && Value=='myeks']].[AutoScalingGroupName, MinSize, MaxSize,DesiredCapacity]" \
    --output table
-----------------------------------------------------------------
|                   DescribeAutoScalingGroups                   |
+------------------------------------------------+----+----+----+
|  eks-ng1-7ec42f6c-6f37-7553-7a87-2d033db4653d  |  3 |  6 |  5 |
+------------------------------------------------+----+----+----+

./eks-node-viewer
5 nodes (5925m/9650m) 61.4% cpu █████████████████████████░░░░░░░░░░░░░░░ $0.208/hour | $
43 pods (12 pending 31 running 37 bound)

ip-192-168-2-70.ap-northeast-2.compute.internal  cpu █████████████████████████████████░░
ip-192-168-1-166.ap-northeast-2.compute.internal cpu ███████████████████████████████████
ip-192-168-3-26.ap-northeast-2.compute.internal  cpu ███████████████████████████████████
ip-192-168-3-157.ap-northeast-2.compute.internal cpu ██░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
ip-192-168-2-176.ap-northeast-2.compute.internal cpu ██░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░

# 디플로이먼트 삭제
kubectl delete -f nginx.yaml && date

# 노드 갯수 축소 : 기본은 10분 후 scale down 됨, 물론 아래 flag 로 시간 수정 가능 >> 그러니 디플로이먼트 삭제 후 10분 기다리고 나서 보자!
# By default, cluster autoscaler will wait 10 minutes between scale down operations, 
# you can adjust this using the --scale-down-delay-after-add, --scale-down-delay-after-delete, 
# and --scale-down-delay-after-failure flag. 
# E.g. --scale-down-delay-after-add=5m to decrease the scale down delay to 5 minutes after a node has been added.

# 터미널1
watch -d kubectl get node
```

**리소스 삭제**

```
위 실습 중 디플로이먼트 삭제 후 10분 후 노드 갯수 축소되는 것을 확인 후 아래 삭제를 해보자! >> 만약 바로 아래 CA 삭제 시 워커 노드는 4개 상태가 되어서 수동으로 2대 변경 하자!
kubectl delete -f nginx.yaml

# size 수정 
aws autoscaling update-auto-scaling-group --auto-scaling-group-name ${ASG_NAME} --min-size 3 --desired-capacity 3 --max-size 3
aws autoscaling describe-auto-scaling-groups --query "AutoScalingGroups[? Tags[? (Key=='eks:cluster-name') && Value=='myeks']].[AutoScalingGroupName, MinSize, MaxSize,DesiredCapacity]" --output table

# Cluster Autoscaler 삭제
kubectl delete -f cluster-autoscaler-autodiscover.yaml
```


### CPA - Cluster Proportional Autoscaler

노드 수 증가에 비례하여 성능 처리가 필요한 애플리케이션(컨테이너/파드)를 수평으로 자동 확장

![image](https://github.com/jiwonYun9332/AWES-1/blob/3694b4842839a7346ce3a1c40b7c50c73655414e/Study/images/86_images.jpg)

```
#
helm repo add cluster-proportional-autoscaler https://kubernetes-sigs.github.io/cluster-proportional-autoscaler

# CPA규칙을 설정하고 helm차트를 릴리즈 필요
helm upgrade --install cluster-proportional-autoscaler cluster-proportional-autoscaler/cluster-proportional-autoscaler

# nginx 디플로이먼트 배포
cat <<EOT > cpa-nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        resources:
          limits:
            cpu: "100m"
            memory: "64Mi"
          requests:
            cpu: "100m"
            memory: "64Mi"
        ports:
        - containerPort: 80
EOT
kubectl apply -f cpa-nginx.yaml

# CPA 규칙 설정
cat <<EOF > cpa-values.yaml
config:
  ladder:
    nodesToReplicas:
      - [1, 1]
      - [2, 2]
      - [3, 3]
      - [4, 3]
      - [5, 5]
options:
  namespace: default
  target: "deployment/nginx-deployment"
EOF

# 모니터링
watch -d kubectl get pod

# helm 업그레이드
helm upgrade --install cluster-proportional-autoscaler -f cpa-values.yaml cluster-proportional-autoscaler/cluster-proportional-autoscaler

# 노드 5개로 증가
export ASG_NAME=$(aws autoscaling describe-auto-scaling-groups --query "AutoScalingGroups[? Tags[? (Key=='eks:cluster-name') && Value=='myeks']].AutoScalingGroupName" --output text)
aws autoscaling update-auto-scaling-group --auto-scaling-group-name ${ASG_NAME} --min-size 5 --desired-capacity 5 --max-size 5
aws autoscaling describe-auto-scaling-groups --query "AutoScalingGroups[? Tags[? (Key=='eks:cluster-name') && Value=='myeks']].[AutoScalingGroupName, MinSize, MaxSize,DesiredCapacity]" --output table

-----------------------------------------------------------------
|                   DescribeAutoScalingGroups                   |
+------------------------------------------------+----+----+----+
|  eks-ng1-7ec42f6c-6f37-7553-7a87-2d033db4653d  |  5 |  5 |  5 |
+------------------------------------------------+----+----+----+

# 노드 4개로 축소
aws autoscaling update-auto-scaling-group --auto-scaling-group-name ${ASG_NAME} --min-size 4 --desired-capacity 4 --max-size 4
aws autoscaling describe-auto-scaling-groups --query "AutoScalingGroups[? Tags[? (Key=='eks:cluster-name') && Value=='myeks']].[AutoScalingGroupName, MinSize, MaxSize,DesiredCapacity]" --output table

-----------------------------------------------------------------
|                   DescribeAutoScalingGroups                   |
+------------------------------------------------+----+----+----+
|  eks-ng1-7ec42f6c-6f37-7553-7a87-2d033db4653d  |  4 |  4 |  4 |
+------------------------------------------------+----+----+----+

# 삭제
helm uninstall cluster-proportional-autoscaler && kubectl delete -f cpa-nginx.yaml

```

**Helm Chart 삭제**

```
helm uninstall -n kube-system kube-ops-view
helm uninstall -n monitoring kube-prometheus-stack
```

**삭제**

```
eksctl delete cluster --name $CLUSTER_NAME && aws cloudformation delete-stack --stack-name $CLUSTER_NAME
```

### Karpenter

![image](https://github.com/jiwonYun9332/AWES-1/blob/60e0c6773ce6235be763ae8bd13a97b27dd1bf15/Study/images/92_image.jpg)

카펜터란?

오픈소스 고성능 Kubernetes 클러스터 오토스케일러이다.

![image](https://github.com/jiwonYun9332/AWES-1/blob/60e0c6773ce6235be763ae8bd13a97b27dd1bf15/Study/images/87_image.jpg)

블랙프라이데이와 같은 특정 기점에 트래픽이 기하급수적으로 상승할 때 빠르게 서버가 증설되어야 한다.

스케일 인/아웃 모두 빠르면서 서비스에 적합한 인스턴스를 비용 효율적으로 운영해야 한다. 이때 사용하는 오토 스케일러를 카펜터로 사용하면 몇 초만에 컴퓨팅 리소스를 제공한다.

![image](https://github.com/jiwonYun9332/AWES-1/blob/60e0c6773ce6235be763ae8bd13a97b27dd1bf15/Study/images/88_image.jpg)

**기존에 Cluster AutoScaler 문제점**

: 하나의 자원에 대해 두 군데에서 각자의 방식으로 관리, 관리 정보가 서로 동기화 되지 않아 다양한 문제가 발생한다.

![image](https://github.com/jiwonYun9332/AWES-1/blob/60e0c6773ce6235be763ae8bd13a97b27dd1bf15/Study/images/89_image.jpg)

Node를 삭제 후 aws의 인스턴스는 삭제되지 않거나

![image](https://github.com/jiwonYun9332/AWES-1/blob/60e0c6773ce6235be763ae8bd13a97b27dd1bf15/Study/images/90_image.jpg)

삭제하고 싶은 노드가 삭제되지 않고, AWS의 우선순위가 제일 높은 노드가 삭제된다.

위 방법을 해결하기 위해서는 삭제 대상의 노드를 drain 시키고, asg에서 해당 노드보다 우선순위가 높은 노드에게 프로텍트 옵션을 주어야 한다.

**카펜터 동작방식**

![image](https://github.com/jiwonYun9332/AWES-1/blob/60e0c6773ce6235be763ae8bd13a97b27dd1bf15/Study/images/91_image.jpg)

카펜터는 스케줄링이 안된 Pod를 발견하면 노드를 생성하며 스케줄링 안된 노드를 발견하면 해당 노드를 제거한다. 이를 각 프로비저닝, 디프로비저닝이라고 한다.

![image](https://github.com/jiwonYun9332/AWES-1/blob/7e7354f0a7a71b68c5a17a62cc4bd2d8738d8760/Study/images/93_image.jpg)

인스턴스 타입은 가드레일 방식으로 선언 가능하다.

- 온디맨드로 띄울 지 스팟으로 띄울 지 선택 가능하다.
- 스팟으로 띄운다면 가능한 다양한 인스턴스를 쓰는 것이 유리하다.

둘 다 선언 했을 경우 최대한 가능한 Spot을 뜨게 된다. Spot이 없다면 자동으로 온디맨드로 띄워서 안전하다.

[참고링크](https://www.youtube.com/watch?v=FPlCVVrCD64&t=15s)

[참고링크-2](https://file.notion.so/f/s/dc43285b-5aed-4cf3-bfc2-509b2a67fbb4/CON405_How-to-monitor-and-reduce-your-compute-costs.pdf?id=2ea72faa-7248-4feb-8710-d143db7607a8&table=block&spaceId=a6af158e-5b0f-4e31-9d12-0d0b2805956a&expirationTimestamp=1685339450315&signature=5P8mu3L-NfMkHs2vj_mwaur8wbdPmZJCsKSyYZnCyvU&downloadName=CON405_How-to-monitor-and-reduce-your-compute-costs.pdf)


**실습**


```
# 환경변수 정보 확인
export | egrep 'ACCOUNT|AWS_|CLUSTER' | egrep -v 'SECRET|KEY'

# 환경변수 설정
export KARPENTER_VERSION=v0.27.5
export TEMPOUT=$(mktemp)
echo $KARPENTER_VERSION $CLUSTER_NAME $AWS_DEFAULT_REGION $AWS_ACCOUNT_ID $TEMPOUT

# CloudFormation 스택으로 IAM Policy, Role, EC2 Instance Profile 생성 : 3분 정도 소요
curl -fsSL https://karpenter.sh/"${KARPENTER_VERSION}"/getting-started/getting-started-with-karpenter/cloudformation.yaml  > $TEMPOUT \
&& aws cloudformation deploy \
  --stack-name "Karpenter-${CLUSTER_NAME}" \
  --template-file "${TEMPOUT}" \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides "ClusterName=${CLUSTER_NAME}"

# 클러스터 생성 : myeks2 EKS 클러스터 생성 19분 정도 소요
eksctl create cluster -f - <<EOF
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: ${CLUSTER_NAME}
  region: ${AWS_DEFAULT_REGION}
  version: "1.24"
  tags:
    karpenter.sh/discovery: ${CLUSTER_NAME}

iam:
  withOIDC: true
  serviceAccounts:
  - metadata:
      name: karpenter
      namespace: karpenter
    roleName: ${CLUSTER_NAME}-karpenter
    attachPolicyARNs:
    - arn:aws:iam::${AWS_ACCOUNT_ID}:policy/KarpenterControllerPolicy-${CLUSTER_NAME}
    roleOnly: true

iamIdentityMappings:
- arn: "arn:aws:iam::${AWS_ACCOUNT_ID}:role/KarpenterNodeRole-${CLUSTER_NAME}"
  username: system:node:{{EC2PrivateDNSName}}
  groups:
  - system:bootstrappers
  - system:nodes

managedNodeGroups:
- instanceType: m5.large
  amiFamily: AmazonLinux2
  name: ${CLUSTER_NAME}-ng
  desiredCapacity: 2
  minSize: 1
  maxSize: 10
  iam:
    withAddonPolicies:
      externalDNS: true

## Optionally run on fargate
# fargateProfiles:
# - name: karpenter
#  selectors:
#  - namespace: karpenter
EOF

# eks 배포 확인
eksctl get cluster
NAME    REGION          EKSCTL CREATED
myeks2  ap-northeast-2  True

eksctl get nodegroup --cluster $CLUSTER_NAME
CLUSTER NODEGROUP       STATUS  CREATED                 MIN SIZE        MAX SIZE        DESIRED CAPACITY        INSTANCE TYPE   IMAGE ID        ASG NAME                                                TYPE
myeks2  myeks2-ng       ACTIVE  2023-05-28T07:14:53Z    1               10              2                       m5.large        AL2_x86_64      eks-myeks2-ng-70c4309a-dac6-6da5-2742-2e1ac15ea645      managed

eksctl get iamidentitymapping --cluster $CLUSTER_NAME
ARN                                                                                             USERNAME                                GROUPS                                  ACCOUNT
arn:aws:iam::485702506058:role/KarpenterNodeRole-myeks2                                         system:node:{{EC2PrivateDNSName}}       system:bootstrappers,system:nodes
arn:aws:iam::485702506058:role/eksctl-myeks2-nodegroup-myeks2-ng-NodeInstanceRole-1HEFSZWHPVCFQ system:node:{{EC2PrivateDNSName}}       system:bootstrappers,system:nodes

eksctl get iamserviceaccount --cluster $CLUSTER_NAME
NAMESPACE       NAME            ROLE ARN
karpenter       karpenter       arn:aws:iam::485702506058:role/myeks2-karpenter
kube-system     aws-node        arn:aws:iam::485702506058:role/eksctl-myeks2-addon-iamserviceaccount-kube-s-Role1-14XMXLEPEM8QR

eksctl get addon --cluster $CLUSTER_NAME
2023-05-28 16:25:53 [ℹ]  Kubernetes version "1.24" in use by cluster "myeks2"
2023-05-28 16:25:53 [ℹ]  getting all addons

# [터미널1] eks-node-viewer
cd ~/go/bin && ./eks-node-viewer

2 nodes (450m/3860m) 11.7% cpu █████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ $0.236/hour | $172.280/month
6 pods (0 pending 6 running 6 bound)

ip-192-168-23-255.ap-northeast-2.compute.internal cpu ██████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  17% (4 pods) m5.large/$0.1180 On-Demand - Ready
ip-192-168-63-157.ap-northeast-2.compute.internal cpu ██░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░   6% (2 pods) m5.large/$0.1180 On-Demand - Ready

# k8s 확인
kubectl cluster-info
kubectl get node --label-columns=node.kubernetes.io/instance-type,eks.amazonaws.com/capacityType,topology.kubernetes.io/zone
kubectl get pod -n kube-system -owide
kubectl describe cm -n kube-system aws-auth
Name:         aws-auth
Namespace:    kube-system
Labels:       <none>
Annotations:  <none>

Data
====
mapRoles:
----
- groups:
  - system:bootstrappers
  - system:nodes
  rolearn: arn:aws:iam::485702506058:role/KarpenterNodeRole-myeks2
  username: system:node:{{EC2PrivateDNSName}}
- groups:
  - system:bootstrappers
  - system:nodes
  rolearn: arn:aws:iam::485702506058:role/eksctl-myeks2-nodegroup-myeks2-ng-NodeInstanceRole-1HEFSZWHPVCFQ
  username: system:node:{{EC2PrivateDNSName}}

mapUsers:
----
[]


BinaryData
====

Events:  <none>

# 카펜터 설치를 위한 환경 변수 설정 및 확인

# 카펜터 설치를 위한 환경 변수 설정 및 확인
export CLUSTER_ENDPOINT="$(aws eks describe-cluster --name ${CLUSTER_NAME} --query "cluster.endpoint" --output text)"
export KARPENTER_IAM_ROLE_ARN="arn:aws:iam::${AWS_ACCOUNT_ID}:role/${CLUSTER_NAME}-karpenter"
echo $CLUSTER_ENDPOINT $KARPENTER_IAM_ROLE_ARN

 EC2 Spot Fleet 사용을 위한 service-linked-role 생성 확인 : 만들어있는것을 확인하는 거라 아래 에러 출력이 정상!
# If the role has already been successfully created, you will see:
# An error occurred (InvalidInput) when calling the CreateServiceLinkedRole operation: Service role name AWSServiceRoleForEC2Spot has been taken in this account, please try a different suffix.
aws iam create-service-linked-role --aws-service-name spot.amazonaws.com || true

# docker logout : Logout of docker to perform an unauthenticated pull against the public ECR
docker logout public.ecr.aws

# karpenter 설치
helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter --version ${KARPENTER_VERSION} --namespace karpenter --create-namespace \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=${KARPENTER_IAM_ROLE_ARN} \
  --set settings.aws.clusterName=${CLUSTER_NAME} \
  --set settings.aws.defaultInstanceProfile=KarpenterNodeInstanceProfile-${CLUSTER_NAME} \
  --set settings.aws.interruptionQueueName=${CLUSTER_NAME} \
  --set controller.resources.requests.cpu=1 \
  --set controller.resources.requests.memory=1Gi \
  --set controller.resources.limits.cpu=1 \
  --set controller.resources.limits.memory=1Gi \
  --wait
  
# 확인
kubectl get-all -n karpenter
kubectl get all -n karpenter
kubectl get cm -n karpenter karpenter-global-settings -o jsonpath={.data} | jq
kubectl get crd | grep karpenter
awsnodetemplates.karpenter.k8s.aws           2023-05-28T07:29:22Z
provisioners.karpenter.sh                    2023-05-28T07:29:22Z

```

**ExternalDNS, kube-ops-view**

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
```

![image](https://github.com/jiwonYun9332/AWES-1/blob/8ad5bd0823878ff568e6e18d0b2578c38640340d/Study/images/94_image.jpg)

**Create Provisioner**

```
#
cat <<EOF | kubectl apply -f -
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: default
spec:
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["spot"]
  limits:
    resources:
      cpu: 1000
  providerRef:
    name: default
  ttlSecondsAfterEmpty: 30
---
apiVersion: karpenter.k8s.aws/v1alpha1
kind: AWSNodeTemplate
metadata:
  name: default
spec:
  subnetSelector:
    karpenter.sh/discovery: ${CLUSTER_NAME}
  securityGroupSelector:
    karpenter.sh/discovery: ${CLUSTER_NAME}
EOF

# 확인
kubectl get awsnodetemplates,provisioners
NAME                                        AGE
awsnodetemplate.karpenter.k8s.aws/default   3s

NAME                               AGE
provisioner.karpenter.sh/default   3s

```

**Add optional monitoring with Grafana**

```
# pause 파드 1개에 CPU 1개 최소 보장 할당
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inflate
spec:
  replicas: 0
  selector:
    matchLabels:
      app: inflate
  template:
    metadata:
      labels:
        app: inflate
    spec:
      terminationGracePeriodSeconds: 0
      containers:
        - name: inflate
          image: public.ecr.aws/eks-distro/kubernetes/pause:3.7
          resources:
            requests:
              cpu: 1
EOF
kubectl scale deployment inflate --replicas 5
kubectl logs -f -n karpenter -l app.kubernetes.io/name=karpenter -c controller

# 스팟 인스턴스 확인!
aws ec2 describe-spot-instance-requests --filters "Name=state,Values=active" --output table
kubectl get node -l karpenter.sh/capacity-type=spot -o jsonpath='{.items[0].metadata.labels}' | jq
kubectl get node --label-columns=eks.amazonaws.com/capacityType,karpenter.sh/capacity-type,node.kubernetes.io/instance-type
NAME                                                STATUS   ROLES    AGE   VERSION                CAPACITYTYPE   CAPACITY-TYPE   INSTANCE-TYPE
ip-192-168-23-255.ap-northeast-2.compute.internal   Ready    <none>   34m   v1.24.13-eks-0a21954   ON_DEMAND                      m5.large
ip-192-168-63-157.ap-northeast-2.compute.internal   Ready    <none>   34m   v1.24.13-eks-0a21954   ON_DEMAND                      m5.large

# Now, delete the deployment. After 30 seconds (ttlSecondsAfterEmpty), Karpenter should terminate the now empty nodes.
kubectl delete deployment inflate
kubectl logs -f -n karpenter -l app.kubernetes.io/name=karpenter -c controller
```

**Consolidation**

```
kubectl delete provisioners default
cat <<EOF | kubectl apply -f -
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: default
spec:
  consolidation:
    enabled: true
  labels:
    type: karpenter
  limits:
    resources:
      cpu: 1000
      memory: 1000Gi
  providerRef:
    name: default
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values:
        - on-demand
    - key: node.kubernetes.io/instance-type
      operator: In
      values:
        - c5.large
        - m5.large
        - m5.xlarge
EOF

#
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inflate
spec:
  replicas: 0
  selector:
    matchLabels:
      app: inflate
  template:
    metadata:
      labels:
        app: inflate
    spec:
      terminationGracePeriodSeconds: 0
      containers:
        - name: inflate
          image: public.ecr.aws/eks-distro/kubernetes/pause:3.7
          resources:
            requests:
              cpu: 1
EOF
kubectl scale deployment inflate --replicas 12
kubectl logs -f -n karpenter -l app.kubernetes.io/name=karpenter -c controller

# 인스턴스 확인
# This changes the total memory request for this deployment to around 12Gi, 
# which when adjusted to account for the roughly 600Mi reserved for the kubelet on each node means that this will fit on 2 instances of type m5.large:
kubectl get node -l type=karpenter
kubectl get node --label-columns=eks.amazonaws.com/capacityType,karpenter.sh/capacity-type
kubectl get node --label-columns=node.kubernetes.io/instance-type,topology.kubernetes.io/zone

# Next, scale the number of replicas back down to 5:
kubectl scale deployment inflate --replicas 5

kubectl logs -f -n karpenter -l app.kubernetes.io/name=karpenter -c controller
2023-05-28T07:57:10.435Z        INFO    controller.provisioner  launching machine with 3 pods requesting {"cpu":"3125m","pods":"6"} from types m5.xlarge        {"commit": "698f22f-dirty", "provisioner": "default"}
2023-05-28T07:57:10.440Z        INFO    controller.provisioner  launching machine with 3 pods requesting {"cpu":"3125m","pods":"6"} from types m5.xlarge        {"commit": "698f22f-dirty", "provisioner": "default"}
2023-05-28T07:57:10.445Z        INFO    controller.provisioner  launching machine with 3 pods requesting {"cpu":"3125m","pods":"6"} from types m5.xlarge        {"commit": "698f22f-dirty", "provisioner": "default"}
2023-05-28T07:57:10.906Z        DEBUG   controller.provisioner.cloudprovider    created launch template {"commit": "698f22f-dirty", "provisioner": "default", "launch-template-name": "karpenter.k8s.aws/12231572537772030512", "launch-template-id": "lt-0d08315f89160cb14"}
2023-05-28T07:57:12.670Z        INFO    controller.provisioner.cloudprovider    launched instance       {"commit": "698f22f-dirty", "provisioner": "default", "id": "i-06684f30e15bb44b6", "hostname": "ip-192-168-89-241.ap-northeast-2.compute.internal", "instance-type": "m5.xlarge", "zone": "ap-northeast-2c", "capacity-type": "on-demand", "capacity": {"cpu":"4","ephemeral-storage":"20Gi","memory":"15155Mi","pods":"58"}}
2023-05-28T07:57:12.671Z        INFO    controller.provisioner.cloudprovider    launched instance       {"commit": "698f22f-dirty", "provisioner": "default", "id": "i-0d3723301c4b5b276", "hostname": "ip-192-168-176-74.ap-northeast-2.compute.internal", "instance-type": "m5.xlarge", "zone": "ap-northeast-2c", "capacity-type": "on-demand", "capacity": {"cpu":"4","ephemeral-storage":"20Gi","memory":"15155Mi","pods":"58"}}
2023-05-28T07:57:12.671Z        INFO    controller.provisioner.cloudprovider    launched instance       {"commit": "698f22f-dirty", "provisioner": "default", "id": "i-0ede3e735f8ca9dcf", "hostname": "ip-192-168-182-123.ap-northeast-2.compute.internal", "instance-type": "m5.xlarge", "zone": "ap-northeast-2c", "capacity-type": "on-demand", "capacity": {"cpu":"4","ephemeral-storage":"20Gi","memory":"15155Mi","pods":"58"}}
2023-05-28T07:57:12.671Z        INFO    controller.provisioner.cloudprovider    launched instance       {"commit": "698f22f-dirty", "provisioner": "default", "id": "i-06089f3935638db46", "hostname": "ip-192-168-156-211.ap-northeast-2.compute.internal", "instance-type": "m5.xlarge", "zone": "ap-northeast-2d", "capacity-type": "on-demand", "capacity": {"cpu":"4","ephemeral-storage":"20Gi","memory":"15155Mi","pods":"58"}}
2023-05-28T07:59:32.279Z        DEBUG   controller      deleted launch template {"commit": "698f22f-dirty", "launch-template": "karpenter.k8s.aws/2612807800920218174"}
2023-05-28T07:59:32.363Z        DEBUG   controller      deleted launch template {"commit": "698f22f-dirty", "launch-template": "karpenter.k8s.aws/12231572537772030512"}
2023-05-28T07:29:32.241Z        DEBUG   controller      discovered kube dns     {"commit": "698f22f-dirty", "kube-dns-ip": "10.100.0.10"}
2023-05-28T07:29:32.242Z        DEBUG   controller      discovered version      {"commit": "698f22f-dirty", "version": "v0.27.5"}
2023/05/28 07:29:32 Registering 2 clients
2023/05/28 07:29:32 Registering 2 informer factories
2023/05/28 07:29:32 Registering 3 informers
2023/05/28 07:29:32 Registering 5 controllers
2023-05-28T07:29:32.243Z        INFO    controller      Starting server {"commit": "698f22f-dirty", "path": "/metrics", "kind": "metrics", "addr": "[::]:8080"}
2023-05-28T07:29:32.243Z        INFO    controller      Starting server {"commit": "698f22f-dirty", "kind": "health probe", "addr": "[::]:8081"}
I0528 07:29:32.344253       1 leaderelection.go:248] attempting to acquire leader lease karpenter/karpenter-leader-election...
2023-05-28T07:29:32.385Z        INFO    controller      Starting informers...   {"commit": "698f22f-dirty"}

kubectl logs -f -n karpenter -l app.kubernetes.io/name=karpenter -c controller
2023-05-28T07:29:32.241Z        DEBUG   controller      discovered kube dns     {"commit": "698f22f-dirty", "kube-dns-ip": "10.100.0.10"}
2023-05-28T07:29:32.242Z        DEBUG   controller      discovered version      {"commit": "698f22f-dirty", "version": "v0.27.5"}
2023/05/28 07:29:32 Registering 2 clients
2023/05/28 07:29:32 Registering 2 informer factories
2023/05/28 07:29:32 Registering 3 informers
2023/05/28 07:29:32 Registering 5 controllers
2023-05-28T07:29:32.243Z        INFO    controller      Starting server {"commit": "698f22f-dirty", "path": "/metrics", "kind": "metrics", "addr": "[::]:8080"}
2023-05-28T07:29:32.243Z        INFO    controller      Starting server {"commit": "698f22f-dirty", "kind": "health probe", "addr": "[::]:8081"}
I0528 07:29:32.344253       1 leaderelection.go:248] attempting to acquire leader lease karpenter/karpenter-leader-election...
2023-05-28T07:29:32.385Z        INFO    controller      Starting informers...   {"commit": "698f22f-dirty"}
2023-05-28T07:57:10.906Z        DEBUG   controller.provisioner.cloudprovider    created launch template {"commit": "698f22f-dirty", "provisioner": "default", "launch-template-name": "karpenter.k8s.aws/12231572537772030512", "launch-template-id": "lt-0d08315f89160cb14"}
2023-05-28T07:57:12.670Z        INFO    controller.provisioner.cloudprovider    launched instance       {"commit": "698f22f-dirty", "provisioner": "default", "id": "i-06684f30e15bb44b6", "hostname": "ip-192-168-89-241.ap-northeast-2.compute.internal", "instance-type": "m5.xlarge", "zone": "ap-northeast-2c", "capacity-type": "on-demand", "capacity": {"cpu":"4","ephemeral-storage":"20Gi","memory":"15155Mi","pods":"58"}}
2023-05-28T07:57:12.671Z        INFO    controller.provisioner.cloudprovider    launched instance       {"commit": "698f22f-dirty", "provisioner": "default", "id": "i-0d3723301c4b5b276", "hostname": "ip-192-168-176-74.ap-northeast-2.compute.internal", "instance-type": "m5.xlarge", "zone": "ap-northeast-2c", "capacity-type": "on-demand", "capacity": {"cpu":"4","ephemeral-storage":"20Gi","memory":"15155Mi","pods":"58"}}
2023-05-28T07:57:12.671Z        INFO    controller.provisioner.cloudprovider    launched instance       {"commit": "698f22f-dirty", "provisioner": "default", "id": "i-0ede3e735f8ca9dcf", "hostname": "ip-192-168-182-123.ap-northeast-2.compute.internal", "instance-type": "m5.xlarge", "zone": "ap-northeast-2c", "capacity-type": "on-demand", "capacity": {"cpu":"4","ephemeral-storage":"20Gi","memory":"15155Mi","pods":"58"}}
2023-05-28T07:57:12.671Z        INFO    controller.provisioner.cloudprovider    launched instance       {"commit": "698f22f-dirty", "provisioner": "default", "id": "i-06089f3935638db46", "hostname": "ip-192-168-156-211.ap-northeast-2.compute.internal", "instance-type": "m5.xlarge", "zone": "ap-northeast-2d", "capacity-type": "on-demand", "capacity": {"cpu":"4","ephemeral-storage":"20Gi","memory":"15155Mi","pods":"58"}}
2023-05-28T07:59:32.279Z        DEBUG   controller      deleted launch template {"commit": "698f22f-dirty", "launch-template": "karpenter.k8s.aws/2612807800920218174"}
2023-05-28T07:59:32.363Z        DEBUG   controller      deleted launch template {"commit": "698f22f-dirty", "launch-template": "karpenter.k8s.aws/12231572537772030512"}
2023-05-28T08:01:03.276Z        INFO    controller.deprovisioning       deprovisioning via consolidation delete, terminating 1 machines ip-192-168-182-123.ap-northeast-2.compute.internal/m5.xlarge/on-demand {"commit": "698f22f-dirty"}

# 인스턴스 확인
kubectl get node -l type=karpenter
NAME                                                 STATUS   ROLES    AGE     VERSION
ip-192-168-156-211.ap-northeast-2.compute.internal   Ready    <none>   4m33s   v1.24.13-eks-0a21954
ip-192-168-89-241.ap-northeast-2.compute.internal    Ready    <none>   4m33s   v1.24.13-eks-0a21954

kubectl get node --label-columns=eks.amazonaws.com/capacityType,karpenter.sh/capacity-type
NAME                                                 STATUS   ROLES    AGE     VERSION                CAPACITYTYPE   CAPACITY-TYPE
ip-192-168-156-211.ap-northeast-2.compute.internal   Ready    <none>   4m35s   v1.24.13-eks-0a21954                  on-demand
ip-192-168-23-255.ap-northeast-2.compute.internal    Ready    <none>   45m     v1.24.13-eks-0a21954   ON_DEMAND
ip-192-168-63-157.ap-northeast-2.compute.internal    Ready    <none>   45m     v1.24.13-eks-0a21954   ON_DEMAND
ip-192-168-89-241.ap-northeast-2.compute.internal    Ready    <none>   4m35s   v1.24.13-eks-0a21954                  on-demand

kubectl get node --label-columns=node.kubernetes.io/instance-type,topology.kubernetes.io/zone
NAME                                                 STATUS   ROLES    AGE     VERSION                INSTANCE-TYPE   ZONE
ip-192-168-156-211.ap-northeast-2.compute.internal   Ready    <none>   4m38s   v1.24.13-eks-0a21954   m5.xlarge       ap-northeast-2d
ip-192-168-23-255.ap-northeast-2.compute.internal    Ready    <none>   45m     v1.24.13-eks-0a21954   m5.large        ap-northeast-2b
ip-192-168-63-157.ap-northeast-2.compute.internal    Ready    <none>   45m     v1.24.13-eks-0a21954   m5.large        ap-northeast-2d
ip-192-168-89-241.ap-northeast-2.compute.internal    Ready    <none>   4m38s   v1.24.13-eks-0a21954   m5.xlarge       ap-northeast-2c

# 삭제
kubectl delete deployment inflate
```

**실습 리소스 삭제**

```
kubectl delete svc -n monitoring grafana
helm uninstall -n kube-system kube-ops-view
helm uninstall karpenter --namespace karpenter

# 위 삭제 완료 후 아래 삭제
aws ec2 describe-launch-templates --filters Name=tag:karpenter.k8s.aws/cluster,Values=${CLUSTER_NAME} |
    jq -r ".LaunchTemplates[].LaunchTemplateName" |
    xargs -I{} aws ec2 delete-launch-template --launch-template-name {}

# 클러스터 삭제
eksctl delete cluster --name "${CLUSTER_NAME}"

#
aws cloudformation delete-stack --stack-name "Karpenter-${CLUSTER_NAME}"

# 위 삭제 완료 후 아래 삭제
aws cloudformation delete-stack --stack-name ${CLUSTER_NAME}
```























