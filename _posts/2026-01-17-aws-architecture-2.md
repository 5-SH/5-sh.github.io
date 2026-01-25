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
애플리케이션 소스코드를 작성하듯이 인프라 구성을 코드로 작성하고 자동화 도구로 배포하는 방식이다.   
IaC를 사용하면 배포를 자동화 해 반복적인 수동 작업을 제거할 수 있고 배포 시간을 단축할 수 있다.   
Git으로 인프라 변경 이력을 추적하고 관리할 수 있고 같은 패턴을 개발, 검증, 운영 등 여러 환경에 쉽게 적용할 수 있다.    

### 3-1. CloudFormation
AWS에서 만든 IaC 서비스이다. AWS 아키텍처를 CloudFormation 파일로 만들고 git에서 관리해 쉽게 배포할 수 있다.   

### 3-2. Elastic Beanstalk
정형화된 AWS 아키텍처를 CloudFormation으로 제공해 사용자가 쉽게 서비스를 배포할 수 있도록 돕는다.   
애플리케이션 배포에 최적화 된 CloudFormation Wrapper 역할이다.   

## 4. DB 성능을 높이는 구조
CloudFormation을 사용해 AWS 인프라를 코드로 배포하고 CloudTrail과 CloudWatch를 서비스 운영과 모니터링에 사용한다.    
한 동안 서비스가 문제 없이 동작 했지만 서비스가 성장하고 사용자의 요청이 늘어나 RDS의 리소스가 부족해 성능이 느려졌다.   
CloudWatch에서 RDS의 CPU 사용률이 60%를 넘어갔다고 알림이 왔다.   
<br/>
RDS 서버의 성능을 좋은 것으로 바꿔 해결(Scale-up)할 수 있지만 한계가 있다.   
RDS 서버가 요청을 버티지 못하고 다운이 되더라도 Multi-AZ가 Failover를 하고 장애를 최소화 하지만 결국 요청을 모두 처리하지 못해 다시 RDS가 다운될 수 있다.   
이 경우 RDS서버를 늘려 요청을 분산(Scale-out)해 문제를 해결해야 한다. AWS는 Read Replica 기능으로 읽기 전용 RDS를 추가할 수 있다.   
보통 애플리케이션은 RDS에 읽기 작업을 주로 요청한다. 쓰기는 Primary DB에 하고 읽기는 Read Replica에서 해 요청 처리량을 늘릴 수 있다.   
RDS에서 Primary DB와 Read Replica의 데이터 동기화는 비동기식 복제를 사용한다.   

### 4-1. Read Replica
AWS는 하나의 Primary DB에서 최대 15개 까지 Read Replica를 생성할 수 있다. MySQL, PostgreSQL, MariaDB, Oracle, SQL Server에서만 지원된다.   
같은 리전, 다른 리전, 다른 AWS 계정에 Read Replica를 생성할 수 있다.   
Read Replica는 읽기만 가능하고 쓰기를 할 수 없다.   
Read Replica도 독립적인 인스턴스 이므로 추가 비용이 발생한다.   
그리고 Read Replica를 독립적인 다른 Primary DB로 승격시킬 수 있다. 장애 발생 시 Read Replica를 Primary DB로 승격해 빠르게 서비스를 복구할 수 있다.   

<figure>
  <img src="https://i.imgur.com/hX2RZWO.png" width="100%" alt=""/>
  <p style="font-style: italic; color: gray;">Read Replica 추가한 구조</p>
</figure>

### 4-2. ElasticCache
Read Replica를 15개 추가 했지만 사용자의 요청이 늘어나 성능향상이 추가로 필요하다.   
RDS 앞에 캐시인 ElasticCache를 추가해 RDS 서버의 부하를 줄일 수 있다.   
ElasticCache는 AWS에서 제공하는 완전관리형 인메모리 캐싱 서비스이다. Redis, Memcached를 지원한다.   
ElasticCache는 인메모리 DB처럼 동작하기 때문에, 애플리케이션에서 ElasticCache를 먼저 조회를 하고 없는 경우 RDS 서버에 요청하도록 코드를 수정해야 한다. 

<figure>
  <img src="https://i.imgur.com/XHd0NNS.png" width="100%" alt=""/>
  <p style="font-style: italic; color: gray;">ElasticCache 추가한 구조</p>
</figure>

## 5. 데이터베이스 분산 구조
Read Replica를 추가하고 ElasticCache를 추가해도 수백만 수준의 요청을 받으면 RDS 서버가 다운되어 서비스에 전면 장애가 발생할 수 있다.   
인스턴스 타입을 변경해 Scale-up 하고 Read Replica로 Scale-out 했지만 한계가 있다.   
이 경우 샤딩을 도입해 문제를 해결할 수 있다.   
MySQL, Maria DB, Oracle 같은 RDBMS는 직접 샤딩을 구현해 해결해야 하지만 AWS DynamoDB는 자동으로 샤딩을 설정해 준다.   
DynamoDB는 무한한 성능에 관리가 필요없고 key/value 형태로 데이터를 저장하는 NoSQL DB이다.   

### 5-1. DynamoDB
DynamoDB는 대규모 데이터와 높은 처리량을 효율적으로 처리하기 위해 데이터를 여러 파티션으로 나눠 분산 저장하고 관리한다.   
파티션은 DynamoDB 내부의 논리적인 단위로서 DynamoDB 내부 단위로 데이터와 처리량을 분산하는 역할을 한다.   
각 파티션은 독립적으로 동작하고, 최대 10GB의 데이터와 초당 1,000개의 쓰기 또는 3,000개의 읽기를 처리할 수 있다.   
DynamoDB는 사용자가 명시적으로 샤딩을 구성하지 않아도 자동으로 파티션을 관리한다.   
데이터와 트래픽이 증가하면 DynamoDB가 자동으로 파티션을 분할하고 데이터와 처리량을 새 파티션들에 분배한다. 애플리케이션은 별도의 수정 없이 자동으로 분산된 데이터에 접근할 수 있다.   
분할된 파티션은 여러 노드에 걸쳐 생성되고 삭제될 수 있고 DynamoDB 내부에서 관리하기 때문에 사용자는 파티션의 위치를 알 필요 없이 DynamoDB 엔드포인트에 연결해 요청만 하면 된다.    
DynamoDB의 샤딩은 파티션 키를 기반으로 동작한다. 같은 파티션 키 값은 같은 파티션에 저장되기 때문에 파티션 키를 적절히 설정해야 한다.   
그리고 모든 파티션은 최소 3개의 AZ에 동기식 복제되므로 항상 Multi-AZ로 동작한다.   
<br/>
DynamoDB는 기존 RDS와 다르게 key/value 형식이므로 독립적인 테이블 먼저 DynamoDB로 옮긴다.   

<figure>
  <img src="https://i.imgur.com/4jyfnVV.png" width="100%" alt=""/>
  <p style="font-style: italic; color: gray;">DynamoDB 추가한 구조</p>
</figure>

### 5-2. DynamoDB Accelerator(DAX)
DAX는 DynamoDB 앞에 배치되는 인메모리 캐시 서비스로서 읽기 성능을 향상시킨다.   
DAX는 DynamoDB에만 사용할 수 있고 DynamoDb와 자동 통합되어 쓰기 시 자동으로 캐시를 무효화 할 수 있다.   
DynamoDB 클라이언트를 DAX로 대체하면 자동으로 캐싱이 된다.    
따라서 ElasticCache 처럼 애플리케이션에서 캐시 hit 유무를 판단하는 조건문이 필요 없고 애플리케이션에서 연결하는 엔드포인트를 DynamoDB에서 DAX로 변경하면 된다.   

<figure>
  <img src="https://i.imgur.com/N5PFf8W.png" width="100%" alt=""/>
  <p style="font-style: italic; color: gray;">DAX 추가한 구조</p>
</figure>

## 6. 암호화, 컨플라이언스, 패치
### 6-1. Key Management Service(KMS)
데이터베이스에 있는 개인 정보가 유출될 경우 소송 등 큰 사고가 발생할 수 있다.  
스토리지에 있는 데이터들을 암호화해야 하고 암호화를 위해 키가 필요하다.   
AWS는 암호화 키를 관리하는 KMS를 제공한다.    
KMS는 암호화 키인 Customer Managed Key(CMK)를 생성, 저장하고 키는 HSM 기반 하드웨어 안에서 보호된다.    
HSM은 암호화 키를 저장, 연상하는 전용 보안 정비이다. 사용자는 HSM에 직접 접근할 수 없고 API를 통해서만 접근할 수 있다. 키는 AWS 밖으로 나갈 수 없다.   
<br/>
4KB 이하의 짧은 데이터는 KMS에서 직접 암호화, 복호화 할 수 있지만 성능과 네트워크 통신 비용이 커 실무에서는 대부분 직접하지 않는다.   
대부분의 경우 Envelop Encryption을 사용한다. Envelop Encryption은 아래 단계로 동작한다.   

1. 앱 → KMS: GenerateDataKey 요청
    - DataKey는 AES 같은 대칭키 이고 실제 데이터를 암호화, 복호화 하는데 사용한다.   
    - 256bit 크기의 키이고 한 번 사용하고 버린다.   
2. KMS(HSM) → 앱: 평문 DataKey, 암호화된 DataKey를 전달한다.
    - 암호화 할 때,
      - 평문 DataKey는 암호화되지 않은 키이고 네트워크를 통해 전달 받을 때 TLS로 보호된다.
      - 암호화를 할 때 앱의 메모리에 잠깐 존재하고 사용 후 즉시 파기한다.
    - 복호화 할 때,
      - 암호화된 DataKey는 평문 DataKey를 KMS Master Key(CMK)로 암호화 한 것이다.
      - 평문이 아니라서 탈취돼도 안전하고 데이터 복호화를 위해 앱에서 저장한다.   
      - 데이터를 복호화 하기 위해 암호화된 DataKey를 KMS로 전달해 평문 DataKey를 받고 복호화에 사용한 뒤 즉시 파기한다.
    - 평문 DataKey를 직접 전달하는 이유는 파일 등으로 저장하지 않고 앱의 메모리 내에서만 존재한다면 안전하다고 간주하기 때문이다. 
    - 즉, "애플리케이션의 런타임은 신뢰 영역이다." 라고 간주한다.

### 6-2. AWS Config
AWS 앱에서 데이터의 암호화, 복호화를 강제하도록 하기 위해 AWS Config를 사용할 수 있다.   
AWS Config는 AWS 내 리소스들의 Complicance를 만들고 지켜지지 않으면 경고 알림을 보내거나 막도록 하는 기능을 제공한다.   
AWS Config는 Config Rule을 사용해 컴플라이언스를 지키는 지 검사한다.   
KMS를 사용한 암호화 유무를 검사하기 위해 S3는 s3-bucket-server-side-encryption-enabled, EBS는 encrypted-volumes, RDS는 rds-storage-encrypted 등의 Config Rule를 사용한다.   

### 6-3. AWS Systems Manager(SSM)
OS, 데이터베이스 등의 보안 취약점이 발견되어 패치가 필요할 수 있다.   
이 때 패치해야할 서버가 매우 많을 때 각 서버에 접근해 명령어를 실행하면 오랜 시간이 걸리고 실수가 발생할 수 있다.   
AWS는 서버들을 전체 관리하기 위해 AWS Systems Manager를 제공한다.   
특정 시간에 업데이트를 계획하는 Maintenance, 패치를 자동화하는 Patch Manager 그리고 모든 서버에 간단한 명령어를 날릴 수 있게 하는 Run Command 라는 서비스가 있다.   
<br/>
SSM으로 취약한 PostgreSQL 버전을 탐지해서 업그레이드 하는 과정은 아래와 같다.   

1. SSM에서 취약한 PostgreSQL 버전을 탐지
    - AWS는 내부적으로 CVE 정보를 갖고 있다.
    - Patch Manager + Amazon Inspector로 취약점을 탐지한다.
    - Inspector가 EC2를 스캔해 버전 정보를 확인하면 CVE 정보와 비교해 탐지한다.
2. 업그레이드 전략 수립
    - SSM Automation + Maintenance Window
    - Maintenance Window는 언제 패치 작업을 수행할지 계획한다.
    - 실제 업그레이드는 SSM Automation Document(Runbook)가 실행한다. 단계별로 성공/실패를 기록하고 실패 시 자동 중단, 알림, 이전 스냅샷으로 복구 등을 지원한다.
    - SSM Automation Document(Runbook)은 AWS 리소스에 대해 미리 정의된 운영 절차를 코드로 실행하는 문서이다.
    ```
    schemaVersion: '0.3'
    description: >
      PostgreSQL security patch upgrade runbook.
      Stops PostgreSQL, takes EBS snapshot, upgrades package,
      restarts service, and verifies version.

    parameters:
      InstanceId:
        type: String
        description: PostgreSQL가 설치된 EC2 인스턴스 ID

      DataVolumeId:
        type: String
        description: PostgreSQL 데이터가 있는 EBS 볼륨 ID

    mainSteps:

      # 1️⃣ PostgreSQL 서비스 중지
      # 데이터 파일 일관성을 보장하기 위해 먼저 DB를 정지
      - name: StopPostgreSQL
        action: aws:runCommand
        inputs:
          DocumentName: AWS-RunShellScript
          InstanceIds:
            - "{{ InstanceId }}"
          Parameters:
            commands:
              - systemctl stop postgresql

      # 2️⃣ 데이터 볼륨 스냅샷 생성
      # 업그레이드 실패 시 복구를 위한 롤백 포인트
      - name: SnapshotDataVolume
        action: aws:createSnapshot
        inputs:
          VolumeId: "{{ DataVolumeId }}"
          Description: "Pre-PostgreSQL-upgrade snapshot"

      # 3️⃣ PostgreSQL 패키지 업그레이드
      # OS 패키지 매니저를 통해 보안 패치 적용
      - name: UpgradePostgreSQL
        action: aws:runCommand
        inputs:
          DocumentName: AWS-RunShellScript
          InstanceIds:
            - "{{ InstanceId }}"
          Parameters:
            commands:
              - yum update -y postgresql

      # 4️⃣ PostgreSQL 서비스 재시작
      # 업그레이드된 바이너리로 DB 기동
      - name: StartPostgreSQL
        action: aws:runCommand
        inputs:
          DocumentName: AWS-RunShellScript
          InstanceIds:
            - "{{ InstanceId }}"
          Parameters:
            commands:
              - systemctl start postgresql

      # 5️⃣ PostgreSQL 버전 확인
      # 실제로 보안 패치가 적용되었는지 검증
      - name: VerifyPostgreSQLVersion
        action: aws:runCommand
        inputs:
          DocumentName: AWS-RunShellScript
          InstanceIds:
            - "{{ InstanceId }}"
          Parameters:
            commands:
              - psql --version

    ```
    - 완료 후 SSM Compliance 상태를 업데이트 한다.