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