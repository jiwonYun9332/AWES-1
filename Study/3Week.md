# 3Week

![logo](https://github.com/jiwonYun9332/AWES-1/blob/914b358b03c0fe27f42a00e3e5f176700dd61213/Study/images/21_logo.jpg)

### 실습환경 준비
Cloudformation Deploy

```
curl -O https://s3.ap-northeast-2.amazonaws.com/cloudformation.cloudneta.net/K8S/eks-oneclick2.yaml
```

```
# Default 네임스페이스 적용
kubectl ns default

# (옵션) contect 이름 변경
NICK=jiwon
kubectl ctx
```

### EFS 마운트 포인트 확인

**EFS(Elastic File System)**

![efs](https://github.com/jiwonYun9332/AWES-1/blob/97234e719ab25910ec8b3a323a2984218b0f109a/Study/images/23_efs.png)

![path](https://github.com/jiwonYun9332/AWES-1/blob/97234e719ab25910ec8b3a323a2984218b0f109a/Study/images/24_path.png)

**Mount**

```
# mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport fs-008165f9e0d190c32.efs.ap-northeast-2.amazonaws.com:/ /mnt/myefs
# df -hT --type nfs4
# mount | grep nfs4
# echo "efs file test" > /mnt/myefs/memo.txt
# cat /mnt/myefs/memo.txt
```

![mount](https://github.com/jiwonYun9332/AWES-1/blob/10305687614f18890747521bb512a1b30bcbe2ad/Study/images/25_mount.png)

**스토리지클래스 및 CSI 노드 확인**

![csinode](https://github.com/jiwonYun9332/AWES-1/blob/28f280b487336576f0bc49bde4a056a4f0b09b2a/Study/images/26_csinode.png)

**노드 정보 확인**

![nodeinformation](https://github.com/jiwonYun9332/AWES-1/blob/28f280b487336576f0bc49bde4a056a4f0b09b2a/Study/images/27_nodeinformation.png)

**노드 IP 확인 및 PrivateIP 변수 지정**

```
N1=$(kubectl get node --label-columns=topology.kubernetes.io/zone --selector=topology.kubernetes.io/zone=ap-northeast-2a -o jsonpath={.items[0].status.addresses[0].address})
N2=$(kubectl get node --label-columns=topology.kubernetes.io/zone --selector=topology.kubernetes.io/zone=ap-northeast-2b -o jsonpath={.items[0].status.addresses[0].address})
N3=$(kubectl get node --label-columns=topology.kubernetes.io/zone --selector=topology.kubernetes.io/zone=ap-northeast-2c -o jsonpath={.items[0].status.addresses[0].address})
echo "export N1=$N1" >> /etc/profile
echo "export N2=$N2" >> /etc/profile
echo "export N3=$N3" >> /etc/profile
echo $N1, $N2, $N3
```

**노드 보안그룹 ID 확인**

```
NGSGID=$(aws ec2 describe-security-groups --filters Name=group-name,Values=*ng1* --query "SecurityGroups[*].[GroupId]" --output text)
aws ec2 authorize-security-group-ingress --group-id $NGSGID --protocol '-1' --cidr 192.168.1.100/32
```

**워커노드 SSH 접속**

```
ssh ec2-user@$N1 hostname
ssh ec2-user@$N2 hostname
ssh ec2-user@$N3 hostname
```

**노드에 툴 설치**

```
ssh ec2-user@$N1 sudo yum install links tree jq tcpdump sysstat -y
ssh ec2-user@$N2 sudo yum install links tree jq tcpdump sysstat -y
ssh ec2-user@$N3 sudo yum install links tree jq tcpdump sysstat -y
```

### AWS LB/ExternalDNS, kube-ops-view 설치

**AWS LB Controller**

```
helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=$CLUSTER_NAME \
  --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller
```

**ExternalDNS**

```
MyDomain=senidin.com
MyDnzHostedZoneId=$(aws route53 list-hosted-zones-by-name --dns-name "${MyDomain}." --query "HostedZones[0].Id" --output text)
echo $MyDomain, $MyDnzHostedZoneId
curl -s -O https://raw.githubusercontent.com/gasida/PKOS/main/aews/externaldns.yaml
MyDomain=$MyDomain MyDnzHostedZoneId=$MyDnzHostedZoneId envsubst < externaldns.yaml | kubectl apply -f -
```

**kube-ops-view**

```
helm repo add geek-cookbook https://geek-cookbook.github.io/charts/
helm install kube-ops-view geek-cookbook/kube-ops-view --version 1.2.2 --set env.TZ="Asia/Seoul" --namespace kube-system
kubectl patch svc -n kube-system kube-ops-view -p '{"spec":{"type":"LoadBalancer"}}'
kubectl annotate service kube-ops-view -n kube-system "external-dns.alpha.kubernetes.io/hostname=kubeopsview.$MyDomain"
echo -e "Kube Ops View URL = http://kubeopsview.$MyDomain:8080/#scale=1.5"
```












































