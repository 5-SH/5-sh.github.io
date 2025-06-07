---
layout: post
title: 스프링 부트와 JPA 활용1
date: 2025-06-04 01:30:00 + 0900
categories: [spring]
tags: [spring, jpa]
---

### 강의 : [실전! 스프링 부트와 JPA 활용1 - 웹 애플리케이션 개발](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81%EB%B6%80%ED%8A%B8-JPA-%ED%99%9C%EC%9A%A9-1/dashboard)

## 질문

### 1. JPA 연관관계 매핑 분석에서 연관관계의 주인을 정해야 하는 이유가 뭐야?

JPA 연관관계 매핑에서 연관관계의 주인을 정하는 이유는 데이터베이스에 저장될 외래 키(Foreign Key)를 어느 엔티티가 관리할지를 명확히 하기 위해서입니다.   
이는 일대다, 다대일, 일대일, 다대다 관계에서 모두 중요합니다.   

<br/>

🔑 왜 "주인(Owner of the relationship)"을 정해야 할까?   
JPA는 객체의 참조를 통해 연관관계를 표현하지만, 데이터베이스는 외래 키(FK) 를 통해 연관관계를 표현합니다.   
객체는 양쪽에서 서로 참조할 수 있으나, DB는 외래 키가 하나의 테이블에만 존재하므로 한 쪽만 외래 키를 관리해야 합니다.   
따라서 JPA에서는 연관관계의 주인을 정해서, 어느 쪽에서 외래 키를 관리할지를 명확히 지정합니다.   

<br/>

📌 예시로 살펴보기
엔티티: Member - Team 관계 (N:1)
```java
@Entity
public class Member {
    @Id @GeneratedValue
    private Long id;

    @ManyToOne
    @JoinColumn(name = "team_id")  // 연관관계의 주인
    private Team team;
}

@Entity
public class Team {
    @Id @GeneratedValue
    private Long id;

    @OneToMany(mappedBy = "team")  // 주인이 아님
    private List<Member> members = new ArrayList<>();
}
```

⚠️ 여기서 중요한 점
- Member.team → 외래 키 컬럼 team_id를 관리하는 연관관계의 주인
- Team.members → mappedBy를 통해 연관관계의 주인이 아님을 명시함 (읽기 전용)

<br/>

✅ 연관관계 주인을 정하지 않으면?
1. 양쪽 모두 외래 키를 관리하려고 해서 충돌이 발생할 수 있음
2. JPA가 어떤 필드를 기준으로 DB에 외래 키를 업데이트할지 결정할 수 없음
3. 의도하지 않은 UPDATE SQL이 발생하거나, 연관관계가 잘못 매핑될 수 있음

<br/>

✅ 실무 팁:
항상 연관관계의 주인을 명확히 정하고, 주인이 아닌 쪽에는 mappedBy를 반드시 선언하세요.   
양방향 관계에서는 주인 쪽에서만 외래 키 변경이 반영됩니다.

### 2. @JoinColumn과 mappedBy를 설명해 줘

@JoinColumn과 mappedBy는 JPA에서 연관관계 매핑 시 서로 다른 역할을 합니다.    
둘은 함께 사용되며, 연관관계의 방향과 주인을 구분짓는 핵심 개념입니다.   

<br/>

✅ @JoinColumn — 외래 키(FK)를 맵핑하고 관리하는 쪽에 사용   
📌 의미:
- "이 컬럼이 외래 키입니다" 라고 JPA에 알려주는 어노테이션
- 보통 연관관계의 주인이 선언합니다.

📋 예시:
```java
@Entity
public class Member {

    @Id @GeneratedValue
    private Long id;

    @ManyToOne
    @JoinColumn(name = "team_id") // team_id 외래키 컬럼 생성 및 관리
    private Team team;
}
```

- 여기서 team_id는 Member 테이블에 생깁니다.
- 외래 키 컬럼을 통해 Member → Team 관계를 설정합니다.
- Member가 연관관계의 주인입니다.

<br/>

✅ mappedBy — 연관관계의 주인이 아님을 선언   
📌 의미:
- 외래 키를 관리하지 않으며, "저쪽 필드가 주인이야" 라고 알려줍니다.
- 즉, 이 필드는 읽기 전용 관계입니다.

📋 예시:
```java
@Entity
public class Team {

    @Id @GeneratedValue
    private Long id;

    @OneToMany(mappedBy = "team") // Member.team이 연관관계의 주인
    private List<Member> members = new ArrayList<>();
}
```
- mappedBy = "team" 은 Member 엔티티의 team 필드를 가리킵니다.
- 이 필드는 단지 연관관계를 양방향으로 보기 위함이며, DB 외래 키에 영향 안 줍니다.

<br/>

📌 실제 DB 테이블 구조 예 (위 Member–Team 예시 기준)

|Member 테이블|Team 테이블|
|---|---|
|id|id|
|team_id (FK)||

- 외래 키 team_id 는 Member 테이블에만 존재
- Team 엔티티는 mappedBy를 사용해서 양방향으로 관계만 표현

### 3. 다대다, 일대일 관계의 예를 설명해 줘

좋습니다! 다대다(@ManyToMany)와 일대일(@OneToOne) 관계는 JPA에서 흔히 사용되지만, 정확한 매핑 방식을 이해하지 않으면 의도하지 않은 결과를 낳기 쉽습니다.    
아래에서 각각 자세히 설명드릴게요.   

<br/>

✅ 1. 다대다 관계 (@ManyToMany)
📌 설명
- A와 B가 서로 여러 개를 가질 수 있는 관계
- 관계를 표현하려면 중간 테이블이 필요
- JPA는 @ManyToMany만으로도 기본 중간 테이블을 자동 생성하지만, 실무에서는 별도 엔티티로 푸는 것이 안전

<br/>

🔸 예: 학생(Student)과 과목(Subject)
```java
@Entity
public class Student {

    @Id @GeneratedValue
    private Long id;

    @ManyToMany
    @JoinTable(name = "student_subject",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "subject_id"))
    private List<Subject> subjects = new ArrayList<>();
}
```
```java
@Entity
public class Subject {

    @Id @GeneratedValue
    private Long id;

    @ManyToMany(mappedBy = "subjects") // 주인이 아님
    private List<Student> students = new ArrayList<>();
}
```

📋 생성되는 테이블
- student
- subject
- student_subject ← 중간 테이블 (FK: student_id, subject_id)

<br/>

⚠ 실무에서 주의   

- 다대다 관계는 종종 중간 테이블에 추가 정보(예: 수강 날짜) 가 필요하므로, 아래처럼 중간 엔티티로 분리하는 것이 일반적입니다:

```java
@Entity
public class Enrollment {
    @Id @GeneratedValue
    private Long id;

    @ManyToOne
    private Student student;

    @ManyToOne
    private Subject subject;

    private LocalDateTime enrolledAt;
}
```

✅ 2. 일대일 관계 (@OneToOne)   

📌 설명   
- A와 B가 서로 정확히 한 개만 연결
- 외래 키는 한쪽 테이블에만 존재
- 양방향 매핑에서는 외래 키를 가진 쪽이 연관관계의 주인

<br/>

🔸 예: 사용자(User)와 여권(Passport)
```java
@Entity
public class User {

    @Id @GeneratedValue
    private Long id;

    @OneToOne
    @JoinColumn(name = "passport_id") // FK가 여기에 있음 → 주인
    private Passport passport;
}
```
```java
@Entity
public class Passport {

    @Id @GeneratedValue
    private Long id;

    @OneToOne(mappedBy = "passport") // 연관관계의 주인 아님
    private User user;
}
```

📋 생성되는 테이블   

|User 테이블|Passport 테이블|
|---|---|
|id|id|
|passport_id (FK)||

<br/>

📌 주의사항   
- @OneToOne 관계에서는 주인 쪽이 어느 쪽이 될지는 설계 목적에 따라 달라지며, 외래 키가 존재하는 곳이 주인
- 성능 상 외래 키는 자주 조회되는 쪽에 두는 것이 좋음

### 4. @ManyToOne 애노테이션을 설명해 줘

@ManyToOne 애노테이션은 JPA에서 가장 많이 사용되는 연관관계 매핑 어노테이션 중 하나입니다.   
이는 N:1 관계, 즉 여러 개의 엔티티가 하나의 엔티티에 연관되는 관계를 표현합니다.   

<br/>

✅ @ManyToOne 이란?
- "다수(N)의 엔티티가 하나(1)의 엔티티에 연관된다"는 관계를 정의
- 예: 여러 명의 Member가 하나의 Team에 소속되어 있는 경우

<br/>

📌 예제 코드   
```java
@Entity
public class Member {

    @Id @GeneratedValue
    private Long id;

    private String username;

    @ManyToOne
    @JoinColumn(name = "team_id") // FK를 매핑할 컬럼
    private Team team;
}
```
- @ManyToOne: Member → Team 다대일 관계
- @JoinColumn(name = "team_id"): 외래 키 이름 지정 (없으면 기본값은 team_id)

🔄 반대편 매핑 (양방향 예시)   

```java
@Entity
public class Team {

    @Id @GeneratedValue
    private Long id;

    private String name;

    @OneToMany(mappedBy = "team") // 주인이 아님
    private List<Member> members = new ArrayList<>();
}
```

⚠ 중요한 특징 및 옵션   
|속성|설명|
|---|---|
|fetch|LAZY(지연로딩, 기본 권장) 또는 EAGER(즉시로딩, 비추천)|
|optional|true/false (null 허용 여부, default = true)|
|cascade|관련 엔티티에 연쇄 작업 적용 가능 (ex: persist, remove 등)|

예:   

```java
@ManyToOne(fetch = FetchType.LAZY, optional = false)
@JoinColumn(name = "team_id")
private Team team;
```

✅ 생성되는 DB 구조   
- Member 테이블에 team_id 외래 키 컬럼이 생깁니다.
- 외래 키 제약 조건이 자동으로 적용됩니다 (nullable 여부 등)

<br/>

📌 실무에서 팁   
- @ManyToOne 은 반드시 LAZY 로딩을 명시하는 것이 좋습니다:

```java
@ManyToOne(fetch = FetchType.LAZY)
```

- 외래 키는 필수 관계라면 nullable = false 로 지정하세요:

```java
@JoinColumn(name = "team_id", nullable = false)
```

### 5. @ManyToOne 은 반드시 LAZY 로딩을 명시하는 것이 좋은 이유가 뭐야?

@ManyToOne(fetch = FetchType.LAZY) 를 명시적으로 지정하는 것이 좋은 이유는 다음과 같습니다:   

<br/>

✅ 1. @ManyToOne의 기본 fetch 전략은 EAGER (즉시 로딩)   
- JPA에서는 @ManyToOne과 @OneToOne 관계는 **기본값이 FetchType.EAGER**입니다.   
- 즉, 연관된 엔티티를 항상 즉시 함께 조회합니다.

```java
@ManyToOne // 기본은 FetchType.EAGER
private Team team;
```

→ 이 경우 Member를 조회하면 항상 Team도 즉시 쿼리로 같이 로딩됩니다.

<br/>

✅ 2. 불필요한 조인 성능 이슈 발생 가능   
예를 들어 List<Member>를 조회할 때 Team이 필요 없는 상황이라도, 쿼리마다 JOIN 하게 되어 쿼리 복잡도 증가 + 성능 저하를 초래합니다.   

```sql
-- 예상치 못한 조인
select m.*, t.* from member m
join team t on m.team_id = t.id;
```

- 결과적으로 N+1 문제로 이어지기 쉽습니다.

<br/>

✅ 3. 연관된 엔티티가 클 경우, 메모리 낭비 가능   
즉시 로딩은 불필요한 객체까지 로딩하므로,   
- 메모리 낭비
- 성능 저하
- 객체 그래프 순환 문제까지 발생할 수 있습니다.

✅ 4. 명시적 LAZY 설정으로 제어권 확보   
- LAZY를 사용하면, 실제로 필요한 시점에만 연관 객체를 조회하기 때문에 효율적입니다.
```java
@ManyToOne(fetch = FetchType.LAZY)
private Team team;
```
- 이후 연관 객체 접근 시점에 쿼리가 발생:

```java
Member member = memberRepository.findById(1L);
Team team = member.getTeam(); // 이 시점에 쿼리 발생 (프록시 → 실제 객체 로딩)
```

<br/>

💡 실무 팁
항상 이렇게 명시하세요:   

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "team_id")
private Team team;
```

- 연관 객체가 꼭 필요한 경우는 JPQL이나 QueryDSL의 fetch join 으로 가져오는 것이 성능적으로 훨씬 유리합니다.

### 6. N+1 문제가 뭐야?

N+1 문제는 JPA, Hibernate 같은 ORM(Object Relational Mapping)에서 자주 발생하는 성능 문제입니다.   

✅ N+1 문제란?   
- 1개의 쿼리(N=1) 로 엔티티 목록을 가져온 후, 각 엔티티에 대해 연관된 데이터를 또 다시 N개의 쿼리로 조회하는 상황입니다.   

즉, 총 N+1번의 쿼리가 실행됩니다.   

<br/>

🔍 예시로 보기   
예: Member 와 Team (다대일 관계)   

```java
@Entity
public class Member {
    @Id @GeneratedValue
    private Long id;

    private String name;

    @ManyToOne(fetch = FetchType.LAZY)
    private Team team;
}

@Entity
public class Team {
    @Id @GeneratedValue
    private Long id;
    private String name;
}
```

✅ 이 코드를 실행하면?   

```java
List<Member> members = memberRepository.findAll();
for (Member m : members) {
    System.out.println(m.getTeam().getName());
}
```

📌 실행되는 쿼리   
1. 1번 쿼리 (N=1) – 모든 Member 조회   
```sql
SELECT * FROM member;
```

2. N번 쿼리 – 각 Member에 대한 Team 조회 (LAZY 로딩이라 한 명씩 조회)
```sql
SELECT * FROM team WHERE id = ?;  -- N번 반복
```

🧨 결과: 총 N+1번의 쿼리 발생   
- 회원이 100명이라면 → 1 + 100 = 101번 쿼리
- 비효율적이고, 대규모 데이터에서 심각한 성능 저하 유발

✅ 해결 방법   
1. fetch join 사용
```java
@Query("SELECT m FROM Member m JOIN FETCH m.team")
List<Member> findAllWithTeam();
```

→ 한 번의 쿼리로 Member + Team 조인해서 조회   

```sql
SELECT m.*, t.* FROM member m
JOIN team t ON m.team_id = t.id;
```

2. @EntityGraph 사용
```java
@EntityGraph(attributePaths = {"team"})
List<Member> findAll();
```

→ 필요한 연관 관계를 한 번에 불러오도록 명시

### 7. fetch join을 설명해 줘

✅ fetch join이란?   
- 연관된 엔티티를 SQL 조인으로 한 번에 조회하여 N+1 문제를 해결합니다.
- JPQL에서만 사용 가능한 기능입니다.

<br/>

📌 문법   
```jpql
SELECT e FROM Entity e JOIN FETCH e.relation
```
- JOIN FETCH 구문을 사용해야만 fetch join입니다.
- 단순한 JOIN은 fetch join이 아닙니다 (지연로딩 유지됨).

<br/>

🔍 예제   
엔티티 구조   
```java
@Entity
public class Member {
    @Id @GeneratedValue
    private Long id;

    private String name;

    @ManyToOne(fetch = FetchType.LAZY)
    private Team team;
}
```

<br/>

일반 JOIN (지연로딩 그대로 유지)   
```java
@Query("SELECT m FROM Member m JOIN m.team")
List<Member> findMembers();
```
- 이 경우 Member는 조회되지만, Team은 여전히 지연 로딩됨 → 이후 N+1 문제 발생

<br/>

✅ fetch join
```java
@Query("SELECT m FROM Member m JOIN FETCH m.team")
List<Member> findMembersWithTeam();
```
- Member 와 연관된 Team 을 한 번의 쿼리로 함께 조회
- 실제 실행되는 SQL:
```sql
SELECT m.*, t.* FROM member m
JOIN team t ON m.team_id = t.id
```

<br/>

📈 장점
|항목|설명|
|---|---|
|성능|쿼리 수를 획기적으로 줄임 (N+1 문제 해결)|
|제어|지연 로딩 전략을 유지하면서 필요한 시점에만 즉시 로딩 가능|
|효율|불필요한 쿼리 제거로 DB 부담 완화|

<br/>

⚠ 주의사항   
1. 컬렉션(OneToMany) 을 fetch join하면 중복 row가 생깁니다. → distinct 필요   

```java
SELECT DISTINCT t FROM Team t JOIN FETCH t.members
```

2. fetch join은 JPQL에서만 사용 가능 (Criteria API, Native SQL에서는 다르게 처리)

3. 페이징과 함께 사용 시 주의: @OneToMany fetch join + Pageable → 메모리에서 페이징 처리됨 (비효율)


### 8. jpql이 뭐야

JPQL은 Java Persistence Query Language의 약자로, JPA에서 엔티티 객체를 대상으로 쿼리하는 객체지향 쿼리 언어입니다.    

<br/>

✅ JPQL이란?   
SQL은 테이블을 대상으로, JPQL은 엔티티를 대상으로 쿼리합니다.   

- JPA의 표준 쿼리 언어
- 객체(Entity)와 그 필드를 기준으로 작성
- 데이터베이스에 독립적인 쿼리를 작성할 수 있게 해 줌

<br/>

✅ JPQL vs SQL      

|구분|SQL|JPQL|
|---|---|---|
|대상|테이블|엔티티|
|컬럼|컬럼 이름|엔티티 필드 이름|
|결과|테이블 row|자바 객체|
|예외|DB에 따라 달라짐|JPA가 처리함 (DB 독립적)|

🔍 예제 비교   
SQL   
```sql
SELECT * FROM member WHERE name = 'kim';
```
JPQL   
```java
@Query("SELECT m FROM Member m WHERE m.name = 'kim'")
List<Member> findByNameKim();
```
- 여기서 Member는 엔티티 클래스, m.name은 필드 이름
- JPA가 내부적으로 SQL로 변환해 실행

<br/>

✅ 주요 문법 요소   
- SELECT, FROM, WHERE, JOIN, ORDER BY, GROUP BY, HAVING
- JOIN FETCH, DISTINCT, SUBQUERY, CASE, IN, BETWEEN, IS NULL 등도 사용 가능

✅ 왜 JPQL을 써야 할까?   
|이유|설명|
|---|---|
|객체 중심 개발|테이블이 아닌 객체 중심으로 쿼리를 작성|
|DB 독립성|다른 DB로 변경해도 대부분 쿼리 수정 불필요|
|연관관계 탐색|조인 없이도 객체 그래프 탐색 가능 (m.team.name)|
|성능 최적화|fetch join, @Query, @EntityGraph 등으로 제어 가능|

✅ 실무 예   
```java
@Query("SELECT o FROM Order o JOIN FETCH o.member m WHERE m.name = :name")
List<Order> findOrdersByMemberName(@Param("name") String name);
```

### 8. @Embeddable 애노테이션을 설명해 줘

@Embeddable은 JPA에서 **값 타입(Value Type)**을 정의할 때 사용하는 애노테이션입니다.   
하나의 엔티티에 속하는, 독립적으로 존재하지 않는 객체를 만들고 싶을 때 사용합니다.   

<br/>

✅ 개념 요약   
- @Embeddable: 값 타입 객체의 클래스에 사용
- @Embedded: 엔티티 내부에서 값을 포함할 때 사용
- DB에는 별도 테이블 없이 엔티티의 컬럼으로 병합됨

<br/>

🔍 예제   
📦 주소 정보를 값 타입으로 만들기   
```java
@Embeddable
public class Address {
    private String city;
    private String street;
    private String zipcode;
}
```
- 👤 Member 엔티티에 Address 포함

```java
@Entity
public class Member {
    @Id @GeneratedValue
    private Long id;

    private String name;

    @Embedded
    private Address address;
}
```

🧾 저장되는 테이블 구조 (컬럼 이름)   
```sql
member (
  id BIGINT,
  name VARCHAR,
  city VARCHAR,
  street VARCHAR,
  zipcode VARCHAR
)
```
- Address 객체는 별도 테이블 없이 Member 테이블의 일부로 포함됨
- 즉, Embedded 값들은 해당 엔티티의 필드처럼 저장됨 

✅ 언제 쓰나?   

|경우|이유|
|---|---|
|주소, 전화번호, 기간|자주 쓰이는 값을 재사용하고 싶을 때|
|객체 지향적 설계|복잡한 값 객체를 하나로 묶어 표현|
|재사용|여러 엔티티에서 같은 값 객체 사용 가능|

✅ 장점   
- 코드 재사용성 ↑
- 연관관계 없음 → 단순한 구조
- 내장 타입을 분리해 객체지향적으로 관리 가능

<br/>

⚠ 주의사항   
- @Embeddable 클래스는 엔티티처럼 @Entity로 관리되지 않음
- @Embeddable 클래스는 기본 생성자 필요
- equals(), hashCode() 재정의 권장 → 값 객체의 비교는 값 기준


### 9. jpa에서 Cascade all이 무엇인지 알려주고 언제 사용하면 좋은지도 알려줘

JPA에서 cascade = CascadeType.ALL은 엔티티 간의 연관 관계에서 관련된 작업을 자동으로 전파하는 기능입니다.   

<br/>

✅ Cascade란?   
- 부모 엔티티의 작업(저장, 삭제 등)을 연관된 자식 엔티티에도 전파하는 기능
- 예를 들어, 부모 엔티티를 저장할 때 자식 엔티티도 자동으로 저장되게 하는 거예요.

<br/>

✅ CascadeType.ALL의 의미   
- CascadeType.ALL은 아래 모든 cascade 동작을 포함합니다:

|타입|설명|
|---|---|
|PERSIST|저장 전파 (자식도 함께 저장)|
|MERGE|병합 전파 (자식도 함께 병합)|
|REMOVE|삭제 전파 (자식도 함께 삭제)|
|REFRESH|새로 고침 전파|
|DETACH|영속성 분리 전파|

즉, CascadeType.ALL은 모든 cascade 동작을 포함한 종합 세트입니다.   

<br/>

🔍 예제   
```java
@Entity
public class Parent {
    @Id @GeneratedValue
    private Long id;

    @OneToMany(mappedBy = "parent", cascade = CascadeType.ALL)
    private List<Child> children = new ArrayList<>();
}

@Entity
public class Child {
    @Id @GeneratedValue
    private Long id;

    @ManyToOne
    private Parent parent;
}
```

```java
Parent parent = new Parent();
Child child1 = new Child();
Child child2 = new Child();

child1.setParent(parent);
child2.setParent(parent);
parent.getChildren().add(child1);
parent.getChildren().add(child2);

em.persist(parent); // 🎯 자식들도 자동 persist!
```

- cascade = CascadeType.ALL 덕분에 child1, child2를 별도로 persist하지 않아도 저장됩니다.

<br/>

✅ 언제 사용하면 좋을까?   
|상황|사용 이유|
|---|---|
|부모-자식 엔티티가 생명주기를 완전히 같이 할 때|한 쪽만 관리하면 되므로 코드 단순화|
|예: 주문(Order)과 주문상품(OrderItem), 게시글(Post)과 댓글(Comment)|함께 저장/삭제되기 때문|

<br/>

⚠ 주의사항   
|주의점|설명|
|---|---|
|생명주기가 다르면 사용 X|예: 회원(Member)과 주소(Address)처럼 독립적이면 Cascade는 부적절|
|무분별한 REMOVE 전파는 위험|자식까지 다 삭제되므로 주의 필요|
|실수로 모든 엔티티 삭제 가능|테스트나 실서비스에서 주의해야 함|


### 10. DAO를 사용해서 개발하면 좋은 점이 뭐야?

DAO(Data Access Object)를 사용하면 애플리케이션의 데이터 접근 로직을 분리할 수 있어서 여러 가지 이점이 있습니다.   

<br/>

✅ DAO란?   
**DAO (Data Access Object)**는 데이터베이스에 접근하는 코드만을 담당하는 객체입니다.   
예: DB 연결, 쿼리 실행, 결과 매핑 등 → DAO가 데이터 저장소와 직접 통신합니다.   

<br/>

✅ DAO를 사용하면 좋은 점   
|장점|설명|
|---|---|
|1. 관심사 분리 (Separation of Concerns)|비즈니스 로직과 DB 처리 코드를 분리함으로써 코드가 깔끔해져요.|
|2. 유지보수 용이|DB 관련 로직이 한 곳에 모여 있어, 쿼리 수정이나 DB 구조 변경 시 대응이 쉬워요.|
|3. 테스트 용이|DAO를 mocking하면 DB 없이도 비즈니스 로직 테스트 가능해요.|
|4. 재사용성|여러 서비스나 컴포넌트가 동일 DAO를 재사용할 수 있어요.|
|5. 유연한 구조 변경|RDB에서 NoSQL로 바꿔야 할 때 DAO만 수정하면 돼요.|
|6. 명확한 책임 분리|서비스 계층은 '무엇을' 할지, DAO는 '어떻게' 할지를 책임져요.|

<br/>

✅ DAO 구조 예시   

```java
// DAO 인터페이스
public interface MemberDao {
    Member findById(Long id);
    void save(Member member);
    void delete(Long id);
}

// 구현체
public class JdbcMemberDao implements MemberDao {
    public Member findById(Long id) {
        // JDBC 코드
    }
    public void save(Member member) {
        // insert 쿼리 실행
    }
}
```

✅ 비즈니스 로직 예시   

```java
public class MemberService {
    private MemberDao memberDao;

    public MemberService(MemberDao memberDao) {
        this.memberDao = memberDao;
    }

    public void register(Member member) {
        memberDao.save(member);  // DB 저장은 DAO에 위임
    }
}
```

✅ DAO는 어디서 주로 쓰이나?   

- 전통적인 Java 애플리케이션 (JDBC, MyBatis 등)
- Spring Framework에서 @Repository로 구현
- ORM (JPA) 사용 시에는 Repository가 DAO 역할 수행

<br/>

✅ 요약   

|항목|설명|
|---|---|
|정의|DB 접근을 담당하는 객체|
|핵심 기능|쿼리 실행, 결과 매핑, 연결 관리|
|장점|관심사 분리, 테스트 용이, 재사용성, 유지보수 편리|
|현대적 대체|JPA의 Repository나 Spring Data JPA가 DAO 역할을 추상화|

### 11. DAO를 사용할 때 장점을 트랜잭션 스크립트 구조 개발했을 때와 비교해서 설명해 줘

DAO 패턴과 트랜잭션 스크립트 패턴은 둘 다 실제 서비스 로직을 구성할 때 쓰이지만, 관심사의 분리 수준, 코드 조직 방식, 유지보수성 등에서 차이가 있어요.   

<br/>

✅ 비교 대상 정의   

|패턴|개념 요약|
|---|---|
|트랜잭션 스크립트|서비스 로직과 DB 접근 로직을 한 메서드에서 처리하는 방식|
|DAO 패턴|DB 접근 로직을 별도 DAO 클래스에 분리하여 캡슐화하는 방식|

<br/>

✅ 코드 예시 비교   
🔸 트랜잭션 스크립트 예   

```java
public void registerMember(String name) {
    Connection conn = getConnection();
    PreparedStatement stmt = conn.prepareStatement("INSERT INTO member (name) VALUES (?)");
    stmt.setString(1, name);
    stmt.executeUpdate();
}
```

- 서비스 로직과 DB 접근이 한 곳에 있음
- 코드가 짧고 직관적이지만, 확장성과 재사용성이 떨어짐

🔹 DAO 패턴 예   

```java
// DAO
public class MemberDao {
    public void save(Member member) {
        Connection conn = getConnection();
        PreparedStatement stmt = conn.prepareStatement("INSERT INTO member (name) VALUES (?)");
        stmt.setString(1, member.getName());
        stmt.executeUpdate();
    }
}

// 서비스
public class MemberService {
    private MemberDao memberDao;

    public void register(Member member) {
        memberDao.save(member); // DB 로직 분리
    }
}
```

- 서비스는 비즈니스에만 집중, DAO는 DB 처리만 담당

<br/>

✅ 비교 정리   

|항목|트랜잭션 스크립트|DAO 사용 시|
|---|---|---|
|책임 분리|없음 → 한 메서드에 서비스 + DB 로직|명확하게 분리됨 (Service ↔ DAO)|
|재사용성|낮음|높음 (여러 서비스에서 DAO 재사용)|
|유지보수|변경 시 복잡도 증가|DAO에만 변경 → 영향도 적음|
|테스트 용이성|어렵다 (DB 의존적)|Mock DAO로 단위 테스트 쉬움|
|확장성|구조 변경 어렵다|계층 구조로 확장 가능|
|표준화|로직마다 다른 코드 스타일 가능성|일관된 데이터 접근 방식 가능|

<br/>

✅ 언제 DAO 패턴이 더 좋은가?   

- 프로젝트 규모가 커질수록
- 팀 개발 시 각 역할을 분리하고자 할 때
- 테스트 자동화가 중요한 환경
- 비즈니스 로직이 복잡하거나 재사용성이 높은 구조를 원할 때

### 12. PlatformTransactionManager와 JDBCTemplate의 차이점을 설명해 줘

PlatformTransactionManager와 JdbcTemplate은 모두 데이터베이스와 관련된 작업을 도와주는 스프링의 핵심 컴포넌트지만, 역할과 책임이 다릅니다.   

<br/>

✅ 요약 비교   

|항목|JdbcTemplate|PlatformTransactionManager|
|---|---|---|
|역할|SQL 실행 도우미 (DB 작업 처리)|트랜잭션 관리 도우미|
|관심사|SQL 실행, 결과 매핑|트랜잭션 시작, 커밋, 롤백|
|주 사용 목적|JDBC 작업을 간편하게 수행|트랜잭션을 프로그래밍적/선언적으로 관리|
|예외 처리|스프링 예외 전환 처리 포함|트랜잭션 실패 시 롤백 수행|
|결합 여부|내부에서 트랜잭션 매니저를 사용할 수 있음|직접 DB 작업은 하지 않음|

<br/>

✅ 역할 설명   
1. 🔹 JdbcTemplate

- 목적: JDBC API를 쉽게 사용할 수 있게 도와주는 도구
- 하는 일:
	- SQL 실행 (SELECT, INSERT, UPDATE, DELETE)
	- 커넥션/스테이트먼트 관리 자동화
	- ResultSet을 객체로 매핑
	- checked 예외 → unchecked 예외로 변환

예시 코드:   

```java
@Autowired
private JdbcTemplate jdbcTemplate;

public List<Member> findAll() {
    return jdbcTemplate.query("SELECT * FROM member",
        new BeanPropertyRowMapper<>(Member.class));
}
```
<br/>

2. 🔸 PlatformTransactionManager

- 목적: 트랜잭션 시작, 커밋, 롤백 등을 관리
- 하는 일:
	- 트랜잭션 경계 설정
	- 트랜잭션 상태 추적
	- 예외 발생 시 자동 롤백
- 다양한 구현체 존재:
	- DataSourceTransactionManager (JDBC 기반)
	- JpaTransactionManager (JPA 기반)
	- HibernateTransactionManager (Hibernate 전용)

<br/>

예시 코드 (프로그래밍 방식):   

```java
TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());
try {
    jdbcTemplate.update("INSERT INTO member(name) VALUES(?)", "kim");
    transactionManager.commit(status);
} catch (RuntimeException e) {
    transactionManager.rollback(status);
    throw e;
}
```

✅ 함께 쓰는 예 (트랜잭션 + SQL)   

```java
@Service
public class MemberService {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Autowired
    private PlatformTransactionManager transactionManager;

    public void saveMemberTransactional(String name) {
        TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());
        try {
            jdbcTemplate.update("INSERT INTO member (name) VALUES (?)", name);
            transactionManager.commit(status);
        } catch (Exception e) {
            transactionManager.rollback(status);
        }
    }
}
```

- 이처럼 JdbcTemplate은 SQL 실행을, PlatformTransactionManager는 트랜잭션 경계 관리를 담당합니다.

<br/>

✅ 결론
|항목|설명|
|---|---|
|JdbcTemplate|JDBC를 쉽게 쓰기 위한 SQL 실행 도우미|
|PlatformTransactionManager|트랜잭션을 시작하고 커밋하거나 롤백하는 컨트롤러|
|함께 사용 가능|JdbcTemplate으로 작업하고 TransactionManager로 트랜잭션을 제어|
