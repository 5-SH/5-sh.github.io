---
layout: post
title: MSA Deploy (4) - ArgoCD
date: 2026-05-10 21:00:00 + 0900
categories: [kubernetes]
tags: [kubernetes, msa]
mermaid: true
---

# 6. ArgoCD
Git 상태를 실제 클러스터 상태와 동일하게 구성하도록 도와주는 Kubernetes용 GitOps CD 도구이다.   
Jenkins에서 Kubernetes에 배포하는 역할까지 담당하고 있지만 ArgoCD를 사용하면 아래와 같이 역할을 나눌 수 있다.   

- Jenkins → 이미지 빌드
- Harbor → 이미지 저장
- ArgoCD → Kubernetes 배포 관리 

```
AS-IS: GitHub → Jenkins → docker build → Harbor push → kubectl apply → Kubernetes

TO-BE: GitHub → Jenkins → docker build → Harbor push → Manifest Repo 수정 → ArgoCD Sync → Kubernetes
```

## 6-1. ArgoCD 설치

1. argocd namespace를 생성

```bash
kubectl create namespace argocd
```

2. 설치 명령

Kubernetes 버전이 아래와 같이 최신이 아니다.   

```bash
[root@m-k8s product-service]# kubectl version --short
Client Version: v1.18.4
Server Version: v1.18.20
```

ArgoCD를 stable 대신 v1.18.20 k8s와 호환이 좋은 v2.4.15 버전으로 설치한다.

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.4.15/manifests/install.yaml
```

3. 설치 확인

```bash
[root@m-k8s product-service]# kubectl get pods -n argocd
NAME                                                READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                     1/1     Running   0          94s
argocd-applicationset-controller-6f64df6b67-7gd5l   1/1     Running   0          95s
argocd-dex-server-56fc55fb54-gw65v                  1/1     Running   0          95s
argocd-notifications-controller-566848f84b-g6wj4    1/1     Running   0          95s
argocd-redis-78d49dc67-zl99t                        1/1     Running   0          94s
argocd-repo-server-65f688d5c9-z7fwz                 1/1     Running   0          94s
argocd-server-74fc764598-kzgbb                      1/1     Running   0          94s
```

## 6-2. ArgoCD 접속 설정

k8s 클러스터에 구성된 MetalLB, ingress-controller를 활용해 로컬 PC에서 ArgoCD로 접속할 수 있도록 한다.   
MetalLB, ingress-controller에서 HTTPS 인증서를 설정하지 않아 HTTP로 통신하고 있다. ArgoCD를 HTTP 모드로 변경한다.   

1. argocd-server HTTP 수정

아래 명령어로 argocd-server의 deployment를 수정한다.   

```bash
kubectl edit deployment argocd-server -n argocd
```

아래와 같이 ```--insecure```를 추가하고 ```kubectl get pods -n argocd```로 pod 재시작을 확인한다.

```yaml
...
containers:
- command:
  - argocd-server
  - --insecure
...
```

2. argocd ingress 적용

아래와 같이 argocd-ingress.yaml을 작성해 ```kubectl apply -f argocd-ingress.yaml```로 적용한다.

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: argocd-ingress
  namespace: argocd

  annotations:
    kubernetes.io/ingress.class: nginx

spec:
  rules:
  - host: argocd.deploy-test.shop
    http:
      paths:
      - path: /
        backend:
          serviceName: argocd-server
          servicePort: 80
```

3. Windows hosts 파일 수정

```text
C:\Windows\System32\drivers\etc\hosts

192.168.1.202 server.deploy-test.shop
192.168.1.202 argocd.deploy-test.shop
```

4. 브라우저에서 접속 확인

<figure>
  <img src="https://i.imgur.com/ATDm0y9.png" width="100%" alt=""/>
  <p style="font-style: italic; color: gray;">ArgoCD 접속</p>
</figure>

5. 초기 비밀번호 확인

ArgoCD admin 계정의 초기 비밀번호는 아래 명령어로 확인할 수 있다.

```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
```

# 7. 모니터링
## 7-1. Prometheus

## 7-2. Grafana

# 8. LGTM 로그 수집

# 9. 그 외
1. Kubernetes 버전 업

2. 시크릿 설정 

3. HTTPS 통신 

4. DB 이중화 연결 

5. 서비스용 DB 계정 생성 

6. 젠킨스, harbor 구성 

7. HPA