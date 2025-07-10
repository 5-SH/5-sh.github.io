---
layout: post
title: 스프링 부트와 JPA 활용3
date: 2025-06-08 20:31:00 + 0900
categories: [spring]
tags: [spring, jpa]
---

### 강의 : [실전! 스프링 부트와 JPA 활용2 - API 개발과 성능 최적화](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81%EB%B6%80%ED%8A%B8-JPA-API%EA%B0%9C%EB%B0%9C-%EC%84%B1%EB%8A%A5%EC%B5%9C%EC%A0%81%ED%99%94/dashboard)

### 1. Rest API 요청을 받기 위한 별도의 DTO를 만든다.

**실무에서는 API 파라미터로 엔티티를 사용하거나 외부에 노출하면 안된다.**   

<br/>

엔티티에 @NotEmpty 같은 API 검증을 위한 로직이 들어가면, 엔티티에 프레젠테이션 계층을 위한 로직이 추가된다.   
실무에서 엔티티를 위한 API가 다양하게 만들어 지는데, 한 엔티티에 각 API를 위한 모든 요청 요구사항을 담기는 어렵다.   

```java
@RestController
@RequiredArgsConstructor
public class MemberApiController {

    private final MemberService memberService;

    @PostMapping("/api/v1/members")
    public CreateMemberResponse saveMemberV1(@RequestBody @Valid Member member) {
        Long id = memberService.join(member);
        return new CreateMemberResponse(id);
    }

    ...
}

@Entity
@Getter @Setter
public class Member {

    @Id @GeneratedValue
    @Column(name = "member_id")
    private Long id;

    @NotEmpty
    private String name;

    @Embedded
    private Address address;

    @OneToMany(mappedBy =  "member")
    private List<Order> order = new ArrayList<>();
}
```

엔티티가 변경되면 API 스펙이 변하기 때문에 엔티티를 API 파라미터로 받지 않고, API 요청 스펙에 맞춰 별도의 DTO를 파라미터로 받는다.   
DTO를 사용해 엔티티와 프레젠테이션 계층 간의 의존성을 끊고 엔티티와 API 스펙을 명확하게 분리한다.   

```java
@RestController
@RequiredArgsConstructor
public class MemberApiController {

    private final MemberService memberService;

    ...

    @PostMapping("/api/v2/members")
    public CreateMemberResponse saveMemberV2(@RequestBody @Valid CreateMemberRequest request) {
        Member member = new Member();
        member.setName(request.getName());

        Long id = memberService.join(member);
        return new CreateMemberResponse(id);
    }

    @Data
    static class CreateMemberRequest {
        private String name;
    }
    ...
}
```

### 2. Rest API 응답을 위한 별도의 DTO를 만든다.

**실무에서는 API 응답으로 엔티티를 사용하거나 외부에 노출하면 안된다.**   

<br/>

엔티티의 모든 값이 노출되고 엔티티가 변경되면 API 스펙이 변한다.    
응답 스펙을 맞추기 위해 @JsonIgnore 등의 로직이 엔티티에 추가된다.    
실무에서는 하나의 엔티티에 대해 API가 용도에 따라 다양하게 만들어지는데, 한 엔티티에 각각의 API를 위한 프레젠테이션 응답 로직을 담기는 어렵다.   

```java
@RestController
@RequiredArgsConstructor
public class MemberApiController {

    private final MemberService memberService;

    @GetMapping("/api/v1/members")
    public List<Member> membersV1() {
        return memberService.findMembers();
    }
    ...
}

@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
public class MemberService {

    private final MemberRepository memberRepository;

    ...

    // 회원 전체 조회
    public List<Member> findMembers() {
        return memberRepository.findAll();
    }

    ...
}

@Repository
@RequiredArgsConstructor
public class MemberRepository {

    private final EntityManager em;

    ...

    public List<Member> findAll() {
        return em.createQuery("select m from Member m", Member.class).getResultList();
    }

    ...
}
```

API 응답 스펙에 맞춰 별도의 DTO를 반환한다.
그리고 컬렉션을 직접 반환하면 API 스펙을 변경하기 어렵기 때문에 별도의 Result 클래스를 생성해 응답한다.   
Result 클래스를 활용하면 ```[{ "name": "userA" }, { "name": "userB" }]``` 대신 ```{ "data": [{ "name": "userA" }, { "name": "userB" }] }```와 같은 형식으로 응답할 수 있다.   
Result 클래스로 컬렉션을 감싸면 향후 필요한 필드를 추가할 수 있다.   

```java
@RestController
@RequiredArgsConstructor
public class MemberApiController {

    private final MemberService memberService;

    @GetMapping("/api/v2/members")
    public Result memberV2() {
        List<Member> members = memberService.findMembers();
        List<MemberDto> list = members.stream().map(m -> new MemberDto(m.getName())).toList();

        return new Result(list);
    }

    @Data
    @AllArgsConstructor
    static class Result<T> {
        private T data;
    }

    @Data
    @AllArgsConstructor
    static class MemberDto {
        private String name;
    }

    ...
}
```

### 3. 지연 로딩과 조회 성능 최적화

주문 + 배송정보 + 회원을 조회하는 API를 만들며 지연 로딩 때문에 발생하는 성능 문제를 단계적으로 해결한다.   

#### 3-1. 엔티티를 직접 노출

아래 코드와 같이 주문 정보를 조회할 때 엔티티를 API 응답으로 사용하면 앞서 설명한 문제 외에 양방향 관계 문제가 발생한다.   
```Member```와 ```Delivery```는 지연 로딩으로 설정되어 있어 Order 목록의 반복문 내 ```order.getMember().getName();```와 ```order.getDelivery().getAddress();``` 코드로 Lazy 강제 초기화를 한다.

```java
@RestController
@AllArgsConstructor
public class OrderSimpleApiController {

    private final OrderRepository orderRepository;
    private final OrderSimpleQueryRepository orderSimpleQueryRepository;

    @GetMapping("/api/v1/simple-orders")
    public List<Order> ordersV1() {
        List<Order> all = orderRepository.findAllByString(new OrderSearch());
        for (Order order : all) {
            order.getMember().getName();
            order.getDelivery().getAddress();
        }
        return all;
    }
    ...
}

@Repository
@RequiredArgsConstructor
public class OrderRepository {

    private final EntityManager em;

    ...

    public List<Order> findAllByString(OrderSearch orderSearch) {

        String jpql = "select o from Order o join o.member m";
        boolean isFirstCondition = true;

        //주문 상태 검색
        if (orderSearch.getOrderStatus() != null) {
            if (isFirstCondition) {
                jpql += " where";
                isFirstCondition = false;
            } else {
                jpql += " and";
            }
            jpql += " o.status = :status";
        }

        //회원 이름 검색
        if (StringUtils.hasText(orderSearch.getMemberName())) {
            if (isFirstCondition) {
                jpql += " where";
                isFirstCondition = false;
            } else {
                jpql += " and";
            }
            jpql += " m.name like :name";
        }

        TypedQuery<Order> query = em.createQuery(jpql, Order.class)
                .setMaxResults(1000);

        if (orderSearch.getOrderStatus() != null) {
            query = query.setParameter("status", orderSearch.getOrderStatus());
        }
        if (StringUtils.hasText(orderSearch.getMemberName())) {
            query = query.setParameter("name", orderSearch.getMemberName());
        }

        return query.getResultList();
    }
    ...
}
```

```/api/v1/simple-orders``` API를 호출하면 아래와 같이 계속해서 회원 정보를 조회한다.   

```
[
    {
        "id": 1,
        "member": {
            "id": 1,
            "name": "userA",
            "address": {
                "city": "서울",
                "street": "1",
                "zipcode": "1111"
            },
            "orders": [
                {
                    "id": 1,
                    "member": {
                        "id": 1,
                        "name": "userA",
                        "address": {
                            "city": "서울",
                            "street": "1",
                            "zipcode": "1111"
                        },
                        "orders": [
                            {
                                "id": 1,
                                "member": {
                                    "id": 1,
                                    "name": "userA",
                                    "address": {
                                        "city": "서울",
                                        "street": "1",
                                        "zipcode": "1111"
                                    },
                                    "orders": [
                                        {
                                            "id": 1,
                                            "member": {
                                                "id": 1,
                                                "name": "userA",
                                                "address": {
                                                    "city": "서울",
                                                    "street": "1",
                                                    "zipcode": "1111"
                                                },
    ...                                                                                                         
```

```Order``` 엔티티와 ```Member``` 엔티티는 다대일 관계이고 양방향으로 참조를 하고 있다.   
그리고 지연 로딩으로 설정되어 있어 계속해서 ```Member```와 ```Order```를 조회하게 된다.   

```java
@Entity
@Table(name = "orders")
@Getter @Setter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Order {

    @Id @GeneratedValue
    @Column(name = "order_id")
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "member_id")
    private Member member;

    ...
}

@Entity
@Getter @Setter
public class Member {

    @Id @GeneratedValue
    @Column(name = "member_id")
    private Long id;

    private String name;

    @Embedded
    private Address address;

    @OneToMany(mappedBy = "member")
    private List<Order> orders = new ArrayList<>();
}
```

이 문제를 해결하려면 양방향으로 참조하는 속성에 ```@JsonIgnore```애노테이션을 추가해 해결한다.   
```@JsonIgnore```애노테이션은 JSON 응답을 만들 때 이 애노테이션이 등록된 속성을 제외하도록 한다.   
jackson 라이브러리에서 JSON 응답 형식을 만들 때 클래스의 속성마다 get 메소드를 호출한다.   
그러나 ```Member``` 엔티티의 ```orders``` 속성은 무시되어 실제 값을 DB에서 로드하지 못하고 지연 로딩을 위한 프록시로 남아 있게 된다.   

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

    @JsonIgnore
    @OneToMany(mappedBy = "member")
    private List<Order> orders = new ArrayList<>();
}
```

```order``` → ```member와``` ```order``` → ```delivery```는 지연로딩 이므로 실제 엔티티 대신에 프록시가 존재한다.   
JSON 응답 형식을 만들 때 jackson 라이브러리는 지연 로딩을 위한 프록시 객체를 어떻게 json으로 생성해야 하는지 몰라 예외가 발생한다.   
아래 에러 로그에서 ```bytebuddy```가 지연 로딩을 위한 프록시 객체이다.   

```
com.fasterxml.jackson.databind.exc.InvalidDefinitionException: No serializer found for class org.hibernate.proxy.pojo.bytebuddy.ByteBuddyInterceptor and no properties discovered to create BeanSerializer (to avoid exception, disable SerializationFeature.FAIL_ON_EMPTY_BEANS) (through reference chain: java.util.ArrayList[0]->jpabook.jpashop.domain.Order["member"]->jpabook.jpashop.domain.Member$HibernateProxy$kYmMvlar["hibernateLazyInitializer"])
	at com.fasterxml.jackson.databind.exc.InvalidDefinitionException.from(InvalidDefinitionException.java:77) ~[jackson-databind-2.18.3.jar:2.18.3]
	at com.fasterxml.jackson.databind.SerializerProvider.reportBadDefinition(SerializerProvider.java:1340) ~[jackson-databind-2.18.3.jar:2.18.3]
	at com.fasterxml.jackson.databind.DatabindContext.reportBadDefinition(DatabindContext.java:414) ~[jackson-databind-2.18.3.jar:2.18.3]
	at com.fasterxml.jackson.databind.ser.impl.UnknownSerializer.failForEmpty(UnknownSerializer.java:53) ~[jackson-databind-2.18.3.jar:2.18.3]
	at com.fasterxml.jackson.databind.ser.impl.UnknownSerializer.serialize(UnknownSerializer.java:30) ~[jackson-databind-2.18.3.jar:2.18.3]
	at com.fasterxml.jackson.databind.ser.BeanPropertyWriter.serializeAsField(BeanPropertyWriter.java:732) ~[jackson-databind-2.18.3.jar:2.18.3]
	at com.fasterxml.jackson.databind.ser.std.BeanSerializerBase.serializeFields(BeanSerializerBase.java:770) ~[jackson-databind-2.18.3.jar:2.18.3]
	at com.fasterxml.jackson.databind.ser.BeanSerializer.serialize(BeanSerializer.java:184) ~[jackson-databind-2.18.3.jar:2.18.3]
	at com.fasterxml.jackson.databind.ser.BeanPropertyWriter.serializeAsField(BeanPropertyWriter.java:732) ~[jackson-databind-2.18.3.jar:2.18.3]
	at com.fasterxml.jackson.databind.ser.std.BeanSerializerBase.serializeFields(BeanSerializerBase.java:770) ~[jackson-databind-2.18.3.jar:2.18.3]
	at com.fasterxml.jackson.databind.ser.BeanSerializer.serialize(BeanSerializer.java:184) ~[jackson-databind-2.18.3.jar:2.18.3]
    ...
```

아래와 같이 ```hibernate5JakartaModule```빈을 등록하면 jackson 라이브러리에서 프록시 객체는 무시한다.

```java
@SpringBootApplication
public class JpashopApplication {

	public static void main(String[] args) {
		SpringApplication.run(JpashopApplication.class, args);
	}

	@Bean
	Hibernate5JakartaModule hibernate5Module() {
		Hibernate5JakartaModule hibernate5JakartaModule = new Hibernate5JakartaModule();
//		hibernate5JakartaModule.configure(Hibernate5JakartaModule.Feature.FORCE_LAZY_LOADING, true);
		return hibernate5JakartaModule;
	}
}
```

```@JsonIgnore```와 ```hibernate5JakartaModule```빈 설정으로 문제를 해결해 아래와 같이 정상적으로 응답을 받을 수 있다.   

```
[
    {
        "id": 1,
        "member": {
            "id": 1,
            "name": "userA",
            "address": {
                "city": "서울",
                "street": "1",
                "zipcode": "1111"
            }
        },
        "orderItems": null,
        "delivery": {
            "id": 1,
            "address": {
                "city": "서울",
                "street": "1",
                "zipcode": "1111"
            },
            "status": null
        },
        "orderDate": "2025-07-08T16:12:21.074519",
        "status": "ORDER",
        "totalPrice": 50000
    },
    {
        "id": 2,
        "member": {
            "id": 2,
            "name": "userB",
            "address": {
                "city": "진주",
                "street": "2",
                "zipcode": "2222"
            }
        },
        "orderItems": null,
        "delivery": {
            "id": 2,
            "address": {
                "city": "진주",
                "street": "2",
                "zipcode": "2222"
            },
            "status": null
        },
        "orderDate": "2025-07-08T16:12:21.135029",
        "status": "ORDER",
        "totalPrice": 220000
    }
]
```

**※ 엔티티를 직접 노출할 때 양방향 연관관계가 걸린 경우 반드시 한 곳에 ```@JsonIgnore``` 처리를 해야한다.**   
**※ 지연 로딩(LAZY)을 피하기 위해 즉시 로딩(EAGER)으로 설정하면, 연관 관계가 필요 없는 경우에도 데이터를 항상 조회해서 성능 문제가 발생할 수 있고 성능 튜닝을 하기 어렵기 때문에 설정하면 안된다.**
**※ 항상 지연 로딩을 기본으로 하고 성능 최적화가 필요한 경우에는 fetch join을 사용한다.**

#### 3-2. 엔티티를 DTO로 변환

엔티티를 조회해서 DTO로 응답하도록 수정한다.   
실행되는 쿼리의 수는 ```ordersV1()```와 동일하다.    
쿼리는 ```order```조회 1번 + ```order → member``` 지연로딩 N번 + ```order → delivery``` 지연로딩 N번 총 1 + N + N번 실행된다.   
예를 들어 ```order```의 결과가 4개면 1 + 4 + 4번 실행된다.    
조회 하려는 값이 영속성 컨텍스트에 존재하는 경우 실행되는 쿼리의 수는 줄어들 수 있다.   

```java
@RestController
@AllArgsConstructor
public class OrderSimpleApiController {

    private final OrderRepository orderRepository;
    private final OrderSimpleQueryRepository orderSimpleQueryRepository;

    ...

    @GetMapping("/api/v2/simple-orders")
    public List<SimpleOrderDto> ordersV2() {
        return orderRepository.findAllByString(new OrderSearch()).stream()
                .map(SimpleOrderDto::new)
                .toList();
    }

    ...

    @Data
    static class SimpleOrderDto {
        private Long orderId;
        private String name;
        private LocalDateTime orderDate;
        private OrderStatus orderStatus;
        private Address address;

        public SimpleOrderDto(Order order) {
            orderId = order.getId();
            name = order.getMember().getName();
            orderDate = order.getOrderDate();
            orderStatus = order.getStatus();
            address = order.getDelivery().getAddress();
        }
    }
}
```

#### 3-3. 엔티티를 DTO로 변환 - 페치 조인 최적화

fetch join을 사용해 쿼리 1번으로 API 응답에 필요한 ```Order``` 엔티티의 정보를 가져온다.   
fetch join으로 쿼리 1번에 ```order → member```, ```order → delivery```를 조회한다.   

```java
@RestController
@AllArgsConstructor
public class OrderSimpleApiController {

    private final OrderRepository orderRepository;
    private final OrderSimpleQueryRepository orderSimpleQueryRepository;

    ...

    @GetMapping("/api/v3/simple-orders")
    public List<SimpleOrderDto> orderV3() {
        return orderRepository.findAllWithMemberDelivery().stream()
                .map(SimpleOrderDto::new)
                .toList();
    }

    ...

    @Data
    static class SimpleOrderDto {
        private Long orderId;
        private String name;
        private LocalDateTime orderDate;
        private OrderStatus orderStatus;
        private Address address;

        public SimpleOrderDto(Order order) {
            orderId = order.getId();
            name = order.getMember().getName();
            orderDate = order.getOrderDate();
            orderStatus = order.getStatus();
            address = order.getDelivery().getAddress();
        }
    }
}

@Repository
@RequiredArgsConstructor
public class OrderRepository {

    private final EntityManager em;

    ...

    public List<Order> findAllWithMemberDelivery() {
        return em.createQuery(
                "select o from Order o" +
                        " join fetch o.member m" +
                        " join fetch o.delivery d", Order.class
        ).getResultList();
    }
    
    ...
}
```

```/api/v3/simple-orders``` API를 호출하면 아래 쿼리 1개가 실행되고 응답을 받는다.   

```sql
select
    o1_0.order_id,
    d1_0.delivery_id,
    d1_0.city,
    d1_0.street,
    d1_0.zipcode,
    d1_0.status,
    m1_0.member_id,
    m1_0.city,
    m1_0.street,
    m1_0.zipcode,
    m1_0.name,
    o1_0.order_date,
    o1_0.status 
from
    orders o1_0 
join
    member m1_0 
        on m1_0.member_id=o1_0.member_id 
join
    delivery d1_0 
        on d1_0.delivery_id=o1_0.delivery_id
```
#### 3-4. JPA에서 DTO로 바로 조회

```Order```엔티티를 DTO로 변환해 응답하는 ```/api/v3/simple-orders``` API와 다르게 JPA DTO로 바로 조회한다.   
```new``` 명령어를 사용해서 JPQL 결과를 바로 DTO로 변환한다.   
필요한 데이터만 직접 선택해 조회하므로 네트웍 용량이 최적화 된다는 장점이 있지만 효과는 미비하다.   
레포지토리 재사용성과 API 스펙에 맞춘 코드가 레포지토리에 들어가는 단점이 있다.   

```java
@RestController
@AllArgsConstructor
public class OrderSimpleApiController {
    
    private final OrderRepository orderRepository;
    private final OrderSimpleQueryRepository orderSimpleQueryRepository;

    ...

    @GetMapping("/api/v4/simple-orders")
    public List<OrderSimpleQueryDto> orderV4() {
        return orderSimpleQueryRepository.findOrderDtos();
    }

    ...
}

@Repository
@RequiredArgsConstructor
public class OrderSimpleQueryRepository {

    private final EntityManager em;

    public List<OrderSimpleQueryDto> findOrderDtos() {
        return em.createQuery(
                "select new jpabook.jpashop.repository.order.simplequery.OrderSimpleQueryDto(o.id, m.name, o.orderDate, o.status, d.address)" +
                        " from Order o" +
                        " join o.member m" +
                        " join o.delivery d", OrderSimpleQueryDto.class
        ).getResultList();
    }
}

@Data
public class OrderSimpleQueryDto {
    private Long orderId;
    private String name;
    private LocalDateTime orderDate;
    private OrderStatus orderStatus;
    private Address address;

    public OrderSimpleQueryDto(Long orderId, String name, LocalDateTime orderDate, OrderStatus orderStatus, Address address) {
        this.orderId = orderId;
        this.name = name;
        this.orderDate = orderDate;
        this.orderStatus = orderStatus;
        this.address = address;
    }
}
```

#### 3-5. 정리

엔티티를 DTO로 변환하거나 DTO로 바로 조회하는 방법은 각각 장단점이 있다.    
상황에 따라서 더 나은 방법을 선택하면 되는데 권장하는 선택 순서는 다음과 같다.   

1. 우선 엔티티를 DTO로 변환하는 방법을 선택한다.
2. 필요하면 페치 조인으로 성능을 최적화 한다. 대부분의 성능 이슈가 해결된다. 
3. 그래도 안되면 DTO로 직접 조회하는 방법을 사용한다.
4. 최후의 방법은 JPA가 제공하는 네이티브 SQL이나 스프링 JDBC Template를 사용해 SQL를 직접 사용한다.
