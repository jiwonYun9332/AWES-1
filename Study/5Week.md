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














