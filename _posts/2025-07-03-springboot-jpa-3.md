---
layout: post
title: 스프링 부트와 JPA 활용3
date: 2025-06-08 20:31:00 + 0900
categories: [spring]
tags: [spring, jpa]
---

### 강의 : [실전! 스프링 부트와 JPA 활용2 - API 개발과 성능 최적화](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81%EB%B6%80%ED%8A%B8-JPA-API%EA%B0%9C%EB%B0%9C-%EC%84%B1%EB%8A%A5%EC%B5%9C%EC%A0%81%ED%99%94/dashboard)

### 1. Rest API 요청을 받기 위한 별도의 DTO를 만든다.

실무에서는 API 파라미터로 엔티티를 사용하거나 외부에 노출하면 안된다.   

<br/>

엔티티에 프레젠테이션 계층을 위한 로직이 추가된다.   
엔티티에 @NotEmpty 같은 API 검증을 위한 로직이 들어간다.   
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

