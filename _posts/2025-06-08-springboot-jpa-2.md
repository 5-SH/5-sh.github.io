---
layout: post
title: 스프링 부트와 JPA 활용2
date: 2025-06-08 20:31:00 + 0900
categories: [spring]
tags: [spring, jpa]
---

### 강의 : [실전! 스프링 부트와 JPA 활용1 - 웹 애플리케이션 개발](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81%EB%B6%80%ED%8A%B8-JPA-%ED%99%9C%EC%9A%A9-1/dashboard)

## 질문

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


### 13. 폼 데이터와 @ModelAttribute, @RequestParam 애노테이션 사용법을 정리해서 알려줘

✅ 1. 공통 전제: HTML 폼에서 전송되는 데이터   
HTML ```<form>```은 기본적으로 POST 방식과 application/x-www-form-urlencoded 콘텐츠 타입으로 데이터를 전송합니다:   

```html
<form action="/submit" method="post">
  <input type="text" name="name">
  <input type="number" name="age">
  <button type="submit">전송</button>
</form>
```

<br/>

✅ 2. @RequestParam 사용   

```java
@PostMapping("/submit")
public String handleSubmit(@RequestParam String name,
                           @RequestParam int age) {
    System.out.println("name = " + name);
    System.out.println("age = " + age);
    return "result";
}
```
- 🔹 각각의 폼 필드를 하나씩 변수로 받음
- 🔹 폼의 <input name="xxx"> 속성값과 변수 이름이 일치해야 함
- 🔹 단순 폼이나 몇 개의 필드만 처리할 때 적합

<br/>

✅ 3. @ModelAttribute 사용   

DTO 클래스 정의   

```java
public class MemberForm {
    private String name;
    private int age;

    // Getter/Setter 필수
}
```

컨트롤러   

```java
@PostMapping("/submit")
public String handleSubmit(@ModelAttribute MemberForm form) {
    System.out.println("name = " + form.getName());
    System.out.println("age = " + form.getAge());
    return "result";
}
```
- 🔹 폼 필드가 많을 때 매우 유리
- 🔹 이름이 일치하면 자동으로 필드에 바인딩됨
- 🔹 객체가 자동으로 Model에 등록되어 뷰에서 ${form.name} 형태로 사용 가능

<br/>

✅ 4. 정리 비교표   

|항목|@RequestParam|@ModelAttribute|
|---|---|---|
|사용 목적|개별 파라미터 바인딩|여러 파라미터 → 객체로 바인딩|
|입력 필드 개수|적을 때 적합|많을 때 적합|
|HTML name 매핑 방식|각 필드마다 직접 매핑|DTO 필드명과 name 일치하면 자동 매핑|
|객체로 받기 가능 여부|❌ 불편 (수동 바인딩 필요)|✅ 자동으로 객체 생성 및 바인딩|
|Model에 자동 등록 여부|❌ 기본은 안 됨|✅ 뷰에서 사용 가능|
|기본값, 필수 여부 설정 가능|✅ required, defaultValue 사용 가능|⚠️ 객체 전체 기준, 개별 속성 제어는 별도로 필요|

✅ 5. 언제 무엇을 써야 할까?    

|상황|추천 방식|
|---|---|---|
|파라미터 수가 적다|@RequestParam|
|여러 필드를 하나의 객체로 받고 싶다|@ModelAttribute|
|뷰에서 객체 전체를 보여줘야 한다|@ModelAttribute|
|JSON 요청이다|@RequestBody 사용 (※ 별도 주제)|

### 14. 엔티티 설계 시 주의점

#### 14-1. 엔티티에는 가급적 Setter를 사용하지 말자

엔티티에 Setter가 모두 열려 있으면 변경 포인트가 너무 많아서 유지보수가 어렵다.    
아래 ItemService.updateItem 코드와 같이 setter를 사용해 수정하면 어떤 곳에서 어떤 값을 수정하는지 추적하기 어렵다.   

```java
@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
public class ItemService {

    ...

    @Transactional
    public void updateItem(Long id, String name, int price, int stockQuantity) {
        Item findItem = findOne(id);
        
        findItem.setName(name);
        findItem.setPrice(price);
        findItem.setStockQuantity(stockQuantity);
    }
}

@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "dtype")
@Getter @Setter
public abstract class Item {

    @Id
    @GeneratedValue
    @Column(name = "item_id")
    private Long id;

    private String name;
    private int price;
    private int stockQuantity;

    ...
}
```

setter를 사용하는 대신 아래와 같이 속성 값을 변경하는 명시적인 메소드를 만들어 사용하는 것이 권장된다.

```java
@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
public class ItemService {

    ...

    @Transactional
    public void updateItem(Long id, String name, int price, int stockQuantity) {
        Item findItem = findOne(id);
        findItem.change(name, price, stockQuantity);
    }
}

@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "dtype")
@Getter
public abstract class Item {

    @Id
    @GeneratedValue
    @Column(name = "item_id")
    private Long id;

    private String name;
    private int price;
    private int stockQuantity;

    ...

    public void change(String name, int price, int stockQuantity) {
        this.name = name;
        this.price = price;
        this.stockQuantity = stockQuantity;
    }
}
```

#### 14-2. 모든 연관관계는 지연로딩으로 설정한다.

- 실무에서 발생하는 많은 장애들을 해결하기 때문에 매우 중요함
- 즉시로딩은 엔티티를 조회할 때 한 번에 연관된 엔티티를 모두 조회하는 것이다.
    - Member 엔티티를 조회할 때 연관된 Order, Delivery, OrderItem, Item 엔티티를 한 번에 조회한다.
- 즉시로딩(EAGER)은 예측이 어렵고, 어떤 SQL이 실행될지 추적하기 어렵다. 특히 JPQL을 실행할 때 N+1 문제가 자주 발생한다.
- 실무에서 모든 연관관계는 지연로딩(LAZY)으로 설정해야 한다.
- 연관된 엔티티를 함꼐 DB에서 조회해야 하면, fetch join 또는 엔티티 그래프 기능을 사용한다.
- @XToOne(OneToOne, ManyToOne) 관계는 기본이 즉시로딩이므로 직접 지연로딩으로 설정해야 한다.

#### 14-3. 컬렉션은 필드에서 초기화 한다.

컬렉션은 필드에서 바로 초기화 하는 것이 null 문제에서 안전하다.   

```java
@Entity
@Getter @Setter
public class Member {

    @Id @GeneratedValue
    @Column(name = "member_id")
    private Long id;

    private String name;

    @Embedded
    private Address address;

    @OneToMany(mappedBy =  "member")
    private List<Order> order = new ArrayList<>();
}
```

하이버네이트는 엔티티를 영속화 할 때 컬렉션을 감싸서 하이버네이트가 제공하는 내장 컬렉션으로 변경한다.   
만약 getOrders() 처럼 임의의 메서드에서 컬렉션을 잘못 생성하면 하이버네이트 내부 메커니즘에 문제가 발생할 수 있다.   
따라서 필들 ㅔ벨에서 생성하는 것이 가장 안전한고 코드도 간결하다.   

```java
Member member = new Member();
System.out.println(member.getOrders().getClass());
em.persist(member);
System.out.println(member.getOrders().getClass());

//출력 결과
// class java.util.ArrayList
// class org.hibernate.collection.internal.PersistentBag
```