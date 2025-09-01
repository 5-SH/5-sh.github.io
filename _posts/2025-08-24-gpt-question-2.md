---
layout: post
title: Chat GPT 질문 2
date: 2025-08-24 23:00:00 + 0900
categories: [question]
tags: [spring, jpa]
---

### 1. containerd와 docker의 차이점

containerd와 Docker는 둘 다 컨테이너 런타임 관련 기술이지만, 하는 역할과 범위가 달라요.   

#### 1-1. Docker

개발자 친화적인 올인원 플랫폼으로서 Docker는 단순한 컨테이너 런타임이 아니라:   

- 이미지 빌드 (docker build)
- 이미지 배포 (docker push/pull)
- 컨테이너 실행/관리 (docker run, docker ps)
- 네트워킹, 볼륨, 플러그인 관리

까지 제공하는 엔드투엔드 도구 세트예요.   
내부적으로 컨테이너 실행을 위해 containerd를 사용해 왔어요.   
즉, Docker = CLI + API + 빌드 도구 + containerd + runc 조합.   

#### 1-2. containerd

컨테이너 실행을 책임지는 런타임으로서 CNCF(Cloud Native Computing Foundation)에 기여된 독립 프로젝트.   

역할은 단순해요:   

- 컨테이너 이미지 가져오기 (pull)
- 이미지 관리
- 컨테이너 실행/중지
- 컨테이너 파일시스템, 네트워킹 기초 지원

자체 CLI(ctr)는 있지만, 개발자 친화적이지 않고 주로 쿠버네티스 같은 오케스트레이터가 containerd API를 통해 컨테이너를 다루어요.   
실행 자체는 containerd가 runc 같은 하위 런타임을 호출해서 이루어져요.   

#### 1-3. 구조 비교

```
[ Docker ]
 ├─ CLI (docker)
 ├─ Docker Engine (API 서버, 빌드 등)
 └─ containerd (컨테이너 관리)
      └─ runc (실제 컨테이너 실행)

[ containerd 단독 사용 ]
 ├─ containerd (컨테이너 관리)
 └─ runc (실행)
```

#### 1-4. 차이 요약

|구분|Docker|containerd|
|---|---|---|
|성격|플랫폼/툴체인|런타임|
|기능 범위|빌드, 배포, 실행, 네트워크, 볼륨 등 포함|컨테이너 이미지 관리 & 실행 중심|
|사용 대상|개발자 (CLI 편리함)|쿠버네티스/시스템 수준|
|의존 관계|내부적으로 containerd를 사용|독립 실행 가능|
|CLI|docker|ctr (불편, 디버깅용)|

#### 1-5. 👉 요약하면,   

- Docker는 개발자를 위한 올인원 도구 세트
- containerd는 쿠버네티스 같은 시스템이 쓰는 저수준 런타임

### 2. 쿠버네티스(Kubernetes)에서 containerd가 Docker를 대체하게된 배경

#### 2-1. 원래 쿠버네티스와 Docker의 관계

초창기 쿠버네티스는 컨테이너 실행을 위해 Docker를 직접 사용했어요.   
Pod를 실행할 때 내부적으로 kubelet → Docker Engine → containerd → runc 이런 구조였죠.   

#### 2-2. 문제점

시간이 지나면서 몇 가지 문제가 드러났어요:

- 중간 계층(오버헤드) 문제: 실제 컨테이너 실행에는 containerd + runc만 있으면 충분한데, Docker는 Docker Engine이라는 무거운 레이어가 하나 더 있었음.
- 쿠버네티스 입장에서는 필요 없는 빌드/CLI 기능도 로드해야 했음.
- CRI(Container Runtime Interface) 불일치
- 쿠버네티스는 런타임을 추상화하기 위해 CRI라는 표준 API를 만들었어요.
- Docker는 CRI를 지원하지 않았음 → dockershim이라는 별도의 번역기 계층을 두어야 했음.
- 관리 복잡성이 증가했음.
- 운영/성능 부담
- Docker는 개발자 친화적이지만, 쿠버네티스가 원하는 건 “컨테이너 실행” 뿐.
- 불필요한 기능이 많아서 리소스 낭비 + 성능 저하 가능성이 있었음.

#### 2-3. containerd로 전환

containerd는 쿠버네티스 CRI를 네이티브로 지원해요. → dockershim이 필요 없어짐.   
단순하고 경량 → 성능과 안정성이 개선됨.   
CNCF(쿠버네티스를 관리하는 재단)에서 containerd도 관리 → 생태계 일관성 ↑.   

#### 2-4. 결과: Docker 지원 종료 (2020 발표, 2021~2022 전환)

Kubernetes v1.20에서 “Dockershim Deprecation” 발표.   
v1.24부터는 공식적으로 Docker 지원 제거.   
지금은 containerd, CRI-O 같은 CRI 호환 런타임을 권장.   

#### 2-5. 현재 상황

- 개발 환경: 여전히 Docker를 많이 씀 (빌드/실행 편리). 
- 운영 환경(쿠버네티스): 주로 containerd 또는 CRI-O 사용.
- Docker로 만든 이미지는 그대로 containerd에서 실행 가능 (이미지 포맷 표준 OCI 때문).

#### 2-6. ✅ 정리하면,
쿠버네티스는 **경량·표준화된 런타임(containerd)**만 필요했기 때문에 “Docker → containerd”로 옮겨간 거예요.   


### 3. 헬름과 커스토마이즈의 차이

쿠버네티스에서 **Helm(헬름)**과 **Kustomize(커스토마이즈)**는 모두 매니페스트 관리 도구인데, 접근 방식이 다릅니다.   

#### 3-1. Helm (헬름)

쿠버네티스의 패키지 매니저 (리눅스의 apt, yum 같은 역할).   
앱을 Chart라는 패키지 형태로 배포/관리.   

주요 특징:   

- 템플릿 기반: values.yaml 파일에 값만 넣으면 템플릿이 채워짐.
- 패키지 레지스트리(Helm Hub 등)에서 차트를 가져와 바로 설치 가능 (helm install).
- 배포/업데이트/롤백 기능 내장.
- 복잡한 애플리케이션도 하나의 Chart로 관리 가능 (예: nginx + redis + db 같이).

단점:   

- 템플릿 복잡성 → 디버깅 어렵고, YAML 재사용 구조가 덜 투명함.
- 학습 난이도가 있음.

#### 3-2. Kustomize (커스토마이즈)

kubectl에 내장된 오버레이(Overlay) 도구.   
매니페스트를 템플릿화하지 않고, YAML을 조합/변형하는 방식.   

주요 특징:   

- kustomization.yaml 파일을 정의해서 base 매니페스트를 overlay로 확장.
- 예: dev, staging, prod 환경마다 레이블/리소스 값만 patch.
- 템플릿 언어가 아니라 YAML을 그대로 유지 → 읽기 쉽고 구조적.
- 쿠버네티스 표준 도구(kubectl apply -k)로 바로 사용 가능.

단점:   

- Helm처럼 패키지 관리(저장소, 버전 관리)는 없음.
- 아주 복잡한 앱 배포에는 다소 부족할 수 있음.

#### 3-3. 비교 정리
|구분|Helm|Kustomize|
|---|---|---|
|성격|패키지 매니저 (쿠버네티스의 apt/yum)|매니페스트 커스터마이징 도구|
|접근 방식|템플릿 + values.yaml|오버레이 방식 (patch, merge)|
|배포|helm install → 릴리스 관리 (rollback 가능)|kubectl apply -k (단순 배포, 롤백 없음)|
|패키지화|Chart 공유 가능 (Helm Hub, private repo)|패키지 개념 없음 (GitOps와 잘 맞음)|
|학습 난이도|상대적으로 높음 (템플릿 언어)|비교적 낮음 (순수 YAML 변형)|
|사용 예시|nginx ingress, prometheus 등 오픈소스 배포|환경별(dev/prod) 설정 차이 관리|

#### 3-4. ✅ 정리

- Helm: "앱 설치/업데이트/패키지 관리까지 해주는 쿠버네티스 앱스토어"   
- Kustomize: "같은 매니페스트를 환경별로 깔끔하게 변형하는 도구"

👉 실제 현업에서는 Helm + Kustomize를 같이 쓰는 경우도 많습니다.

- Helm 차트로 앱을 설치
- Kustomize로 환경별 overlay 적용


### 4. 쿠버네티스 매니페스트 그리고 Helm Chart 예시 vs Kustomize Overlay 예시

#### 4-1. 매니페스트(Manifest)란?

쿠버네티스에서 리소스를 정의하는 YAML/JSON 문서를 말해요.   
“내가 원하는 상태(Desired State)”를 쿠버네티스 API 서버에 전달하는 역할.   
쿠버네티스는 이 매니페스트를 읽고, 실제 클러스터 상태를 맞추기 위해 동작합니다.   

예: Deployment 매니페스트 (nginx 실행)   

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
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
        image: nginx:1.25
        ports:
        - containerPort: 80
```

👉 이 파일을 kubectl apply -f deployment.yaml 하면, 쿠버네티스는 nginx:1.25 컨테이너가 2개 떠 있도록 유지해줍니다.   

#### 4-2. Helm Chart 예시

Helm은 매니페스트를 템플릿 + 값(values.yaml) 구조로 관리합니다.   

values.yaml

```yaml
replicaCount: 3
image:
  repository: nginx
  tag: "1.25"

deployment.yaml (템플릿)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {% raw %}{{ .Chart.Name }}{% endraw %}
spec:
  replicas: {% raw %}{{ .Values.replicaCount }}{% endraw %}
  template:
    metadata:
      labels:
        app: {% raw %}{{ .Chart.Name }}{% endraw %}
    spec:
      containers:
      - name: {% raw %}{{ .Chart.Name }}{% endraw %}
        image: {% raw %}{{ .Values.image.repository }}:{% raw %}{{ .Values.image.tag }}{% endraw %}
        ports:
        - containerPort: 80
```

- 👉 helm install myapp ./mychart -f values.yaml 하면 템플릿이 채워져서 쿠버네티스에 배포됩니다.
- 👉 Helm은 릴리스 단위로 관리하기 때문에, 버전업/롤백이 쉽습니다.

#### 4-3. Kustomize Overlay 예시

Kustomize는 매니페스트를 그대로 두고, 환경별로 patch하는 방식입니다.   

base/deployment.yaml (공통)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25

overlays/prod/kustomization.yaml
resources:
  - ../../base

patches:
  - target:
      kind: Deployment
      name: nginx
    patch: |
      - op: replace
        path: /spec/replicas
        value: 3
```

- 👉 kubectl apply -k overlays/prod 하면 replicas가 3으로 적용된 상태가 배포됩니다.
- 👉 템플릿 문법 없이 YAML 그대로 관리 가능.

#### 4-4. 한 줄 정리

- 매니페스트 = 쿠버네티스 리소스를 정의한 YAML/JSON
- Helm = 매니페스트를 패키지(Chart)로 만들고, 템플릿 + values 방식으로 재사용
- Kustomize = 매니페스트를 그대로 두고, 환경별 차이를 patch/overlay

혹시 제가 여기서 **"언제 Helm을 쓰고, 언제 Kustomize를 쓰는 게 좋은지 (실무 시나리오별)"**까지 정리해드릴까요?

### 5. 실무에서 Helm과 Kustomize를 사용하는 시나리오

#### 5-1. ✅ Helm이 적합한 경우

##### 5-1-1. 오픈소스 애플리케이션 설치

예: nginx ingress controller, prometheus, grafana, cert-manager   
이미 Helm Chart로 배포되는 경우가 많아서 그냥 helm install로 설치하면 끝.   
➝ 패키지 관리자로 쓰기 딱 좋음.   

##### 5-1-2. 복잡한 애플리케이션 배포

여러 리소스(Deployment, Service, ConfigMap, Secret, CRD 등)를 한꺼번에 배포할 때 유용.    
Chart 안에서 모든 리소스 관계를 관리 가능.   
버전/릴리스 관리 필요하면 ```helm upgrade```, ```helm rollback```으로 손쉽게 버전 관리.  
운영 환경에서 장애 시 빠른 롤백에 유리.   

##### 5-1-3. ✅ Kustomize가 적합한 경우

- 환경별 설정 차이 관리(dev/staging/prod)
- 공통 매니페스트(Base) + 환경별 Overlay (replica 수, 이미지 태그, 리소스 제한 등).
- 템플릿 언어 없이 YAML 그대로 유지 가능 → GitOps에 특히 적합.
- 단순 커스터마이징: ConfigMap, Secret, Label, Annotation 등 간단한 차이만 반영할 때.
- 별도 툴 설치 없이 kubectl apply -k로 바로 실행 가능.
- GitOps 워크플로우: ArgoCD, Flux 같은 GitOps 툴과 잘 맞음.
- 선언적 관리 → Git에 기록된 매니페스트 그대로 동작.

##### 5-1-4. ✅ Helm + Kustomize 같이 쓰는 경우

실무에서는 두 개를 섞어서 쓰기도 합니다:   
- Helm: 오픈소스 앱을 설치 (예: helm install ingress-nginx)
- Kustomize: 그 설치된 매니페스트를 환경별로 overlay (예: dev는 replica=1, prod는 replica=3)

이런 조합이 흔해요 → Helm은 패키지, Kustomize는 환경별 차이 관리   

#### 5-2. 📝 요약

- Helm = 앱 설치·배포·업데이트 관리에 강함 (쿠버네티스의 apt/yum)
- Kustomize = 매니페스트 커스터마이징·환경별 설정 관리에 강함 (overlay 도구)

실무에서는 Helm + Kustomize를 같이 쓰는 경우 많음


### 6. 헬름의 chart 구조

#### 6-1. Helm Chart란?

- 쿠버네티스 애플리케이션을 패키지화한 디렉토리 구조로서 매니페스트(YAML) + 템플릿 + 설정값(values) + 메타데이터 등을 모아놓은 것   
- ```helm install myapp ./mychart``` 하면 Chart 전체가 쿠버네티스에 배포됨

#### 6-2. 기본 Chart 디렉토리 구조

helm create mychart 하면 생성되는 예시:   

```
mychart/
├── Chart.yaml          # Chart 메타데이터 (이름, 버전 등)
├── values.yaml         # 기본 설정값 (사용자가 override 가능)
├── charts/             # 의존성 chart 저장 (subcharts)
├── templates/          # 쿠버네티스 매니페스트 템플릿 모음
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── _helpers.tpl    # 템플릿 함수/공통 정의
│   └── NOTES.txt       # 설치 후 사용자에게 안내 메시지
└── .helmignore         # 빌드 시 제외할 파일/폴더 (.git, README 등)
```

#### 6-3. 주요 파일 설명

##### 6-3-1. 🟢 Chart.yaml

Chart의 메타데이터 (앱 이름, 버전, 설명 등)

```yaml
apiVersion: v2
name: mychart
description: A Helm chart for Kubernetes
type: application
version: 0.1.0
appVersion: "1.16.0"
```

##### 6-3-2. 🟢 values.yaml

사용자가 설정할 수 있는 기본 값으로서 템플릿에서 .Values로 접근

```yaml
replicaCount: 2
image:
  repository: nginx
  tag: "1.25"
service:
  type: ClusterIP
  port: 80
```

##### 6-3-3. 🟢 templates/

실제 쿠버네티스 매니페스트 템플릿으로서 Go 템플릿 문법 사용

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {% raw %}{{ .Chart.Name }}{% endraw %}
spec:
  replicas: {% raw %}{{ .Values.replicaCount }}{% endraw %}
  template:
    spec:
      containers:
      - name: nginx
        image: "{% raw %}{{ .Values.image.repository }}:{% raw %}{{ .Values.image.tag }}"{% endraw %}
```

##### 6-3-4. 🟢 _helpers.tpl

재사용 가능한 헬퍼 함수 모음

```
{% raw %}{{- define "mychart.fullname" -}}{% endraw %}
{% raw %}{{ .Release.Name }}}{% endraw %}-{% raw %}{{ .Chart.Name }}{% endraw %}
{% raw %}{{- end }}{% endraw %}
```

##### 6-3-5. 🟢 NOTES.txt

Chart 설치 후 사용자에게 보여줄 메시지

```
1. Get the application URL by running these commands:
   kubectl get svc {% raw %}{{ include "mychart.fullname" . }}{% endraw %}
```

#### 6-4. 실행 흐름

1. ```helm install myapp ./mychart -f custom-values.yaml``` 실행
2. Helm이 values.yaml + custom-values.yaml 병합
3. templates/의 매니페스트 템플릿을 값으로 렌더링
4. 최종 YAML을 쿠버네티스 API 서버에 적용

#### 6-5. ✅ 요약

- Chart.yaml → 메타데이터
- values.yaml → 기본 설정값
- templates/ → 매니페스트 템플릿
- charts/ → 의존성 Chart
- _helpers.tpl → 공통 함수
- NOTES.txt → 설치 안내 메시지

나의 말:
### 7. argo cd 배포화면에 나오는 rs

Argo CD에서 애플리케이션을 배포하면 Deployment를 보통 사용하게 되는데, 이때 쿠버네티스가 자동으로 ReplicaSet (RS) 을 생성합니다.   

#### 7-1. ✅ ReplicaSet(RS)란?

쿠버네티스 리소스 중 하나로, 특정 수의 **Pod 레플리카(복제본)**가 항상 실행되도록 보장해주는 역할.   
Deployment가 직접 Pod를 관리하지 않고, ReplicaSet을 생성해서 관리하게 되어 있어요.   

즉: Argo CD → Deployment 생성 → ReplicaSet 생성 → Pod 생성   

#### 7-2. ✅ 동작 예시

##### 7-2-1. Argo CD에서 nginx Deployment 적용

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
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
        image: nginx:1.25
```

##### 7-2-2. 쿠버네티스 내부 동작

1) Deployment nginx 생성

2) Deployment가 자동으로 ReplicaSet 생성 (예: nginx-7c9d4c5b6d)

3) ReplicaSet이 replicas=3을 보고 Pod 3개 실행 (nginx-7c9d4c5b6d-abcde …)

##### 7-2-3. ✅ ReplicaSet과 Deployment 관계

ReplicaSet은 Pod를 일정 개수 유지하는 역할만 함.   
Deployment는 상위 개념으로, 새로운 ReplicaSet 생성 (버전 변경 시) 롤링 업데이트 / 롤백 관리 / 전략적 배포 (RollingUpdate, Recreate 등) 수행   

즉,

- ReplicaSet = “Pod 복제본 유지 관리”  
- Deployment = “ReplicaSet을 버전별로 관리 + 배포 전략 수행”

#### 7-3. ✅ Argo CD에서 보이는 RS

Argo CD UI에서 Deployment를 배포하면, 그 하위에 ReplicaSet이 생성된 걸 볼 수 있어요.   
예를 들어: nginx Deployment → nginx-7c9d4c5b6d ReplicaSet → nginx-7c9d4c5b6d-xxxxx Pod들   

#### 7-4. 👉 정리

Argo CD에서 보이는 RS는 ReplicaSet이에요.   
Deployment가 생성한 Pod를 실제로 관리하는 리소스.   
버전이 바뀌면 새 ReplicaSet이 생기고, 이전 ReplicaSet은 점점 Pod 수가 줄어듭니다.   


### 8. Deployment 업데이트 시 ReplicaSet이 2개 이상 생기고 롤링 업데이트 되는 과정을 단계별로 정리

#### 8-1. 초기 상태

Argo CD로 nginx:1.25 버전의 Deployment를 배포했다고 가정해요.

```
Deployment (nginx)
 └─ ReplicaSet (nginx-7c9d4c5b6d)  → replicas=3
      └─ Pod 3개 (nginx:1.25)
```

#### 8-2. 새로운 버전 배포 (nginx:1.26)

Argo CD에서 매니페스트가 업데이트되면, Deployment는 기존 ReplicaSet을 유지하면서 새 ReplicaSet을 생성합니다.   

```
Deployment (nginx)
 ├─ ReplicaSet (nginx-7c9d4c5b6d)  → replicas=3 (구 버전, nginx:1.25)
 └─ ReplicaSet (nginx-65f7c8a9b7)  → replicas=0 (신 버전, nginx:1.26)
```

#### 8-3. 롤링 업데이트 진행

Deployment의 기본 전략은 RollingUpdate입니다.   
즉, Pod을 한꺼번에 모두 교체하지 않고, 순차적으로 바꿔요.   

예: maxUnavailable=1, maxSurge=1인 경우:   
- 새로운 ReplicaSet에서 Pod 1개 생성 (nginx:1.26)
- 기존 ReplicaSet에서 Pod 1개 제거 (nginx:1.25 → -1개)
- 이 과정을 반복 → 점점 신 버전 Pod 수가 늘고, 구 버전 Pod 수가 줄어듬

#### 8-4. 최종 상태

업데이트가 완료되면 구 버전 ReplicaSet은 Pod 수가 0이 되고, 새 ReplicaSet만 남습니다.   

```
Deployment (nginx)
 ├─ ReplicaSet (nginx-7c9d4c5b6d)  → replicas=0 (구 버전)
 └─ ReplicaSet (nginx-65f7c8a9b7)  → replicas=3 (신 버전, nginx:1.26)
```

👉 하지만 구 ReplicaSet 자체는 롤백할 때 다시 사용 가능하도록 삭제되지 않고 남아 있음.

#### 8-5. 그림으로 요약

```
초기
Deployment
 └─ RS (old) → Pod: v1 x3

업데이트 시작
Deployment
 ├─ RS (old) → Pod: v1 x2
 └─ RS (new) → Pod: v2 x1

업데이트 중
Deployment
 ├─ RS (old) → Pod: v1 x1
 └─ RS (new) → Pod: v2 x2

업데이트 완료
Deployment
 ├─ RS (old) → Pod: v1 x0
 └─ RS (new) → Pod: v2 x3
```

#### 8-6. ✅ 정리

- Deployment는 버전별로 ReplicaSet을 생성해 Pod를 관리.   
- 롤링 업데이트 시 구 ReplicaSet은 점점 Pod 수가 줄고, 새 ReplicaSet은 Pod 수가 늘어남.
- 최종적으로 구 ReplicaSet은 Pod 0개 상태로 남아 있어 롤백에 사용 가능.


### 9. maxUnavailable, maxSurge

maxUnavailable과 maxSurge는 Deployment의 롤링 업데이트(RollingUpdate) 전략에서 Pod를 얼마나 동시에 줄이고 늘릴 수 있는지를 제어하는 옵션이에요.

#### 9-1. RollingUpdate 전략 기본 구조

Deployment에서 업데이트 전략은 두 가지가 있어요:   

- Recreate : 기존 Pod 전부 삭제 후 새 Pod 생성 (다운타임 발생 가능)
- RollingUpdate : 기존 Pod을 조금씩 줄이고 새 Pod을 조금씩 늘려서 무중단 업데이트

RollingUpdate일 때 두 가지 핵심 파라미터 maxUnavailable, maxSurge가 등장합니다.

#### 9-2. maxUnavailable

- 업데이트 중에 최소 몇 개까지 Pod가 줄어들 수 있는지 (Unavailable 허용치).
- 숫자 또는 비율(%)로 지정 가능.

예: replicas=10, maxUnavailable=2 → 업데이트 중 최소 8개는 항상 살아 있어야 함.   

#### 9-3. maxSurge

- 업데이트 중에 최대 몇 개까지 Pod를 추가로 더 만들 수 있는지 (Surge 허용치).   
- 숫자 또는 비율(%)로 지정 가능.

예: replicas=10, maxSurge=3 → 업데이트 중 순간적으로 최대 13개 Pod까지 늘어날 수 있음.   

#### 9-4. 예시 시나리오

Deployment 매니페스트 예시:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 1
```

replicas=5 라고 하면:

- 업데이트 시작 시, 새 Pod 1개 추가 (+1, maxSurge=1) → 총 6개
- 기존 Pod 1개 제거 (-1, maxUnavailable=1) → 다시 5개
- 이 과정을 반복해서 순차적으로 교체

#### 9-5. 그림으로 이해하기
```
설정: replicas=3, maxUnavailable=1, maxSurge=1
초기: Pod v1 x3
업데이트 시작
- 새 Pod v2 +1 (총 4개)
- 기존 Pod v1 -1 (총 3개)
- 반복
최종: Pod v2 x3
```

👉 항상 최소 2개 Pod는 살아 있고, 최대 4개까지 늘어날 수 있음.

#### 9-6. ✅ 요약

- maxUnavailable → 업데이트 중 동시에 죽어도 되는 Pod 개수
- maxSurge → 업데이트 중 동시에 더 늘려도 되는 Pod 개수
- 둘을 조합해서 무중단 배포 보장 + 리소스 효율성을 조절함

### 10. maxUnavailable=0 vs maxSurge=0 인 경우를 비교

두 설정은 업데이트 방식이 완전히 달라지기 때문에 운영 환경에서 중요한 의미가 있어요.

#### 10-1. 📌 Case 1: maxUnavailable = 0

👉 업데이트 중 Pod가 절대 줄어들지 않음. 즉, 기존 Pod를 죽이기 전에 새 Pod를 먼저 띄움.   

```
예: replicas=3, maxUnavailable=0, maxSurge=1

초기: v1 Pod x3

업데이트:
- 새 Pod v2 +1 (surge 허용) → 총 4개
- 기존 Pod v1 -1 → 총 3개
- 반복

최종: v2 Pod x3
```

##### 10-1-1. ✅ 장점

- 항상 원하는 replica 수(3개)가 보장됨 → 무중단 배포에 안전
- 다운타임 위험 최소화

##### 10-1-2. ❌ 단점

- 순간적으로 Pod 수가 늘어남 → 리소스(메모리/CPU) 여유 필요
- 클러스터 자원이 부족하면 배포가 지연될 수 있음

#### 10-2. 📌 Case 2: maxSurge = 0

👉 업데이트 중 Pod를 더 이상 늘리지 않음. 즉, 기존 Pod를 줄이고 나서 새 Pod를 띄움.   

```
예: replicas=3, maxUnavailable=1, maxSurge=0

초기: v1 Pod x3

업데이트:
- 기존 Pod v1 -1 (unavailable 허용) → 총 2개
- 새 Pod v2 +1 → 총 3개
- 반복

최종: v2 Pod x3
```

##### 10-2-1. ✅ 장점

- Pod 개수가 늘어나지 않음 → 리소스 절약
- 작은 클러스터에서도 안정적으로 동작

##### 10-2-2. ❌ 단점

- 업데이트 도중 가용 Pod 수가 줄어들 수 있음 → 서비스 순간적 성능 저하 가능성 있음
- maxUnavailable=1이라면 replicas=3일 때 최소 2개만 살아있게 됨

#### 10-3. 📌 비교 요약
|설정|동작 방식|장점|단점|적합한 상황|
|---|---|---|---|---|
|maxUnavailable=0|기존 Pod 줄이지 않고 새 Pod 먼저 생성|무중단 배포 보장|순간적으로 리소스 더 필요|서비스 다운타임 절대 허용 불가|
|maxSurge=0|기존 Pod 줄이고 나서 새 Pod 생성|리소스 절약|순간적으로 Pod 수 줄어듦 (성능 저하 가능)|자원이 빡빡한 클러스터|

#### 10-4. ✅ 정리하면

- maxUnavailable=0 → 다운타임 방지 우선
- maxSurge=0 → 리소스 절약 우선
