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

JPA의 ```new```명령어를 사용해 DTO로 직접 조회할 수 있다. 사용 방식은 JPA와 동일하다.   

```java
public interface MemberRepository extends JpaRepository<Member, Long>, MemberRepositoryCustom {
    ...
    @Query("select new study.datajpa.dto.MemberDto(m.id, m.username, t.name) from Member m join m.team t")
    List<MemberDto> findMemberDto();
    ...
}

@Data
public class MemberDto {

    private Long id;
    private String username;
    private String teamName;

    public MemberDto(Long id, String username, String teamName) {
        this.id = id;
        this.username = username;
        this.teamName = teamName;
    }

    public MemberDto(Member member) {
        this.id = member.getId();
        this.username = member.getUsername();
    }
}
```

#### 3-5. 파라미터 바인딩

파라미터 바인딩은 위치 또는 이름 기반으로 사용할 수 있다.   
위치 기반으로 사용할 경우 코드 가독성이 떨어지고 휴먼 에러가 발생할 가능성이 높기 때문에 사용하지 않고 이름 기반으로 사용한다.    
그리고 ```Collection```타입으로 in절을 지원한다.   

```java
public interface MemberRepository extends JpaRepository<Member, Long>, MemberRepositoryCustom {
    ...
    @Query("select m from Member m where m.username = :name")
    Member findMembers(@Param("name") String name);

    @Query("select m from Member m where m.username in :names")
    List<Member> findByNames(@Param("names") Collection<String> names);
    ...
}
```

#### 3-6. 반환 타입

스프링 데이터 JAP는 여러 반환 타입을 지원한다.   
리턴 타입에 대한 자세한 내용은 [공식 문서](https://docs.spring.io/spring-data/jpa/reference/repositories/query-return-types-reference.html#appendix.query.return.types)에서 확인할 수 있다.   

```java
List<Member> findByUsername(String name);
Member findByUsername(String name);
Optional<Member> findByUsername(String name);
```

반환 타입이 컬렉션일 때 조회 결과가 없으면 빈 컬렉션을 반환한다.   
단건 조회에서 결과가 없으면 ```null```을 반환하고 결과가 2건 이상이면 ```javax.persistence.NonUniqueResultException``` 예외가 발생한다.   

단건으로 지정한 메서드를 호출하면 스프링 데이터 JPA는 내부적으로 JPQL의 ```getSingleResult()``` 메서드를 호출한다. 이 메서드는 조회 결과가 없으면 ```javax.persistence.NoResultException``` 예외가 발생하는데, 개발자 입장에서 다루기 불편하기 떄문에 스프링 데이터 JPA는 이 예외를 무시하고 ```null```을 반환한다.   

#### 3-7. 순수 JPA 페이징과 정렬

순수 JPA에서 ```setFirstResult(offset)```, ```setMaxResults(limit)``` 메소드를 사용해 페이징을 사용할 수 있다.    
나이가 10살이고 이름으로 내림차순한 결과를 페이지 당 3건, 첫 번째 페이지를 조회할 수 있는 JPA 페이징 레포지토리 코드는 다음과 같다.   

```java
public List<Member> findByAge(int age, int offset, int limit) {
    return em.createQuery("select m from Member m where m.age = :age order by m.username desc")
            .setParameter("age", age)
            .setFirstResult(offset)
            .setMaxResults(limit)
            .getResultList();
}

public long totalCount(int age) {
    return em.createQuery("select count(m) from Member m where m.age = :age", Long.class)
            .setParameter("age", age)
            .getSingleResult();
}
```

#### 3-8. 스프링 데이터 JPA 페이징과 정렬

스프링 데이터 JPA는 페이징을 위해 페이징과 정렬 파라메터 그리고 특별한 반환 타입을 사용한다.   

##### 페이징과 정렬 파라메터

- ```org.springframework.data.domain.Sort```: 정렬 기능
- ```org.springframework.data.domain.Pageable```: 페이징 기능(내부에 ```Sort``` 포함)

##### 특별한 반환 타입

- ```org.springframework.data.doamin.Page```: 추가 count 쿼리 결과를 포함하는 페이징
- ```org.springframework.data.domain.Slice```: 추가 count 쿼리 없이 다음 페이지만 확인 가능(내부적으로 limit + 1 조회)
- ```List```: 추가 count 쿼리 없이 결과만 반환   

##### 페이징과 정렬 사용 예

```java
Page<Member> findByUsername(String name, Pageable pageable); // count 쿼리 사용
Slice<Member> findByUsername(String name, Pageable pageable); // count 쿼리 사용 안 함
List<Member> findByUsername(String name, Pageable pageable); // count 쿼리 사용 안 함
List<Member> findByUsername(String name, Sort sort);
```

순수 JPA로 페이징을 사용할 때와 같은 조건인 나이가 10살이고 이름으로 내림차순한 결과를 페이지 당 3건, 첫 번째 페이지를 조회할 수 있는 스프링 데이터 JPA는 다음과 같다.   

##### Page 사용 예제 정의 코드

```java
public interface MemberRepository extends JpaRepository<Member, Long> {
    Page<Member> findByAge(int age, Pageable pageable);
}
```

##### Page 사용 테스트 코드

```java
@Test
public void page() throws Exception {

    memberRepository.save(new Member("member1", 10));
    memberRepository.save(new Member("member2", 10));
    memberRepository.save(new Member("member3", 10));
    memberRepository.save(new Member("member4", 10));
    memberRepository.save(new Member("member5", 10));
   
    PageRequest pageRequest = PageRequest.of(0, 3, Sort.by(Sort.Direction.DESC, 
"username"));
    Page<Member> page = memberRepository.findByAge(10, pageRequest);
    //then
    List<Member> content = page.getContent(); //조회된 데이터
    assertThat(content.size()).isEqualTo(3); //조회된 데이터 수 
    assertThat(page.getTotalElements()).isEqualTo(5); //전체 데이터 수
    assertThat(page.getNumber()).isEqualTo(0); //페이지 번호
    assertThat(page.getTotalPages()).isEqualTo(2); //전체 페이지 번호
    assertThat(page.isFirst()).isTrue(); //첫번째 항목인가?
    assertThat(page.hasNext()).isTrue(); //다음 페이지가 있는가?
}
```

- 두 번쨰 파라메터로 받은 ```Pageable```은 인테퍼이스이다. 실제 사용할 떄는 이 인터페이스를 구현한 ```org.springframework.data.domain.PageRequest``` 객체를 사용한다.   
- ```PageRequest``` 생성자의 첫 번째 파라메터에는 현재 페이지, 두 번째 파라메터에는 조회할 데이터 수를 입력한다. 추가로 정렬 정보도 파라메터로 사용할 수 있다. 그리고 현재 페이지는 0 부터 시작한다.

##### Page 인터페이스

```java
public interface Page<T> extends Slice<T> {
    int getTotalPages(); // 전체 페이지 수
    long getTotalElements(); // 전체 데이터 수
    <U> Page<U> map(Function<? super T, ? extends U> converter); // 변환기
}
```

##### Slice 인터페이스

```java
public interface Slice<T> extends Streamable<T> {
    int getNumber();    // 현재 페이지
    int getSize();      // 페이지 크기
    int getNumberOfElements();  // 현재 페이지에 나올 데이터 수
    List<T> getContent();   // 조회된 데이터
    boolean hasContent();   // 조회된 데이터 존재 여부
    Sort getSort();     // 정렬 정보
    boolean isFirst();  // 현재 페이지가 첫 페이지 인지 여부
    boolean isLast();   // 현재 페이지가 마지막 페이지 인지 여부
    boolean hasNext();  // 다음 페이지 여부
    boolean hasPrevious();  // 이전 페이지 여부
    Pageable getPageable(); // 페이지 요청 정보
    Pageable nextPageable();    // 다음 페이지 객체
    Pageable previousPageable();    // 이전 페이지 객체
    <U> Slice<U> map(Function<? super T, ? extends U> converter); // 변환기
}
```

##### count 쿼리를 다음과 같이 분리 가능

left join을 활용한 복잡한 쿼리를 사용할 때 데이터는 left join이 필요하지만, 카운트는 left join이 필요없다.   
주어진 쿼리에서 count 함수를 사용하기 때문에 카운트 값을 구할 때 불필요한 left join이 발생한다.   
카운트 쿼리를 분리해 불필요한 left join을 줄여 성능 최적화를 할 수 있고 실무에서 자주 사용된다.   

```java
@Query(value = "select m from Member m",
        countQuery = "select count(m.username) from Member m")
Page<Member> findMemberAllCountBy(Pageable pageable);
```

##### 페이지를 유지하면서 엔티티를 DTO로 변환하기

변환기인 map 메소드를 사용해 페이지 정보를 유지하면서 결과만 DTO 객체로 변환할 수 있다.

```java
Page<Member> page = memberRepository.findByAge(10, pageRequest);
Page<MemberDto> dtoPage = page.map(m -> new MemberDto());
```

##### 스프링부트 3 - 하이버네이트 6 left join 최적화

스프링부트 3 이상을 사용하면 하이버네이트 6가 적용된다.   
```select m from Member m left join m.team t``` 이 JPQL에서 ```Team```의 정보를 조회하지 않기 때문에,   
```Member```와 ```Team```의 left join은 실제로 필요하지 않다.   
```Member```만 조회하는 것과 같기 때문에 JPA는 최적화를 해서 join이 없이 ```Member``` 만으로 쿼리를 만든다.   
최적화를 무시하고 ```Member```와 ```Team```을 한 번에 조회하려면 아래와 같이 ```fetch join```을 사용해야 한다.   

```select m from Member m left join fetch m.team t```

#### 3-9. 벌크성 수정 쿼리

##### JPA를 사용한 벌크성 수정 쿼리

```java
@Repository
public class MemberJpaRepository {
    ...
    public int bulkAgePlus(int age) {
        return em.createQuery("update Member m set m.age = m.age + 1 where m.age >= :age")
                .setParameter("age", age)
                .executeUpdate();
    }
}
```

##### 스프링 데이터 JPA를 사용한 벌크성 수정 쿼리

```java
public interface MemberRepository extends JpaRepository<Member, Long> {    
    ...
    @Modifying(clearAutomatically = true)
    @Query("update Member m set m.age = m.age + 1 where m.age >= :age")
    int bulkAgePlus(@Param("age") int age);
    ...
}
```

스프링 데이터 JPA에서 ```@Query```를 쓰면 기본적으로 select 쿼리로 인식한다.   
그래서 ```UPDATE```, ```CREATE```, ```DELETE``` 쿼리를 수행할 때는 ```@Modifying``` 애노테이션을 사용해야 한다.   
사용하지 않으면 ```org.hibernate.hql.internal.QueryExecutionRequestException: Not supported for DML operations``` 예외가 발생한다.   
```@Modifying```이 붙은 쿼리는 ```executeUpdate()```를 호출하고 영향을 받은 row 수를 ```int``` 타입으로 반환한다.   
```@Modifying``` 애노테이션을 사용할 때는 트랜잭션이 꼭 필요하고 트랜잭션 없이 호출되면 예외가 발생한다.   

<br/>

벌크성 쿼리는 영속성 컨텍스트를 무시하고 실행하기 때문에 영속성 컨텍스트에 있는 엔티티의 상태와 DB의 엔티티 상태가 달라질 수 있다.   
벌크성 쿼리 실행 후 ```findById```로 다시 조회하면 영속성 컨텍스트에 과거 값이 남아서 문제가 될 수 있다.    
벌크성 쿼리를 실행하고 나서 다시 조회해야 하면 꼭 영속성 컨텍스트를 초기화 해야한다.   
```@Modifying(clearAutomatically = true)```를 사용하면 벌크성 쿼리를 실행하고 나서 영속성 컨텍스트를 초기화 한다.   

#### 3-10. @EntityGraph

member → team과 같은 지연로딩 관계일 때 member 엔티티에서 team의 데이터를 조회할 때 마다 쿼리가 실행된다.   
이 때 불필요한 쿼리가 호출되는 N+1 문제가 발생하고 해결을 위해 JPQL에서 fetch join을 사용한다.   

스프링 데이터 JPA는 JPA가 제공하는 엔티티 그래프 기능을 편리하게 사용하도록 도와준다.    
이 기능을 사용하면 JPQL 없이 fetch join을 사용할 수 있다.   

```java
public interface MemberRepository extends JpaRepository<Member, Long> { 
    ...
    // 공통 메서드 오버라이드
    @Override
    @EntityGraph(attributePaths = { "team" })
    List<Member> findAll();

    // JPQL + 엔티티 그래프
    @EntityGraph(attributePaths = { "team" })
    @Query("select m from Member m")
    List<Member> findMemberEntityGraph();

    // 메서드 이름으로 쿼리 + 엔티티 그래프
    @EntityGraph(attributePaths = { "team" })
    List<Member> findEntityGraphByUsername(@Param("username") String username);
    ...
}
```

EntityGraph가 기본적으로 INNER JOIN을 사용하는 fetch join과 다른 점은 LEFT OUTER JOIN을 사용하는 것이다.   
왜냐하면 연관 엔티티가 없더라도 부모 엔티티는 조회되도록 하기 위해서이다.   

#### 3-11. JPA Hint & Lock

JPA 쿼리 힌트는 Hibernate와 같은 JPA 구현체에 추가적인 실행 지시사항을 제공하는 기능이다.   
SQL 힌트가 아니라 JPA 구현체에서 제공하는 힌트이다.   

일반적으로 읽기 전용 조회 최적화에 가장 많이 사용하고 아래와 같은 경우에도 사용한다.   

- 읽기 전용 데이터를 영속성 컨텍스트에서 관리하지 않게 해서 성능 향상(Dirty check 최소화)
- 특정 쿼리를 캐시에서 제외하거나 포함
- Hibernate 등의 구현체에 최적화를 지시

자주 쓰이는 힌트 종류는 아래와 같다.   

|힌트|이름|설명|
|---|---|---|
|org.hibernate.readOnly|true면 읽기 전용 → dirty checking 하지 않음|
|org.hibernate.cacheable|true면 2차 캐시에 캐싱|
|org.hibernate.fetchSize|JDBC fetch size 설정|
|org.hibernate.timeout|JDBC 쿼리 timeout 설정|
|javax.persistence.query.timeout|밀리초 단위 쿼리 timeout 설정 (JPA 표준)|

```java
public interface MemberRepository extends JpaRepository<Member, Long> { 
    ...
    @QueryHints(value = @QueryHint( name = "org.hibernate.readOnly", value = "true"))
    Member findReadOnlyByUsername(@Param("username") String username);
    ...
}

@SpringBootTest
@Transactional
@Rollback(false)
class MemberRepositoryTest {

    @Autowired MemberRepository memberRepository;
    @Autowired TeamRepository teamRepository;
    @PersistenceContext
    EntityManager em;
    ...
    @Test
    public void queryHint() {
        Member member1 = new Member("member1", 10);
        memberRepository.save(member1);
        em.flush();
        em.clear();

        Member findMember = memberRepository.findReadOnlyByUsername("member1");
        findMember.setUsername("member2"); // @QueryHint가 readOnly true라서 반영 안 됨

        em.flush(); // Update Query가 실행되지 않음
    }
    ...
}
```

##### 쿼리 힌트 Page 추가 예제

반환 타입으로 ```Page``` 인터페이스를 적용하면 추가로 호출하는 페이징을 위한 count 쿼리도 쿼리 힌트 적용(기본값 ```true```)   

```java
@QueryHints(value = { @QueryHint(name = "org.hibernate.readOnly", value = "true")}, forCounting = true)
 Page<Member> findByUsername(String name, Pageable pageable);
```
