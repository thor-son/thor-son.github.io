---
layout: post
title: eks ingress-nginx 구축
subtitle: aws eks에 ingress-nginx구축하고, client source-ip 받기
categories: kubernetes
tag: [eks, aws, kubernetes, ingress]
---


> aws eks 환경에 ingress-nginx를 구축하고, source-ip를 받아 보자.  
  
  
  
aws eks에서 ingress-nginx를 설치할 경우, service type=LoadBalancer를 nlb로 구성해준다.
<a href="https://kubernetes.github.io/ingress-nginx/deploy/#aws" target="_blank">link</a>


### eks ingress-nginx 구축
<a href="https://kubernetes.github.io/ingress-nginx/deploy/#network-load-balancer-nlb" target="_blank">link</a>
  
  
**kubectl apply로 구축**  

```console
kubectl apply -f <https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.4.0/deploy/static/provider/aws/deploy.yaml>
```
  
  
**service 확인**
```console
$ kubectl get svc -n ingress-nginx

NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP                                                                          PORT(S)                      AGE                                                                              80/TCP                       4h6m
ingress-nginx-controller             LoadBalancer   10.x.x.x         
a92d5~~.elb.ap-northeast-2.amazonaws.com   80:30037/TCP,443:31998/TCP   4h24m
ingress-nginx-controller-admission   ClusterIP      10.x.x.x   <none>
```

  
   
**ingress 생성**
ingress.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-test
  namespace : ingress-nginx
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: hsson.com
    http:
      paths:
      - path: /foo
        pathType: Prefix
        backend:
          service:
            name: clusterip
            port:
              number: 80
```

```console
$ kubectl apply -f ingress.yaml
```
  
  
  
  
### 트러블슈팅
  
   
ingress.yaml 적용했지만 address 가 안나오지 않았다.

```console
$ kubectl get ingress -A

NAMESPACE       NAME           CLASS    HOSTS   ADDRESS   PORTS   AGE
ingress-nginx   ingress-test   <none>   *                 80      39m
```
  
ingress-nginx-controller pod의 로그를 보니, IngressClass 쪽의 이슈로 확인된다.

```console
$ kubectl logs ingress-nginx-controller-5cc4f7674-x6s6l -n ingress-nginx

W1007 01:50:48.650866       6 controller.go:258] ignoring ingress ingress-test in default based on annotation : ingress does not contain a valid IngressClass
I1007 01:50:48.650887       6 main.go:100] "successfully validated configuration, accepting" ingress="default/ingress-test"
I1007 01:50:48.659468       6 store.go:426] "Ignoring ingress because of error while validating ingress class" ingress="default/ingress-test" error="ingress does not contain a valid IngressClass"
```  
  
  
  
**해결방법**
  
ingress.yaml에 `ingressClassName`을 입력하고 적용  
_ingressclass는 nlb를 설치할 때, 같이 설치가 되었다._

```yaml
spec:
  ingressClassName: nginx       <-- 추가
  rules:
....
```  
  
  
  
또는
  
ingressClass 에 대한 문서를 보니, default 로 class를 지정할 수 있었다.  
<a href="https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.2/guide/ingress/ingress_class/" target="_blank">link</a>

```console
$ kubectl get ingressclasses --namespace=ingress-nginx

NAME    CONTROLLER             PARAMETERS   AGE
nginx   k8s.io/ingress-nginx   <none>       75m

$ kubectl edit ingressclasses nginx --namespace=ingress-nginx

```
  
```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"    <--- 추가
    kubectl.kubernetes.io/last-applied-configuration: |
      ...
  creationTimestamp: "2022-10-07T01:25:31Z"
...

```
  
  
![ingressClass](/assets/images/ingressClass.png)
  
  
  
  
  
ingress-contoller 의 pod log가 아래와 같이 출력이되면 ingress가 잘 설정 된 것이다.
```console
$ kubectl logs ingress-nginx-controller-5cc4f7674-x6s6l -n ingress-nginx

I1007 02:33:20.545431       6 admission.go:149] processed ingress via admission controller {testedIngressLength:1 testedIngressTime:0.058s renderingIngressLength:1 renderingIngressTime:0s admissionTime:21.7kBs testedConfigurationSize:0.058}
I1007 02:33:20.545477       6 main.go:100] "successfully validated configuration, accepting" ingress="ingress-nginx/ingress-test"
I1007 02:33:20.554901       6 store.go:430] "Found valid IngressClass" ingress="ingress-nginx/ingress-test" ingressclass="nginx"
I1007 02:33:20.555472       6 event.go:285] Event(v1.ObjectReference{Kind:"Ingress", Namespace:"ingress-nginx", Name:"ingress-test", UID:"250f71d0-db2c-4382-a5de-7da9f49c73f2", APIVersion:"networking.k8s.io/v1", ResourceVersion:"151742", FieldPath:""}): type: 'Normal' reason: 'Sync' Scheduled for sync
I1007 02:33:20.555596       6 controller.go:168] "Configuration changes detected, backend reload required"
I1007 02:33:20.640413       6 controller.go:185] "Backend successfully reloaded"
I1007 02:33:20.640663       6 event.go:285] Event(v1.ObjectReference{Kind:"Pod", Namespace:"ingress-nginx", Name:"ingress-nginx-controller-5cc4f7674-x6s6l", UID:"b368e19a-99b4-488a-83a6-707c62de6dd0", APIVersion:"v1", ResourceVersion:"142195", FieldPath:""}): type: 'Normal' reason: 'RELOAD' NGINX reload triggered due to a change in configuration
```  
  
  
  
http 요청을 해보면,
```console
curl -H "HOST:hsson.com" <http://a92~~.elb.ap-northeast-2.amazonaws.com/foo>
```  
  
  
  
client 정보출력해주는 svc 에서 아래와 같은 출력이 나왔다.
```plaintext
HEADERS RECEIVED:
accept=*/*
host=hsson.com
user-agent=curl/7.54.0
**x-forwarded-for=219.241.x.x  <-- client source ip 알맞게 됨**
x-forwarded-host=hsson.com
x-forwarded-port=80
x-forwarded-proto=http
x-forwarded-scheme=http
x-real-ip=219.241.x.x
x-request-id=f8c29d4ff3e342bfd68b478a1e8b6587
x-scheme=http
```  
