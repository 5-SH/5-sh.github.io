---
layout: post
title: MSA Deploy (3) - devOps & Monitoring
date: 2026-05-05 20:00:00 + 0900
categories: [kubernetes]
tags: [kubernetes, msa]
mermaid: true
---

# 5. devOps 환경
virtualbox에 vagrant로 프로비저닝한 환경에 harbor와 jenkins를 구성하기엔 리소스가 모자랄 것으로 예상해 로컬 PC 기반 CI/CD + k8s 배포 파이프라인 구축   

```
[개발 코드]
   ↓
[Jenkins (빌드)]
   ↓
[Docker 이미지 생성]
   ↓
[Harbor (이미지 저장소)]
   ↓
[Kubernetes (이미지 pull → 배포)]
```
## 5-1. 작업 순서
1. Harbor 구축 (이미지 저장소)
- Docker Registry 역할
- k8s가 여기서 이미지 pull

2. Jenkins 구축 (CI/CD 서버)
- 코드 빌드
- Docker 이미지 생성
- Harbor로 push

3. Docker ↔ Harbor 연결
- docker login
- 이미지 push 테스트

4. Kubernetes ↔ Harbor 연결
- imagePullSecret 설정
- Pod에서 Harbor 이미지 pull 가능하게

5. Jenkins Pipeline 구성 (Jenkinsfile)
- git pull
- build
- docker build
- docker push
- kubectl apply

## 5-2. harbor

### 5-2-1. 설치

harbor는 nginx(gateway), harbor-core(API), harbor-db(postgres), redis, registry로 구성된 마이크로 서비스 구조   
여러 컨테이너를 묶어서 실행하기 위해 docker compose로 설치   

```bash
wsl --install -d Ubuntu
```

Docker Desktop WSL integration 기능이 Ubuntu를 Docker CLI를 사용 가능한 상태로 연결해 Ubuntu WSL에서 작업한 내용이 docker-desktop에 전달됨   

```
[Ubuntu WSL]  ← 작업하는 환경
      ↓
[docker CLI]
      ↓
[docker-desktop WSL (Docker Engine)]
      ↓
[컨테이너 실행]
```

### docker-desktop WLS 특징

- 도커를 돌리기 위한 내부 OS
- 실제 Docker 서버(엔진)
- Docker 엔진 전용 내부 환경
- 최소 구성 (busybox 수준)
- 패키지 설치 거의 불가
- sudo, apt, systemctl 없음
- Harbor 같은 복잡한 서비스 설치 불가

### Ubuntu WSL이 필요한 이유

- 실제 서버처럼 사용하는 리눅스
- 완전한 리눅스 환경
- apt 사용 가능
- sudo 가능
- 패키지 설치 가능
- Harbor / Jenkins 설치 가능

### Ubuntu WSL 초기설정

```PowerShell
wsl.exe -d Ubuntu
```

```bash
# harbor 다운로드
wget https://github.com/goharbor/harbor/releases/download/v2.15.0/harbor-online-installer-v2.15.0.tgz

# 압축 해제 
tar xzvf harbor-online-installer-v2.15.0.tgz

# 설정 파일 생성
cd harbor
cp harbor.yml.tmpl harbor.yml
```

마스터, 워커 노드에서 접근할 수 있도록 harbor.yml에서 hostname을 수정하고 https를 비활성화

```yaml
vim harbor.yml

hostname: localhost
...
# https:
#   port: 443
#   certificate: /your/cert/path
#   private_key: /your/key/path
```

설치 후 harbor 위치 이동
```bash
mv /mnt/c/harbor ~/
```

설치 성공 확인

```bash
gram@DESKTOP-TQV062L:~/harbor$ docker ps
CONTAINER ID   IMAGE                                 COMMAND                  CREATED          STATUS                             PORTS                       NAMES
11379e03240b   goharbor/harbor-jobservice:v2.15.0    "/harbor/entrypoint.…"   15 seconds ago   Up 9 seconds (health: starting)                                harbor-jobservice
b2111739189c   goharbor/nginx-photon:v2.15.0         "nginx -g 'daemon of…"   15 seconds ago   Up 12 seconds (health: starting)   0.0.0.0:80->8080/tcp        nginx
09257b053a54   goharbor/harbor-core:v2.15.0          "/harbor/entrypoint.…"   15 seconds ago   Up 13 seconds (health: starting)                               harbor-core
b8a8112c071a   goharbor/harbor-registryctl:v2.15.0   "/home/harbor/start.…"   15 seconds ago   Up 13 seconds (health: starting)                               registryctl
82c7b42d56bd   goharbor/registry-photon:v2.15.0      "/home/harbor/entryp…"   15 seconds ago   Up 13 seconds (health: starting)                               registry
a8a3a86ebcab   goharbor/harbor-portal:v2.15.0        "nginx -g 'daemon of…"   15 seconds ago   Up 13 seconds (health: starting)                               harbor-portal
00a9c89521b9   goharbor/redis-photon:v2.15.0         "redis-server /etc/r…"   15 seconds ago   Up 13 seconds (health: starting)                               redis
f048b958637b   goharbor/harbor-db:v2.15.0            "/docker-entrypoint.…"   15 seconds ago   Up 13 seconds (health: starting)                               harbor-db
1c504cd1c846   goharbor/harbor-log:v2.15.0           "/bin/sh -c /usr/loc…"   15 seconds ago   Up 14 seconds (health: starting)   127.0.0.1:1514->10514/tcp   harbor-log
```

로컬PC에서 harbor 접속 확인   

<figure>
  <img src="https://i.imgur.com/BluwlkW.png" width="100%" alt=""/>
  <p style="font-style: italic; color: gray;">harbor 접속</p>
</figure>

Harbor 설치 환경   

- Windows + WSL2(Ubuntu)
- Docker Desktop
- Harbor v2.15.0
- Kubernetes v1.18.4
- Docker runtime 18.09.9

### 5-2-2. Member 서비스 이미지 push

MSA 환경에서 Docker 이미지를 중앙 저장소로 관리하기 위해 Harbor를 구축하고,    
Vagrant 기반 Kubernetes Cluster에서 Harbor 이미지를 Pull 하도록 구성했다.   

구성 환경은 아래와 같다.

```text
[ Windows PC ]
 └─ Docker Desktop
     └─ Harbor (192.168.56.1:8080)

[ Vagrant Kubernetes Cluster ]
 ├─ m-k8s
 ├─ w1-k8s
 ├─ w2-k8s
 └─ w3-k8s
```

1. Harbor API 확인   

```bash
curl http://192.168.56.1:8080/api/v2.0/ping
# 정상 응답:
Pong
```

2. Harbor UI에 접속해 프로젝트 생성

<figure>
  <img src="https://i.imgur.com/hCUlWBd.png" width="100%" alt=""/>
  <p style="font-style: italic; color: gray;">프로젝트 생성</p>
</figure>

3. Docker insecure registry 설정

Harbor를 HTTP로 구성했기 때문에 Docker에서 insecure registry 설정이 필요하다.
적용 후 Docker Desktop을 재시작 해야한다.   

<figure>
  <img src="https://i.imgur.com/oml8Gw1.png" width="100%" alt=""/>
  <p style="font-style: italic; color: gray;">http 설정</p>
</figure>

<figure>
  <img src="https://i.imgur.com/lqIVBI8.png" width="100%" alt=""/>
  <p style="font-style: italic; color: gray;">docker 설정</p>
</figure>

4. Docker 이미지 Push
- 로컬에서 member-service 이미지 확인
<figure>
  <img src="https://i.imgur.com/f57Je8R.png" width="100%" alt=""/>
  <p style="font-style: italic; color: gray;">docker 이미지</p>
</figure>

- Harbor 용 태그 생성

```bash
PS C:\WINDOWS\system32> docker tag member-service:latest 192.168.56.1:8080/deploy-test-member/member-service:latest
PS C:\WINDOWS\system32> docker images
REPOSITORY                                            TAG       IMAGE ID       CREATED         SIZE
product-service                                       latest    0004eb27476e   3 weeks ago     433MB
ordering-service                                      latest    46f21531d0c3   3 weeks ago     434MB
192.168.56.1:8080/deploy-test-member/member-service   latest    fcaee27bcf5f   4 weeks ago     421MB
member-service                                        latest    fcaee27bcf5f   4 weeks ago     421MB
goharbor/redis-photon                                 v2.15.0   4ed296911e8b   7 weeks ago     173MB
goharbor/harbor-registryctl                           v2.15.0   4b2b3294a46c   7 weeks ago     166MB
goharbor/registry-photon                              v2.15.0   5b02673a2fa2   7 weeks ago     87.7MB
goharbor/harbor-log                                   v2.15.0   9e71d0bc1e73   7 weeks ago     170MB
goharbor/harbor-jobservice                            v2.15.0   2fe35692621f   7 weeks ago     185MB
goharbor/harbor-core                                  v2.15.0   87a862e24da8   7 weeks ago     210MB
goharbor/harbor-portal                                v2.15.0   b3939ccdd07f   7 weeks ago     166MB
goharbor/harbor-db                                    v2.15.0   e66097841983   7 weeks ago     274MB
goharbor/prepare                                      v2.15.0   029f75920a4b   7 weeks ago     199MB
goharbor/nginx-photon                                 v2.15.0   77728976f2e8   7 weeks ago     158MB
redis                                                 7.4       65750d044ac8   16 months ago   117MB
apache/kafka                                          3.8.0     b610bd8a193a   21 months ago   382MB
mysql                                                 8.0.38    6c54cbcf775a   22 months ago   572MB
```

- Harbor push

```bash
PS C:\WINDOWS\system32> docker login 192.168.56.1:8080
Username: admin

i Info → A Personal Access Token (PAT) can be used instead.
          To create a PAT, visit https://app.docker.com/settings


Password:

Login Succeeded
PS C:\WINDOWS\system32> docker push 192.168.56.1:8080/deploy-test-member/member-service:latest
The push refers to repository [192.168.56.1:8080/deploy-test-member/member-service]
d06ca5ee8d0b: Layer already exists
d24210e52761: Layer already exists
f7409d4c66cc: Layer already exists
08e98c779fb9: Layer already exists
0a828e8088fe: Layer already exists
344ae0b6479e: Layer already exists
989e799e6349: Layer already exists
latest: digest: sha256:cf421bb1ce160dc0b08b5f24118c47e3e565b05dc22facb1c57b1471276c7b48 size: 1786
```

- 이미지 push 확인

<figure>
  <img src="https://i.imgur.com/kZr5zw3.png" width="100%" alt=""/>
  <p style="font-style: italic; color: gray;">docker 이미지</p>
</figure>


## 5-3. jenkins

## 5-4. argocd

# 6. 모니터링
1. grafana

2. lgtm

3. slack

# 7. 그 외
1. 시크릿 설정

2. HTTPS 통신

3. DB 이중화 연결

4. 서비스용 DB 계정 생성

5. 젠킨스, harbor 구성

6. HPA
