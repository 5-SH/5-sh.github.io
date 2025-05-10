---
layout: post
title: Java - optional
date: 2025-04-25 21:00:00 + 0900
categories: [java]
tags: [lambda, stream, optional, functional programming]
---
### 강의 : [김영한의 실전 자바 - 고급 3편, 람다, 스트림, 함수형 프로그래밍](https://www.inflearn.com/course/%EA%B9%80%EC%98%81%ED%95%9C%EC%9D%98-%EC%8B%A4%EC%A0%84-%EC%9E%90%EB%B0%94-%EA%B3%A0%EA%B8%89-3/dashboard)

## 1. Optional이 필요한 이유

### 1-1. NullPointerException 문제

null 참조에 대해 메서드를 호출하면 NullPointerException이 발생해 프로그램이 예기치 않게 종료될 수 있다.    
null 체크를 하기 위해 if (obj != null) { ... } 같은 조건문이 자주 발생하게 된다.   
그리고 메서드 시그니처(String findNameById(Long id))만으로 null을 반환하는지 알 수 없다.    

<br/>

이런 문제를 해결하기 위해 자바 8부터 Optional 클래스를 도입했다.   
Optional은 "값이 있을 수도 있고 없을 수도 있음"을 명시적으로 표현한다.   
Optional을 사용하면 "빈 값"을 표현할 때, null 대신 Optional.empty()를 사용해 의도를 드러낼 수 있다.    
그리고 null 체크 로직을 간결하게 만들어 NullPointException이 발생할 수 있는 부분을 쉽게 파악할 수 있다.    

<br/>

### 1-2. Optional 소개

Optional<T>는 존재할 수도 존재하지 않을 수도 있는 값을 감싸는 컨테이너 클래스이다.   
직접 null을 다루는 대신 Optional 객체로 감싸서 Optional.empty() 또는 Optional.of(value) 형태로 다룬다.   

```java
package java.util;

public final class Optional<T> {
    private final T value;
    ...
}
```

#### AS-IS

```java
public class OptionalStartMain1 {

    private static final Map<Long, String> map = new HashMap<>();

    static {
        map.put(1L, "Kim");
        map.put(2L, "Seo");
    }

    public static void main(String[] args) {
        findAndPrint(1L);
        findAndPrint(2L);
    }

    static void findAndPrint(Long id) {
        String name = findByName(id);

        if (name != null) System.out.println(id + ": " + name.toUpperCase());
        else System.out.println(id + ": " + "UNKNOWN");
    }

    static String findByName(Long id) {
        return map.get(id);
    }
}
```

#### Optional 사용

```java
public class OptionalStartMain2 {

    private static final Map<Long, String> map = new HashMap<>();

    static {
        map.put(1L, "Kim");
        map.put(2L, "Seo");
    }

    public static void main(String[] args) {
        findAndPrint(1L);
        findAndPrint(2L);
    }

    static void findAndPrint(Long id) {
        Optional<String> optName = findByName(id);
        String name = optName.orElse("UNKNOWN");
        System.out.println(id + ": " + name.toUpperCase());
    }

    static Optional<String> findByName(Long id) {
        String findName = map.get(id);
        Optional<String> optName = Optional.ofNullable(findName);

        return optName;
    }
}
```

## 2. Optional 생성과 값 획득

### 2-1. Optional 생성

- Optional.of(T value): 내부 값이 확실히 null이 아닐 때 사용. null을 전달하면 NullPointerException 발생
- Optional.ofNullable(T value): 값이 null일 수도 있고 아닐 수도 있을 때 사용. null 이면 Optional.empty()를 반환
- Optional.empty(): 값이 없음을 표현할 때 사용

```java
public class OptionalCreationMain {
    public static void main(String[] args) {
        // 1. of() : 값이 null이 아님이 확실할 때 사용, null이면 NullPointerException 발생
        String nonNullValue = "Hello Optional!";
        Optional<String> opt1 = Optional.of(nonNullValue);
        System.out.println("opt1 = " + opt1);

        // 2. ofNullable() : 값이 null일 수도 아닐 수도 있을 때
        Optional<String> opt2 = Optional.ofNullable("Hello!");
        Optional<String> opt3 = Optional.ofNullable(null);
        System.out.println("opt2 = " + opt2);
        System.out.println("opt3 = " + opt3);

        // 3. empty() : 비어있는 Optional을 명시적으로 생성
        Optional<String> opt4 = Optional.empty();
        System.out.println("opt4 = " + opt4);
    }
}
```

### 2-2. Optional 값 획득

