---
layout: post
title: AWS Architecture (3)
date: 2026-01-25 21:00:00 + 0900
categories: [aws]
tags: [aws, architecture]
mermaid: true
---

# AWS Architecture (3)
## 1. Peering Connection
쇼핑몰 서비스에 이어 여행 전문 쇼핑몰 서비스를 만드려한다.   
서비스의 구조는 쇼핑몰 서비스와 유사해 VPC를 새로 만들고 쇼핑몰 서비스의 CloudForamtion으로 아키텍처를 배포한다.   
여행 전문 쇼핑몰에서 기존 쇼핑몰의 데이터베이스에서 사용자, 제품 정보 등을 가져오기 위해 VPC 간 연결이 필요하다.   
AWS는 다른 계정, 리전의 VPC를 연결할 수 있는 Peering Connection 서비스를 제공한다.   

<figure>
  <img src="https://i.imgur.com/V7CyKaw.png" width="60%" alt=""/>
  <p style="font-style: italic; color: gray;">2개의 VPC 연결</p>
</figure>

<br/>

연결할 VPC가 늘어나면 Peering Connection 서비스의 수도 n(n-1)/2 만큼 늘어난다.   
여러 VPC를 연결하려면 Peering Connection 만으론 한계가 있다.   

<figure>
  <img src="https://i.imgur.com/ZYTUAt4.png" width="60%" alt=""/>
  <p style="font-style: italic; color: gray;">3개의 VPC 연결</p>
</figure>
<figure>
  <img src="https://i.imgur.com/nk1nyI8.png" width="60%" alt=""/>
  <p style="font-style: italic; color: gray;">4개의 VPC 연결</p>
</figure>
<br/>

## 2. Transit Gateway
Transit Gateway는 VPC의 허브 같은 역할을 담당한다.   
모든 VPC가 서로 통신할 수 있도록 해서 Peering Connection으로 구성하면 복잡할 네트워크를 간단하게 구성해 준다.   
VPC마다 Transit Gateway Attachment를 만들어 Transit Gateway에 등록하고 Transit Gateway Route Table을 설정해 사용할 수 있다.   

<figure>
  <img src="https://i.imgur.com/SwEf1tt.png" width="60%" alt=""/>
  <p style="font-style: italic; color: gray;">Transit Gateway로 VPC 연결</p>
</figure>

## 3. Peering Connection을 사용해 Transit Gateway 간 연결
VPC도 Peering Connection이 있는 것 처럼, Transit Gateway도 Peering Connection이 있다.   
아래와 같이 서울 리전과 도쿄 리전에 Transit Gateway를 만들어 VPC를 연결하고 Trasit Gateway를 연결해 더 대규모의 네트워크를 만들 수 있다.   
서울 리전 Transit Gateway에 도쿄 리전과 연결할 Transit Gateway를 설정한 Transit Gateway Attachment를 등록하면 도쿄 리전에 Transit Gateway 연결 요청을 보낸다.   
도쿄 리전의 Transit Gateway Attachment에서 peering 요청을 수락하고 서울, 도쿄 리전의 Transit Gateway Route Table을 설정하면 연결이 된다.   

<figure>
  <img src="https://i.imgur.com/aiLJSfB.png" width="60%" alt=""/>
  <p style="font-style: italic; color: gray;">Transit Gateway로 간 연결</p>
</figure>

## 4. Site-To-Site VPN
다른 회사와 협업하기 위해 데이터를 공유하려 한다.   
데이터베이스를 공유하기 위해 다른 회사가 AWS를 사용하면 VPC Peering으로 간단히 연결하면 되지만 그렇지 않다면 VPN으로 연결해야 한다.   
AWS는 Site-To-Site VPN이라는 기능을 제공한다. 이 기능을 사용하면 VPN 관련 소프트웨어 설치와 설정을 하지 않고 VPN 연결을 할 수 있다.   
그러나 통신 속도가 느린 문제가 발생할 수 있다.   

<figure>
  <img src="https://i.imgur.com/satu18G.png" width="60%" alt=""/>
  <p style="font-style: italic; color: gray;">Site-To-Site VPN</p>
</figure>

## 5. Direct Connect(DX)
앞선 환경에서 통신 속도가 느린 경우 협업할 회사의 데이터센터와 인터넷으로 연결하는 대신 전용선을 연결해 사용할 수 있는 DX 서비스를 제공한다.   

<figure>
  <img src="https://i.imgur.com/zwgGima.png" width="60%" alt=""/>
  <p style="font-style: italic; color: gray;">DX 연결</p>
</figure>

## 6. 해외 리전으로 확장
유럽에 서비스를 확장하기 위해 파리 리전에 서울 리전과 똑같은 인프라를 배포하려 한다.   
인터넷은 전세계에서 다 접속이 되기 때문에 굳이 파리 리전에 인프라를 배포해서 서비스 하지 않아도 될 수 있다.   
그러나 물리적인 통신 속도 때문에 실사용 고객과 가까운 리전에 서비스를 배포하는 것이 좋다.   
그리고 나라마다 서비스와 데이터의 법이 다르고 데이터를 해외 서버에 저장하지 못하는 나라도 있다.   
파리 리전에 CloudFormation으로 서울 리전과 같은 아키텍처를 쉽게 배포할 수 있다.   
그리고 서울 리전과 파리 리전의 VPC를 Peering Connection으로 연결해 서로 통신할 수 있다.   

<figure>
  <img src="https://i.imgur.com/c7F1KUp.png" width="60%" alt=""/>
  <p style="font-style: italic; color: gray;">파리 리전 배포</p>
</figure>

## 7. AWS 리전 장애
서울 리전 자체에 장애가 나면 AZ 설정과 상관없이 전체 서비스가 장애가 날 수 있다.   
서울 리전이 장애난 경우 사용자의 요청을 파리 리전으로 보내 서비스를 계속 운영할 수 있다.   
Route54 서비스는 리전의 장애를 모니터링 하다가 한쪽에 장애가 발생하면 트래픽을 다른 리전으로 보내주는 라우팅을 지원한다.   

<figure>
  <img src="https://i.imgur.com/G51789T.png" width="60%" alt=""/>
  <p style="font-style: italic; color: gray;">Route53으로 트래픽 제어</p>
</figure>


