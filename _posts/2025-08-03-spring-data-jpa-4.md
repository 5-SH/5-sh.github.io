---
layout: post
title: 스프링 데이터 JPA 4
date: 2025-08-03 18:00:00 + 0900
categories: [spring]
tags: [spring, jpa]
---

### 강의 : [실전! 스프링 데이터 JPA](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-%EB%8D%B0%EC%9D%B4%ED%84%B0-JPA-%EC%8B%A4%EC%A0%84)

### 5. 스프링 데이터 JPA 분석

#### 5-1. 스프링 데이터 JPA 구현체 분석

```org.springframework.data.jpa.repository.support.SimpleJpaRepository```가 스프링 데이터 JPA의 구현체이다.   

```java
@Repository
@Transactional(readOnly = true)
public class SimpleJpaRepository<T, ID> implements JpaRepositoryImplementation<T, ID> {

    ...
    public Optional<T> findById(ID id) {
        Assert.notNull(id, "The given id must not be null");
        Class<T> domainType = this.getDomainClass();
        if (this.metadata == null) {
            return Optional.ofNullable(this.entityManager.find(domainType, id));
        } else {
            LockModeType type = this.metadata.getLockModeType();
            Map<String, Object> hints = this.getHints();
            return Optional.ofNullable(type == null ? this.entityManager.find(domainType, id, hints) : this.entityManager.find(domainType, id, type, hints));
        }
    }

    @Deprecated
    public T getOne(ID id) {
        return (T)this.getReferenceById(id);
    }

    ...

    public List<T> findAll() {
        return this.getQuery((Specification)null, (Sort)Sort.unsorted()).getResultList();
    }

    public List<T> findAllById(Iterable<ID> ids) {
        Assert.notNull(ids, "Ids must not be null");
        if (!ids.iterator().hasNext()) {
            return Collections.emptyList();
        } else if (!this.entityInformation.hasCompositeId()) {
            Collection<ID> idCollection = toCollection(ids);
            ByIdsSpecification<T> specification = new ByIdsSpecification<T>(this.entityInformation);
            TypedQuery<T> query = this.getQuery(specification, (Sort)Sort.unsorted());
            return query.setParameter(specification.parameter, idCollection).getResultList();
        } else {
            List<T> results = new ArrayList();

            for(ID id : ids) {
                this.findById(id).ifPresent(results::add);
            }

            return results;
        }
    }

    ...
}
```

자주 사용하는 ```findById(ID id)``` 메소드를 살펴보면, JPA에서 제공하는 EntityManager를 그대로 사용해서 구현한 것을 알 수 있다.   
```getOne(ID id)```, ```findAll()```, ```findAllById()``` 등 그 외의 메소드도 동일하게 구현되어 있다.   

<br/>

```SimpleJpaRepository```에 ```@Repository```와 ```@Transactional``` 애노테이션이 적용되어 있다.   

```@Repository``` 애노테이션은 ```SimpleJpaRepository```을 스프링 빈 컴포넌트 대상으로 만들어 스프링 컨테이너에 등록한다.   
그리고 영속성 계층의 세부 기술(JPA, JDBC...) 마다 다른 예외를 스프링이 추상화한 예외로 변환한다.    
서비스나 컨트롤러 계층에선 스프링 프레임워크가 제공하는 예외를 받기 때문에 영속성 계층의 세부 기술을 변경해도 상위 계층을 수정하지 않아도 된다.   

<br/>

```SimpleJpaRepository```에 ```@Transactional(readOnly = true)```로 등록되어 있고 ```save```, ```delete``` 같은 등록, 수정, 삭제 메서드는 ```@Transactional```로 재정의했다.   
그래서 스프링 데이터 JPA를 사용하면 모든 JPA의 데이터 변경은 트랜잭션 안에서 일어나고, 별도의 트랜잭션 설정 없이 등록, 수정, 삭제를 사용할 수 있다.   

<br/>

```SimpleJpaRepository```는 대부분 조회와 관련된 메소드를 갖고 있어 ```@Transactional(readOnly = true)```를 등록한다.   
```readOnly = true``` 옵션을 사용하면 플러시를 생략해서 약간의 성능 향상을 얻을 수 있고 엔티티를 변경 하더라도 데이터베이스에 반영하지 않는다.   

<br/>

```save()``` 메서드는 새로운 엔티티면 ```persist```를 호출하고 새로운 엔티티가 아니면 ```merge```를 호출한다.   
```merge```를 호출하면 데이터베이스에 조회를 해서 새로운 엔티티를 가져온 다음 ```save()``` 메서드에 파라미터로 넘긴 ```entity```의 값으로 다 교체를 한다. 그리고 트랜잭션이 끝날 때 데이터베이스에 반영한다.   

가급적 데이터 변경은 ```merge```를 사용하면 안되고 변경감지를 사용해야 한다.   
```merge```는 데이터를 반영할 때 쓰는 것이 아니라 영속 상태를 벗어난 엔티티가 다시 영속 상태가 되어야 할 때 사용한다.   

```java
@Repository
@Transactional(readOnly = true)
public class SimpleJpaRepository<T, ID> implements JpaRepositoryImplementation<T, ID> {
    
    ...
    @Transactional
    public <S extends T> S save(S entity) {
        Assert.notNull(entity, "Entity must not be null");
        if (this.entityInformation.isNew(entity)) {
            this.entityManager.persist(entity);
            return entity;
        } else {
            return (S)this.entityManager.merge(entity);
        }
    }
    ...

}
```

#### 5-2. 새로운 엔티티를 구별하는 방법

```SimpleJpaRepository```의 ```save()``` 메소드에서 새로운 엔티티 인지 확인할 때 ```this.entityInformation.isNew(entity)``` 코드를 사용한다.   

새로운 엔티티를 판단하는 기본 전략은 아래와 같다.   

- 식별자가 객체일 때 ```null``로 판단
- 식별자가 자바 기본 타입일 때 ```0```으로 판단
- ```Persistable``` 인터페이스를 구현해서 판단 로직 변경 가능

위 코드로 테스트를 하면, id가 ```null``` 이므로 ```this.entityManager.persist(entity);``` 메서드가 실행된다.   

엔티티의 식별자에 ```@GeneratedValue``` 애노테이션이 등록되면 식별자 값을 자동으로 등록해 주는데, EntityManager의 ```persist``` 메소드를 호출할 때
등록된다.   
따라서 ```this.entityInformation.isNew(entity)``` 코드를 실행할 때 엔티티의 식별자는 ```null```이고 ```this.entityManager.persist(entity);``` 메소드를 실행한 다음 식별자가 할당된다.   

```java
@Entity
@Getter
public class Item {

    @Id @GeneratedValue
    private Long id;
}

public interface ItemRepository extends JpaRepository<Item, Long> {}

@SpringBootTest
class ItemRepositoryTest {

    @Autowired ItemRepository itemRepository;

    @Test
    public void save() {
        Item item = new Item();
        itemRepository.save(item);
    }
}
```

그러나 아래와 같이 엔티티의 식별자가 ```@GeneratedValue``` 애노테이션을 사용하지 않는 경우에는 ```save()``` 메소드가 예상대로 동작하지 않는다.   

```java
@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Item {

    @Id
    private String id;

    public Item(String id) {
        this.id = id;
    }
}

@SpringBootTest
class ItemRepositoryTest {

    @Autowired ItemRepository itemRepository;

    @Test
    public void save() {
        Item item = new Item("A");
        itemRepository.save(item);
    }
}
```

```Item``` 엔티티의 식별자 값을 "A"로 직접 할당했기 때문에 ```this.entityInformation.isNew(entity)``` 코드를 실행하면 ```return (S)this.entityManager.merge(entity);``` 코드가 실행된다.   
```merge```를 사용하면 우선 데이터베이스에 조회 요청을 하고 데이터베이스에 없으면 새로운 엔티티로 인지하므로 매우 비효율 적이다.   
이런 경우 ```Persistable```을 사용해 새로운 엔티티 확인 여부를 직접 구현해 해결한다.   

엔티티 클래스에서 ```Persistable``` 인터페이스를 구현하면 새로운 엔티티인지 확인하는 ```isNew()``` 메서드를 직접 구현해야 한다.   
그리고 등록시간(```@CreatedDate```)을 조합해서 사용하면 이 필드로 새로운 엔티티 여부를 쉽게 확인할 수 있다.   

```java
@Entity
@EntityListeners(AuditingEntityListener.class)
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Item implements Persistable<String> {

    @Id
    private String id;

    @CreatedDate
    private LocalDateTime createdDate;

    public Item(String id) {
        this.id = id;
    }

    @Override
    public String getId() {
        return id;
    }

    @Override
    public boolean isNew() {
        return createdDate == null;
    }
}
```

위와 같이 엔티티에 ```Persistable``` 인터페이스를 구현한 다음 테스트를 하면 ```this.entityInformation.isNew(entity)``` 코드를 실행한 다음 ```this.entityManager.persist(entity);``` 메서드가 실행되는 것을 확인할 수 있다.   
