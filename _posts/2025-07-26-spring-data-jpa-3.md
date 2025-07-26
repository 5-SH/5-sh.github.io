---
layout: post
title: 스프링 데이터 JPA 3
date: 2025-07-26 22:36:00 + 0900
categories: [spring]
tags: [spring, jpa]
---

### 강의 : [실전! 스프링 데이터 JPA](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-%EB%8D%B0%EC%9D%B4%ED%84%B0-JPA-%EC%8B%A4%EC%A0%84)

### 4. 확장 기능

#### 4-1. 사용자 정의 레포지토리 구현

스프링 데이터 JPA 레포지토리는 ```JpaRepository``` 인터페이스만 정의하고 구현체는 스프링이 자동 생성한다.   
```JpaRepository``` 인터페이스를 직접 구현하려면 구현할 기능이 너무 많다.    
주로 JPA 직접 사용(```EntityManager```), JDBC Template 사용, MyBatis 사용, 데이터베이스 커넥션 직접 사용, Querydsl 사용의 이유로 ```JpaRepository``` 인터페이스를 직접 구현한다.   

```java
// 사용자 정의 인터페이스
public interface MemberRepositoryCustom {
    List<Member> findMemberCustom();
}

// 사용자 정의 인터페이스 구현 클래스
@RequiredArgsConstructor
public class MemberRepositoryImpl implements MemberRepositoryCustom {

    private final EntityManager em;

    @Override
    public List<Member> findMemberCustom() {
        return em.createQuery("select m from Member m", Member.class).getResultList();
    }
}

// 사용자 정의 인터페이스 상속
public interface MemberRepository extends JpaRepository<Member, Long>, MemberRepositoryCustom {
    ...
}

// 사용자 정의 메서드 호출 코드
List<Member> result = memberRepository.findMemberCustom();
```

사용자 정의 레포지토리를 만들기 위해 ```JpaRepository``` 인터페이스를 모두 구현할 필요는 없다.   
사용자 정의할 인터페이스를 만들고 상속한 다음 구현하면 스프링 데이터 JPA가 자동으로 추가해 준다.   
스프링 데이터 JPA가 사용자가 정의한 인터페이스를 구현한 클래스를 찾을 수 있도록 레포지토리 인터페이스 이름 + Impl(```MemberRepositoryImpl```) 또는 사용자 정의 인터페이스 이름 + Impl(```MemberRepositoryCustomImpl```)을 사용한다.   

사용자 정의 레포지토리 대신 임의의 레포지토리를 만들어도 된다.    
별도의 레포지토리 클래스를 만들고 스프링 빈으로 등록해서 직접 사용할 수 있다.    
이 경우 빈으로 등록된 레포지토리는 스프링 데이터 JPA와 관계 없이 별도로 동작한다.   

#### 4-2. Auditing

운영의 편의를 위해 엔티티를 생성, 변경할 때 변경한 사람과 시간을 관리한다.   

##### 순수 JPA 사용

```java
@Getter
@MappedSuperclass
public class JpaBaseEntity {

    @Column(updatable = false)
    private LocalDateTime createdDate;
    private LocalDateTime updatedDate;

    @PrePersist
    public void prePersist() {
        LocalDateTime now = LocalDateTime.now();
        createdDate = now;
        updatedDate = now;
    }

    @PreUpdate
    public void preUpdate() {
        updatedDate = LocalDateTime.now();
    }
}

@Entity
@Getter @Setter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@ToString(of = { "id", "username", "age" })
public class Member extends JpaBaseEntity {
    ...
}
```

```@PrePersist``` 애노테이션은 ```EntityManger.persist()``` 메소드 호출 전 실행될 함수를 정의할 때 사용하고,   
```@PreUpdate``` 애노테이션은 ```EntityManager.flush()``` 메소드 호출 전 실행될 함수를 정의할 때 사용한다.   
```@MappedSuperclass```는 JPA에서 공통 필드를 갖는 추상 엔티티 클래스를 만들 때 사용한다.   
이 애노테이션이 붙은 클래스는 엔티티로 매핑되지 않지만, 이 클래스를 상속하는 자식 엔티티 클래스에 필드 매핑 정보가 전달된다.   

##### 스프링 데이터 JPA 사용

```java
@EnableJpaAuditing
@SpringBootApplication
public class DataJpaApplication {

	public static void main(String[] args) {
		SpringApplication.run(DataJpaApplication.class, args);
	}

	@Bean
	public AuditorAware<String> auditorProvider() {
		return () -> Optional.of(UUID.randomUUID().toString());
	}
}

@EntityListeners(AuditingEntityListener.class)
@Getter
@MappedSuperclass
public class BaseTimeEntity {

    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdDate;

    @LastModifiedDate
    private LocalDateTime lastModifiedDate;
}

@EntityListeners(AuditingEntityListener.class)
@Getter
@MappedSuperclass
public class BaseEntity extends BaseTimeEntity {

    @CreatedBy
    @Column(updatable = false)
    private String createdBy;

    @LastModifiedBy
    private String lastModifiedBy;
}
```

등록자, 수정자를 처리해 주는 ```AuditorAware``` 스프링 빈을 등록하고 ```DataJpaApplication```에 ```@EnableJpaAuditing```도 함께 등록한다.   
예제에선 등록자, 수정자로 UUID를 사용하지만 실무에서는 세션 정보나 스프링 시큐리티 로그인 정보에서 ID를 받는다.   

대부분의 엔티티는 등록시간, 수정시간이 필요하지만 등록자, 수정자는 필요 없을 수도 있다.   
그래서 다음과 같이 Base 타입을 분리하고 원하는 타입을 선택해서 상속한다.   

```@EntityListeners(AuditingEntityListener.class)``` 애노테이션을 사용하면 ```@PrePersist```, ```@PreUpdate``` 애노테이션을 사용해 저장하는 메소드를 구현하지 않고 ```createdBy```, ```lastModifiedBy```, ```createdDate```, ```lastModifiedDate```을 저장할 수 있다.   
```@EntityListeners(AuditingEntityListener.class)``` 애노테이션을 사용하기 위해선 ```@EnableJpaAuditing```을 등록해야 한다.   
그리고 ```AuditorAware``` 빈이 등록되어 있어야 ```@CreatedBy```, ```@LastModifiedBy``` 애노테이션을 사용할 수 있다.   

#### 4-3. Web 확장 - 도메인 클래스 컨버터

도메인 클래스 컨버터를 사용하면 엔티티를 조회하기 위해 컨트롤러에서 받은 ID 값으로 엔티티 조회를 하지 않아도 된다.    
컨트롤러 파라메터에 조회할 엔티티의 타입을 선언하면 엔티티의 아이디로 엔티티 객체를 찾아서 바인딩 해준다.   
도메인 클래스 컨버터도 내부적으로 레포지토리를 사용해서 엔티티를 찾는다.   

##### 도메인 클래스 컨버터 사용 전

```java
@RestController
@RequiredArgsConstructor
public class MemberController {
    private final MemberRepository memberRepository;
    
    @GetMapping("/members/{id}")
    public String findMember(@PathVariable("id") Long id) {
        Member member = memberRepository.findById(id).get();
        return member.getUsername();
    }
}
```

##### 도메인 클래스 컨버터 사용 후

```java
@RestController
@RequiredArgsConstructor
public class MemberController {
    private final MemberRepository memberRepository;
 
    @GetMapping("/members/{id}")
    public String findMember(@PathVariable("id") Member member) {
        return member.getUsername();
    }
}
```

도메인 클래스 컨버터로 받은 엔티티는 트랜잭션이 없는 범위에서 엔티티를 조회했기 때문에 dirty check가 되지 않기 때문에 단순 조회용으로만 사용해야 한다.   

#### 4-4. Web 확장 - 페이징과 정렬

스프링 데이터가 제공하는 페이징과 정렬 기능을 스프링 MVC에서 사용할 수 있다.   

```java
@GetMapping("/members")
public Page<Member> list(Pageable pageable) {
    Page<Member> page = memberRepository.findAll(pageable);
    return page;
}
```

```/members?page=0&size=3&sort=id,desc&sort=username,desc```로 요청을 보내면 ```page```, ```size```, ```sort``` 인자를 사용해 ```org.springframework.data.domain.PageRequest``` 객체를 생성해 컨트롤러에 전달한다.   
기본 페이지 사이즈는 20, 최대 페이지 사이즈는 2000으로 기본 설정이 되어 있다.    
다음과 같이 ```@PageableDefault``` 애노테이션을 사용해 변경할 수 있다.   

```java
@RequestMapping(value = "/members_page", method = RequestMethod.GET)
public String list(@PageableDefault(size = 12, sort = "username", direction = Sort.Direction.DESC) Pageable pageable) {
    ...
}
```

페이징 정보가 둘 이상이면 ```@Qualifier``` 애노테이션을 사용해 접두사로 구분할 수 있다.    
```/members?member_page=0&order_page=1```으로 요청을 하면 아래 코드와 같이 받을 수 있다.   

```java
public String list(
    @Qualifier("member") Pageable memberPageable,
    @Qualifier("order") Pageable orderPageable, ...
```

##### Page 내용을 DTO로 변환하기   

엔티티를 API로 노출하면 다양한 문제가 발생하기 때문에 DTO로 변환해서 반환해야 한다.   
Page는 엔티티를 DTO로 변환할 수 있도록 ```map()``` 메소드를 제공한다.   

```java
@Data
public class MemberDto {
    private Long id;
    private String username;
    
    public MemberDto(Member m) {
        this.id = m.getId();
        this.username = m.getUsername();
    }
    ...
}

@GetMapping("/members")
public Page<MemberDto> list(Pageable pageable) {
    return memberRepository.findAll(pageable).map(MemberDto::new);
}
```
