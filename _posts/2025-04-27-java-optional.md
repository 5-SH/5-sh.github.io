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

```Optional<T>```는 존재할 수도 존재하지 않을 수도 있는 값을 감싸는 컨테이너 클래스이다.   
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

1. ```isPresent()```, ```isEmpty()```
    - 값이 있으면 true, 없으면 false 반환

2. ```get()```
    - 값이 있는 경우 그 값을 반환
    - 값이 없으면 NoSuchElementException 발생
    - ```orElse```, ```orElse...``` 메서드를 대신 사용하는 것을 권장

3. ```orElse(T other)```
    - 값이 있으면 그 값을 반환, 값이 없으면 other를 반환

4. ```orElseGet(Supplier<? extends T> supplier)```
    - 값이 있으면 그 값을 반환, 없으면 other를 반환

5. ```orElseThrow(...)```
    - 값이 있으면 그 값을 반환, 값이 없으면 지정한 예외를 던짐

6. ```or(Supplier<? extends Optional<? extends T>> supplier)```
    - 값이 있으면 해당 값의 Optional을 그대로 반환
    - 값이 없으면 supplier가 제공하는 다른 Optional 반환

```java
public class OptionalRetrievalMain {
    
    public static void main(String[] args) {
        Optional<String> optValue = Optional.of("Hello");
        Optional<String> optEmpty = Optional.empty();

        // isPresent() : 값이 있으면 true
        System.out.println("optValue.isPresent() = " + optValue.isPresent());
        System.out.println("optEmpty.isPresent() = " + optEmpty.isPresent());
        // isEmpty() : 값이 없으면 true
        System.out.println("optEmpty.isEmpty() = " + optEmpty.isEmpty());

        // get() : 직접 내부 값을 꺼냄, 값이 없으면 예외 (NoSuchElementException)
        String getValue = optValue.get();
        System.out.println("getValue = " + getValue);
        // String getValue2 = optEmpty.get();  // 예외 발생

        // 값이 있으면 그 값, 없으면 지정된 기본 값 사용
        String value1 = optValue.orElse("기본값");
        String empty1 = optValue.orElse("기본값");
        System.out.println("value1 = " + value1);
        System.out.println("empty1 = " + empty1);

        // 값이 없을 때만 람다(supplier)가 실행되어 기본 값 생성
        String value2 = optValue.orElseGet(() -> {
            System.out.println("람다 호출 - optValue");
            return "New Value";
        });
        String empty2 = optEmpty.orElseGet(() -> {
            System.out.println("람다 호출 - optEmpty");
            return "New Value";
        });
        System.out.println("value2 = " + value2);
        System.out.println("empty2 = " + empty2);

        // 값이 있으면 반환 없으면 예외 발생
        String value3 = optValue.orElseThrow(() -> new RuntimeException("값이 없습니다"));
        System.out.println("value3 = " + value3);

        try {
            String empty3 = optEmpty.orElseThrow(() -> new RuntimeException("값이 없습니다"));
            System.out.println("empty3 = " + empty3);
        } catch (Exception e) {
            System.out.println("예외 발생: " + e.getMessage());
        }

        // Optional을 반환
        Optional<String> result1 = optValue.or(() -> Optional.of("Fallback"));
        System.out.println("result1 = " + result1);
        Optional<String> result2 = optEmpty.or(() -> Optional.of("Fallback"));
        System.out.println("result2 = " + result2);
    }
}
```

### 2-3. Optional 값 처리

Optional에서는 값이 있을 떄와 없을 때를 처리하기 위한 메서드들을 제공한다.   
이를 활용해 null 체크 로직 없이 값을 다룰 수 있다.   

1. ```ifPresent(Consumer<? super T> action)```
    - 값이 존재하면 action 실행, 값이 없으면 아무것도 안 함

2. ```ifPresentOrElse(Consumer<? super T> action, Runnalbe emptyAction)```
    - 값이 존재하면 action 실행, 값이 없으면 emtpyAction 실행

3. ```map(Function<? super T, ? extends U> mapper)```
    - 값이 있으면 mapper를 적용한 결과(```Optional<U>```) 반환
    - 값이 없으면 Optional.empty() 반환

4. ```flatMap(Function<? super T, ? extends Optional<? extends U>> mapper)```
    - map과 유사하지만, Optional을 반환할 때 중첩되지 않고 flat 해서 반환

5. ```filter(Predicate<? super T> predicate) ```
    - 값이 있고 조건을 만족하면 그대로 반환
    - 조건을 만족하지 않거나 비어 있으면 Optional.empty() 반환

6. ```stream()```
    - 값이 있으면 단일 요소를 담은 Stream<T> 반환
    - 값이 없으면 빈 스트림 반환

```java
public class OptionalProcessingMain {

    public static void main(String[] args) {
        Optional<String> optValue = Optional.of("Hello");
        Optional<String> optEmpty = Optional.empty();

        // 값이 존재하면 Consumer 실행, 없으면 아무 일도 하지 않음
        optValue.ifPresent(v -> System.out.println("optValue 값: " + v));
        optEmpty.ifPresent(v -> System.out.println("optEmpty 값: " + v));

        // 값이 있으면 Consumer 실행, 없으면 Runnable 실행
        optValue.ifPresentOrElse(
                v -> System.out.println("optValue 값: " + v),
                () -> System.out.println("optValue는 비어있음")
        );
        optEmpty.ifPresentOrElse(
                v -> System.out.println("optEmpty 값: " + v),
                () -> System.out.println("optEmpty 비어있음")
        );

        // 값이 있으면 Function 적용 후 Optional로 반환, 없으면 Optional.empty()
        Optional<Integer> lengthOpt1 = optValue.map(String::length);
        System.out.println("optValue.map(String::length) = " + lengthOpt1);
        Optional<Integer> lengthOpt2 = optEmpty.map(String::length);
        System.out.println("optEmpty.map(String::length) = " + lengthOpt2);
        
        // map()과 비슷하지만 Optional을 반환하는 경우 중첩을 제거
        Optional<Optional<String>> nestedOpt = optValue.map(v -> Optional.of(v));
        System.out.println("optValue = " + optValue);
        System.out.println("nestedOpt = " + nestedOpt);
        Optional<String> flattenedOpt = optValue.flatMap(s -> Optional.of(s));
        System.out.println("optValue = " + optValue);
        System.out.println("flattenedOpt = " + flattenedOpt);

        // 값이 있고 조건을 만족하면 그 값을 그대로, 불만족 시 Optional.empty()
        Optional<String> filtered1 = optValue.filter(s -> s.startsWith("H"));
        Optional<String> filtered2 = optValue.filter(s -> s.startsWith("X"));
        System.out.println("filter(H) = " + filtered1);
        System.out.println("filter(X) = " + filtered2);

        // 값이 있으면 단일 요소 스트림, 없으면 빈 스트림
        optValue.stream().forEach(s -> System.out.println("optValue.stream() = " + s));
        optEmpty.stream().forEach(s -> System.out.println("optEmpty.stream() = " + s));
    }
}
```

## 3. orElse()와 orElseGet() 차이

### 3-1. 즉시 평가와 지연 평가

#### 즉시 평가

연산을 정의하는 시점에 연산을 실행한다.    
아래 코드에서 logger의 debug 모드를 비활성화 해서 "value100() + value200()" 연산을 할 필요가 없지만 실행된 것을 확인할 수 있다.   

```java
public class Logger {

    private boolean isDebug = false;

    public boolean isDebug() {
        return isDebug;
    }
    public void setDebug(boolean debug) {
        isDebug = debug;
    }
    // DEBUG로 설정한 경우만 출력 - 데이터를 받음
    public void debug(Object message) {
        if (isDebug) {
            System.out.println("[DEBUG] " + message);
        }
    }
}

public class LogMain1 {

    public static void main(String[] args) {
        Logger logger = new Logger();
        logger.setDebug(true);
        logger.debug(value100() + value200());

        System.out.println("=== 디버그 모드 끄기 ===");
        logger.setDebug(false);
        logger.debug(value100() + value200());
    }

    private static int value200() {
        System.out.println("value200 호출");
        return 200;
    }

    private static int value100() {
        System.out.println("value100 호출");
        return 100;
    }
}

/**
value100 호출
value200 호출
[DEBUG] 300
=== 디버그 모드 끄기 ===
value100 호출
value200 호출
*/
```

#### 지연 평가
람다 함수를 사용하면 연산을 정의하는 시점과 연산을 실행하는 시점을 분리할 수 있다.   
람다 함수를 사용하면 연산이 필요할 때 평가를 할 수 있다.   


```java
public class Logger {

    ...

    // 추가
    public void debug(Supplier<?> supplier) {
        if (isDebug) {
            System.out.println("[DEBUG] " + supplier.get());
        }
    }
}

public class LogMain3 {

    public static void main(String[] args) {
        Logger logger = new Logger();
        logger.setDebug(true);
        logger.debug(() -> value100() + value200());

        System.out.println("=== 디버그 모드 끄기 ===");
        logger.setDebug(false);
        logger.debug(() -> value100() + value200());
    }

    private static int value200() {
        System.out.println("value200 호출");
        return 200;
    }

    private static int value100() {
        System.out.println("value100 호출");
        return 100;
    }
}
```

### 3-2. orElse() vs orElseGet()

```java
public class OrElseGetMain {

    public static void main(String[] args) {
        Optional<Integer> optValue = Optional.of(100);
        Optional<Integer> optEmpty = Optional.empty();

        System.out.println("단순 계산");
        Integer i1 = optValue.orElse(10 + 20); // 10 + 20 계산 후 버림
        Integer i2 = optEmpty.orElse(10 + 20); // 10 + 20 계산 후 사용
        System.out.println("i1 = " + i1);
        System.out.println("i2 = " + i2);

        // 값이 있으면 그 값, 없으면 지정된 기본값 사용
        System.out.println("=== orElse ===");
        System.out.println("값이 있는 경우");
        Integer value1 = optValue.orElse(createData());
        System.out.println("value1 = " + value1);
        System.out.println("값이 없는 경우");
        Integer empty1 = optEmpty.orElse(createData());
        System.out.println("empty1 = " + empty1);

        // 값이 있으면 그 값, 없으면 지정된 람다 사용
        System.out.println("=== orElseGet ===");
        System.out.println("값이 있는 경우");
        Integer value2 = optValue.orElseGet(() -> createData());
        System.out.println("value2 = " + value2);
        System.out.println("값이 없는 경우");
        Integer empty2 = optEmpty.orElseGet(() -> createData());
        System.out.println("empty2 = " + empty2);
    }
    public static int createData() {
        System.out.println("데이터를 생성합니다...");
        try {
            Thread.sleep(3000);
        }
        catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
        int createValue = new Random().nextInt(100);
        System.out.println("데이터 생성이 완료되었습니다. 생성 값: " + createValue);
        return createValue;
    }
}

/**
단순 계산
i1 = 100
i2 = 30
=== orElse ===
값이 있는 경우
데이터를 생성합니다...
데이터 생성이 완료되었습니다. 생성 값: 72
value1 = 100
값이 없는 경우
데이터를 생성합니다...
데이터 생성이 완료되었습니다. 생성 값: 6
empty1 = 6
=== orElseGet ===
값이 있는 경우
value2 = 100
값이 없는 경우
데이터를 생성합니다...
데이터 생성이 완료되었습니다. 생성 값: 10
empty2 = 10 
*/
```

## 4. Optional 베스트 프랙티스

### 4-1. 반환 타입으로만 사용하고, 필드에는 가급적 쓰지 말기

Optional은 주로 메서드의 반환 값에 대해 "값이 없을 수도 있음"을 표현하기 위해 도입되었다.   
클래스의 필드에 Optional을 직접 두는 것을 권장하지 않는다.   

```java
public class Product {
    // 안티 패턴: 필드를 Optional로 선언
    private Optional<String> name; 
    
    // ... constructor, getter, etc.
}
```

위 코드에서 name 필드는 null, ```Optional.empty()```, ```Optional.of(value)```를 가질 수 있다.   
name에 null이 할당되어 사용되면 NullPointerException이 발생할 수 있어 혼란이 가중된다.   
그래서 필드는 Optional을 사용하지 않는다.    

### 4-2. 메서드 매개변수로 Optional을 사용하지 말기

자바 공식 문서에 Optional은 메서드의 반환 값으로 사용하기를 권장하고 매개변수로 사용하지 말라고 명시되어 있다.   
메서드를 호출하는 쪽에서 null 전달 대신 ```Optional.empty()```를 전달해야 하는 부담이 생긴다.   
이 경우에 null을 사용하든 ```Optional.empty()```를 사용하든 차이가 없어 가독성만 떨어진다.   

```java
public void processOrder(Optional<Long> orderId) {
    if (orderId.isPresent()) {
        System.out.println("Order ID: " + orderId.get());
    } else {
        System.out.println("Order ID is empty!");
    }
}
```

위 코드의 함수를 호출할 때 processOrder(orderId.empty()) 처럼 호출해야 하는데,   
processOrder 함수 내에선 똑같이 조건문으로 null이 들어왔는지 확인해야 해서 processOrder(null)과 차이가 없다.   

### 4-3. 컬렉션(Collection)이나 배열 타입을 Optional로 감싸지 말기

```List<T>```, ```Set<T>```, ```Map<K, V>``` 등 컬렉션 자체는 비어있는 상태를 표현할 수 있다.   
따라서 ```Optional<List<T>>``` 처럼 다시 감싸면 ```Optional.empty()```와 ```Collections.emptyList()```가 이중 표현이 되어 혼란을 야기한다.   

```java
public Optional<List<String>> getUserRoles(String userId) {
    List<String> userRolesList ...;
    if (foundUser) {
        return Optional.of(userRolesList);
    } else {
        return Optional.empty();
    }
 }

Optional<List<String>> optList = getUserRoles("someUser");
if (optList.isPresent()) {
    // ...
}
```

위 코드에서 Optional이 비어있는지 체크해야 하고 userRolesList가 비어있는지 추가로 체크해야 한다.   

### 4-4. isPresent()와 get() 조합을 직접 사용하지 않기

Optional의 ```get()```메서드는 가급적 사용하지 않아야 한다.    
```if (opt.isPresent()) { ... opt.get() ..} else { ... }```는 사실상 null 체크와 다를게 없고 NoSucheElementException이 발생할 수 있다.   
```orElse```, ```orElseGet```, ```orElseThrow```, ```ifPresentOfElse```, ```map```, ```filter```등의 메서드를 대신 활용해 간결하고 안전하게 처리할 수 있다.   

```java
public static void main(String[] args) {
    Optional<String> optStr = Optional.ofNullable("Hello");
 
    if (optStr.isPresent()) {
        System.out.println(optStr.get());
    } else {
        System.out.println("Nothing");
    }
}
```

위 코드와 같이 Optional을 사용하기 보단 아래 코드와 같이 사용하는 것을 권장한다.   

```java
public static void main(String[] args) {
    Optional<String> optStr = Optional.ofNullable("Hello");
    // 1) orElse
    System.out.println(optStr.orElse("Nothing"));
    // 2) ifPresentOrElse
    optStr.ifPresentOrElse(
        System.out::println,
        () -> System.out.println("Nothing")
    );
    // 3) map
    int length = optStr.map(String::length).orElse(0);
    System.out.println("Length: " + length);
}
```

### 4-5. orElseGet(), orElse() 차이를 이해하기 

```orElse(T other)```는 항상 other를 직시 생성하거나 계산하고(즉시 평가) ```orElseGet(Supplier<? extends T>)```는 필요할 때만 Supplier를 호출한다(지연 평가).   
비용이 크지 않은 대체 값이라면 orElse()를 사용해도 되지만, 복잡하고 비용이 큰 객체 생성이 필요한 경우 orElseGet()를 사용한다.   

### 4-6. 무조건 Optional이 좋은 것은 아니다

Optional을 무분별하게 사용하면 코드의 복잡성을 높인다.   
항상 값이 있는 상황이나 값이 없을 때 예외를 던지는 것이 더 자연스러운 상황에선 Optional을 사용하지 않는 것이 낫다.

## 5. Optional 기본형 타입 지원

```OptionalInt```, ```OptionalLong```, ```OptionalDouble```과 같은 기본현 타입의 Optional을 지원한다.   
Optional은 래퍼 객체를 생성하기 때문에 아주 약간의 성능 상의 손해가 생긴다.   
예외적으로 미세한 성능을 극도로 추구하는 경우 Optional 기본형 타입을 사용하면 성능 상의 손해를 줄일 수 있다.   
그러나 일반적인 상황에서는 주로 Optional을 사용한다.   