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
