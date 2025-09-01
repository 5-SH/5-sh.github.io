---
layout: post
title: Chat GPT 질문 3
date: 2025-09-01 21:30:00 + 0900
categories: [question]
tags: [spring, jpa]
---

### 1. 쿠버네티스에서 롤링 업데이트 중 구 버전 Pod가 종료될 때 처리 중인 요청이 어떻게 되는지?

#### 1-1. Pod 종료 과정

Pod가 종료될 때 쿠버네티스는 Graceful Termination 절차를 따릅니다.   

1) SIGTERM 신호 전송

2) kubelet이 Pod 내 컨테이너에 SIGTERM 신호를 보내 “종료 준비 시작”을 알림.

3) 컨테이너는 이 시그널을 받아 현재 처리 중인 요청을 완료하거나 정리 작업을 수행할 수 있음.

4) Termination Grace Period

5) Pod spec에 terminationGracePeriodSeconds 값에 따라 종료 대기 시간 부여 (기본 30초).

6) 이 시간 동안 컨테이너는 현재 요청을 처리하고 종료 준비 완료 가능.

7) Pod 종료 완료

8) grace period 내에 종료가 안 되면 SIGKILL을 보내 강제 종료.

#### 1-2. Service와 연결된 Pod

쿠버네티스 Service는 Pod의 Endpoints 목록을 관리함.   
Pod가 종료 신호를 받으면, Service Endpoint 목록에서 즉시 제외되지는 않고, kube-proxy가 트래픽을 전달할 수 있음.   
대부분의 경우 로드 밸런서/Ingress Controller가 Pod 상태를 체크하여 종료 중인 Pod로 새 요청이 가지 않도록 조절.   

#### 1-3. 처리 중인 요청

- 컨테이너 애플리케이션이 SIGTERM을 잘 처리하면: 요청 완료 후 종료 → 사용자는 정상 응답 수신   
- 애플리케이션이 SIGTERM 무시하거나 긴 요청 처리 중이면: grace period 종료 후 강제 종료(SIGKILL), 현재 처리 중이던 요청은 중단 → 클라이언트는 오류 발생 가능 (HTTP 5xx)   

#### 1-4. 실무 팁

##### 1-4-1. HTTP 서버에서 SIGTERM 처리

- Node.js: process.on('SIGTERM', ...)
- Java(Spring Boot): DisposableBean 또는 @PreDestroy
- Python(Flask/FastAPI): signal handler 등록

##### 1-4-2. 적절한 terminationGracePeriodSeconds 설정

- 애플리케이션 요청 처리 최대 시간 + 여유 5~10초 정도

##### 1-4-3. Readiness Probe 활용

- Pod 종료 시작 시 Readiness 상태를 false로 변경 → 서비스가 더 이상 새 요청을 보내지 않음
- 기존 요청만 처리 후 안전하게 종료 가능

#### 1-5. 정리

|단계|처리 방식|
|---|---|
|Pod 종료 시작|SIGTERM 전송, Readiness false → 신규 요청 차단|
|Graceful termination|현재 요청 처리 완료|
|Grace period 종료|아직 처리 중인 요청이 있으면 강제 종료(SIGKILL) → 오류 발생 가능|

#### 1-6. ✅ 결론:

- 앱이 SIGTERM을 제대로 처리하고, readiness probe를 사용하면 사용자는 오류 없이 요청 완료 가능
- 그렇지 않으면 처리 중 요청 일부는 실패할 수 있음

### 2. Spring Boot 애플리케이션이 SIGTERM 신호를 처리하는 과정

#### 2-1. Pod 종료 시 이벤트 순서

쿠버네티스에서 Pod를 종료하면 다음 단계가 발생합니다.

1. SIGTERM 전송

- kubelet이 컨테이너에 SIGTERM을 보냄.
- Spring Boot 애플리케이션이 이 신호를 수신.

2. Readiness 제거

- kubelet은 즉시 해당 Pod를 Endpoints(또는 EndpointsSlice)에서 제거 → 새로운 요청이 더 이상 들어오지 않음.

3. Grace Period 대기

- Pod 스펙의 terminationGracePeriodSeconds 동안 대기 (기본 30초).
- 이 시간 안에 기존 요청을 모두 마무리해야 함.

4. SIGKILL 전송

- grace period 내에 종료되지 않으면 SIGKILL로 강제 종료됨.

#### 2-2. Spring Boot에서 SIGTERM 처리 방식

Spring Boot(내부적으로는 Tomcat/Jetty/Undertow)는 SIGTERM을 받으면:   

- 새로운 요청 수락 중단 (stop accepting new connections)
- 이미 처리 중인 요청은 계속 실행 → 완료될 때까지 기다림 (Graceful Shutdown 지원)

Spring Boot 2.3+ (기본적으로 Graceful Shutdown 지원)

application.properties 에서 설정 가능:

```yaml
server.shutdown=graceful
spring.lifecycle.timeout-per-shutdown-phase=30s
```

- server.shutdown=graceful → 실행 중인 요청이 끝날 때까지 대기
- spring.lifecycle.timeout-per-shutdown-phase → 최대 대기 시간 (k8s terminationGracePeriodSeconds와 맞추면 안전)

#### 2-3. 흐름 요약 (Spring Boot + Kubernetes)

1. kubelet이 Pod에 SIGTERM 보냄
2. Spring Boot는 새 연결은 거절하고, 진행 중인 요청은 끝까지 처리
3. readinessProbe가 실패 처리됨 → Service에서 Pod 제거됨
4. 모든 요청이 끝나거나 timeout이 지나면 프로세스 종료

즉, 올바르게 설정하면 사용자는 오류 응답을 받지 않고 정상 응답을 받음 ✅

#### 2-4. 주의할 점

- Readiness Probe 미구현 → Pod가 여전히 트래픽을 받아서 요청이 중간에 끊길 수 있음 ❌
- Grace period가 너무 짧음 → 긴 요청 처리 중에 SIGKILL로 끊겨 5xx 에러 발생 가능 ❌

따라서 Readiness Probe + Graceful Shutdown 설정 + 충분한 terminationGracePeriodSeconds 조합이 필요.

#### 2-5. 👉 정리하면:
Spring Boot는 SIGTERM을 받으면 자동으로 처리 중인 요청을 완료하고 종료하는 메커니즘을 갖고 있고, 쿠버네티스와 잘 맞물려 동작해요.   

### 3. ser.shutdown=graceful 설정 시 Spring Boot의 내부 동작

server.shutdown=graceful을 켜면 Spring Boot가 종료 신호를 받았을 때의 내부 종료 절차가 “단계적·대기형(드레인)”으로 바뀝니다.    
핵심은 “새 연결은 막고, 진행 중 요청은 끝까지 처리한 뒤, 그 다음 컴포넌트들을 순서대로 내린다”는 거예요.   

#### 3-1. 무슨 일이 단계별로 일어나나?

##### 3-1-1. 전제

설정

```yaml
server.shutdown=graceful
spring.lifecycle.timeout-per-shutdown-phase=30s   # 각 종료 단계 대기 최대시간(기본 30초 권장)
```

쿠버네티스에서는 보통 terminationGracePeriodSeconds를 이 값보다 같거나 조금 크게 잡습니다.

##### 3-1-2. SIGTERM 수신 → 애플리케이션 종료 시퀀스 시작

- JVM의 Shutdown Hook을 통해 ApplicationContext.close()가 호출됩니다.   
- 이때부터 Spring Boot는 **“그레이스풀 종료 모드”**로 진입합니다.   

##### 3-1-3. 웹 서버(임베디드 서블릿/리액티브) 먼저 드레인

- 새 연결 수락 중지: 내장 톰캣/제티/언더토우/리액터 Netty가 더 이상 새 연결을 받지 않습니다(커넥터/리스너 일시정지).   
- Keep-Alive/HTTP/2도 정리: 기존 연결로는 새 요청을 더 받지 않도록 하고,   
- 진행 중 요청은 완료까지 대기: 이미 들어와 처리 중인 HTTP 요청·서블릿 비동기 작업·리액티브 파이프라인을 대기 시간 내에서 마칠 때까지 기다립니다.   
- 이 대기 시간의 상한이 ```spring.lifecycle.timeout-per-shutdown-phase```예요.   
-시간 내에 모두 끝나면 정상 종료(성공 드레인), 남아 있으면 강제 중단 단계로 넘어갑니다.   

포인트   

- “새 요청 차단”과 “진행 중 요청 대기”가 분리되어 있어서 사용자 오류를 최소화합니다.   
- WebSocket/SSE 등 장수 연결은 서버가 더 이상 유지하지 못하므로 시간이 지나면 끊깁니다(필요시 별도 타임아웃 설계).   

##### 3-1-4. 다음 단계 컴포넌트들을 “단계(Phase)” 순서로 종료

Spring은 SmartLifecycle의 phase 개념으로 외부 유입 차단 → 내부 소비 중지 → 리소스 해제 순서의 자연스러운 정지를 보장합니다.

일반적인 순서 느낌:   

1. 수신단(웹서버) 정지 → 새 트래픽 유입 차단
2. 메시지 리스너/스케줄러(@Scheduled, Kafka/Rabbit/JMS 등) 정지 → 더 이상 내부적으로도 작업이 늘어나지 않게
3. 서비스/리포지토리 계층 종료 준비
4. 데이터소스/커넥션 풀 등 리소스 해제
5. 빈 소멸: @PreDestroy / DisposableBean 실행

각 단계마다 해당 단계의 작업이 끝나길 지정 시간만큼 기다렸다가 다음 단계로 이동합니다(설정값이 그 “단계별 최대 대기 시간”).

##### 3-1-5. 시간 초과 시 어떻게?

- 대기 시간 내에 끝나지 않은 요청/작업이 있으면 그 단계는 타임아웃으로 넘어가고, 컨테이너/스레드풀이 강제 종료 수순으로 들어갑니다.
- 쿠버네티스 입장에선 grace period가 끝나면 SIGKILL이 올 수 있으므로,
terminationGracePeriodSeconds ≥ spring.lifecycle.timeout-per-shutdown-phase (+여유 5~10초) 권장.

#### 3-2. 실무 팁 (쿠버네티스와의 맞물림)

- 신규 요청 완벽 차단을 원하면, 종료 시작 시 Readiness를 즉시 false로(앱에서 처리).   
    - 예: ```@PreDestroy```나 ```ContextClosedEvent```에서 플래그 내려 ```/actuator/health```(custom readiness) 를 DOWN(503)으로 응답.
- 긴 작업(대용량 업로드, 배치 트리거, 장수 스트림)이 있다면 타임아웃 정책과 중단 가능 설계를 함께 두세요.   
- 메시지 컨슈머(Kafka 등)는 오프셋 커밋/작업 재시도가 안전하도록 정지 순서를 조정(리스너 먼저 멈춤 → 남은 메시지 처리 → 데이터 커밋).   

#### 3-3. 최소 구성 예시   

application.yml

```yaml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

Readiness 토글(예)

```java
@Component
public class ReadinessHealth implements org.springframework.boot.actuate.health.HealthIndicator {
  private volatile boolean ready = true;
  public void setReady(boolean r) { this.ready = r; }
  @Override public Health health() { return ready ? Health.up().build() : Health.down().build(); }
}

@Component
class ShutdownHook {
  private final ReadinessHealth readiness;
  ShutdownHook(ReadinessHealth readiness) { this.readiness = readiness; }

  @PreDestroy
  public void onShutdown() {
    // 1) 새 요청 차단(쿠버네티스 readiness에서 제외되도록)
    readiness.setReady(false);
    // 2) 이 시점부터는 Boot가 진행 중 요청 드레인 → 이후 단계별 종료
  }
}
```

#### 3-4. 한 줄 요약

```server.shutdown=graceful```은 **“수신 차단 → 진행 중 요청 대기 → 컴포넌트 단계별 종료(시간 제한 포함)”**로 종료 동작을 바꿔 무중단에 가까운 종료를 가능하게 합니다.   
쿠버네티스 사용 시에는 Readiness false 전환 + 적절한 grace/timeout 정합이 완성 키트예요.   

### 4. 롤링 업데이트 중 Pod 종료 시 요청이 안전하게 처리되도록 하는 예시(실제 쿠버네티스 매니페스트와 코드)

#### 4-1. Pod 매니페스트 설정

Deployment 예시

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app: myapp
    spec:
      terminationGracePeriodSeconds: 30  # Graceful termination 시간
      containers:
      - name: myapp
        image: myapp:1.0
        ports:
        - containerPort: 8080
        readinessProbe:              # Readiness Probe
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

포인트   

- terminationGracePeriodSeconds: 30 → 종료 시 최대 30초 동안 요청 처리 가능
- readinessProbe → 종료 신호 시 Service가 해당 Pod로 새 요청을 보내지 않도록 함

#### 4-2. 애플리케이션 코드 예시

Node.js (Express)

```javascript
const express = require('express');
const app = express();
let shuttingDown = false;

app.get('/health', (req, res) => {
  if (shuttingDown) return res.status(503).send('Shutting down');
  res.send('OK');
});

app.get('/data', (req, res) => {
  setTimeout(() => {            // 처리 중인 요청 시뮬레이션
    res.send('Data processed');
  }, 5000);
});

process.on('SIGTERM', () => {
  console.log('SIGTERM received, shutting down gracefully');
  shuttingDown = true;
  // 기존 요청 완료 후 종료
  setTimeout(() => {
    process.exit(0);
  }, 10000); // 예: 최대 10초 기다림
});

app.listen(8080, () => console.log('Server started'));
```

포인트   

- SIGTERM 감지 → Pod 종료 시작 시 실행
- shuttingDown 플래그 → Readiness Probe처럼 새로운 요청 차단
- setTimeout → 기존 요청 처리 완료 후 종료

#### 4-3. 동작 과정 요약

1. Argo CD/kubectl apply로 새로운 버전 배포
2. 기존 Pod에 SIGTERM 전송
3. Readiness Probe false → 서비스가 신규 요청 차단
4. 기존 요청 처리 완료
5. Grace period(terminationGracePeriodSeconds) 내 종료
6. 새 ReplicaSet Pod가 가용 상태로 올라와 트래픽 처리

#### 4-4. 그림으로 이해

```
Deployment v1 Pod
 ├─ SIGTERM sent
 ├─ Readiness false → 신규 요청 차단
 ├─ 기존 요청 처리 완료
 └─ Pod 종료

Deployment v2 Pod
 └─ 새 Pod 생성 → 트래픽 처리
```

##### 4-5. ✅ 핵심 포인트

- Readiness Probe + SIGTERM 처리 → 롤링 업데이트 중에도 사용자 요청 안전
- terminationGracePeriodSeconds → 최대 대기 시간 설정
- 롤링 업데이트 전략 (maxUnavailable, maxSurge)과 함께 사용하면 무중단 배포 가능

### 5. Argo CD를 사용해 롤링 업데이트할 때 Pod 상태가 어떻게 변하는지 시각화 그림으로 정리

#### 5-1. 시나리오

- Deployment: replicas=3
- 롤링 업데이트: 기존 v1 → 새로운 v2
- maxUnavailable=1, maxSurge=1
- terminationGracePeriodSeconds=30
- Readiness Probe 사용

#### 5-2. 초기 상태

```
Deployment (v1)
└─ Pod-1 (v1) [Ready]
└─ Pod-2 (v1) [Ready]
└─ Pod-3 (v1) [Ready]
```

Argo CD UI에서는 Deployment → ReplicaSet → Pod 구조로 보임.

#### 5-3. 업데이트 시작

새 ReplicaSet(v2) 생성

```
Deployment
├─ ReplicaSet (v1) → Pod-1~3
└─ ReplicaSet (v2) → Pod-4 (starting)
```

- maxSurge=1 → 순간적으로 Pod 4개까지 증가 가능   
- 새 Pod가 준비 상태(Ready) 될 때까지 기존 Pod는 그대로 처리 중   

#### 5-4. 새 Pod 준비 완료 → 기존 Pod 점진적 종료

```
Deployment
├─ ReplicaSet (v1) → Pod-1~2 [Ready]
├─ ReplicaSet (v1) → Pod-3 [Terminating, Graceful termination]
└─ ReplicaSet (v2) → Pod-4 [Ready]
```

- Pod-3: SIGTERM 받고, Readiness false → 새로운 요청 차단
- 기존 요청 처리 후 종료

#### 5-5. 반복 업데이트

```
Deployment
├─ ReplicaSet (v1) → Pod-1 [Ready]
├─ ReplicaSet (v1) → Pod-2 [Terminating]
└─ ReplicaSet (v2) → Pod-4~5 [Ready]
```

- 기존 Pod 하나씩 종료
- 새 Pod 하나씩 증가 → 항상 최소 2개 이상 Pod 가용

#### 5-6. 최종 상태

```
Deployment
├─ ReplicaSet (v1) → Pod-1~3 [0 Pod] (남아 있음, 롤백용)
└─ ReplicaSet (v2) → Pod-4~6 [Ready]
```

- 모든 트래픽은 새 ReplicaSet(v2)으로 처리
- 구 ReplicaSet(v1)은 Pod 0개 상태로 존재 → 롤백 가능

Argo CD UI 기준 구조

```
Application (myapp)
└─ Deployment (nginx)
    ├─ ReplicaSet (nginx-v1) [Pod x0..3]
    └─ ReplicaSet (nginx-v2) [Pod x3]
```

- Pod가 Ready 되면 초록색
- Terminating 중이면 회색/노란색
- 신규 Pod 생성 중이면 주황색 표시

롤링 업데이트 진행 상황을 UI에서 직관적으로 확인 가능

#### 5-7. ✅ 핵심 포인트

- SIGTERM + Readiness Probe → 요청 안전 처리
- maxUnavailable / maxSurge → Pod 수 관리
- Argo CD UI → Deployment → ReplicaSet → Pod 트리 구조로 상태 확인 가능
- 롤링 업데이트 중에도 다운타임 없이 배포 가능

### 6. Readiness Probe 처리

Readiness Probe 처리 쿠버네티스와 애플리케이션 코드가 서로 협력해서 동작합니다.   

#### 6-1. 기본 원리

1. 쿠버네티스가 Pod 상태를 확인
2. kubectl get pods에서 READY 상태를 관리
3. Service/Ingress가 트래픽을 보내기 전에 Pod가 준비되었는지 판단

#### 6-2. 애플리케이션이 준비 상태를 알려주는 방법

- HTTP GET, TCP Socket, Command 실행 등 Probe 방식 선택 가능   
- 애플리케이션에서 “준비 완료” 상태를 반환하면 쿠버네티스가 Ready로 표시

즉:   

```
[애플리케이션 코드] → 준비 완료 여부 제공
[kubelet / kube-proxy] → Probe로 확인 → Service에 트래픽 전달
```

#### 6-2. Probe 종류와 구현 방식

|Probe 종류|동작 방식|구현 필요 여부|
|---|---|---|
|HTTP GET|특정 URL 요청 → 200 OK 반환하면 Ready|URL 핸들러 구현 필요|
|TCP Socket|특정 포트 연결 가능 여부 확인|포트 열려 있어야 함 (특별한 코드 필요 없음)|
|Command|컨테이너 내부 명령 실행 → 0 반환 시 Ready|명령어 구현 필요|

#### 6-3. 예시

##### 6-3-1. HTTP Readiness Probe

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

쿠버네티스가 /health로 요청   
애플리케이션 코드에서 /health 핸들러 구현 필요   

Node.js 예시:   

```javascript
app.get('/health', (req, res) => {
  if (db.isConnected()) {
    res.status(200).send('OK');
  } else {
    res.status(503).send('Not Ready');
  }
});
```

- ✅ 준비 완료: 200 → Pod Ready → 서비스가 트래픽 보냄
- ❌ 준비 안 됨: 503 → Pod Not Ready → 트래픽 차단

##### 6-3-2. TCP Socket Readiness Probe   

```yaml
readinessProbe:
  tcpSocket:
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

애플리케이션이 포트를 열고 있으면 Ready   
HTTP 핸들러 구현 필요 없음   

Command Readiness Probe

```yaml
readinessProbe:
  exec:
    command:
      - cat
      - /tmp/ready
  initialDelaySeconds: 5
  periodSeconds: 5
```

/tmp/ready 파일이 존재하면 Ready   

애플리케이션에서 종료 신호 시 파일 삭제 등으로 Ready 상태 변경 가능   

#### 6-4. 결론

- 쿠버네티스: Probe를 주기적으로 실행, Pod Ready/Not Ready 판단, Service 트래픽 제어
- 개발자: 애플리케이션 코드에서 “준비 완료 여부”를 HTTP, TCP, Command 등으로 노출해야 함

즉, 쿠버네티스가 Probe를 호출하고, 앱은 상태를 알려주는 역할을 담당하는 구조예요.   

### 7. readiness probe을 구현하지 않으면 rolling update를 할 때 사용자의 요청이 처리되지 않을 수 있는지?

Readiness Probe를 구현하지 않으면 롤링 업데이트 중 사용자 요청이 안전하게 처리되지 않을 위험이 있어요. 

#### 7-1. Readiness Probe가 없는 경우

쿠버네티스는 Pod가 Ready인지 여부를 모름   

Deployment가 롤링 업데이트를 진행하면:   

1. 새 Pod를 생성

2. 기존 Pod를 종료 시작

하지만 서비스(Service)는 아직 종료 신호를 받은 Pod를 계속 Endpoint로 포함할 수 있음.   
→ 그 Pod로 새 요청이 들어갈 수 있음   

#### 7-2. 발생할 수 있는 문제

Pod가 초기화 중인데 요청 수신   

- 새 Pod가 준비되지 않았는데 트래픽이 전달되면 5xx 오류 발생 가능   

기존 Pod가 SIGTERM으로 종료 중인데 요청 수신   

- 기존 Pod는 종료 준비 중
- Graceful termination을 처리하지 않으면 진행 중인 요청이 강제 종료 → 오류 발생

#### 7-3. 실제 예시

```
replicas=3, maxUnavailable=1, maxSurge=1
```

- 기존 Pod 중 1개 종료 시작   
- Readiness Probe 없음 → 서비스는 아직 종료되는 Pod를 Endpoint로 유지
- 클라이언트 요청이 종료 중 Pod로 들어감 → 처리 실패 가능

즉, 롤링 업데이트 중 잠깐이라도 다운타임이나 5xx 오류 발생 가능   

#### 7-4. Readiness Probe의 역할

- Pod 종료 시, Readiness Probe를 false로 설정   
- 서비스가 해당 Pod로 트래픽을 더 이상 보내지 않음   
- 기존 요청만 처리 → 무중단 배포 가능   

```
[Deployment 롤링업데이트]
       │
       ├─ 새 Pod 생성
       │
       ├─ 기존 Pod 종료 시작 → Readiness false
       │
       ├─ 기존 요청 완료 후 Pod 종료
       │
       └─ 새 Pod Ready → 트래픽 처리
```

#### 7-5. ✅ 결론

- Readiness Probe 없으면 롤링 업데이트 중 요청 실패 가능
- Readiness Probe + Graceful Termination을 함께 구현해야 사용자에게 오류 없이 배포 가능

### 8. Spring Boot 애플리케이션의 Readiness Probe 설정

Spring Boot 자체는 Readiness Probe를 기본으로 활성화하지 않습니다.   

#### 8-1. Spring Boot Actuator

Spring Boot에서 헬스 체크, 메트릭, 상태 확인 등을 제공하는 모듈이 ```spring-boot-starter-actuator```입니다.   
기본적으로 ```/actuator/health``` 엔드포인트를 제공합니다.    

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info
  endpoint:
    health:
      show-details: never
```

```/actuator/health``` 요청 시 200 OK 반환 → Ready/Alive 여부 확인 가능   

하지만 쿠버네티스와 자동 연동되지는 않음.   
Pod 종료 시 자동으로 Readiness 상태를 false로 바꾸는 기능은 제공되지 않습니다.   

#### 8-2. Pod 종료(SIGTERM) 처리

Spring Boot 2.x 이상에서는 Graceful Shutdown 기능을 지원합니다.   

```yaml
server:
  shutdown: graceful
spring.lifecycle.timeout-per-shutdown-phase: 30s
```

Pod가 종료될 때:   

- Spring Boot가 SIGTERM 감지
- HTTP 서버 연결을 종료하지 않고 현재 요청 처리 후 종료
- 하지만 Readiness Probe를 false로 바꿔서 신규 요청을 차단하는 기능은 별도 구현 필요

#### 8-3. Kubernetes Readiness Probe와 연계

보통 Spring Boot + Kubernetes 환경에서는 다음과 같이 설정합니다.   

Deployment 예시

```yaml
readinessProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

- /actuator/health가 200이면 Ready → 서비스가 트래픽 보냄
- Pod 종료 시작 시 Spring Boot에서 Readiness Probe를 false로 바꾸려면 Actuator Health 상태를 동적으로 바꾸는 코드 필요

#### 8-4. 동적 Readiness 구현 예시 (Spring Boot)

```java
@Component
public class ReadinessHealthIndicator implements HealthIndicator {

    private boolean ready = true;

    public void setReady(boolean ready) {
        this.ready = ready;
    }

    @Override
    public Health health() {
        return ready ? Health.up().build() : Health.down().build();
    }
}
```

- 종료 시점에 setReady(false) 호출
- 쿠버네티스가 Probe 확인 → 신규 요청 차단

#### 8-5. ✅ 요약
|기능|지원 여부|
|---|---|
|/actuator/health|기본 제공 (Spring Boot Actuator)|
|Graceful shutdown|제공 (Spring Boot 2.x 이상)|
|Readiness Probe 상태 자동 변경|제공하지 않음 → 개발자가 구현 필요|

즉, Spring Boot 자체만으로는 “Pod 종료 시 Readiness false”가 자동 처리되지 않으므로, 무중단 배포를 위해서는 HealthIndicator + Graceful Shutdown 조합으로 구현해야 합니다.   

### 9. Spring Boot에서 setReady(false)를 호출하는 적절한 시점과 위치

#### 9-1. 호출 시점

setReady(false)는 Pod가 종료 신호(SIGTERM)를 받을 때 호출해야 합니다.   

이유:   

- 쿠버네티스는 Readiness Probe를 보고 서비스로 트래픽을 보냄
- Pod가 종료되기 전에 Readiness를 false로 바꿔야 신규 요청이 들어오지 않음
- 기존 요청만 처리 후 안전하게 종료 가능

즉, Graceful Shutdown 시작 시점에 호출   

#### 9-2. 호출 위치

- Spring Boot 2.x 이상 (Graceful Shutdown 지원)   
- Spring Boot에서는 애플리케이션 종료 시점에 hook 등록 가능   
- @PreDestroy 또는 SmartLifecycle/DisposableBean을 활용

예시: @PreDestroy

```java
@Component
public class ReadinessManager {

    private final ReadinessHealthIndicator readiness;

    public ReadinessManager(ReadinessHealthIndicator readiness) {
        this.readiness = readiness;
    }

    @PreDestroy
    public void onShutdown() {
        // 종료 시작 시 Readiness false
        readiness.setReady(false);
        System.out.println("Pod shutting down: Readiness set to false");
        // 기존 요청 처리 시간 확보
        try {
            Thread.sleep(10000); // 예: 10초 대기
        } catch (InterruptedException ignored) {}
    }
}
```

동작 순서   

1. Pod 종료 신호(SIGTERM) 수신
2. Spring Boot Graceful Shutdown 시작 → @PreDestroy 실행
3. readiness.setReady(false) → /actuator/health가 down 상태 반환
4. Service가 신규 요청을 더 이상 보내지 않음
5. 기존 요청 처리 후 Pod 종료

#### 9-3. 추가 팁

- Readiness와 Liveness 분리
    - Readiness: 트래픽 수신 가능 여부
    - Liveness: Pod 살아 있는지 여부 → 종료 시 Liveness는 그대로 true, Readiness만 false로 바꿔야 함

- Termination Grace Period 활용
    - Deployment의 terminationGracePeriodSeconds를 충분히 설정 → 기존 요청이 끝날 시간 확보
    - Thread.sleep 같은 직접 대기보다는 실제 서버에서 처리 중인 요청이 완료될 때까지 기다리는 방식이 더 안전

#### 9-4. ✅ 요약

|항목|권장 처리|
|---|---|
|호출 시점|Pod 종료(SIGTERM) 직후|
|호출 위치|@PreDestroy, DisposableBean, SmartLifecycle 등 종료 hook|
|목적|신규 요청 차단 → 기존 요청 처리 후 안전하게 종료|
|보조|terminationGracePeriodSeconds로 충분한 종료 시간 확보|


### 10. Argo CD의 “EP”

#### 10-1. Endpoints(EP)란?

- 쿠버네티스 Service와 Pod를 연결하는 실제 네트워크 엔드포인트 목록   
- Service는 클라이언트가 접근할 수 있는 가상 주소를 제공하고
- Endpoints는 실제 Pod IP:Port 목록을 담고 있음
    - Service (ClusterIP) → Endpoints → 실제 Pod

예시:

```
Service myapp-svc
└─ Endpoints
   ├─ 10.244.0.5:8080
   ├─ 10.244.0.6:8080
   └─ 10.244.0.7:8080
```

- Endpoints에 등록된 Pod만 Service를 통해 트래픽 수신 가능
- Pod가 Ready 상태가 아니면 Endpoints에서 제외됨

#### 10-2. Argo CD UI에서 보이는 EP

- Argo CD에서 Deployment나 Service를 선택하면 **연결된 Endpoints(Ep)**를 볼 수 있음
- Pod가 준비 완료(Ready) 상태이면 Endpoints에 표시
- Pod가 Terminating이거나 Readiness false이면 Endpoints에서 제거 → 트래픽 차단

#### 10-3. Endpoints와 Readiness Probe 관계

- Readiness Probe가 false → 해당 Pod는 Endpoints 목록에서 제거
- 결과: Service가 신규 요청을 보내지 않음 → 안전하게 롤링 업데이트 가능

```
Deployment 롤링 업데이트
┌─────────────┐
│ Service     │
└─────────────┘
       │
       ▼
┌─────────────┐
│ Endpoints   │  ← Ready Pod만 포함
└─────────────┘
       │
       ▼
┌─────────────┐
│ Pod         │
└─────────────┘
```

#### 10-4. 요약

|항목|의미|
|---|---|
|EP|Endpoints, Service와 연결된 실제 Pod IP:Port 목록|
|역할|Service → Pod 트래픽 라우팅|
|Readiness Probe 관계|Ready=false → EP에서 제거 → 신규 요청 차단|
|Argo CD UI|Deployment → EP → 실제 Pod 상태 확인 가능|

즉, Argo CD에서 보이는 EP는 현재 서비스로 트래픽을 받을 수 있는 실제 Pod 리스트라고 이해하면 돼요.   

### 11. endpointslice

#### 11-1. EndpointsSlice란?

- Kubernetes 1.17 이상에서 도입된 확장된 Endpoints 리소스
- 기존 Endpoints는 Service와 연결된 Pod IP 목록을 하나의 오브젝트에 모두 담음
- Pod 수가 많아지면 Endpoints 오브젝트가 커져 API 서버 성능 저하 가능

EndpointsSlice 특징   

- Service와 연결된 Pod IP:Port 목록을 작은 단위(slice)로 나눠서 관리
- 한 Service에 여러 EndpointsSlice가 존재할 수 있음
- 대규모 클러스터에서 성능과 확장성 개선 목적

#### 11-2. EndpointsSlice 구조

예시: Service myapp에 6개의 Pod가 있을 때   

```
EndpointsSlice-1
└─ Pod 10.244.0.5:8080
└─ Pod 10.244.0.6:8080
└─ Pod 10.244.0.7:8080

EndpointsSlice-2
└─ Pod 10.244.0.8:8080
└─ Pod 10.244.0.9:8080
└─ Pod 10.244.0.10:8080
```

- EndpointsSlice마다 최대 100 Pod 등 조각 단위로 관리
- Service는 여러 Slice를 합쳐서 트래픽을 라우팅

#### 11-3. Endpoints vs EndpointsSlice
|항목|Endpoints|EndpointsSlice|
|Pod 수|모든 Pod를 하나의 오브젝트에 저장|여러 Slice로 나눠 저장|
|대규모 지원|비효율적|효율적 (수천~만 Pod)|
|트래픽 라우팅|동일|동일|
|쿠버네티스 버전|모든 버전|1.17+|
|Argo CD에서 표시|EP로 보임|기본적으로는 Slice 단위로 표시되지 않고, EP와 연계됨|

#### 11-4. EndpointsSlice와 Readiness Probe

- Readiness=false → 해당 Pod는 EndpointsSlice에서 제거
- 서비스 트래픽은 Ready Pod만 받음
- 기존 Endpoints와 동작 원리는 동일하지만, 대규모 환경에서 성능과 확장성이 개선됨

#### 11-5. 요약

- Endpoints = Service와 연결된 Pod IP 목록 (전통적인 방식)
- EndpointsSlice = Endpoints를 여러 조각으로 나눈 버전 → 대규모 클러스터 최적화
- Readiness Probe + Graceful Shutdown과 함께 사용하면 롤링 업데이트 중에도 안전하게 트래픽 분배 가능

