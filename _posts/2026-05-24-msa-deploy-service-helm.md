---
layout: post
title: MSA Deploy (5) - Helm
date: 2026-05-24 06:30:00 + 0900
categories: [kubernetes]
tags: [kubernetes, msa]
mermaid: true
---

# 현재 구조도

```mermaid
flowchart TD

    %% =====================================================
    %% Developer
    %% =====================================================

    DEV[Developer]

    %% =====================================================
    %% GitHub
    %% =====================================================

    subgraph GITHUB["GitHub"]

        direction TB

        APP_REPO["Application Repositories
        - deploy-test-member
        - deploy-test-ordering
        - deploy-test-product"]

        MANIFEST_REPO["GitOps Repository
        - deploy-test-manifest"]

    end

    %% =====================================================
    %% Local PC Windows
    %% =====================================================

    subgraph LOCAL["Local PC Windows"]

        direction TB

        subgraph DOCKER["Docker Desktop"]

            direction TB

            JENKINS["Jenkins Container"]

            HARBOR["Harbor Registry
            192.168.56.1:8080"]

            subgraph PIPELINE["Jenkins CI Pipeline"]

                direction TB

                GIT_CLONE["1. Git Clone"]

                GRADLE["2. Gradle Build"]

                DOCKER_BUILD["3. Docker Build"]

                HARBOR_PUSH["4. Harbor Push"]

                UPDATE_MANIFEST["5. deployment.yaml
                image tag 변경"]

            end

        end

        HOSTS["hosts 설정
        server.deploy-test.shop
        argocd.deploy-test.shop"]

    end

    %% =====================================================
    %% Oracle VM + Kubernetes
    %% =====================================================

    subgraph VM["Oracle VM VirtualBox + Vagrant"]

        direction TB

        subgraph K8S["Kubernetes Cluster"]

            direction TB

            %% =============================================
            %% Infra
            %% =============================================

            subgraph INFRA["Cluster Infra"]

                direction TB

                METALLB["MetalLB
                192.168.1.200~210"]

                INGRESS["Nginx Ingress Controller"]

                ARGOCD["ArgoCD"]

            end

            %% =============================================
            %% Services
            %% =============================================

            subgraph DEPLOY["Namespace : deploy-test"]

                direction TB

                MEMBER_SVC["member-service"]
                MEMBER_DEPLOY["member-depl"]

                ORDERING_SVC["ordering-service"]
                ORDERING_DEPLOY["ordering-depl"]

                PRODUCT_SVC["product-service"]
                PRODUCT_DEPLOY["product-depl"]

            end

            %% =============================================
            %% Worker Nodes
            %% =============================================

            subgraph NODES["Worker Nodes"]

                direction TB

                W1["w1-k8s"]

                W2["w2-k8s"]

                W3["w3-k8s"]

            end

            %% =============================================
            %% Database
            %% =============================================

            subgraph DB["Namespace : deploy-test-data"]

                direction TB

                MYSQL_MASTER["mysql-master"]

                MYSQL_SLAVE["mysql-slave"]

            end

        end

    end

    %% =====================================================
    %% User
    %% =====================================================

    USER["Browser / API Client"]

    %% =====================================================
    %% CI Flow
    %% =====================================================

    DEV -->|git push| APP_REPO

    APP_REPO --> JENKINS

    JENKINS --> GIT_CLONE
    GIT_CLONE --> GRADLE
    GRADLE --> DOCKER_BUILD
    DOCKER_BUILD --> HARBOR_PUSH

    HARBOR_PUSH --> HARBOR

    JENKINS --> UPDATE_MANIFEST

    UPDATE_MANIFEST --> MANIFEST_REPO

    %% =====================================================
    %% GitOps Flow
    %% =====================================================

    MANIFEST_REPO --> ARGOCD

    ARGOCD --> MEMBER_DEPLOY
    ARGOCD --> ORDERING_DEPLOY
    ARGOCD --> PRODUCT_DEPLOY

    %% =====================================================
    %% Deployment Scheduling
    %% =====================================================

    MEMBER_DEPLOY --> W2
    ORDERING_DEPLOY --> W1
    PRODUCT_DEPLOY --> W3

    %% =====================================================
    %% Image Pull
    %% =====================================================

    W1 -. image pull .-> HARBOR
    W2 -. image pull .-> HARBOR
    W3 -. image pull .-> HARBOR

    %% =====================================================
    %% Service Traffic Flow
    %% =====================================================

    USER --> HOSTS

    HOSTS --> METALLB

    METALLB --> INGRESS

    INGRESS --> MEMBER_SVC
    INGRESS --> ORDERING_SVC
    INGRESS --> PRODUCT_SVC

    MEMBER_SVC --> MEMBER_DEPLOY
    ORDERING_SVC --> ORDERING_DEPLOY
    PRODUCT_SVC --> PRODUCT_DEPLOY

    %% =====================================================
    %% DB Connection
    %% =====================================================

    MEMBER_DEPLOY --> MYSQL_MASTER
    ORDERING_DEPLOY --> MYSQL_MASTER
    PRODUCT_DEPLOY --> MYSQL_MASTER

    MYSQL_SLAVE --> MYSQL_MASTER

    %% =====================================================
    %% ArgoCD UI
    %% =====================================================

    USER -->|argocd.deploy-test.shop| ARGOCD
```

# 7. Helm 도입

## 7-1. Helm 도입 및 ArgoCD GitOps 환경 개선

기존에는 Jenkins에서 Kubernetes Manifest(```deployment.yaml```)를 직접 수정하는 방식으로 배포를 수행다.   
하지만 이 방식은 서비스가 늘어날수록 yaml 파일 관리가 복잡해지고 환경별 설정 분리가 어려워지는 문제가 있었다.   
이를 해결하기 위해 Helm Chart 기반 구조로 전환하고, ArgoCD와 연동하여 GitOps 방식으로 개선했다.   

### 기존 구조
기존에는 아래와 같은 흐름이었다.

```text
Jenkins
  ↓
Docker Image Build
  ↓
Harbor Push
  ↓
deployment.yaml 직접 수정
  ↓
Git Push
  ↓
ArgoCD Sync
  ↓
kubectl apply
```

manifest repository 구조는 다음과 같았다.

```text
deploy-test-manifest/
 ├ member/
 │   ├ deployment.yaml
 │   └ service.yaml
 │
 ├ ordering/
 │   ├ deployment.yaml
 │   └ service.yaml
 │
 └ product/
     ├ deployment.yaml
     └ service.yaml
```

## 7-2. Helm 도입 목적

Helm 도입의 주요 목적은 다음과 같다.

* Kubernetes yaml 템플릿화
* values.yaml 기반 설정 분리
* 서비스별 재사용 가능한 배포 구조 구성
* GitOps 환경 단순화
* 운영 환경 확장 용이성 확보

## 7-3. Helm 설치 

현재 Kubernetes Cluster는 Oracle VM VirtualBox 기반 Vagrant 환경에 구성되어 있다.   
Helm 명령어는 Kubernetes API 접근이 가능한 환경에서 실행해야 하고 m-k8s Master Node에서 Helm을 설치했다.   

1. Helm 설치 스크립트 다운로드
> ```bash
> curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
> ```

2. 실행 권한 부여
> ```bash
> chmod 700 get_helm.sh
> ```

3. Helm 설치
> ```bash
> ./get_helm.sh
> ```

4. 설치 확인
> ```bash
> helm version
> ```
> 
> 정상 출력 예시:
> 
> ```bash
> version.BuildInfo{Version:"v3.18.1", ...}
> ```

## 7-4. Helm Chart 생성

로컬 Git 작업 환경에서 Helm Chart를 생성한다.

1. Member Chart 생성
> ```bash
> helm create member-chart
> ```
> 
> 생성 결과:
> 
> ```text
> member-chart/
>  ├ charts/
>  ├ templates/
>  ├ values.yaml
>  └ Chart.yaml
> ```

2. 불필요 파일 제거
> 기본 생성되는 테스트 파일들을 제거하였다.
> 
> ```bash
> rm -rf templates/tests
> rm templates/hpa.yaml
> rm templates/ingress.yaml
> rm templates/serviceaccount.yaml
> rm templates/NOTES.txt
> ```

3. values.yaml 수정
> 기존 ```deployment.yaml```, ```service.yaml```에 있던 값들을 ```values.yaml```로 이동하였다.   
> 
> ```yaml
> # member-chart/values.yaml
> 
> replicaCount: 1
> 
> image:
>   repository: 192.168.56.1:8080/deploy-test-member/member-service
>   pullPolicy: Always
>   tag: "19"
> 
> imagePullSecrets: []
> nameOverride: ""
> fullnameOverride: ""
> 
> serviceAccount:
>   create: true
>   annotations: {}
>   name: ""
> 
> podAnnotations: {}
> 
> podSecurityContext: {}
> 
> securityContext: {}
> 
> service:
>   type: ClusterIP
>   port: 80
>   targetPort: 8080
> 
> ingress:
>   enabled: false
>   className: ""
>   annotations: {}
>   hosts:
>     - host: chart-example.local
>       paths:
>         - path: /
>           pathType: ImplementationSpecific
>   tls: []
> 
> resources:
>   limits:
>     cpu: "250m"
>     memory: "500Mi"
>   requests:
>     cpu: "100m"
>     memory: "250Mi"
> 
> autoscaling:
>   enabled: false
>   minReplicas: 1
>   maxReplicas: 100
>   targetCPUUtilizationPercentage: 80
> 
> nodeSelector: {}
> 
> tolerations: []
> 
> affinity: {}
> 
> env:
>   DB_HOST: "mysql-master-0.mysql-master.deploy-test-data.svc.cluster.local"
>   DB_PW: "rootpass"
> ```

4. deployment.yaml, service.yaml 템플릿 수정
> ```templates/deployment.yaml```, ```template/service.yaml``` 내부를 Helm 변수 기반으로 수정한다.
> 
> ```yaml
> # member-chart/templates/deployment.yaml
> 
> apiVersion: apps/v1
> kind: Deployment
> 
> metadata:
>   name: member-depl
>   namespace: deploy-test
> 
> spec:
>   replicas: {{ .Values.replicaCount }}
> 
>   selector:
>     matchLabels:
>       app: member
> 
>   template:
>     metadata:
>       labels:
>         app: member
> 
>     spec:
>       imagePullSecrets:
>         - name: harbor-secret
> 
>       containers:
>         - name: member-container
>           image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
>           imagePullPolicy: {{ .Values.image.pullPolicy }}
>           ports:
>             - containerPort: 8080
> 
>           resources:
>             limits:
>               cpu: {{ .Values.resources.limits.cpu }}
>               memory: {{ .Values.resources.limits.memory }}
>             requests:
>               cpu: {{ .Values.resources.requests.cpu }}
>               memory: {{ .Values.resources.requests.memory }}
> 
>           env:
>             - name: DB_HOST
>               value: {{ .Values.env.DB_HOST }}
>             - name: DB_PW
>               value: {{ .Values.env.DB_PW }}
> 
>           readinessProbe:
>             httpGet:
>               path: /health
>               port: 8080
>             initialDelaySeconds: 10
>             periodSeconds: 10
> ```
> 
> ```yaml
> apiVersion: v1
> kind: Service
> 
> metadata:
>   name: member-service
>   namespace: deploy-test
> 
> spec:
>   type: {{ .Values.service.type }}
>   ports:
>     - protocol: TCP
>       port: {{ .Values.service.port }}
>       targetPort: {{ .Values.service.targetPort }}
>   selector:
>     app: member
> ```

## 7-5. Helm Chart 기반 GitOps Repository 구조 변경

기존에는 서비스별 Kubernetes Manifest 파일을 직접 관리했었다.   
기존 구조는 다음과 같았다.   

```
deploy-test-manifest/
 ├ member/
 │   ├ deployment.yaml
 │   └ service.yaml
 │
 ├ ordering/
 │   ├ deployment.yaml
 │   └ service.yaml
 │
 └ product/
     ├ deployment.yaml
     └ service.yaml
```

이 구조에서는 다음과 같은 문제가 있었다.   

* deployment.yaml 수정이 반복됨
* 서비스별 YAML 중복 증가
* 환경(dev/stage/prod) 분리 어려움
* 이미지 태그 변경 시 deployment.yaml 직접 수정 필요
* 운영 환경 확장 시 유지보수 복잡도 증가

이를 해결하기 위해 Helm Chart 기반 구조로 변경하였다.

## 7-6. Helm 기반 구조로 변경

변경 후 Repository 구조는 다음과 같다.

```
deploy-test-manifest/
 │
 ├ member-chart/
 │   ├ Chart.yaml
 │   ├ values.yaml
 │   └ templates/
 │       ├ deployment.yaml
 │       └ service.yaml
 │
 ├ ordering-chart/
 │   ├ Chart.yaml
 │   ├ values.yaml
 │   └ templates/
 │       ├ deployment.yaml
 │       └ service.yaml
 │
 └ product-chart/
     ├ Chart.yaml
     ├ values.yaml
     └ templates/
         ├ deployment.yaml
         └ service.yaml
```

---

### 7-6-1. 각 파일 역할

1. Chart.yaml
> Helm Chart의 메타데이터 파일이다.   
> Chart 이름, 버전 등의 정보를 저장한다.   
> 
> 예시:
> 
> ```yaml
> apiVersion: v2
> name: member-chart
> description: member service helm chart
> type: application
> version: 0.1.0
> appVersion: "1.0"
> ```

2. values.yaml
> 서비스 설정값을 저장하는 파일이다.   
> 실제 Kubernetes YAML에서 사용할 변수들을 정의한다.   
> 
> 예시:
> 
> ```yaml
> replicaCount: 1
> 
> image:
>   repository: 192.168.56.1:8080/deploy-test-member/member-service
>   tag: "19"
>   pullPolicy: Always
> 
> service:
>   type: ClusterIP
>   port: 80
>   targetPort: 8080
> 
> resources:
>   limits:
>     cpu: 250m
>     memory: 500Mi
> 
>   requests:
>     cpu: 100m
>     memory: 250Mi
> 
> env:
>   DB_HOST: mysql-master-0.mysql-master.deploy-test-data.svc.cluster.local
>   DB_PW: rootpass
> ```
> 
> Jenkins는 이 values.yaml 내부의 image tag 값만 변경한다.

3. templates/deployment.yaml
> Kubernetes Deployment Template 파일이다.   
> Helm 변수 문법을 사용한다.   
> 
> 예시:
> 
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> 
> metadata:
>   name: member-depl
> 
> spec:
>   replicas: {{ .Values.replicaCount }}
> 
>   selector:
>     matchLabels:
>       app: member
> 
>   template:
>     metadata:
>       labels:
>         app: member
> 
>     spec:
>       imagePullSecrets:
>         - name: harbor-secret
> 
>       containers:
>         - name: member-container
> 
>           image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
>           imagePullPolicy: {{ .Values.image.pullPolicy }}
> 
>           ports:
>             - containerPort: 8080
> 
>           env:
>             - name: DB_HOST
>               value: "{{ .Values.env.DB_HOST }}"
> 
>             - name: DB_PW
>               value: "{{ .Values.env.DB_PW }}"
> ```

4. templates/service.yaml
> Kubernetes Service Template 파일이다.   
> 
> 예시:
> 
> ```yaml
> apiVersion: v1
> kind: Service
> 
> metadata:
>   name: member-service
> 
> spec:
>   type: {{ .Values.service.type }}
> 
>   selector:
>     app: member
> 
>   ports:
>     - port: {{ .Values.service.port }}
>       targetPort: {{ .Values.service.targetPort }}
> ```


### 7-6-2. Helm Template Rendering 동작 방식

ArgoCD는 Helm Chart를 발견하면 내부적으로 다음을 수행한다.   

```text
Git Clone
  ↓
helm template
  ↓
deployment.yaml 생성
  ↓
service.yaml 생성
  ↓
kubectl apply
```

즉 Git Repository에는 Template만 저장되고 실제 Kubernetes yaml ArgoCD가 동적으로 생성한다.   

---

### 7-6-3. Helm 구조로 변경 후 Jenkins 역할 변화

기존에는 Jenkins가 deployment.yaml 자체를 수정한다.   

기존 방식:

```bash
sed -i "s|image:.*|image: xxx|g" deployment.yaml
```

하지만 Helm 도입 이후에는 values.yaml의 tag 값만 변경한다.   

변경 후:

```bash
sed -i "s/tag:.*/tag: \"${BUILD_NUMBER}\"/" values.yaml
```

---

### 7-6-4. Helm 도입 후 최종 구조

현재 최종 GitOps 구조는 아래와 같고 기존보다 훨씬 운영 친화적인 구조로 개선되었다.   

```text
Jenkins
  ↓
Docker Image Build
  ↓
Harbor Push
  ↓
values.yaml tag 변경
  ↓
Git Push
  ↓
ArgoCD Sync
  ↓
Helm Template Rendering
  ↓
Kubernetes Rolling Update
```

## 7-7. Helm Install 수행

m-k8s에서 Helm Install 수행   

```bash
helm install member-test ./member-chart -n deploy-test
```

1. 발생한 문제 1 - Kubernetes Cluster Unreachable
> 다음과 같은 에러가 발생했다.  
> 
> ```bash
> Error: INSTALLATION FAILED:
> Kubernetes cluster unreachable:
> unable to parse the server version:
> invalid character '<' looking for beginning of value
> ```
> 
> 원인은 Helm 명령어를 로컬 WSL Ubuntu에서 수행했기 때문이었다.   
> WSL에는 kubeconfig가 설정되어 있지 않아 Kubernetes API 접근이 불가능했다.   
> 
> Helm 명령어는 Kubernetes API 접근이 가능한 m-k8s에서 수행해 해결했다.

2. 발생한 문제 2 - Helm Ownership Metadata 에러
> 다음 에러가 발생하였다.   
> 
> ```bash
> Error: rendered manifests contain a resource that already exists
> ```
> 
> 세부 메시지:
> 
> ```bash
> Service "member-service" exists and cannot be imported
> ```
> 
> 원인은 기존 kubectl apply 방식으로 생성된 Deployment/Service가 이미 존재하고 있었기 때문이다.   
> Helm은 자신이 생성하지 않은 리소스를 관리 대상으로 가져갈 수 없다.    
> 
> 기존 리소스를 삭제해 해결했다.   
> 기존 ReplicaSet과 Pod까지 함께 삭제되는지 확인한다.   
> 
> ```bash
> kubectl delete deployment member-depl -n deploy-test
> kubectl delete service member-service -n deploy-test
> ```

3. Helm Install 재수행
> ```bash
> helm install member-test ./member-chart -n deploy-test
> ```
> 
> 정상 배포 완료.

## 7-8. ArgoCD와 Helm 연동

ArgoCD는 Helm을 Native 지원한다.   
그래서 ArgoCD Application에서 path만 Helm Chart 위치로 지정하면 자동으로 Helm Template Rendering을 수행한다.   

### 기존 ArgoCD 구조

```yaml
path: member
```

### 변경 후

```yaml
path: member-chart
```


### ArgoCD 내부 동작

ArgoCD는 내부적으로 다음을 수행한다.   

```text
Git Clone
  ↓
helm template
  ↓
Manifest Rendering
  ↓
kubectl apply
```

따라서 별도의 Helm CLI 설치 없이도 Helm 기반 GitOps 운영이 가능하다.   

## 7-9. Jenkins Pipeline 수정

기존에는 deployment.yaml을 직접 수정했었다.   

기존 방식:

```bash
sed -i "s|image:.*|image: xxx|g" deployment.yaml
```

---

### Helm 기반 변경

이제는 values.yaml의 tag 값만 수정하도록 변경한다.   

```bash
sed -i "s/tag:.*/tag: \"${BUILD_NUMBER}\"/" values.yaml
```

---

### Jenkinsfile 수정

기존:

```bash
cd deploy-test-manifest/member
```

변경 후:

```bash
cd deploy-test-manifest/member-chart
```

```
# jenkinsfile

pipeline {

    agent any

    environment {

        IMAGE_NAME = "192.168.56.1:8080/deploy-test-member/member-service"
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Git Clone') {

            steps {

                git branch: 'main',
                    credentialsId: 'github-ssh',
                    url: 'https://github.com/5-SH/deploy-test-member.git'
            }
        }

        stage('Gradle Build') {

            steps {

                sh '''
                chmod +x gradlew
                ./gradlew clean build
                '''
            }
        }

        stage('Docker Build') {

            steps {

                sh '''
                docker build -t $IMAGE_NAME:$IMAGE_TAG .
                '''
            }
        }

        stage('Harbor Login') {

            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'harbor-account',
                        usernameVariable: 'HARBOR_USER',
                        passwordVariable: 'HARBOR_PASS'
                    )
                ]) {

                    sh '''
                    docker login 192.168.56.1:8080 \
                    -u $HARBOR_USER \
                    -p $HARBOR_PASS
                    '''
                }
            }
        }

        stage('Harbor Push') {

            steps {

                sh '''
                docker push $IMAGE_NAME:$IMAGE_TAG
                '''
            }
        }

        stage('Update Manifest Repo') {

            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'github-token',
                        usernameVariable: 'GIT_USER',
                        passwordVariable: 'GIT_TOKEN'
                    )
                ]) {

                    sh '''
                    rm -rf deploy-test-manifest

                    git clone https://${GIT_USER}:${GIT_TOKEN}@github.com/5-SH/deploy-test-manifest.git

                    cd deploy-test-manifest/member-chart

                    sed -i "s/tag:.*/tag: \\"${BUILD_NUMBER}\\"/" values.yaml

                    git config user.email "jenkins@deploy-test.com"
                    git config user.name "jenkins"

                    git add values.yaml
                    git diff --cached --quiet || git commit -m "update member image ${BUILD_NUMBER}"

                    git push
                    '''
                }
            }
        }
    }
}
```

## 7-10. 최종 GitOps 흐름

최종 구조는 아래와 같이 변경되었다.   

```text
Jenkins
  ↓
Gradle Build
  ↓
Docker Build
  ↓
Harbor Push
  ↓
values.yaml 수정
  ↓
Git Push
  ↓
ArgoCD 자동 감지
  ↓
Helm Template Rendering
  ↓
Kubernetes Rolling Update
```

# 8. 모니터링
## 8-1. Prometheus

## 8-2. Grafana

# 9. LGTM 로그 수집

# 10. 그 외
1. Kubernetes 버전 업

2. 시크릿 설정 

3. HTTPS 통신 

4. DB 이중화 연결 

5. 서비스용 DB 계정 생성 

6. 젠킨스, harbor 구성 

7. HPA