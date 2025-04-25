---
layout: post
title: 임계구역 해결 방법
date: 2022-08-06 02:00:00 + 0900
categories: [os]
tags: [os, critical section, monitor]
---
# 임계구역 해결 방법
출처1 : [임계구역 해결방법 결론은 하나 'Queue'! - 널널한 개발자 TV](https://www.youtube.com/watch?v=1I_DQosMkhQ&list=PLXvgR_grOs1DGFOeD792kHlRml0PhCe9l&index=17&ab_channel=%EB%84%90%EB%84%90%ED%95%9C%EA%B0%9C%EB%B0%9C%EC%9E%90TV)    
출처2 : [동기화, 모니터 : Synchronization, Monitor [운영체제]](https://luv-n-interest.tistory.com/441)


## 1. 임계구역은 어떤 경우에 발생하는가?
<figure>
  <img src="https://user-images.githubusercontent.com/13375810/183125719-8bae2489-9508-4f8d-bd5a-57150e8dc47e.png" width="55%"/>
  <p style="font-style: italic; color: gray;">▲ 임계구역이 발생하는 경우</p>
</figure>

### 1-1
단일 연결 리스트로 주소록을 구현하고 전역 변수에 저장 했습니다.    
그리고 사용자의 요청을 받아 주소록 조회, 추가, 삭제, 동기화 작업을 하는 UI 스레드와    
DB 에 네트워크 I/O 로 주소록을 저장, 삭제, 변경, 조회하는 Service 스레드가 존재합니다.    
<br/>
주소록에 ___1→2→3→4___ 가 저장되어 있고 UI 스레드에서 전체 조회 작업을 진행 중 이었습니다.    
그리고 동시에 Service 스레드에서 ___2-1___ 노드를 추가하려 합니다.   
<br/>
추가 작업은 ___2___ 노드의 링크를 ___2-1___ 노드로 옮기고 ___2-1___ 노드의 링크를 ___3___ 노드로 옮기는 여러 작업을 포함합니다.   
Service 스레드에서 ___2___ 노드의 링크를 ___2-1___ 노드로 연결하는 작업을 완료하기 전에    
UI 스레드가 ___2___ 노드에서 ___3___ 노드로 이동해 읽는다면,    
UI 스레드는 ___1→2→3→4___ , Service 스레드는 ___1→2-1→3→4___ 를 다른 주소록의 상태를 가집니다.    
이 경우 두 스레드는 동시성을 만족하지 못합니다.    
<br/>
비슷한 상황이 삭제 상황에서 발생한다면, UI 스레드는 이미 삭제된 노드를 읽어 프로세스가 죽어버릴 수 있습니다.

### 1-2
따라서 주소록이 저장된 전역 단일 연결 리스트 자료구조에 UI 스레드의 전체 읽기가 실행되는 동안   
Service 스레드에서 추가, 삭제, 변경이 일어나면 안됩니다.   
<br/>
DB 같은 자료구조에 접근하는 경우 DBMS 에서 동시성 제어를 해줘 이런 경우를 신경쓰지 않아도 되지만,   
일반적인 자료구조를 여러 스레드에서 동시에 접근 시 동시성 제어를 위해 작업 원자성 부여 등 처리를 해줘야 합니다.

## 2. 전역 변수로 잠금(Spin Lock)
<figure>
  <img src="https://user-images.githubusercontent.com/13375810/183125401-09782064-91d3-4802-8765-cabddd21be95.png" width="55%"/>
  <p style="font-style: italic; color: gray;">▲ Spin Lock</p>
</figure>

## 3. 잘 알려진 해결 방법
### 3-1
<figure>
  <img src="https://user-images.githubusercontent.com/13375810/183134225-2af724ff-c0ef-4955-bf9c-0e6f5fc09017.png" width="55%"/>
  <p style="font-style: italic; color: gray;">▲ 스레드-큐 구조</p>
</figure>

스레드 또는 프로세스에서 동기화 목표를 달성하기 위해서 __Queue__ 를 사용해야 합니다.   
작업을 수행하는 스레드들은 큐에서 작업을 하나씩 꺼내온다. 그리고 작업은 다른 외부 스레드에서 큐에 추가합니다.   
<br/>
각 작업 수행 스레드에서 큐를 가져다 연산하는 것이 아니라 큐를 관리하는 스레드에 작업을 요청합니다.    
여기서 큐는 임계구역이 되고, __여러 작업 처리 스레드의 임계구역 접근을 단일 스레드로 제한__ 해 동시성을 확보합니다.

### 3-2
아래와 같은 좀 더 나은 구조를 생각해 볼 수 있습니다.

<figure>
  <img src="https://user-images.githubusercontent.com/13375810/183226567-8cdf1aa1-4963-4baf-9825-ef3ba60fc0eb.png" width="55%"/>
  <p style="font-style: italic; color: gray;">▲ 좀 더 나은 스레드-큐 구조</p>
</figure>

작업에 따라 코드를 분리하고 작업 큐를 별도로 둡니다.   
그리고 작업 별 스레드들은 작업 큐를 통해 처리할 작업을 주고 받습니다.   

## 4. 모니터
<figure>
  <img src="https://user-images.githubusercontent.com/13375810/183226546-ace52d83-4e11-466b-a06b-b61aec3a14c4.png" width="55%"/>
  <p style="font-style: italic; color: gray;">▲ Monitor</p>
</figure>

- 세마포어를 실제로 구현했습니다.
- 순차적으로만 사용할 수 있는 공유자원 그룹, 공유자원을 할당합니다.
- 모니터 외부 프로세스는 모니터 내부 프로스에 직접 접근할 수 없습니다.
- __한 순간에 한 프로세스만 모니터에 진입할 수 있습니다.__
- monitor 구조체로 공유 데이터를 선언합니다.   
  공유 데이터는 변수와 변수를 조작할 수 있는 프로시저 또는 함수를 포함합니다.    
  공유 데이터의 내부의 변수는 내부 프로시저 통해서만 접근할 수 있습니다.
  ```
  monitor monitor-name {
    procedure p1(..) {...}
    procedure p2(..) {...}
    procedure p3(..) {...}
    function f1(..) {...}
    function f2(..) {...}
  }
  ```
- 조건 변수를 활용해 동기화 메커니즘을 제공해야 합니다.   
  condition 은 wait(), signal() 함수만 사용 가능합니다.    
  ``` javascript
  condition x, y;
  x.wait();     // 자원이 없을 경우 대기
  y.signal();   // 사용 중인 자원을 반납하고 대기하고 있는 프로세스를 불러옴
  ```
- 내부의 프로시저가 동시에 접근되지 않아 Lock 을 걸 필요가 없습니다.