---
layout: post
title: 스프링 부트와 JPA 활용3
date: 2025-07-10 19:10:00 + 0900
categories: [spring]
tags: [spring, jpa]
---

### 강의 : [실전! 스프링 부트와 JPA 활용2 - API 개발과 성능 최적화](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81%EB%B6%80%ED%8A%B8-JPA-API%EA%B0%9C%EB%B0%9C-%EC%84%B1%EB%8A%A5%EC%B5%9C%EC%A0%81%ED%99%94/dashboard)

### 4. 컬렉션 조회 최적화

주문내역에서 추가로 주문한 상품 정보를 추가로 조회한다.   
```Order```에서 컬렉션으로 참조하는 ```OrderItem```과 ```Item```이 필요하다.   

이전 문서의 "3. 지연 로딩과 조회 성능 최적화"에선 OneToOne, ManyToOne 관계만 있었다.   
이번에는 컬렉션인 OneToMany를 조회하고 최적화 하는 방법을 알아본다.   

### 4-1. 엔티티 직접 노출

```orderItem```, ```item```을 Lazy 강제 초기화 한다.   
```order → member```, ```order → delivery``` 관계와 동일하게 순환 참조로 인한 문제가 발생하지 않도록 한 곳에 ```@JsonIgnore```를 추가한다.   

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

    @BatchSize(size = 1000)
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    private List<OrderItem> orderItems = new ArrayList<>();

    ...
}

@Entity
@Getter @Setter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class OrderItem {

    @Id @GeneratedValue
    @Column(name = "order_item_id")
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "item_id")
    private Item item;

    @JsonIgnore
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id")
    private Order order;

    ...
}

@RestController
@RequiredArgsConstructor
public class OrderApiController {

    private final OrderRepository orderRepository;

    @GetMapping("/api/v1/orders")
    public List<Order> ordersV1() {
        List<Order> all = orderRepository.findAllByString(new OrderSearch());
        for (Order order : all) {
            order.getMember().getName();
            order.getDelivery().getAddress();

            List<OrderItem> orderItems = order.getOrderItems();
            orderItems.stream().forEach(o -> o.getItem().getName());
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

```/api/v1/orders``` API 요청을 하면 아래와 같이 ```orderItem```과 ```item```이 포함된 응답을 받을 수 있다.   
앞서 설명한 대로 엔티티를 직접 노출하기 때문에 좋은 방법은 아니다.   

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
        "orderItems": [
            {
                "id": 1,
                "item": {
                    "id": 1,
                    "name": "JPA1 BOOK",
                    "price": 10000,
                    "stockQuantity": 99,
                    "categories": null,
                    "author": null,
                    "isbn": null
                },
                "orderPrice": 10000,
                "count": 1,
                "totalPrice": 10000
            },
            {
                "id": 2,
                "item": {
                    "id": 2,
                    "name": "JPA2 BOOK",
                    "price": 20000,
                    "stockQuantity": 98,
                    "categories": null,
                    "author": null,
                    "isbn": null
                },
                "orderPrice": 20000,
                "count": 2,
                "totalPrice": 40000
            }
        ],
        "delivery": {
            "id": 1,
            "address": {
                "city": "서울",
                "street": "1",
                "zipcode": "1111"
            },
            "status": null
        },
        "orderDate": "2025-07-10T00:41:18.076796",
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
        "orderItems": [
            {
                "id": 3,
                "item": {
                    "id": 3,
                    "name": "SPRING1 BOOK",
                    "price": 20000,
                    "stockQuantity": 97,
                    "categories": null,
                    "author": null,
                    "isbn": null
                },
                "orderPrice": 20000,
                "count": 3,
                "totalPrice": 60000
            },
            {
                "id": 4,
                "item": {
                    "id": 4,
                    "name": "SPRING2 BOOK",
                    "price": 40000,
                    "stockQuantity": 96,
                    "categories": null,
                    "author": null,
                    "isbn": null
                },
                "orderPrice": 40000,
                "count": 4,
                "totalPrice": 160000
            }
        ],
        "delivery": {
            "id": 2,
            "address": {
                "city": "진주",
                "street": "2",
                "zipcode": "2222"
            },
            "status": null
        },
        "orderDate": "2025-07-10T00:41:18.257801",
        "status": "ORDER",
        "totalPrice": 220000
    }
]
```

### 4-2. 엔티티를 DTO로 변환

```SimpleOrderDto```와 같이 DTO로 API 응답을 하기 위해 ```OrderDto```를 만든다.   
```OrderDto```는 ```orderItem```을 응답하기 위해 속성이 추가했다.   
```orderItems```에서 연결해 조회하는 ```orderItem```도 프레젠테이션 계층과 엔티티의 의존 관계를 끊기 위해 ```OrderItemDto```를 만들어 응답한다.   

```java
@RestController
@RequiredArgsConstructor
public class OrderApiController {

    private final OrderRepository orderRepository;

    ...

    @GetMapping("/api/v2/orders")
    public List<OrderDto> ordersV2() {
        return orderRepository.findAllByString(new OrderSearch())
                .stream()
                .map(OrderDto::new)
                .toList();
    }

    ...
    @Data
    static class OrderDto {

        private Long orderId;
        private String name;
        private LocalDateTime orderDate;
        private OrderStatus orderStatus;
        private Address address;
        private List<OrderItemDto> orderItems;

        public OrderDto(Order order) {
            orderId = order.getId();
            name = order.getMember().getName();
            orderDate = order.getOrderDate();
            orderStatus = order.getStatus();
            address = order.getDelivery().getAddress();
            orderItems = order.getOrderItems().stream().map(OrderItemDto::new).toList();
        }
    }

    @Data
    static class OrderItemDto {

        private String itemName;
        private int orderPrice;
        private int count;

        public OrderItemDto(OrderItem orderItem) {
            itemName = orderItem.getItem().getName();
            orderPrice = orderItem.getOrderPrice();
            count = orderItem.getCount();
        }
    }
}
```

```/api/v2/orders``` API를 요청하면 아래와 같이 응답을 받는다.   
엔티티를 직접 응답받지 않고 DTO로 변환해 받기 때문에 엔티티의 수정으로 인해 API 명세가 바뀌지 않고    
API 명세에 대한 조건이 엔티티 코드에 추가되지 않는다.   

<br/>

그러나 지연로딩으로 너무 많은 쿼리가 실행된다.   
쿼리는 ```order```조회 1번 + ```member``` 지연로딩 N번 + ```delivery``` 지연로딩 N번 + ```orderItem``` 지연로딩 N번 + ```item``` 지연로딩 M 번 총 1 + N + N + N + M번 실행된다.   
예를 들어 ```order```의 결과가 2개이고 ```orderItem```에 등록된 ```item```이 2개 이면 1 + 2 + 2 + 2 + 2*2번 실행된다.    

```
[
    {
        "orderId": 1,
        "name": "userA",
        "orderDate": "2025-07-10T00:41:18.076796",
        "orderStatus": "ORDER",
        "address": {
            "city": "서울",
            "street": "1",
            "zipcode": "1111"
        },
        "orderItems": [
            {
                "itemName": "JPA1 BOOK",
                "orderPrice": 10000,
                "count": 1
            },
            {
                "itemName": "JPA2 BOOK",
                "orderPrice": 20000,
                "count": 2
            }
        ]
    },
    {
        "orderId": 2,
        "name": "userB",
        "orderDate": "2025-07-10T00:41:18.257801",
        "orderStatus": "ORDER",
        "address": {
            "city": "진주",
            "street": "2",
            "zipcode": "2222"
        },
        "orderItems": [
            {
                "itemName": "SPRING1 BOOK",
                "orderPrice": 20000,
                "count": 3
            },
            {
                "itemName": "SPRING2 BOOK",
                "orderPrice": 40000,
                "count": 4
            }
        ]
    }
]
```

### 4-3. 엔티티를 DTO로 변환 - 페치 조인 최적화

지연로딩 때문에 많은 쿼리를 보내는 문제를 해결하기 위해 페치 조인을 사용한다.   

```java
@RestController
@RequiredArgsConstructor
public class OrderApiController {

    private final OrderRepository orderRepository;

    ...

    @GetMapping("/api/v3/orders")
    public List<OrderDto> ordersV3() {
        return orderRepository.findAllWithItem()
                .stream()
                .map(OrderDto::new)
                .toList();
    }
    ...   
}

@Repository
@RequiredArgsConstructor
public class OrderRepository {

    private final EntityManager em;

    ...

    public List<Order> findAllWithItem() {
        return em.createQuery(
                "select distinct o from Order o" +
                        " join fetch o.member m" +
                        " join fetch o.delivery d" +
                        " join fetch o.orderItems oi" +
                        " join fetch oi.item i", Order.class)
                .getResultList();
    }
    ...
}
```

페치 조인을 사용하면 아래 쿼리 하나로 API 응답을 할 수 있다.   

```
select
    distinct o1_0.order_id,
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
    oi1_0.order_id,
    oi1_0.order_item_id,
    oi1_0.count,
    i1_0.item_id,
    i1_0.dtype,
    i1_0.name,
    i1_0.price,
    i1_0.stock_quantity,
    i1_0.artist,
    i1_0.etc,
    i1_0.author,
    i1_0.isbn,
    i1_0.actor,
    i1_0.director,
    oi1_0.order_price,
    o1_0.status 
from
    orders o1_0 
join
    member m1_0 
        on m1_0.member_id=o1_0.member_id 
join
    delivery d1_0 
        on d1_0.delivery_id=o1_0.delivery_id 
join
    order_item oi1_0 
        on o1_0.order_id=oi1_0.order_id 
join
    item i1_0 
        on i1_0.item_id=oi1_0.item_id
```

이전 문서에서 설명한 다대일, 일대일 페치 조인 최적화와 다르게 JPQL 쿼리에 ```distinct```가 추가되어 있다.   
일대다 관계에서 페치 조인을 사용하면 아래와 같이 row가 증가하고 중복된 데이터가 생긴다.    

![Image](https://i.imgur.com/UN7Jfb3.png)

JPA의 distinct는 SQL에 distinct를 추가하고 같은 엔티티가 조회되면 애플리케이션에서 중복을 걸러준다.   
실행하는 쿼리에 ```distinct``` 명령어를 추가 하더라도 일대다 조인 이므로 row가 증가하고 중복된 데이터가 생긴다.   
중복을 걸러주는 실질적인 동작은 애플리케이션에서 ```Order``` 엔티티의 키 값을 사용해 수행한다.   

<br/>

그리고 일대다 관계에서 컬렉션 페치 조인을 사용하면 페이징을 사용할 수 없다.   
페치 조인을 하면 row가 증가하고 중복된 데이터가 생겨 ```Order```기준으로 페이징 처리를 할 수 없기 때문이다.   
컬렉션 페치 조인을 하면서 페이징 요청을 하면 하이버네이트는 경고 로그를 남기며   
모든 데이터를 DB에서 읽어오고 메모리에서 페이징하기 때문에 애플리케이션의 메모리가 부족해 장애가 발생할 수 있다.   
페이징 테스트를 위해 아래 코드와 같이 ```findAllWithItem()``` 함수에 ```offset```과 ```limit```를 추가해 ```/api/v3/orders``` API를 요청한다.   

```java
@Repository
@RequiredArgsConstructor
public class OrderRepository {

    private final EntityManager em;

    ...

    public List<Order> findAllWithItem() {
        return em.createQuery(
                "select distinct o from Order o" +
                        " join fetch o.member m" +
                        " join fetch o.delivery d" +
                        " join fetch o.orderItems oi" +
                        " join fetch oi.item i", Order.class)
                .setFirstResult(1)
                .setMaxResults(100)
                .getResultList();
    }
    ...
}
```

API를 실행하면 아래와 같이 ```2025-07-10T10:39:37.126+09:00  WARN 22192 --- [nio-8080-exec-1] org.hibernate.orm.query                  : HHH90003004: firstResult/maxResults specified with collection fetch; applying in memory``` 경고 로그를 남긴다.   
그리고 실행한 쿼리에 조회나 페이징 조건 없어 모든 데이터를 읽어오는 것을 확인할 수 있다.   

```
2025-07-10T10:39:32.796+09:00  INFO 22192 --- [nio-8080-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2025-07-10T10:39:32.797+09:00  INFO 22192 --- [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2025-07-10T10:39:32.815+09:00  INFO 22192 --- [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 17 ms
2025-07-10T10:39:37.126+09:00  WARN 22192 --- [nio-8080-exec-1] org.hibernate.orm.query                  : HHH90003004: firstResult/maxResults specified with collection fetch; applying in memory
2025-07-10T10:39:37.393+09:00 DEBUG 22192 --- [nio-8080-exec-1] org.hibernate.SQL                        : 
    select
        distinct o1_0.order_id,
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
        oi1_0.order_id,
        oi1_0.order_item_id,
        oi1_0.count,
        i1_0.item_id,
        i1_0.dtype,
        i1_0.name,
        i1_0.price,
        i1_0.stock_quantity,
        i1_0.artist,
        i1_0.etc,
        i1_0.author,
        i1_0.isbn,
        i1_0.actor,
        i1_0.director,
        oi1_0.order_price,
        o1_0.status 
    from
        orders o1_0 
    join
        member m1_0 
            on m1_0.member_id=o1_0.member_id 
    join
        delivery d1_0 
            on d1_0.delivery_id=o1_0.delivery_id 
    join
        order_item oi1_0 
            on o1_0.order_id=oi1_0.order_id 
    join
        item i1_0 
            on i1_0.item_id=oi1_0.item_id
2025-07-10T10:39:37.443+09:00  INFO 22192 --- [nio-8080-exec-1] p6spy                                    : #1752111577443 | took 15ms | statement | connection 12| url jdbc:h2:tcp://localhost/~/jpashop
select distinct o1_0.order_id,d1_0.delivery_id,d1_0.city,d1_0.street,d1_0.zipcode,d1_0.status,m1_0.member_id,m1_0.city,m1_0.street,m1_0.zipcode,m1_0.name,o1_0.order_date,oi1_0.order_id,oi1_0.order_item_id,oi1_0.count,i1_0.item_id,i1_0.dtype,i1_0.name,i1_0.price,i1_0.stock_quantity,i1_0.artist,i1_0.etc,i1_0.author,i1_0.isbn,i1_0.actor,i1_0.director,oi1_0.order_price,o1_0.status from orders o1_0 join member m1_0 on m1_0.member_id=o1_0.member_id join delivery d1_0 on d1_0.delivery_id=o1_0.delivery_id join order_item oi1_0 on o1_0.order_id=oi1_0.order_id join item i1_0 on i1_0.item_id=oi1_0.item_id
select distinct o1_0.order_id,d1_0.delivery_id,d1_0.city,d1_0.street,d1_0.zipcode,d1_0.status,m1_0.member_id,m1_0.city,m1_0.street,m1_0.zipcode,m1_0.name,o1_0.order_date,oi1_0.order_id,oi1_0.order_item_id,oi1_0.count,i1_0.item_id,i1_0.dtype,i1_0.name,i1_0.price,i1_0.stock_quantity,i1_0.artist,i1_0.etc,i1_0.author,i1_0.isbn,i1_0.actor,i1_0.director,oi1_0.order_price,o1_0.status from orders o1_0 join member m1_0 on m1_0.member_id=o1_0.member_id join delivery d1_0 on d1_0.delivery_id=o1_0.delivery_id join order_item oi1_0 on o1_0.order_id=oi1_0.order_id join item i1_0 on i1_0.item_id=oi1_0.item_id;
```

<br/>


##### <참고>
Hibernate 6버전 부터는 JPQL에 ```distinct```를 추가하지 않아도 중복이 제거된다.   

```
출처: https://docs.jboss.org/hibernate/orm/6.0/migration-guide/migration-guide.html

...
DISTINCT
Starting with Hibernate ORM 6 it is no longer necessary to use distinct in JPQL and HQL to filter out the same parent entity references when join fetching a child collection. The returning duplicates of entities are now always filtered by Hibernate.

Which means that for instance it is no longer necessary to set QueryHints#HINT_PASS_DISTINCT_THROUGH to false in order to skip the entity duplicates without producing a distinct in the SQL query.

From Hibernate ORM 6, distinct is always passed to the SQL query and the flag QueryHints#HINT_PASS_DISTINCT_THROUGH has been removed.
...
```

### 4-4. 엔티티를 DTO로 변환 - 페이징과 한계 돌파

컬렉션을 페치 조인하면 일대다 조인이 발생해 데이터가 예측할 수 없이 증가한다.    
일대다에서 '일'을 기준으로 페이징을 하는데 '다'를 기준으로 row가 생성되어 페이징을 할 수 없다.   
그리고 컬렉션을 페치 조인 하면서 페이징을 요청하면 Hibernate는 DB의 모든 데이터를 읽어서 메모리에서 페이징을 하기 때문에 장애로 이어질 수 있다.   

<br/>

페이징 + 컬렉션 엔티티를 함께 조회하기 위해 아래 방법을 사용한다.   

1. OneToOne, ManyToOne 관계를 모두 페치 조인 한다. ToOne 관계는 row 수를 증가시키지 않아 페이징 쿼리에 영향을 주지 않는다.   
2. 컬렉션은 지연 로딩으로 조회한다.   
3. 지연 로딩 성능 최적화를 위해 hibernate ```default_batch_fetch_size```, ```@BatchSize```를 적용한다.   
    - ```default_batch_fetch_size```: 글로벌 설정
    - ```@BatchSize```: 애노테이션이 붙은 속성 개발 최적화
    - 이 옵션을 사용하면 컬렉션이나 프록시 객체를 설정한 size 만큼 한꺼번에 IN 쿼리로 조회한다.

```java
@RestController
@RequiredArgsConstructor
public class OrderApiController {

    private final OrderRepository orderRepository;

    ...

    @GetMapping("/api/v3.1/orders")
    public List<OrderDto> ordersV3_page(
            @RequestParam(value = "offset", defaultValue = "0") int offset,
            @RequestParam(value = "limit", defaultValue = "0") int limit) {
        return orderRepository.findAllWithMemberDelivery(offset, limit)
                .stream()
                .map(OrderDto::new)
                .toList();
    }

    ...
}

@Repository
@RequiredArgsConstructor
public class OrderRepository {

    private final EntityManager em;

    ...

    public List<Order> findAllWithMemberDelivery(int offset, int limit) {
        return em.createQuery(
                "select o from Order o" +
                        " join fetch o.member m" +
                        " join fetch o.delivery d", Order.class)
                .setFirstResult(offset)
                .setMaxResults(limit)
                .getResultList();
    }
}
```

ToOne 관계에 있는 ```Member```, ```Delivery```는 페치 조인하고 페이징 요청을 한다.   

```java
@RestController
@RequiredArgsConstructor
public class OrderApiController {

    private final OrderRepository orderRepository;

    ...

    @GetMapping("/api/v3.1/orders")
    public List<OrderDto> ordersV3_page(
            @RequestParam(value = "offset", defaultValue = "0") int offset,
            @RequestParam(value = "limit", defaultValue = "0") int limit) {
        return orderRepository.findAllWithMemberDelivery(offset, limit)
                .stream()
                .map(OrderDto::new)
                .toList();
    }

    @Data
    static class OrderDto {

        private Long orderId;
        private String name;
        private LocalDateTime orderDate;
        private OrderStatus orderStatus;
        private Address address;
        private List<OrderItemDto> orderItems;

        public OrderDto(Order order) {
            orderId = order.getId();
            name = order.getMember().getName();
            orderDate = order.getOrderDate();
            orderStatus = order.getStatus();
            address = order.getDelivery().getAddress();
            orderItems = order.getOrderItems().stream().map(OrderItemDto::new).toList();
        }
    }

    @Data
    static class OrderItemDto {

        private String itemName;
        private int orderPrice;
        private int count;

        public OrderItemDto(OrderItem orderItem) {
            itemName = orderItem.getItem().getName();
            orderPrice = orderItem.getOrderPrice();
            count = orderItem.getCount();
        }
    }
}
```

```OrderDto```의 ```orderItems = order.getOrderItems().stream().map(OrderItemDto::new).toList();``` 코드에서 컬렉션인 ```orderItems```를 지연로드 한다.    
이어서 ```OrderItemDto```에서 ```Item```을 지연로드 한다.   

```yaml
spring:
  datasource:
    url: jdbc:h2:tcp://localhost/~/jpashop
    username: sa
    password: 1234
    driver-class-name: org.h2.Driver

  jpa:
    hibernate:
      ddl-auto: none
    properties:
      hibernate:
        #        show_sql: true
        format_sql: true
        default_batch_fetch_size: 100

logging.level:
  org.hibernate.SQL: debug
#  org.hibernate.orm.jdbc.bind: trace
```

application.yml 파일에 ```default_batch_fetch_size: 100``` 속성을 추가한다.    
애플리케이션 전역에 설정되고 컬렉션이나 프록시 객체를 설정한 100개 만큼 한 꺼번에 IN 쿼리로 조회한다.

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

    @BatchSize(size = 1000)
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    private List<OrderItem> orderItems = new ArrayList<>();

    ...
}
```

```@BatchSize``` 애노테이션을 사용하면 특정 속성에 설정한 만큼 한 꺼번에 IN 쿼리로 조회한다.    
```orderItems```는 한 꺼번에 1000개를 IN 쿼리로 조회한다.   

<br/>

위 방법을 사용하면 페치 조인을 사용하며 쿼리 호출 수가 1 + N에서 1 + 1로 최적화 된다.   
```/api/v3.1/orders?offset=1&limit=100``` API 요청을 보낼 때 실제로 호출되는 쿼리는 아래와 같다.   
offset 값이 1 이므로 주문은 한 개만 조회된다.   

```
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
offset
    ? rows 
fetch
    first ? rows only


select
    oi1_0.order_id,
    oi1_0.order_item_id,
    oi1_0.count,
    oi1_0.item_id,
    oi1_0.order_price 
from
    order_item oi1_0 
where
    oi1_0.order_id=?


select
    i1_0.item_id,
    i1_0.dtype,
    i1_0.name,
    i1_0.price,
    i1_0.stock_quantity,
    i1_0.artist,
    i1_0.etc,
    i1_0.author,
    i1_0.isbn,
    i1_0.actor,
    i1_0.director 
from
    item i1_0 
where
    i1_0.item_id in (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ..., ?)
```

##### <참고> 
```default_batch_fetch_size```
의 크기는 적당한 사이즈를 골라야 하는데, 100~1000 사이를 선택하는 
것을 권장한다. 이 전략을 SQL IN 절을 사용하는데, 데이터베이스에 따라 IN 절 파라미터를 1000으로 제한하기
도 한다. 1000으로 잡으면 한번에 1000개를 DB에서 애플리케이션에 불러오므로 DB에 순간 부하가 증가할 수 
있다. 하지만 애플리케이션은 100이든 1000이든 결국 전체 데이터를 로딩해야 하므로 메모리 사용량이 같다. 
1000으로 설정하는 것이 성능상 가장 좋지만, 결국 DB든 애플리케이션이든 순간 부하를 어디까지 견딜 수 있는
지로 결정하면 된다.