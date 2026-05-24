---
layout: post
title: MSA Deploy (4) - ArgoCD
date: 2026-05-14 22:30:00 + 0900
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
> ```bash
> kubectl create namespace argocd
> ```

2. 설치 명령
> Kubernetes 버전이 아래와 같이 최신이 아니다.   
> 
> ```bash
> [root@m-k8s product-service]# kubectl version --short
> Client Version: v1.18.4
> Server Version: v1.18.20
> ```
> 
> ArgoCD를 stable 대신 v1.18.20 k8s와 호환이 좋은 v2.4.15 버전으로 설치한다.
> 
> ```bash
> kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.4.15/manifests/install.yaml
> ```

3. 설치 확인
> ```bash
> [root@m-k8s product-service]# kubectl get pods -n argocd
> NAME                                                READY   STATUS    RESTARTS   AGE
> argocd-application-controller-0                     1/1     Running   0          94s
> argocd-applicationset-controller-6f64df6b67-7gd5l   1/1     Running   0          95s
> argocd-dex-server-56fc55fb54-gw65v                  1/1     Running   0          95s
> argocd-notifications-controller-566848f84b-g6wj4    1/1     Running   0          95s
> argocd-redis-78d49dc67-zl99t                        1/1     Running   0          94s
> argocd-repo-server-65f688d5c9-z7fwz                 1/1     Running   0          94s
> argocd-server-74fc764598-kzgbb                      1/1     Running   0          94s
> ```

## 6-2. ArgoCD 접속 설정

k8s 클러스터에 구성된 MetalLB, ingress-controller를 활용해 로컬 PC에서 ArgoCD로 접속할 수 있도록 한다.   
MetalLB, ingress-controller에서 HTTPS 인증서를 설정하지 않아 HTTP로 통신하고 있다. ArgoCD를 HTTP 모드로 변경한다.   

1. argocd-server HTTP 수정
> 아래 명령어로 argocd-server의 deployment를 수정한다.   
> 
> ```bash
> kubectl edit deployment argocd-server -n argocd
> ```
> 
> 아래와 같이 ```--insecure```를 추가하고 ```kubectl get pods -n argocd```로 pod 재시작을 확인한다.
> 
> ```yaml
> ...
> containers:
> - command:
>   - argocd-server
>   - --insecure
> ...
> ```

2. argocd ingress 적용
> 아래와 같이 argocd-ingress.yaml을 작성해 ```kubectl apply -f argocd-ingress.yaml```로 적용한다.
> 
> ```yaml
> apiVersion: networking.k8s.io/v1beta1
> kind: Ingress
> metadata:
>   name: argocd-ingress
>   namespace: argocd
> 
>   annotations:
>     kubernetes.io/ingress.class: nginx
> 
> spec:
>   rules:
>   - host: argocd.deploy-test.shop
>     http:
>       paths:
>       - path: /
>         backend:
>           serviceName: argocd-server
>           servicePort: 80
> ```

3. Windows hosts 파일 수정
> ```text
> C:\Windows\System32\drivers\etc\hosts
> 
> 192.168.1.202 server.deploy-test.shop
> 192.168.1.202 argocd.deploy-test.shop
> ```

4. 브라우저에서 접속 확인
> <figure>
>   <img src="https://i.imgur.com/ATDm0y9.png" width="100%" alt=""/>
>   <p style="font-style: italic; color: gray;">ArgoCD 접속</p>
> </figure>

5. 초기 비밀번호 확인
> ArgoCD admin 계정의 초기 비밀번호는 아래 명령어로 확인할 수 있다.
> 
> ```bash
> kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
> ```

## 6-3. ArgoCD 적용

1. Github Repository 생성
> ArgoCD Application 에서 서비스 배포 및 관리를 위해 참조할 레포지토리를 생성한다.   
> 아래와 같은 구조로 서비스 별 deployment.yaml, service.yaml을 작성해 저장한다.   
> 
> ```
> deploy-test-manifest(https://github.com/5-SH/deploy-test-manifest.git)
>  ├── member
>  │    └── deployment.yaml
>  │    └── service.yaml
>  │
>  ├── ordering
>  │    └── deployment.yaml
>  │    └── service.yaml
>  │
>  └── product
>       └── deployment.yaml
>       └── service.yaml
> ```

2. Jenkins 역할 변경
> 기존에는 Jenkins에서 Image push 다음 서비스 별 레포지토리에 있는 depl_svc.yaml 파일을 사용해 ```kubectl apply``` 까지 수행했다.    
> ArgoCD 이후에는 Jenkins는 Image push 다음 Manifest repository에서 서비스 별 deployment.yaml 또는 service.yaml을 수정한다.    
> 그리고 서비스의 배포는 ArgoCD에서 Manifest repository를 참조해 직접 수행한다.   
> 
> 서비스 별 Jenkins pipeline에서 아래와 같이 Kubernetes Deploy stage를 삭제하고 Manifest repository를 수정하는 stage를 추가한다.      
> 
> ```groovy
> pipeline {
> 
>     agent any
> 
>     environment {
> 
>         IMAGE_NAME = "192.168.56.1:8080/deploy-test-member/member-service"
>         IMAGE_TAG = "${BUILD_NUMBER}"
>     }
> 
>     stages {
> 
>         stage('Git Clone') {
> 
>             steps {
> 
>                 git branch: 'main',
>                     credentialsId: 'github-ssh',
>                     url: 'https://github.com/5-SH/deploy-test-member.git'
>             }
>         }
> 
>         stage('Gradle Build') {
> 
>             steps {
> 
>                 sh '''
>                 chmod +x gradlew
>                 ./gradlew clean build
>                 '''
>             }
>         }
> 
>         stage('Docker Build') {
> 
>             steps {
> 
>                 sh '''
>                 docker build -t $IMAGE_NAME:$IMAGE_TAG .
>                 '''
>             }
>         }
> 
>         stage('Harbor Login') {
> 
>             steps {
> 
>                 withCredentials([
>                     usernamePassword(
>                         credentialsId: 'harbor-account',
>                         usernameVariable: 'HARBOR_USER',
>                         passwordVariable: 'HARBOR_PASS'
>                     )
>                 ]) {
> 
>                     sh '''
>                     docker login 192.168.56.1:8080 \
>                     -u $HARBOR_USER \
>                     -p $HARBOR_PASS
>                     '''
>                 }
>             }
>         }
> 
>         stage('Harbor Push') {
> 
>             steps {
> 
>                 sh '''
>                 docker push $IMAGE_NAME:$IMAGE_TAG
>                 '''
>             }
>         }
> 
>         stage('Update Manifest Repo') {
> 
>             steps {
> 
>                 withCredentials([
>                     usernamePassword(
>                         credentialsId: 'github-token',
>                         usernameVariable: 'GIT_USER',
>                         passwordVariable: 'GIT_TOKEN'
>                     )
>                 ]) {
> 
>                     sh '''
>                     rm -rf deploy-test-manifest
> 
>                     git clone https://${GIT_USER}:${GIT_TOKEN}@github.com/5-SH/deploy-test-manifest.git
> 
>                     cd deploy-test-manifest/member
> 
>                     sed -i "s|image:.*|image: 192.168.56.1:8080/deploy-test-member/member-service:${BUILD_NUMBER}|g" deployment.yaml
> 
>                     git config user.email "jenkins@deploy-test.com"
>                     git config user.name "jenkins"
> 
>                     git add .
>                     git commit -m "update member image ${BUILD_NUMBER}"
> 
>                     git push
>                     '''
>                 }
>             }
>         }
>     }
> }
> ```
> 
> Harbor Push stage에서 Jenkins에서 서비스를 빌드할 때 마다 자동으로 증가하는 ```BUILD_NUMBER``` 값을 이미지의 버전으로 사용해 Harbor에 Push 한다.   
> 그리고 Update Manifest Repo Stage에서는 deployment.yaml에 Harbor에 push 한 ```BUILD_NUMBER``` 버전으로 이미지 pull 하도록 수정한다.   

3. ArgoCD Application 생성
> Application은 k8s에 어떤 서비스를 어떤 방식으로 배포할지 정의하는 객체이다.   
> ArgoCD는 Git(deploy-test-manifest)을 계속 감시하다가 변경을 감지하면 k8s 클러스터의 상태와 비교해 차이가 있으면 자동/수동으로 배포한다.   
>아래 서비스 별 application yaml을 ```kubectl apply -f {service}-app.yaml``` 명령어로 적용한다.   
> 
> ```bash
> # member-app.yaml
> apiVersion: argoproj.io/v1alpha1
> kind: Application
> 
> metadata:
>   name: member-app
>   namespace: argocd
> 
> spec:
>   project: default
> 
>   source:
>     repoURL: https://github.com/5-SH/deploy-test-manifest.git
>     targetRevision: main
>     path: member
> 
>   destination:
>     server: https://kubernetes.default.svc
>     namespace: deploy-test
> 
>   syncPolicy:
>     automated:
>       prune: true
>       selfHeal: true
> 
>     syncOptions:
>       - CreateNamespace=true
> 
> # ordering-app.yaml
> apiVersion: argoproj.io/v1alpha1
> kind: Application
> 
> metadata:
>   name: ordering-app
>   namespace: argocd
> 
> spec:
>   project: default
> 
>   source:
>     repoURL: https://github.com/5-SH/deploy-test-manifest.git
>     targetRevision: main
>     path: ordering
> 
>   destination:
>     server: https://kubernetes.default.svc
>     namespace: deploy-test
> 
>   syncPolicy:
>     automated:
>       prune: true
>       selfHeal: true
> 
>     syncOptions:
>       - CreateNamespace=true
> 
> # product-app.yaml
> apiVersion: argoproj.io/v1alpha1
> kind: Application
> 
> metadata:
>   name: product-app
>   namespace: argocd
> 
> spec:
>   project: default
> 
>   source:
>     repoURL: https://github.com/5-SH/deploy-test-manifest.git
>     targetRevision: main
>     path: product
> 
>   destination:
>     server: https://kubernetes.default.svc
>     namespace: deploy-test
> 
>   syncPolicy:
>     automated:
>       prune: true
>       selfHeal: true
> 
>     syncOptions:
>       - CreateNamespace=true
> ```

4. ArgoCD 배포
> 아래 흐름으로 ArgoCD에서 서비스를 배포한다.   
> 
> ```
> 개발자 Git Push
>         ↓
>      Jenkins Build
>         ↓
>    Docker Image 생성
>         ↓
>       Harbor Push
>         ↓
>  GitOps Repo Manifest 수정
>         ↓
>        ArgoCD Sync
>         ↓
>    Kubernetes 자동 반영
> ```
> 
> <figure>
>   <img src="https://i.imgur.com/3KySyp5.png" width="100%" alt=""/>
>   <p style="font-style: italic; color: gray;">ArgoCD 적용</p>
> </figure>
