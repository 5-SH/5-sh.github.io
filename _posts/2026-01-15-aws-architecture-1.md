---
layout: post
title: AWS Architecture (1)
date: 2026-01-15 19:30:00 + 0900
categories: [aws]
tags: [aws, architecture]
mermaid: true
---

# AWS Architecture (1)
## 1. 예전의 On-Premises 구조

```mermaid

graph LR
    LB[Load Balancer]

    subgraph WebAppCluster [Web + App Servers]
        direction TB
        WebApp1[Web + App 1]
        WebApp2[Web + App 2]
        WebApp3[Web + App ...]
    end

    DB[Database Server]

    LB --> WebAppCluster
    WebApp1 --> DB
    WebApp2 --> DB
    WebApp3 --> DB

 ```

쇼핑몰 서비스를 개발한다. 서비스 이용자가 늘어나 Web + App 서버를 증설하고 LB를 사용해 부하를 분산했다.   
서버를 증설할 때 마다 서버를 주문하고 설치하고 서비스를 올리고 LB를 설정하는 작업을 매번 해야한다.   
그리고 서버들을 관리할 데이터센터 같은 장소가 필요하다.   
<br/>
특별 할인 등의 이벤트로 서비스 이용자가 급증할 것으로 예상될 때 얼마나 많은 서버를 준비해야 하는지 예측하기 어렵다.   
서버를 많이 준비했지만 서비스 이용자가 줄어들면 서버 비용이 낭비된다.   
<br/>
서버를 관리하는 데이터센터 같은 장소에서 천재지변이 일어나면 서비스가 전면 중단될 수 있다.   
이런 구조를 On-Premises라 하고 이 구조의 문제를 해결하기 위해 클라우드 구조를 사용하게 되었다.   

## 2. AWS 클라우드 구조로 옮겨가기
### 2-1. 주로 사용하는 AWS 인스턴스
#### 2-1-1. Elastic Compute Cloud(EC2)
EC2는 AWS의 가상 서버 서비스이다.   
원하는 CPU, 메모리, 저장공간 등의 사양을 선택해 서버를 생성하고 운영할 수 있다.   

#### 2-1-2. Elastic Load Balancer(ELB)
ELB는 여러 대의 EC2 인스턴스에 요청을 분산해 주는 로드 밸런서 서비스이다.   
ELB를 만들 때 가용성을 위해 가용영역(AZ)를 2개 이상 선택해야 한다.   
그래야 한 AZ가 장애로 중단되면 다른 AZ의 ELB에서 서비스를 계속해서 운영할 수 있다.   

#### 2-1-3. Relational Database Service(RDS)
DB는 RDS 서비스를 사용하면 된다.   
Oracle, mysql, mariadb, postgresql, Aurora, MSSQL을 선택할 수 있다.   
그리고 Database Migration Service(DMS)로 원본 데이터를 RDS에 쉽게 옮길 수 있다.   

#### 2-1-4. Route53
AWS에서 제공하는 DNS 웹 서비스이다. 도메인 이름을 IP 주소로 변환한다.   
Route53에서 도메인을 직접 구매하고 관리할 수 있다.   
사용자의 요청을 최적의 서버로 자동으로 트래픽을 라우팅 한다.   
그리고 엔드포인터의 상태를 헬스체크해 문제가 발생한 서버로 트래픽을 차단하고 정상 서버로만 트래픽을 보내 가용성을 높일 수 있다.   

#### 2-1-5. Auto Scaling Group(ASG)
트래픽, 서버 부하에 따라 EC2 인스턴스들의 자동 스케일링(확장 및 축소)을 관리한다.   
CPU 사용률, 네트워크 트래픽 등의 지표로 서버 부하에 따라 적절한 수 만큼 인스턴스를 자동으로 조절할 수 있다.   
그리고 장애가 발생한 인스턴스를 자동으로 교체하는 기능도 지원한다.   
<br/>
ASG는 Auto Scaling 기능을 실제로 적용하는 고정된 논리 단위이다.   
하나의 ASG는 하나 이상의 EC2 인스턴스 집합을 의미한다.   
ASG에서 최소/최대 인스턴스 개수, auto scaling 정책, launch template, 적용할 load balancer와 target group을 설정한다.   

#### 2-1-6. Launch Template
EC2 인스턴스를 시작할 때 사용하는 설정들을 미리 정해놓은 템플릿이다.   
ASG에서 주로 사용하고 인스턴스 타입, Security Group, AMI를 결정한다.   

### 2-2. AWS 글로벌 인프라 용어
#### 2-2-1. Availability Zone(가용영역)
1개 이상의 데이터센터를 논리적 그룹으로 묶어 놓은 인프라이다.   
물리적으로 분리된 데이터센터이고 한 AZ에 장애가 발생해도 다른 AZ는 영향을 받지 않는다.   

#### 2-2-2. Region
3개 이상의 가용영역(AZ)을 논리적으로 그룹했다.   
서울, 도쿄, 파리 등의 리전이 있고 AWS는 하나의 리전을 여러 개의 가용영역으로 나눠 운영한다.   

#### 2-2-3. VPC
AWS에서 사용자만의 사설 네트워크를 만들어 주는 서비스이다.   
서브넷, 라우팅 테이블, 인터넷 게이트웨이, NAT 게이트웨이, 보안그룹(SG) 등을 직접 설정해 네트워크 구조를 자유롭게 설계할 수 있다.   
VPC 내 리소스는 외부 네트워크와 격리되어 있어 보안성이 높다.   

#### 2-2-4. CIDR 주소체계
IP 주소를 관리하는 체계이다.   
10.0.0.0/16 일 때 16은 prefix 이고 10.0.0.0을 2진수로 바꿨을 때 왼쪽에서 16번 째 까지 고정한다.   
10.0.0.0/16으로 VPC를 만들면 VPC에 만들어지는 모든 인스턴스의 IP는 10.0으로 시작한다.   

#### 2-2-5. Subnet
VPC를 용도 별로 나눠 사용하기 위한 Sub Network이다.   
외부에서 접근 가능한 Public Subnet과 외부에서 접근이 불가능한 Private Subnet을 각 가용영역마다 1개 씩 만든다.   
그래서 한 가용영역에 장애가 나면 다른 가용영역에서 서비스를 계속 운영할 수 있다.   

#### 2-2-6. Internet Gateway(IGW)
IGW는 특정 VPC와 연결되어 있어야 하고 VPC와 인터넷 간의 양방향 트래픽을 가능하게 해준다.   
인스턴스가 인터넷과 통신하려면 IGW에 Public IP 또는 Elastic IP가 할당되어 있어야 한다. Elastic IP는 고정된 퍼블릭 IPv4 주소이다.   
Public Subnet의 인스턴스들이 인터넷이 되려면 IGW를 VPC에 붙이고 Route table을 설정해야 한다.   
Route Table은 네트워크 범위가 작은 것 부터 먼저 적용된다.(Longest Prefix Match)   
인터넷에 들어오는 트래픽은 VPC가 알아서 인스턴스에 전달하지만 처리되고 나가는 트래픽들은 내부의 서비스나 인스턴스에 보내야 하는지, 인터넷으로 보내야 하는지 알 수 없어 Route Table로 구분해야 한다.   

#### 2-2-7. NAT Gateway
Private Subnet는 외부에서 접근할 수 없다. 그러나 Private Subnet의 인스턴스들도 보안 업데이트 등을 위해 인터넷과 연결이 필요하다.   
Public Subnet에 NAT Gateway를 만들면 Private Subnet의 인스턴스들이 인터넷을 할 수 있다.   
NAT Gateway는 인터넷에서 들어오는 트래픽은 차단하고 인터넷으로 나가는 트래픽만 허용하기 때문에 외부에서 Private Subnet에 접근할 수 없다.   
IP 공유기와 비슷하고 동작하고 가용성을 위해 가용영역을 Public Subnet 마다 NAT gateway를 만들어야 한다.   

#### 2-2-8. Security Group(SG)
EC2 인스턴스나 다른 AWS 리소스로 들어오고 나가는 네트워크 트래픽을 제어하는 가상 방화벽이다.   
SG는 특정 VPC 내에서 생성되고 같은 VPC 내 여러 인스턴스에 적용할 수 있다.   
SG를 만들 때 보안을 위해 필요한 포트만 열어야 하고 나가는 포트는 자동으로 열린다.   
아래 구조에서 Security Group은 ELB, EC2, WEB, APP, RDS 용으로 5개를 만든다.   

#### 2-2-9. Security Chain
각 인스턴스들이 바로 앞에 있는 인스턴스들의 트래픽만 허용한다.   
RDS SG에서 APP 서버에서 오는 트래픽만 허용하고 APP 서버는 WEB 서버에서 오는 트래픽만 허용한다. 그리고 WEB 서버는 ELB에서 오는 트래픽만 허용하도록 설정한다.   
SG의 Inbound Rule 수정으로 설정한다.   

### AWS로 만든 구조
<figure>
  <img src="https://i.imgur.com/KYgZxKq.png" width="100%" alt=""/>
  <p style="font-style: italic; color: gray;">On-Premises 구조를 AWS 구조로 변환</p>
</figure>  

## 3. 단일 장애 지점(Single point of failure) 개선 구조
서비스를 AWS 클라우드 구조로 옮겨 고가용성을 확보 했지만 사용자의 요청이 더 많아지며 전체 시스템이 다운되는 장애가 발생했다.   
APP이나 WEB은 사용자의 요청에 맞게 인스턴스가 늘어나고 ELB를 통해 트래픽을 분산해줘 시스템 다운의 원인이 아니었다.   
하나의 RDS 인스턴스에 많은 요청이 몰려 RDS가 다운되며 서비스가 전체 다운이 되었다.   
RDS는 전체 시스템에 영향을 미치는 단일 장애 지점이고 이중화를 통해 문제를 해결해야 한다.   

### 3-1. Multi-AZ
모든 읽기, 쓰기 요청은 메인 RDS 인스턴스에 수행된다.   
서브 RDS 인스턴스는 복제본 역할로 평상시에는 직접 쿼리를 받지 않고 대기만 한다.   
메인 RDS 인스턴스에서 장애가 발생하면 RDS는 자동으로 서브 RDS 인스턴스를 메인 RDS 인스턴스로 승격해 중단 없이 데이터베이스를 운영한다.   
클라이언트는 몇 분 내에 자동으로 새로운 메인 RDS 인스턴스에 연결된다.   
장애 발생 시 자동 failover로 다운타임을 최소화 하고 동기 복제 방식으로 데이터가 완전히 복제되어 손실이 없다.   
<br/>
같은 DB 인스턴스가 두 개라서 비용이 두 배로 발생한다.   
고가용성 및 장애 복구용으로 설계되어 읽기 부하 분산 용인 Read Replica와는 다르다.   
Read Replica는 읽기 부하 분산용으로 비동기 복제를 사용해 일관성 보장이 안되고 자동 failover 기능이 없다.   

### 3-2. 동기/비동기 복제
#### 3-2-1. 동기 복제
메인 DB와 서브 DB가 데이터를 동일하게 쓰기 성공해야 완료 응답을 한다.   
그래서 완벽하게 일관성을 유지할 수 있지만 성능이 비동기 복제보다 느릴 수 있다.   

#### 3-2-2. 비동기 복제
메인 DB의 변경사항을 바로 서브 DB로 전송하는 것이 아니라 약간의 지연을 두고 변경 내용을 전송한다.   
그래서 동기화 하는데 지연 시간(Lag)이 존재한다.   

### 3-3. RDS를 이중화 한 구조
<figure>
  <img src="https://i.imgur.com/0SdAdEC.png" width="100%" alt=""/>
  <p style="font-style: italic; color: gray;">RDS를 이중화 한 구조</p>
</figure>  

## 4. 웹서버 개선 구조
사용자에게 정적인 자료를 제공하는 웹서버에 등록할 자료가 많아지면 똑같은 자료가 웹서버에 중복되어 스토리지 비용이 늘어난다.   
중앙 집중식으로 정적 자료를 저장하고 사용자에게 보내주는 구조가 효율적이다.   
프론트엔드 자료는 웹서버 대신 S3 서비스를 사용해 배포한다.   

### 4-1. S3
파일을 배포하면 인터넷을 통해 사용자들이 받아갈 수 있는 웹하드, 구글 드라이브 같은 서비스이다.   
스태틱 웹사이트 호스팅 기능이 있어 웹서버의 기능을 대체할 수 있다.   
리액트, 뷰 등 자바스크립트 프레임워크를 S3에 배포해 사용한다.   
AWS가 모든걸 관리하는 오나전 고나리형 서비스 이므로 운영자는 신경쓰지 않아도 된다.   
Static website hosing에서 사용자가 접속할 index.html, 에러 발생 시 사용할 화면을 설정해 웹서버를 대신해 사용할 수 있다.   

### 4-2. S3와 APP의 통신
Private Subnet에 있는 APP 서버가 S3와 통신할 때 AWS 내부망을 사용하지 않고 인터넷을 통해 통신한다.   
S3는 퍼블릭 AWS 서비스 이므로 인터넷을 통해 접근하고 인터넷과 연결되지 않은 Private Subnet의 APP에서 S3와 통신하려면 NAT Gateway를 통해 인터넷과 통신해야 한다.   
VPC Endpoint를 사용하면 APP에서 내부망을 통해 S3와 통신할 수 있다.   

### 4-3. VPC Endpoint 
VPC 안에 있는 리소스들이 인터넷을 거치지 않고 AWS 서비스에 직접 접근할 수 있도록 하는 네트워크 기능이다.   
Private Subnet의 인스턴스들이 NAT, Interget Gateway를 통하지 않아 비용과 복잡성을 줄일 수 있다.   

### 4-4. 웹서버 개선구조
<figure>
  <img src="https://i.imgur.com/pce66XW.png" width="100%" alt=""/>
  <p style="font-style: italic; color: gray;">웹서버 개선 구조</p>
</figure>  

## 5. 캐시를 추가한 구조
서비스가 해외로 확장되고 사용자가 늘어나 요청이 많아지며 서비스의 속도가 느리다는 클레임이 늘어났다.   
서비스는 서울 리전에 있는데 해외에서 요청이 들어오면 물리적인 거리가 멀어 느릴 수 있다.   
사용자와 가까운 위치의 장비에 자주 요청하는 데이터를 저장해 데이터센터까지 요청이 오지 않도록 캐시 서버를 구축해 문제를 해결할 수 있다.   
이 기능을 CDN(Content Delivery Network)라 하고 AWS는 CloudFront라는 서비스로 제공한다.   

### 5-1. CloudFront
사용자에게 전달할 컨텐츠를 Edge location에 캐싱해서 사용자에게 가장 가까운 서버에서 제공한다.    
Edge location은 전 세계 200개 이상 퍼져있다.   
S3, EC2, ALB 등의 원본 서버(Origin)에서 컨텐츠를 가져온다.   
Route53에서 도메인을 CLoudFront에 연결하면 CloudFront는 Origin에서 데이터를 찾고 업으면 Origin에 요청해 캐싱하고 요청에 응답한다.   

### 5-2. CloudFront를 추가한 구조
<figure>
  <img src="https://i.imgur.com/y5IVuwF.png" width="100%" alt=""/>
  <p style="font-style: italic; color: gray;">웹서버 개선 구조</p>
</figure>  