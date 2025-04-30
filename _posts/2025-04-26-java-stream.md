---
layout: post
title: Java - stream
date: 2025-04-25 21:00:00 + 0900
categories: [java]
tags: [lambda, stream, optional, functional programming]
---
### 강의 : [김영한의 실전 자바 - 고급 3편, 람다, 스트림, 함수형 프로그래밍](https://www.inflearn.com/course/%EA%B9%80%EC%98%81%ED%95%9C%EC%9D%98-%EC%8B%A4%EC%A0%84-%EC%9E%90%EB%B0%94-%EA%B3%A0%EA%B8%89-3/dashboard)

## 1. 스트림 API - 기본

### 1-1. 스트림 API 시작

map, filter 기능을 제공하는 MyStreamV3을 만들어 List<T> Collection에 원하는 연산을 선언적으로 구현할 수 있게 지원했다.    
MyStreamV3을 사용해 작업을 어떻게 수행해야 하는지 보다 무엇을 수행해야 하는지에 집중할 수 있고 데이터가 물 흐르듯이 처리 되었다.   
자바도 MyStreamV3과 동일하지만 더 정교하고 많은 기능을 제공하는 스트림 API를 제공한다.   

<br/>

스트림은 데이터 처리 추상화 도구로서 컬렉션/배열 등의 요소들을 일련의 파이프라인으로 연결해 가공, 필터링, 집계할 수 있다.   
스트림은 **가독성**, **선언형 코드**, **지연 연산에 따른 최적화** 라는 장점을 제공한다.   

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

### 1-2. 스트림 API란?
Stream은 자바 8부터 추가된 기능으로 데이터의 흐름을 추상화해서 다루는 도구이다.   
Collection 또는 배열 등의 요소들을 연산 파이프라인을 통해 연속적인 형태로 처리할 수 있게 해준다.   
스트림은 5가지 특징을 가진다.   

- 데이터 소스를 변경하지 않음(Immutable) : 스트림 API의 연산들은 원본 컬렉션을 변경하지 않고 결과를 새로 생성한다.
- 일회성 : 한 번 사용한 스트림은 다시 사용할 수 없고 새로 스트림을 생성해야 한다.
- 지연 연산 : 중간 연산은 필요할 때 까지 동작하지 않고 최종 연산이 실행될 때 한 번에 처리된다.
- 병렬 처리 : 병렬 스트림을 쉽게 만들 수 있어 병렬 연산을 단순한 코드로 작성할 수 있다.

### 1-3. 일괄 처리 vs 파이프라인

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

### 1-4. 지연 연산

MyStreamV3는 즉시 연산을 하고 스트림 API는 최종 연산을 수행할 때만 작동한다.   
MyStreamV3는 filter, map 같은 중간 연산이 호출될 때 마다 바로 연산을 수행하고 각각의 결과를 메모리에 저장한다.   
그리고 최종 연산이 없어도 중간 연산이 동작해 필요 이상의 연산이 수행될 수 있다.   
스트림 API는 최종 연산이 호출되기 전까지는 실제로 연산이 일어나지 않는다.    
중간 연산 코드를 만나면 람다를 내부에 저장하고 최종 연산을 만났을 때 순차적으로 한 번에 수행한다.   
불필요한 연산을 생략하고 중간 연산 결과를 만들지 않아 메모리를 효율적으로 사용할 수 있다.   

<br/>

아래 코드는 배열에서 짝수를 필터링하고 10을 곱한 결과에서 첫 번째 요소를 가져오는 코드이다.   
MyStreamV3는 즉시 연산을 하기 때문에 배열의 모든 값에 대해 filter, map 연산을 하고 각 연산의 결과를 메모리에 저장한다.   
그리고 결과 배열에서 첫 번째 요소를 선택해 반환한다.   
스트림 API는 지연 연산을 하기 때문에 배열의 요소 마다 filter, map 연산을 저장하고 findFirst() 최종 연산을 만났을 때 연산을 실행한다.   
그리고 첫 번째 결과를 받았을 때 연산을 종료한다.   

MyStreamV3의 즉시 연산 순서는 다음과 같다.   

```
1. data(1, 2, 3, 4, 5, 6)
2. filter(1, 2, 3, 4, 5, 6) -> 2, 4, 6
3. map(2, 4, 6) -> 20, 40, 60
4. getFirst(20, 40, 60) -> 20
```

스트림 API의 지연 연산 순서는 다음과 같다.   

```
1. map(filter(data(1))) -> findFirst() -> false
2. map(filter(data(2))) -> findFirst() -> 20 -> get() -> 20
```

```java
public class LazyEvalMain3 {

    public static void main(String[] args) {
        List<Integer> data = List.of(1, 2, 3, 4, 5, 6);
        ex1(data);
        ex2(data);
    }

    private static void ex2(List<Integer> data) {
        Integer result = data.stream()
                .filter(i -> i % 2 == 0)
                .map(i -> i * 10)
                .findFirst()
                .get();

        System.out.println("Stream API result = " + result);
    }

    private static void ex1(List<Integer> data) {
        Integer result = MyStreamV3.of(data)
                .filter(i -> i % 2 == 0)
                .map(i -> i * 10)
                .getFirst();

        System.out.println("MyStreamV3 result = " + result);
    }
}
```

## 2. 스트림 API - 기능

### 2-1. 스트림 생성

|생성 방법|코드 예시|특징|
|---|---|---|
|컬렉션|list.stream()|List, Set 등 컬렉션에서 스트림 생성|
|배열|Arrays.stream(arr)|배열에서 스트림 생성|
|Stream.of(...)|Stream.of("a", "b", "c")|직접 요소를 입력해 스트림 생성|
|무한 스트림(iterate)|Stream.iterate(0, n -> n + 2)|무한 스트림 생성 (초깃값 + 함수)

### 2-2. 스트림 중간 연산

|연산|설명|예시|
|---|---|---|
|filter|조건에 맞는 요소만 남김|stream.filter(n -> n > 5)|
|map|요소를 다른 형태로 변환|stream.map(n -> n * 2)|
|flatMap|중첩 구조 스트림을 일차원으로 평탄화|stream.flatMap(list -> list.stream())|
|distinct|중복 요소 제거|stream.distinct()|
|sorted|요소 정렬|stream.sorted() / stream.sorted(Comparator.reverseOrder())|
|peek|중간 처리 (로그, 디버깅)|stream.peek(System.out::println)|
|limit|앞에서 N개의 요소만 추출|stream.limit(5)|
|skip|앞에서 N개의 요소를 건너뛰고 이후 요소만 추출|stream.skip(5)|
|takeWhile|조건을 만족하는 동안 요소 추출|stream.takeWhile(n -> n < 5)|
|dropWhile|조건을 만족하는 동안 요소를 버리고 이후 요소 추출|stream.dropWhile(n -> n < 5)|

### 2-3. 스트림 최종 연산

|연산|설명|예시|
|---|---|---|
|collect|Collactor를 사용해 결과 수집|stream.collect(Collectors.toList())|
|toList|스트림을 불변 리스트로 수집|stream.toList()|
|toArray|스트림을 배열로 변환|stream.toArray(Integer[]::new)|
|forEach|각 요소에 대해 동작 수행(반환값 없음)|stream.forEach(System.out::println)|
|count|요소 개수 반환|long count = stream.count()|
|reduce|누적 함수를 사용해 모든 요소를 단일 결과로 합침, 초깃값이 없으면 Optional로 반환|int sum = steram.reduce(0, Integer::sum)|
|min / max|최솟값, 최댓값을 Optional로 반환|stream.min(Integer::compareTo), stream.max(Integer::compareTo)|
|findFirst|조건에 맞는 첫 번째 요소 (Optional 반환)|stream.findFirst()|
|findAny|조건에 맞는 아무 요소 (Optional 반환)|stream.findAny()|
|anyMatch|하나라도 조건을 만족하는지 (boolean)|stream.allMatch(n -> n > 0)|
|allMatch|모두 조건을 만족하는지 (boolean)|stream.allMatch(n -> n > 0)|

### 2-4. 기본형 특화 스트림

기본형 특화 스트림은 int, long, double 자료형에 합계, 평균, 최솟값, 최댓값 등 정수와 관련된 편리한 연산을 제공한다.   

|스트림 타입|대상 원시 타입|생성 예시|
|---|---|---|
|IntStream|int|IntStream.of(1, 2, 3), IntStream.range(1, 10), mapToInt(...)|
|LongStream|long|LongStream.of(10L, 20L), LongStream.range(1, 10), mapToLong(...)
|DoubleStream|double|DoubleStream.of(3.14, 2.78), DoubleStream.generate(Matho::random), mapToDouble(...)|

#### 주요 기능 및 메서드

|메서드 / 기능|설명|예시|
|---|---|---|
|sum()|모든 요소의 합계를 구한다|int total = IntStream.of(1, 2, 3).sum()|
|average()|모든 요소의 평균을 구한다|dobule avg = IntStream.range(1, 5).average().getAsDouble()|
|summary Staticstics()|최솟값, 최댓값, 합계, 개수, 평균 등이 담긴 IntSummaryStaticstics(또는 Long/Double) 객체 반환|IntSummaryStatistics stats = IntStream.range(1, 5).summaryStatistics()|
|mapToLong(), mapToDouble()|타입 변환: IntStream -> LongStream, DoubleStream ...|LongStream ls = IntStream.of(1, 2).mapToLong(i -> i * 10L)|
|mapToObj()|객체 스트림으로 변환 : 기본형 → 참조형|Stream<String> s = IntStream.range(1, 5).mapToObj(i -> "No: " + i)|
|boxed()|기본형 특화 스트림을 박싱(Wrapper)된 객체 스트림으로 변환|Stream<Integer> si = IntStream.range(1, 5).boxed()|
|min(), max(), count()|합계, 최솟값, 최댓값, 개수를 반환(타입 별로 int/long/double 반환)|long count = LongStream.of(1, 2, 3).count()|

### 2-5. 성능 - for문 vs 스트림 vs 기본형 특화 스트림

스트림은 박싱/언박싱 오버헤드가 발생해 for문이 스트림보다 1.5~2배 정도 빠르다.   
기본형 특화 스트림은 for문에 가까운 성능을 낸다.   
그러나 대부분의 일반적인 애플리케이션에서는 거의 차이가 없다.    
수천만 건 이상 반복하는 루프를 많이 수행해야 의미있는 속도 차이가 발생할 수 있다.    
실무에서 극단적인 성능이 필요한 경우가 아니라면 코드의 가독성과 유지보수성을 위해 스트림 API를 사용하는 것이 보통 더 나은 선택이다. 

## 3. 스트림 API - 컬렉터

스트림이 중간 연산을 거쳐 최종 연산으로 데이터를 처리할 때 리스트나 맵 같은 자료 구조에 담거나 통계 데이터를 내는 등의 결과가 필요할 수 있다.    
스트림 API의 collect 연산은 결과를 만들어 내는 최종 연산이고 이 때 최종 연산에 Collectors를 활용한다.   
collect(Collector<? super T, A, R> collector) 형태를 주로 사용하고 Collectors 클래스 안에 준비된 여러 메서드를 통해서 다양한 수집 방식을 적용할 수 있다.   

#### Collectors의 주요 기능 

|기능|메서드 예시|설명|반환 타입|
|---|---|---|---|
|List로 수집|toList(), toUnmodifiableList()|스트림 요소를 List로 모은다. toUnmodifiableList()는 불변 리스트를 만든다.|List<T>|
|Set으로 수집|toSet(), toCollection(HashSet::new)|스트림 요소를 Set으로 모은다. 중복 요소는 자동으로 제거된다. 특정 Set 타입으로 모으려면 toCollection() 사용.|Set<T>|
|Map으로 수집|toMap(keyMapper, valueMapper), toMap(KeyMapper, valueMapper, mergeFunction, mapSupplier)|스트림 요소를 Map에 (키, 값) 형태로 수집한다. 중복 키가 생기면 mergeFunction으로 해결하고, mapSupplier로 맵 타입을 지정할 수 있다.|Map<K, V>|
|그룹화|groupingBy(classifier), groupingBy(classifier, downstreamCollector)|특정 기준 함수(classfier)에 따라 그룹별로 스트림 요소를 묶는다. 각 그룹에 대해 추가로 적용할 다운 슽릠 컬렉터를 지정할 수 있다.|Map<K, List<T>>, Map<K, R>|
|분할|partitioningBy(prediate), partitioningBy(predicate, downstreamCollector)|predicate 결과가 true와 false 두 가지로 나뉘어, 2개 그룹으로 분할한다.|Map<Boolean, List<T>>, Map<Boolean, R>|
|통계|counting(), summingInt(), averagingInt(), summarizingInt() 등|요소의 개수, 합계, 평균, 최소, 최댓값 등을 구하거나, IntSummaryStatistics 같은 통계 객체로도 모을 수 있다.|Long, Integer, Double, IntSummaryStatistics 등|
|리듀싱|reducing(...)|스트림의 reduce()와 유사하게, Collector 환경에서 요소를 하나로 합치는 연산을 할 수 있다.|Optional<T> 혹은 다른 타입|
|문자열 연결|joining(delimiter, prefix, suffix)|문자열 스트림을 하나로 합쳐서 연결한다. 구분자(delimiter), 접두사(prefix), 접미사(suffix) 등을 붙일 수 있다.|String|
|매핑|mapping(mapper, downstream)|각 요소를 다른 값으로 변환(mapper)한 뒤 다운스트림 컬렉터로 넘긴다.|다운스트림 결과 타입에 따름|


```java
public class Collertors {

    public static void main(String[] args) {
        basic();
        map();
        group();
        minmax();
        summing();
        reducing();
    }

    private static void reducing() {
        List<String> names = List.of("a", "b", "c", "d");

        String joined1 = names.stream().collect(Collectors.reducing((s1, s2) -> s1 + ", " + s2)).get();
        System.out.println("joined1 = " + joined1);

        String joined2 = names.stream().reduce((s1, s2) -> s1 + ", " + s2).get();
        System.out.println("joined2 = " + joined2);

        String joined3 = names.stream().collect(Collectors.joining(", "));
        System.out.println("joined3 = " + joined3);

        String joined4 = String.join(", ", "a", "b", "c", "d");
        System.out.println("joined4 = " + joined4);
    }

    private static void summing() {
        Long count1 = Stream.of(1, 2, 3).collect(Collectors.counting());
        System.out.println("count1 = " + count1);

        long count2 = Stream.of(1, 2, 3).count();
        System.out.println("count2 = " + count2);

        Double average1 = Stream.of(1, 2, 3).collect(Collectors.averagingInt(i -> i));
        System.out.println("average1 = " + average1);

        double average2 = Stream.of(1, 2, 3).mapToInt(i -> i).average().getAsDouble();
        System.out.println("average2 = " + average2);

        double average3 = IntStream.of(1, 2, 3).average().getAsDouble();
        System.out.println("average3 = " + average3);

        IntSummaryStatistics stats = Stream.of("Apple", "Banana", "Tomato").collect(Collectors.summarizingInt(String::length));
        System.out.println(stats.getCount());
        System.out.println(stats.getSum());
        System.out.println(stats.getMin());
        System.out.println(stats.getMax());
        System.out.println(stats.getAverage());
    }

    private static void minmax() {
        Integer max1 = Stream.of(1, 2, 3).collect(Collectors.maxBy((i1, i2) -> i1.compareTo(i2))).get();
        System.out.println("max1 = " + max1);

        Integer max2 = Stream.of(1, 2, 3).max((i1, i2) -> i1.compareTo(i2)).get();
        System.out.println("max2 = " + max2);

        Integer max3 = Stream.of(1, 2, 3).max(Integer::compareTo).get();
        System.out.println("max3 = " + max3);

        int max4 = IntStream.of(1, 2, 3).max().getAsInt();
        System.out.println("max4 = " + max4);
    }

    private static void group() {
        List<String> names = List.of("Apple", "Avocado", "Banana", "Blueberry", "Cherry");
        Map<String, List<String>> grouped = names.stream().collect(Collectors.groupingBy(name -> name.substring(0, 1)));
        System.out.println("grouped = " + grouped);
    }

    private static void map() {
        Map<String, Integer> map1 = Stream.of("Apple", "Banana", "Tomato")
                .collect(
                        Collectors.toMap(
                                name -> name,
                                name -> name.length()));
        System.out.println("map1 = " + map1);

        // 키 중복 시 런타임 에러 발생
//        Map<String, Integer> map2 = Stream.of("Apple", "Apple", "Tomato")
//                .collect(
//                        Collectors.toMap(
//                                name -> name,
//                                name -> name.length()));
//        System.out.println("map2 = " + map2);

        Map<String, Integer> map3 = Stream.of("Apple", "Apple", "Tomato")
                .collect(
                        Collectors.toMap(
                                name -> name,
                                name -> name.length(),
                                (oldVal, newVal) -> oldVal + newVal));
        System.out.println("map3 = " + map3);

        LinkedHashMap<String, Integer> map4 = Stream.of("Apple", "Apple", "Tomato")
                .collect(
                        Collectors.toMap(
                                name -> name,
                                String::length,
                                (oldVal, newVal) -> oldVal + newVal,
                                LinkedHashMap::new));
        System.out.println("map4 = " + map4);
    }

    private static void basic() {
        List<String> list = Stream.of("Java", "Spring", "JPA").collect(Collectors.toList());
        System.out.println("list = " + list);

        List<Integer> unmodifiableList = Stream.of(1, 2, 3).collect(Collectors.toUnmodifiableList());
        // 리스트 수정 시 런타임 에러 발생
        // unmodifiableList.add(4);
        System.out.println("unmodifiableList = " + unmodifiableList);

        Set<Integer> set = Stream.of(1, 2, 2, 3, 3, 3).collect(Collectors.toSet());
        System.out.println("set = " + set);

        TreeSet<Integer> treeSet = Stream.of(3, 4, 5, 2, 1).collect(Collectors.toCollection(TreeSet::new));
        System.out.println("treeSet = " + treeSet);
    }
}
```

## 4. 스트림 API - 다운스트림 컬렉터

groupingBy(...)를 사용하면 그룹 별로 요소들을 묶을 수 있다. 이 때, 다운스트림 컬렉터를 사용하면 그룹 내부를 다시 모으거나 집계할 수 있다.   
다운스트림 컬렉터는 그룹화된 이후 각 그룹 내부에서 추가적인 연산 또는 결과물(예: 평균, 합계, 최댓값, 최솟값, 통계, 다른 타입으로 변환 등)을 정의한다.   
다운스트림 컬렉터는 Collectors.groupingBy(...) 또는 Collectors.partitioningBy(...)에서 두 번째 인자로 전달되는 Collector 이다.   
다운스트림 컬렉터를 명시하지 않으면 Colelctors.toList()가 적용되어 그룹 별 요소들을 List로 모은다.   

#### 다운스트림 컬렉터의 종류

|Collector|사용 메서드 예시|설명|예시 반환 타입|
|---|---|---|---|
|counting()|Collectors.counting()|그룹 내 요소들의 개수를 센다.|Long|
|summingInt() 등|Collectors.summingInt(...), Collectors.averagingDouble(...)|그룹 내 요소들의 특정 속성 평균 값을 구한다.|Double|
|minBy(), maxBy()|Collectors.minBy(Comparator), Collectors.maxBy(Comparator)|그룹 내 최소, 최댓 값을 구한다.|Optional<T>|
|summarizingInt() 등|Collectors.summarizingInt(...), Collectors.summarizingLong(...)|개수, 합계, 평균, 최소, 최댓값을 동시에 구할 수 있는 SummaryStatistics 객체를 반환한다.|IntSummaryStatistics 등|
|mapping()|Collectors.mapping(변환 함수, 다운스트림)|각 요소를 다른 값으로 변환한 뒤, 변환된 값들을 다시 다른 Collector로 수집할 수 있게 한다.|다운스트림 반환 타입에 따라 달라짐|
|collectingAndThen()|Collectors.collectingAndThen(다른 컬렉터, 변환 함수)|다운스트림 컬렉터의 결과를 최종적으로한 번 더 가공(후처리)할 수 있다.|후처리 후의 타입|
|reducing()|Collectors.reducing(초깃 값, 변환 함수, 누적 함수), Collectors.reducing(누적 함수)|스트림의 reduce()와 유사하게, 그룹 내 요소들을 하나로 합치는 로직을 정의할 수 있다.|누적 로직에 따라 달라짐|
|toList(), toSet()|Collectors.toList(), Collectors.toSet()|그룹 내(혹은 스트림 내) 요소를 리스트나 집합으로 수집한다. toCollection(...)으로 구현체 지정 가능|List<T>, Set<T>|

```java
public class DownStreamMain {

    public static void main(String[] args) {
        List<Student> students = List.of(
                new Student("Kim", 1, 85),
                new Student("Park", 1, 70),
                new Student("Lee", 2, 70),
                new Student("Han", 2, 90),
                new Student("Hoon", 3, 90),
                new Student("Ha", 3, 89)
        );

        downStream1(students);
        downStream2(students);
    }

    private static void downStream2(List<Student> students) {
        // 학년 별로 학생들을 그룹화
        Map<Integer, List<Student>> collect1 = students.stream()
                .collect(Collectors.groupingBy(Student::getGrade));
        System.out.println("collect1 = " + collect1);

        // 학년 별로 가장 점수가 높은 학생(reducing)
        Map<Integer, Optional<Student>> collect2 = students.stream().collect(
                Collectors.groupingBy(
                        Student::getGrade,
                        Collectors.reducing((s1, s2) -> s1.getScore() > s2.getScore() ? s1 : s2)));
        System.out.println("collect2 = " + collect2);

        // 학년 별로 가장 점수가 높은 학생(maxBy)
        Map<Integer, Optional<Student>> collect3 = students.stream().collect(
                Collectors.groupingBy(
                        Student::getGrade,
                        Collectors.maxBy(Comparator.comparingInt(Student::getScore))));
        System.out.println("collect3 = " + collect3);

        // 학년 별로 가장 점수가 높은 학생의 이름(collectingAndThen + maxBy)
        Map<Integer, String> collect4 = students.stream().collect(
                Collectors.groupingBy(
                        Student::getGrade,
                        Collectors.collectingAndThen(
                                Collectors.maxBy(Comparator.comparingInt(Student::getScore)),
                                sOpt -> sOpt.get().getName())
                ));
        System.out.println("collect4 = " + collect4);
    }

    private static void downStream1(List<Student> students) {
        // 학년 별로 학생들을 그룹화
        Map<Integer, List<Student>> collect1 = students.stream().collect(Collectors.groupingBy(Student::getGrade));
        System.out.println("collect1 = " + collect1);

        // 학년 별로 학생들의 이름을 출력
        Map<Integer, List<String>> collect2 = students.stream().collect(
                Collectors.groupingBy(
                        Student::getGrade,
                        Collectors.mapping(
                                Student::getName,
                                Collectors.toList())));
        System.out.println("collect2 = " + collect2);

        // 학년 별로 학생들의 수를 출력
        Map<Integer, Long> collect3 = students.stream().collect(
                Collectors.groupingBy(
                        Student::getGrade,
                        Collectors.counting()));
        System.out.println("collect3 = " + collect3);

        // 학년 별로 학생들의 평균 성적 출력
        Map<Integer, Double> collect4 = students.stream().collect(
                Collectors.groupingBy(
                        Student::getGrade,
                        Collectors.averagingInt(Student::getScore)));
        System.out.println("collect4 = " + collect4);
    }
}
```

#### mapping() vs collectingAndThen()

mapping()은 그룹화된 요소 하나하나를 변환하는데 사용하고, collectingAndThen()은 이미 만들어진 전체 그룹의 결과를 한 번 더 처리해 결과를 만드는데 사용한다.   

|구분|mapping()|collectingAndThen()|
|---|---|---|
|주된 목적|그룹 내 개별 요소를 변환한 뒤, 해당 변환결과를 다른 Collector로 수집|그룹 내 요소들을 이미 한 번 수집한 결과를 추가 가공하거나 최종 타입으로 변환|
|처리 방식|그룹화 → 각 요소를 변환 → List, Set 등으로 수집|그룹화 → 최댓 값/최솟 값/합계 등 수집 → 결과를 후처리(예: Optional → String)|
|대표 예시|mapping(Student::getName, toList())|collectingAndThen(maxBy(...), optional → optional.map(...))