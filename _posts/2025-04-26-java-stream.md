---
layout: post
title: Java - stream
date: 2025-04-25 21:00:00 + 0900
categories: [java]
tags: [lambda, stream, optional, functional programming]
---
### 강의 : [김영한의 실전 자바 - 고급 3편, 람다, 스트림, 함수형 프로그래밍](https://www.inflearn.com/course/%EA%B9%80%EC%98%81%ED%95%9C%EC%9D%98-%EC%8B%A4%EC%A0%84-%EC%9E%90%EB%B0%94-%EA%B3%A0%EA%B8%89-3/dashboard)

# 1. 스트림 API - 기본

## 1-1. 스트림 API 시작

map, filter 기능을 제공하는 MyStreamV3을 만들어 List<T> Collection에 원하는 연산을 선언적으로 구현할 수 있게 지원했다.    
MyStreamV3을 사용해 작업을 어떻게 수행해야 하는지 보다 무엇을 수행해야 하는지에 집중할 수 있고 데이터가 물 흐르듯이 처리 되었다.   
자바도 MyStreamV3과 동일하지만 더 정교하고 많은 기능을 제공하는 스트림 API를 제공한다.   

```java
public class StreamStartMain {

    public static void main(String[] args) {
        List<String> names = List.of("Apple", "Banana", "Berry", "Tomato");

        Stream<String> stream = names.stream();
        List<String> result = stream
                .filter(name -> name.startsWith("B"))
                .map(String::toUpperCase)
                .toList();

        System.out.println("=== 외부 반복 ===");
        for (String s : result) System.out.println(s);

        System.out.println("=== forEach, 내부 반복 ===");
        names.stream()
                .filter(name -> name.startsWith("B"))
                .map(String::toUpperCase)
                .forEach(System.out::println);
    }
}
```

- 중간 연산 : map, filter 메소드와 같이 스트림에서 요소를 걸러내거나 다른 형태로 변환하는 기능이다.   
- 최종 연산 : toList 메소드와 같이 중간 연산에서 정의한 연산의 최종 결과를 만들어 반환한다.   
- 내부 반복 : forEach 메소드와 같이 스트림에 담긴 요소들을 내부적으로 반복해 가며 람다로 전달한 동작을 수행한다. 내부 반복을 사용하면 스트림이 알아서 반복문을 수행하기 때문에 개발자가 직접 for/while 문을 작성하지 않아도 된다.   

## 1-2. 스트림 API란?
Stream은 자바 8부터 추가된 기능으로 데이터의 흐름을 추상화해서 다루는 도구이다.   
Collection 또는 배열 등의 요소들을 연산 파이프라인을 통해 연속적인 형태로 처리할 수 있게 해준다.   
스트림은 5가지 특징을 가진다.   

- 데이터 소스를 변경하지 않음(Immutable) : 스트림 API의 연산들은 원본 컬렉션을 변경하지 않고 결과를 새로 생성한다.
- 일회성 : 한 번 사용한 스트림은 다시 사용할 수 없고 새로 스트림을 생성해야 한다.
- 지연 연산 : 중간 연산은 필요할 때 까지 동작하지 않고 최종 연산이 실행될 때 한 번에 처리된다.
- 병렬 처리 : 병렬 스트림을 쉽게 만들 수 있어 병렬 연산을 단순한 코드로 작성할 수 있다.

## 1-3. 일괄 처리 vs 파이프라인

아래 코드와 같이 배열에서 짝수를 필터링하고 10을 곱한 값으로 매핑하는 연산을 MyStreamV3와 스트림 API로 실행하면,   
결과는 같지만 연산을 처리하는 과정은 다르다.   

```java
public class LazyEvalMain1 {

    public static void main(String[] args) {
        List<Integer> data = List.of(1, 2, 3, 4, 5, 6);
        ex1(data);
        ex2(data);
    }

    private static void ex2(List<Integer> data) {
        List<Integer> result = data.stream()
                .filter(i -> i % 2 == 0)
                .map(i -> i * 10)
                .toList();

        System.out.println("Stream API result = " + result);
    }

    private static void ex1(List<Integer> data) {
        List<Integer> result = MyStreamV3.of(data)
                .filter(i -> i % 2 == 0)
                .map(i -> i * 10)
                .toList();

        System.out.println("MyStreamV3 result = " + result);
    }
}
```

MyStreamV3는 일괄 처리 방식이고 자바 스트림 API는 파이프라인 방식이다.   
일괄 처리는 각 단계 마다 모든 데이터에 대해 연산을 하고 결과를 만든 다음 단계로 넘긴다.   
파이프라인 처리는 한 요소 마다 모든 연산을 적용한다.   

<br/>

MyStreamV3의 일괄 처리 연산 순서는 다음과 같다.

```
1. data(1, 2, 3, 4, 5, 6)
2. filter(1, 2, 3, 4, 5, 6) -> 2, 4, 6
3. map(2, 4, 6) -> 20, 40, 60
4. list(20, 40, 60)
```

스트림 API의 파이프라인 처리 연산 순서는 다음과 같다.

```
1. data(1) -> filter(1) -> false
2. data(2) -> filter(2) -> map(2) -> 20 -> list(20)
3. data(3) -> filter(3) -> false
4. data(4) -> filter(4) -> map(4) -> 40 -> list(20, 40)
5. data(5) -> filter(5) -> false
6. data(6) -> filter(6) -> map(6) -> 60 -> list(20, 40, 60)
```

## 1-4. 지연 연산

# 2. 스트림 API - 기능

# 3. 스트림 APi - 컬렉터