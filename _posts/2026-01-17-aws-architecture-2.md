---
layout: post
title: AWS Architecture (2)
date: 2026-01-17 04:55:00 + 0900
categories: [aws]
tags: [aws, architecture]
mermaid: true
---

# AWS Architecture (2)
## 1. 보안을 위한 서비스
### 1-1. CloudTrail
AWS 계정 내에서 발생하는 API 호출 및 이벤트 기록을 자동으로 수집하고 저장해, 누가 언제 어떤 작업을 했는지 추적해 주는 서비스이다.   

### 1-2. Identity and Access Management(IAM)
AWS 사용자를 관리하는 서비스이다.   
Root 계정은 꼭 필요할 때 말고는 사용하지 않아야 하고 Multi-factor authentication(MFA)를 설정해 one-time 패스워드로 인증하는 것을 권장한다.   
Root 계정을 직접 사용하지 않고 IAM의 Role을 사용해 AdministratorAcess 권한이 있는 Admin 사용자를 만들어 운영에 사용하는 것이 권장된다.   

### 1-3. Access key, Secret Access key
AWS는 Management Console로 브라우저를 통하는 방법, Terminal에서 CLI 명령어를 사용하는 방법 그리고 SDK로 프로그래밍 해 사용하는 방법이 있다.   
Management Console은 Username/Password로 로그인하고 CLI, SDK는 Access key와 Secret Access Key를 사용해 접속한다.   
IAM의 Role 기능으로 임시자격 증명을 받아 서비스들 간에 Key 없이 호출할 수 있다.    

### 1-4. Secrets Manager
코드에 DB에 접근하기 위한 Username, Password가 있어 보안에 취약하다.   
Secrets Manager로 key, password 같은 문자열을 안전하게 관리할 수 있다.   
Secrets Manager에 DB password를 저장하고 코드에서 DB에 연결할 때 AWS API를 사용해 Secrets Manager에 DB password를 받아 연결한다.   

## 2. 모니터링, 알림 서비스
### 2-1. CloudWatch
CPU, 디스크, 네트워크 등의 운영에 필요한 지표를 모니터링 할 수 있다.   

### 2-2. SNS
CloudWatch의 운영 지표가 일정 수치를 넘어가면 알림을 받을 수 있다.   

## 3. Infrastructure as a Code(IaC)
### 3-1. CloudFormation

### 3-2. Elastic Beanstalk