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


eksctl get addon --cluster ${CLUSTER_NAME}
kubectl get pod -n kube-system -l 'app in (ebs-csi-controller,ebs-csi-node)'
kubectl get csinodes













