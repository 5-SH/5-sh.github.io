---
layout: post
title: 스프링 부트와 JPA 활용5
date: 2025-07-22 19:12:00 + 0900
categories: [spring]
tags: [spring, jpa]
---

### 강의 : [실전! 스프링 부트와 JPA 활용2 - API 개발과 성능 최적화](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81%EB%B6%80%ED%8A%B8-JPA-API%EA%B0%9C%EB%B0%9C-%EC%84%B1%EB%8A%A5%EC%B5%9C%EC%A0%81%ED%99%94/dashboard)


### 4. 영속성 컨텍스트와 트랜잭션

#### 4-1. 영속성 컨텍스트와 트랜잭션 유무에 따른 CRUD 동작

JPA에서 조회는 영속성 컨텍스트가 살아 있으면 트랜잭션이 없어도 조회가 가능하다.    
그러나 삽입, 수정, 삭제는 영속성 컨텍스트와 트랜잭션이 살아있어야 한다.   
영속성 컨텍스트가 없을 때 지연로딩을 하면 ```LazyInitializationException```이 발생하고   
트랜잭션이 없을 때 삽입, 수정, 삭제를 하면 ```TransactionRequiredException```이 발생한다.   

#### 4-2. Spring Data JPA에서 영속성 컨텍스트와 트랜잭션의 라이프사이클

Spring Data JPA에서 ```@Transactional```이 적용된 메서드가 실행되면,    
스프링이 트랜잭션을 시작하고 이 트랜잭션 범위 내에서 영속성 컨텍스트(EntityManager)가 생성되어 함께 열린다.   
메서드가 끝나면 ```flush()```와 commit/rollback을 호출하고 트랜잭션과 영속성 컨텍스트를 종료한다.   
서비스 계층 코드는 대부분 ```@Transactional```을 사용하고 ```JpaRepository```의 기본 메서드들은 내부적으로 ```@Transactional```이 붙어있다.   
따라서 JPA 서비스 계층 코드와 레포지토리 계층 코드는 영속성 컨텍스트와 트랜잭션 범위가 동일하게 동작한다.   
이렇게 동작하는 이유는 스프링의 트랜잭션 동기화된 EntityManager(```@PersistenceContext```)를 사용하기 때문이다.

#### 4-3. 트랜잭션 동기화된 EntityManager(```@PersistenceContext```)

스프링이 트랜잭션과 ```EntityManager```의 생명주기를 자동으로 연결해준다.   
```@PersistenceContext```는 스프링이 관리하는 트랜잭션 범위의 EntityManager를 주입받기 위한 표준 애노테이션이다.   
이 EntityManager는 트랜잭션이 시작될 때 생성되어 트랜잭션이 끝날 때 자동으로 닫히며 스레드 안에서 안전하게 바인딩된다.   
스프링은 내부적으로 ```TransactionSynchronizationManager```를 사용해 트랜잭션이 시작되면 해당 스레드에 EntityManager를 바인딩한다.   
EntityManager는 Thread-Safe 하지 않지만 ```@PersistenceContext```로 주입받는 EntityManager는 현재 스레드의 트랜잭션 범위에 맞게 관리된 EntityManager만 주입되기 때문에 Thread-Safe하게 사용할 수 있다.   

```java
@Repository
public class MemberJpaRepository {

    @PersistenceContext
    private EntityManager em;

//    @Transactional
    public Member save(Member member) {
        em.persist(member);
        return member;
    }

    public void delete(Member member) {
        em.remove(member);
    }

    public List<Member> findAll() {
        return em.createQuery("select m from Member m", Member.class).getResultList();
    }
    ...
}
```

### 5. OSIV와 성능 최적화

모든 엔티티의 로딩 전략은 지연로딩(LAZY)으로 설정해야 한다.    
즉시로딩(EAGER)는 어떤 SQL이 실행될지 예측과 추적이 어렵고 N+1 문제가 자주 발생하기 때문이다.    
지연로딩이 실행되려면 영속성 컨텍스트가 살아있어야 하고 영속성 컨텍스트는 트랜잭션과 라이프사이클이 같다.   
따라서 서비스나 레포지토리 계층의 상위에 있는 컨트롤러나 뷰어에서는 지연로딩을 할 수 없다.   
이 문제를 해결하기 위해 OSIV를 사용한다.   

<br/>

하이버네이트는 Open Session In View, JPA는 Open EntityManager In View 라고 부르고 관례상 OSIV로 통일해 사용한다.    
OSVI는 커넥션 시작 지점부터 API 응답이 끝날 때 까지 영속성 컨텍스트와 데이터베이스의 커넥션을 유지해 View Template나 API Controller에서 지연 로딩이 가능하도록 한다.   

<br/>

영속성 컨텍스트는 기본적으로 데이터베이스 커넥션을 유지해서 오랜시간 동안 데이터베이스 커넥션을 사용한다.
예를 들어 Controller에서 외부 API를 호출하면 외부 API 대기 시간 만큼 커넥션 리소스를 유지한다.   
따라서 실시간 트래픽이 많을 경우 커넥션이 모자라 장애로 이어질 수 있다.     

#### 5-2. OSIV ON
```spring.jpa.open-in-view: true```(기본 값)    

스프링의 Servlet Filter 레벨에서 EntityManager를 열고 응답이 끝날 때 닫는다.   
따라서 컨트롤러나 뷰에서 EntityManager가 살아 있기 때문에 예외 없이 지연로딩을 사용할 수 있다.   

#### 5-3. OSIV OFF

```spring.jpa.open-in-view: false```   

OSVI를 끄면 ㅌ랜잭션을 종료할 때 영속성 컨텍스트를 닫고 데이터베이스 커넥션도 반환한다. 따라서 커넥션 리소스를 낭비하지 않는다.   
그러나 OSIV를 끄면 View Template나 API Controller에서 지연로딩이 동작하지 않고, 모든 지연로딩을 트랜잭션 안에서 처리해야 한다.   

##### 5-4. 커맨드와 쿼리 분리

OSIV를 켜면 데이터베이스 커넥션 리소스 부족 문제로 장애가 발생할 수 있기 때문에 보통 사용하지 않는다.   
OSIV를 끈 상태에서는 Command와 Query를 분리해 복잡성을 관리할 수 있다.   

<br/>

보통 비즈니스 로직은 특정 엔티티 몇 개를 등록하거나 수정하므로 성능이 크게 문제되지 않는다.   
그러나 복잡한 화면을 출력하기 위한 쿼리는 성능을 최적화 하는 것이 중요하다.   
그러나 복잡성에 비해 비즈니스 로직에 큰 영향을 주지 않는다.   

예를 들어 다음과 같이 분리해 유지보수한다.
- OrderService
    - OrderService: 핵심 비즈니스 로직
    - OrderQueryService: 화면이나 API에 맞춘 서비스 (주로 읽기 전용 트랜잭션 사용)

보통 서비스 계층에서 트랜잭션을 시작하고 두 서비스 모두 트랜잭션을 유지하면서 지연 로딩을 사용할 수 있다.   