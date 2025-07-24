---
layout: post
title: 스프링 데이터 JPA 2
date: 2025-07-23 04:42:00 + 0900
categories: [spring]
tags: [spring, jpa]
---

### 강의 : [실전! 스프링 데이터 JPA](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-%EB%8D%B0%EC%9D%B4%ED%84%B0-JPA-%EC%8B%A4%EC%A0%84)

### 3. 쿼리 메소드 기능

#### 3-1. 메소드 이름으로 쿼리 생성

스프링 데이터 JPA는 메소드 이름을 분석해서 JPQL을 생성하고 실행한다.   

```java
@Repository
public class MemberJpaRepository {

    @PersistenceContext
    private EntityManager em;

    ...
    public List<Member> findByUsernameAndAgeGreaterThan(String username, int age) {
        return em.createQuery("select m from Member m where m.username = :username and m.age > :age", Member.class)
                .setParameter("username", username)
                .setParameter("age", age)
                .getResultList();
    }
    ...
}
```

위와 같이 작성된 순수 JPA 코드를 스프링 데이터 JPA는 메소드 선언 만으로 만들 수 있다.    
두 메소드는 동일한 쿼리를 수행한다.   

```java
public interface MemberRepository extends JpaRepository<Member, Long>, MemberRepositoryCustom {

    List<Member> findByUsernameAndAgeGreaterThan(String username, int age);
    ...
}
```

스프링 데이터 JPA가 제공하는 쿼리 메소드 기능은 아래와 같다.   
...에는 식별하기 위한 내용이 들어갈 수 있다.

- 조회: find...By, read...By, query...By, get...By
- COUNT: count...By 반환타입 ```long```
- EXISTS: exists...By 반환타임 ```boolean```
- 삭제: delete...By, remove...By 반환타입 ```long```
- DISTINCT: findDistinct, findMemberDistinctBy
- LIMIT: findFirst3, findFirst, findTop, findTop3

이 외에도 And, Betwwen, LessThan, GreaterThan 등의 키워드가 있다.    
자세한 내용은 [스프링 Docs](https://docs.spring.io/spring-data/jpa/reference/jpa/query-methods.html#jpa.query-methods.query-creation)에서 확인할 수 있다.   

#### 3-2. JPA NamedQuery

JPA의 NamedQuery를 호출하기 위한 순수 JPA 코드는 아래와 같다.   

```java
@Entity
@Getter @Setter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@ToString(of = { "id", "username", "age" })
@NamedQuery(
        name = "Member.findByUsername",
        query = "select m from Member m where m.username = :username"
)
public class Member extends BaseEntity {

    @Id
    @GeneratedValue
    @Column(name = "member_id")
    private Long id;
    private String username;
    private int age;
    ...
}

@Repository
public class MemberJpaRepository {

    @PersistenceContext
    private EntityManager em;

    ...
    public List<Member> findByUsername(String username) {
        return em.createNamedQuery("Member.findByUsername", Member.class)
                .setParameter("username", username)
                .getResultList();
    }
    ...
}
```

스프링 데이터 JPA에선 아래 코드와 같이 호출할 수 있다.   

```java
public interface MemberRepository extends JpaRepository<Member, Long>, MemberRepositoryCustom {
    ...
    @Query(name = "Member.findByUsername")
    List<Member> findByUsername(@Param("username") String username);
    ...
}
```

```@Query```애노테이션을 생략하고 메서드 이름만으로 Named Query를 호출할 수 있는데, "도메인 클래스 + . + 메소드 이름"으로 찾는다.   
실행할 Named Query가 없으면 메서드 이름으로 쿼리를 생성해 사용한다.   

JPA Named 쿼리는 애플리케이션 실행 시점에 문법 오류를 발견할 수 있는 장점이 있지만,
실무에선 보통 Named Query를 등록해 사용하는 경우는 드물고 ```@Query```를 사용해 Repository 메소드에 직접 정의해 사용한다.   

#### 3-3. @Query, Repository 메소드에 쿼리 정의하기

```@org.springframework.data.jpa.repository.Query``` 애노테이션을 사용하고 애플리케이션 실행 시점에 문법 오류를 발견할 수 있다.   

```java
public interface MemberRepository extends JpaRepository<Member, Long>, MemberRepositoryCustom {
    ...
    @Query("select m from Member m where m.username = :username and m.age = :age")
    List<Member> findUser(@Param("username") String username, @Param("age") int age);
    ...
}
```

#### 3-4. @Query, 값, DTO 조회하기


#### 3-5. 파라미터 바인딩


#### 3-6. 반환 타입


#### 3-7. 순수 JPA 페이징과 정렬


#### 3-8. 스프링 데이터 JPA 페이징과 정렬


#### 3-9. 벌크성 수정 쿼리


#### 3-10. @EntityGraph


#### 3-11. JPA Hint & Lock