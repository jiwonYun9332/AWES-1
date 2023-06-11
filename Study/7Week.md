
# 7Week - 7주차 실습

![image](https://github.com/jiwonYun9332/AWES-1/blob/ba891951a2e3d48a6534bc1ce66dd04c3a0a98ee/Study/images/118_image.jpg)

### 1. AWS Controller for Kubernetes (ACK)

**AWS Controller for Kubernetes이란?**

```
The idea behind AWS Controllers for Kubernetes (ACK) is to enable Kubernetes users to describe the desired state of AWS resources using the Kubernetes API and configuration language.
```

![image](https://github.com/jiwonYun9332/AWES-1/blob/2f7701c25a215f9476dd2ee8714fea259f59b801/Study/images/119_image.jpg)

AWS에는 익숙하지 않은 엔지니어들을 위해 Kubernetes 안에서 AWS 리소스들을 관리할 수 있는 Controller이다.

aws 서비스 리소스를 k8s에서 직접 정의하고 사용 할 수 있다.

1. kube-apiserver -> ack-s3-controller에 요청 전달
사용자는 kubectl를 사용하여 aws s3 버킷을 생성 할 수 있다.

2. ack-s3-controller가 IRSA 권한이 있는 경우, AWS S3 API를 통해 버킷 생성
쿠버네티스 api는 ack-s3-controller에 요청을 전달하고, ack-s3-controller(IRSA)이 aws s3 api를 통해 버킷을 생성하게 된다.






### 2. Flux

**flux란?**

![image](https://github.com/jiwonYun9332/AWES-1/blob/cc7a4c70cddfbf8ccb26bc7d3d7975c53f876fbe/Study/images/120_image.jpg)

flux는 쿠버네티스를 위한 gitops 도구이다. flux는 git에 있는 쿠버네티스를 manifest를 읽고, 쿠버네티스에 manifest를 배포한다.

**flux와 argocd 비교**

flux는 argocd와 같은 역할을 하고 있어 비교대상으로 자주 언급된다.

argocd는 kustomize를 사용할 때 디폴트 설정이 부족하기 때문에, argocd configmap에서 kustomize옵션을 한땀한땀 설정해야 한다.
반면에  flux는 바로 kustomize를 사용하도록 설정이 되어 있다.

flux는 쿠버네티스 operator패턴을 사용한다. 사용자가 쿠버네티스에 배포할 내용을 flux CRD로 정의하면, flux controller가 CRD를 읽어 리소스를 쿠버네티스에 배포한다.

![image](https://github.com/jiwonYun9332/AWES-1/blob/cc7a4c70cddfbf8ccb26bc7d3d7975c53f876fbe/Study/images/121_image.jpg)

핵심 CRD는 소스(source)와 애플리케이션(application)입니다. 소스는 git, helm registry 등 manifest 주소를 설정합니다. 
애플리케이션은 helm 또는 kustomize를 사용하여 manifest를 쿠버네티스에 배포한다.

flux 와 argocd기능 비교


3. ArgoCD

4. Crossplane

5. eksdemo

