---
layout: post
title: 가상 메모리와 스왑 메모리
date: 2021-11-02 15:00:00 + 0900
categories: OS
ref: OS, memory, swap
---

# 가상 메모리와 스왑 메모리
스왑 메모리는 프로세스 단위이다. 메모리에 P1, P2, ... P10 10개 프로세스가 돌고 있는데 남은 메모리 공간이 없다.   
이 때 P11 을 실행 시키고 싶을때 메모리 확보를 위해 일부 프로세스를 디스크로 내리고 P11 을 메모리에 올리는 과정을 **스와핑**이라고 한다.   
스왑 과정은 시간이 오래 걸리는 작업이고 필요한 공간이 가변적이다.    
따라서 디스크 일부 영역을 파티션으로 나눠 스와핑에 사용한다.   

<br />

가상 메모리는 프로세스가 물리 메모리의 크기에 상관없이 사용할 수 있도록 한다.    
4KB 크기의 페이지로 프로세스를 나누고 프로세스가 실제로 필요하거나 사용하게 될 부분의 페이지들만 메모리로 올려 사용한다.   
메모리 공간이 모자라거나 자주 사용되지 않으면 다시 디스크로 내려간다.   
그리고 페이지는 프로세스 간에 공유될 수 있다.   
이 과정을 **페이징**이라 하고 페이지 테이블을 통해 실제 메모리에 사상된다.   

<br />

htop 에서 VIRT, RES, SWAP 항목은 각각 다음과 같은 내용을 나타낸다.
- VIRT : 가상메모리 공간의 크기로 프로세스에서 필요로 하는 총 공간의 크기이다.   
여기에 RSS, SWAP, 공유 메모리 등이 포함되어 있다.   
 그리고 프로세스는 힙 할당 등을 위해 실제 자신이 필요로 하는 양보다 넉넉하게 가지고 있다.
- RSS : 프로세스에서 실제로 메모리를 점유해 사용하고 있는 공간의 크기이다.
- SWAP : 프로세스에서 디스크 스왑 영역을 점유해 사용하고 있는 공간의 크기이다.   

<br />

VIRT = RSS + SWAP 이 아니다. 왜냐하면 프로세스간 공유하는 메모리도 있고 실제 사용하지 않고 잡아둔 메모리 공간도 있기 때문이다.     
그리고 VIRT 의 페이지 프레임들이 모두 실행되는 것은 아니다.      
디스크에 잠들어 있는 것들도 있고 그 중 현재 프로세스 실행에 필요한 페이지는 RES 와 SWAP 에 올라가 사용되고 있기 때문이다.