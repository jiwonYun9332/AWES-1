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
```









