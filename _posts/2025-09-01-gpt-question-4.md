---
layout: post
title: Chat GPT 질문 4
date: 2025-09-01 23:00:00 + 0900
categories: [question]
tags: [spring, jpa]
---

### 1. Spring Boot Actuator와 k8s readiness probe 연계

Spring Boot Actuator와 Kubernetes Readiness Probe는 자동으로 100% 연계되는 건 아니에요.    
하지만 Actuator가 제공하는 헬스 체크 엔드포인트를 readinessProbe에서 활용하는 방식으로 자연스럽게 붙일 수 있습니다.   

#### 1-1. Spring Boot Actuator의 기본 동작

Actuator를 의존성에 추가하면 /actuator/health 엔드포인트가 생깁니다.   
기본 /actuator/health는 애플리케이션이 정상 기동 중인지만 알려줍니다.   

예:

```json
{
  "status": "UP"
}
```

DB, Kafka, Redis 같은 의존성 빈이 있다면 자동으로 헬스체크에 포함됩니다.   

#### 1-2. Kubernetes Readiness Probe

readinessProbe는 "Pod가 트래픽을 받을 준비가 되었는가?"를 판단합니다.   

보통 아래처럼 Deployment 매니페스트에 넣습니다:   

```yaml
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
```

#### 1-3. Spring Boot ↔ Kubernetes 연계

Spring Boot 2.3+ 부터는 Kubernetes 친화적 프로브 개념이 들어왔습니다.   

/actuator/health/liveness   
/actuator/health/readiness   

management.endpoint.health.probes.enabled=true 설정을 하면 자동으로 위 두 엔드포인트가 활성화됩니다.   

따라서 Kubernetes에서 별도의 코드 작성 없이도 readinessProbe를 /actuator/health/readiness에 맞추면 바로 연동됩니다.   

#### 1-4. 자동 연계의 한계

Spring Boot가 readinessProbe를 직접 쿠버네티스 매니페스트에 넣어주진 않습니다.   
즉, Deployment.yaml에서 readinessProbe를 설정하는 건 개발자/운영자가 직접 해줘야 합니다.   
다만 엔드포인트(/actuator/health/readiness)는 Spring Boot가 제공하므로, Kubernetes 쪽 설정만 맞추면 자동으로 연계되는 느낌을 받을 수 있습니다.   

#### 1-5. ✅ 정리

Spring Boot Actuator가 readiness/liveness 전용 엔드포인트를 제공합니다.   
하지만 Kubernetes 매니페스트에서 readinessProbe를 명시해야 하며, Spring Boot가 자동 등록해주진 않습니다.   
따라서 "자동 연계"라기보다는, Spring Boot가 프로브 엔드포인트를 제공 → Kubernetes가 이를 활용하는 구조라고 이해하면 정확합니다.   

### 2. Actuator 버전에 따른 readiness probe 설정

#### 2-1. Spring Boot 2.3 ~ 2.6

spring-boot-starter-actuator 의존성을 추가하면 /actuator/health 는 기본 제공 ✅   
하지만 Kubernetes 친화적인 /health/liveness, /health/readiness 엔드포인트는 기본적으로 꺼져 있음 ❌   

따라서 반드시 아래 설정을 해줘야 활성화됩니다:

```yaml
management.endpoint.health.probes.enabled=true
```

이 설정을 켜야 /actuator/health/liveness, /actuator/health/readiness 경로가 생깁니다.   

#### 2-2. Spring Boot 2.7+

Spring Boot 2.7부터는 management.endpoint.health.probes.enabled=true 가 기본값이 true 입니다 ✅   
즉, actuator 의존성만 추가하면 /actuator/health/readiness 와 /actuator/health/liveness 가 자동 활성화됩니다.   

#### 2-3. Spring Boot 3.x

Spring Boot 3.x (Spring Boot 3.0, 3.1, 3.2) 도 동일하게 기본 활성화 상태입니다.   
즉, 별도의 설정 없이도 readiness/liveness 엔드포인트가 생깁니다.   
다만, 커스터마이징을 원한다면 management.endpoint.health.probes.enabled=false 로 꺼줄 수도 있습니다.   

#### 2-4. ✅ 정리

- Spring Boot 2.3 ~ 2.6 → management.endpoint.health.probes.enabled=true 필요   
- Spring Boot 2.7+ → 기본적으로 활성화됨, 따로 설정 안 해도 됨   