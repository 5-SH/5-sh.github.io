---
layout: post
title: Java - 함수형 프로그래밍
date: 2025-04-25 21:00:00 + 0900
categories: [java]
tags: [lambda, stream, optional, functional programming]
---
### 강의 : [김영한의 실전 자바 - 고급 3편, 람다, 스트림, 함수형 프로그래밍](https://www.inflearn.com/course/%EA%B9%80%EC%98%81%ED%95%9C%EC%9D%98-%EC%8B%A4%EC%A0%84-%EC%9E%90%EB%B0%94-%EA%B3%A0%EA%B8%89-3/dashboard)

## 1. 디폴트 메서드

### 1-1. 디폴트 메서드 소개

자바는 처음부터 인터페이스와 구현을 명확하게 분리한 언어였다. 인터페이스는 구현없이 메서드의 시그니처만을 정의하는 용도로 사용 되었다.   

- 인터페이스 목적: 코드의 계약을 정의하고 클래스가 어떤 메서드를 반드시 구현하도록 강제해 명세와 구현을 분리하는 것이 주된 목적
- 엄격한 규칙: 인터페이스에 선언되는 메서드는 기본적으로 모두 추상 메서드 였으며, 인터페이스 내에서 구현 내용을 포함할 수 없었다. ```static final``` 필드와 ```abstract``` 메서드 선언만 가능했다.
- 결과: 클래스는 여러 인터페이스를 구현하며 객체지향적인 설계와 다형성을 극대화할 수 있었다.   

자바 8 이전까지는 인터페이스에 새로운 메서드를 추가하면, 인터페이스를 구현한 모든 클래스에서 그 메서드를 구현해야 했다.   
예를 들어 자바가 버전 업을 하면서 ```Collection```, ```List``` 같은 인터페이스에 새로운 기능을 추가했고 개발자가 자바의 버전을 업그레이드 하는 순간 컴파일 오류들이 발생할 것이다.   
이런 문제를 방지하기 위해 자바는 하위호환성을 가장 높은 우선순위에 두고 자바 8에서 디폴트 메서드가 도입되었다.    

#### 디폴트 메서드의 도입 이유

- 하위 호환성 보장: 인터페이스에 새로운 메서드를 추가해도 기존 코드가 깨지지 않도록 하기 위한 목적
- 라이브러리 확장성: 자바가 제공하는 표준 라이브러리에 정의된 인터페이스에 새 메서드를 추가해도 사용자들이나 서드파티 라이브러리 구현체가 일일이 수정하지 않아도 되도록 만듬, 예를 들어 ```List``` 인터페이스에 ```sort(...)``` 메서드가 추가 되었지만 기존의 모든 ```List``` 구현체를 수정하지 않아도 됨
- 람다와 스트림 API 연계: 자바 8에서 함께 도입된 람다와 스트림 API를 보다 편리하게 활용하기 위해 인터페이스에서 구현 로직을 제공, ```Collection``` 인터페이스에 ```stream()``` 디폴트 메서드 추가, ```Iterable``` 인터페이스에 ```forEach``` 디폴트 메서드 추가   
- 설계 유연성 확장: 디폴트 메서드를 통해 인터페이스에서도 일부 공통 동작 방식을 정의할 수 있다.

```java
public interface MyInterface {
    void existingMethod();

    default void newMethod() {
        System.out.println("새로 추가된 디폴트 메서드입니다.");
    }
}
```

### 1-2. 디폴트 메서드의 올바른 사용법

1. 하위 호환성을 위해 최소한으로 사용
2. 인터페이스는 여전히 추상화의 역할
    - 디폴트 메서드는 하위 호환을 위한 기능이므로 공통으로 쓰기 쉬운 간단한 로직을 제공하는 정도가 이상적이다.
    - 로직 구현은 별도 클래스에 두고 인터페이스는 계약의 역할에 충실한 것이 좋다.
3. 다중 상속 시 충돌 문제
    - 하나의 클래스가 여러 인터페이스를 동시에 구현할 때, 서로 다른 인터페이스에 동일한 시그니처의 디폴트 메서드가 존재하면 충돌이 일어난다.   
    - 이 경우 구현 클래스에서 반드시 메서드를 재정의해야 하고 직접 구현 로직을 작성하거나 어떤 인터페이스의 디폴트 메서드를 쓸 것인지 명시해야 한다.   
4. 디폴트 메서드에 상태를 두지 않기
    - 인스턴스 변수를 활용하거나 여러 차례 호출 시 상태에 따라 동작이 달라지는 등의 동작은 지양해야 한다. 
    - 이런 로직은 클래스로 옮기는 것이 더 적절하다.

## 2. 병렬 스트림

### 2-1. 단일 스트림

아래 ```HeavyJob```클래스는 오래 걸리는 작업을 시뮬레이션 하는데, 각 작업은 1초 정도 소요된다고 가정한다.    
입력 값에 10을 곱한 결과를 반환하며, 작업이 실행될 때 마다 로그를 출력한다.   

```java
public class HeavyJob {

    public static int heavyTask(int i) {
        MyLogger.log("calculate " + i + " -> " + i * 10);
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
        return i * 10;
    }

    public static int heavyTask(int i, String name) {
        MyLogger.log("[" + name + "] " + i + " -> " + i * 10);
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
        return i * 10;
    }
}
```

이 작업을 단일 스트림으로 처리하면 1초 걸리는 작업을 8번 순차로 호출하므로 약 8초가 걸린다.   

```java
public class ParallelMain1 {
    public static void main(String[] args) {
        long startTime = System.currentTimeMillis();

        int sum = IntStream.rangeClosed(1, 8)
                .map(HeavyJob::heavyTask)
                .reduce(0, (a, b) -> a + b);

        long endTime = System.currentTimeMillis();
        log("time: " + (endTime - startTime) + "ms, sum: " + sum);
    }
}
```

```
22:50:46.520 [     main] calculate 1 -> 10
22:50:47.527 [     main] calculate 2 -> 20
22:50:48.536 [     main] calculate 3 -> 30
22:50:49.537 [     main] calculate 4 -> 40
22:50:50.554 [     main] calculate 5 -> 50
22:50:51.556 [     main] calculate 6 -> 60
22:50:52.571 [     main] calculate 7 -> 70
22:50:53.572 [     main] calculate 8 -> 80
22:50:54.591 [     main] time: 8085ms, sum: 360
```

### 2-2. 스레드 풀 사용

아래와 같이 작업을 두 개로 분할해 스레드 풀에서 실행하고 Future로 각각 결과를 받아 합친다.   
2개의 스레드가 병렬로 연산을 처리해 약 4초가 걸린다.   

```java
public class ParallelMain3 {

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        ExecutorService es = Executors.newFixedThreadPool(2);

        long startTime = System.currentTimeMillis();

        // 1. 작업을 분할한다.
        SumTask task1 = new SumTask(1, 4);
        SumTask task2 = new SumTask(5, 8);

        // 2. 분할한 작업을 처리한다.
        Future<Integer> future1 = es.submit(task1);
        Future<Integer> future2 = es.submit(task2);

        // 3. join - 처리한 결과를 합친다. get: 결과가 나올 때 까지 대기한다.
        Integer result1 = future1.get();
        Integer result2 = future2.get();

        int sum = result1 + result2;
        long endTime = System.currentTimeMillis();
        log("time: " + (endTime - startTime) + "ms, sum: " + sum);

        es.close();
    }

    static class SumTask implements Callable<Integer> {
        int startValue;
        int endValue;

        public SumTask(int startValue, int endValue) {
            this.startValue = startValue;
            this.endValue = endValue;
        }

        @Override
        public Integer call() {
            log("작업 시작");
            int sum = 0;
            for (int i = startValue; i <= endValue; i++) {
                int calculated = HeavyJob.heavyTask(i);
                sum += calculated;
            }

            log("작업 완료 result=" + sum);

            return sum;
        }
    }
}
```

```
22:55:21.915 [pool-1-thread-2] 작업 시작
22:55:21.915 [pool-1-thread-1] 작업 시작
22:55:21.930 [pool-1-thread-1] calculate 1 -> 10
22:55:21.930 [pool-1-thread-2] calculate 5 -> 50
22:55:22.935 [pool-1-thread-1] calculate 2 -> 20
22:55:22.935 [pool-1-thread-2] calculate 6 -> 60
22:55:23.937 [pool-1-thread-1] calculate 3 -> 30
22:55:23.937 [pool-1-thread-2] calculate 7 -> 70
22:55:24.953 [pool-1-thread-1] calculate 4 -> 40
22:55:24.953 [pool-1-thread-2] calculate 8 -> 80
22:55:25.955 [pool-1-thread-1] 작업 완료 result=100
22:55:25.955 [pool-1-thread-2] 작업 완료 result=260
22:55:25.955 [     main] time: 4069ms, sum: 360
```

### 2-3. Fork/Join 패턴

스레드는 한 번에 하나의 작업을 처리할 수 있다. 효율적으로 처리하기 위해 하나의 큰 작업을 여러 스레드가 처리할 수 있는 작은 단위의 작업으로 분할하고 각각의 스레드가 처리한 뒤 분할된 결과를 하나로 모아야 한다.   
이렇게 분할(Fork) → 처리(Execute) → 모음(Join)의 단계로 이뤄진 멀티스레딩 패턴을 Fork/Join 패턴이라고 한다.   

### 2-4. Fork/Join 프레임워크

#### 2-4-1. 소개

##### 분할 정복 전략

- 큰 작업(task)을 작은 단위로 재귀적으로 분할(fork)
- 각 작은 작업의 결과를 합쳐(join) 최종 결과를 생성
- 멀티코어 환경에서 작업을 효율적으로 분산 처리

##### 작업 훔치기 알고리즘

- 각 스레드는 자신 만의 작업 큐를 가짐
- 작업이 없는 스레드는 다른 바쁜 스레드의 큐에서 작업을 훔쳐와서 대신 처리
- 부하 균현을 자동으로 조절해 효율성 향상

#### 2-4-2. 주요 클래스

##### ForkJoinPool

- Fork/Join 작업을 실행하는 특수한 ```ExecutorService``` 스레드 풀
- 작업 스케줄링 및 스레드 관리를 담당
- 기본적으로 사용 가능한 프로세서 수 만큼 스레드 생성

##### ForkJoinTask

- ```ForkJoinTask```는 Fork/Join 작업의 기본 추상 클래스이다.
- 개발자는 결과를 반환하는 작업인 RecursiveTask<V> 또는 결과를 반환하지 않는 작업인 RecursiveAction 클래스를 구현해 사용한다.   
- RecursiveTask/RecursiveAction은 compute() 메서드를 재정의해서 필요한 작업 로직을 작성한다.
- 보통 작업 범위가 작으면 직접 처리하고 크면 둘로 분할해 각각 병렬 처리하도록 구현한다.

##### fork()/join() 메서드

- fork(): 현재 스레드에서 다른 스레드로 작업을 분할해 보내는 동작(비동기 실행)
- join(): 분할된 작업이 끝날 때까지 기다린 후 결과를 가져오는 동작

#### 2-4-3. Fork/Join 프레임워크 활용

작업할 배열을 받아 ```THRESHOLD``` 만큼 분할하고 연산한 뒤 결과를 모으는 SumTask 클래스를 만들었다.    
SumTask 클래스는 ```RecursiveTask<Integer>``` 인터페이스의 ```compute()``` 메서드를 구현했다.   

```java
public class SumTask extends RecursiveTask<Integer> {
    private static final int THRESHOLD = 2;

    private final List<Integer> list;

    public SumTask(List<Integer> list) {
        this.list = list;
    }

    @Override
    protected Integer compute() {
        if (list.size() <= THRESHOLD) {
            log("[처리 시작] " + list);
            int sum = list.stream()
                    .mapToInt(HeavyJob::heavyTask)
                    .sum();
            log("[처리 완료] " + list + " -> sum: " + sum);
            return sum;
        } else {
            // 작업 범위가 크면 반으로 나누어 병렬 처리
            int mid = list.size() / 2;
            List<Integer> leftList = list.subList(0, mid);
            List<Integer> rightList = list.subList(mid, list.size());
            log("[분할] " + list + " -> LEFT" + leftList + ", RIGHT" + rightList);

            SumTask leftTask = new SumTask(leftList);
            SumTask rightTask = new SumTask(rightList);

            // 왼쪽 작업은 다른 스레드에서 처리
            leftTask.fork();
            // 오른쪽 작업은 현재 스레드에서 처리
            int rightResult = rightTask.compute();

            // 왼쪽 작업 결과를 기다림
            int leftResult = leftTask.join();

            // 왼쪽과 오른쪽 결과를 합침
            int joinSum = leftResult + rightResult;
            log("LEFT[" + leftResult + "] + RIGHT[" + rightResult + "] -> sum: " + joinSum);

            return joinSum;
        }

    }
}
```

```THRESHOLD``` 값을 2로 줄여 실행하면 결과는 아래와 같다.   

```java
public class ForkJoinMain1 {

    public static void main(String[] args) {
        List<Integer> data = IntStream.rangeClosed(1, 8).boxed().toList();

        log("[생성] " + data);

        // ForkJoinPool 생성 및 작업 수행
        ForkJoinPool pool = new ForkJoinPool(10);

        long startTime = System.currentTimeMillis();
        SumTask task = new SumTask(data);

        int result = pool.invoke(task);
        pool.close();
        long endTime = System.currentTimeMillis();
        log("time: " + (endTime - startTime) + "ms, sum: " + result);
        log("pool: " + pool);
    }
}
```

```
23:12:18.472 [     main] [생성] [1, 2, 3, 4, 5, 6, 7, 8]
23:12:18.475 [ForkJoinPool-1-worker-1] [분할] [1, 2, 3, 4, 5, 6, 7, 8] -> LEFT[1, 2, 3, 4], RIGHT[5, 6, 7, 8]
23:12:18.475 [ForkJoinPool-1-worker-1] [분할] [5, 6, 7, 8] -> LEFT[5, 6], RIGHT[7, 8]
23:12:18.475 [ForkJoinPool-1-worker-2] [분할] [1, 2, 3, 4] -> LEFT[1, 2], RIGHT[3, 4]
23:12:18.475 [ForkJoinPool-1-worker-1] [처리 시작] [7, 8]
23:12:18.475 [ForkJoinPool-1-worker-2] [처리 시작] [3, 4]
23:12:18.475 [ForkJoinPool-1-worker-4] [처리 시작] [1, 2]
23:12:18.475 [ForkJoinPool-1-worker-3] [처리 시작] [5, 6]
23:12:18.500 [ForkJoinPool-1-worker-1] calculate 7 -> 70
23:12:18.500 [ForkJoinPool-1-worker-3] calculate 5 -> 50
23:12:18.500 [ForkJoinPool-1-worker-4] calculate 1 -> 10
23:12:18.500 [ForkJoinPool-1-worker-2] calculate 3 -> 30
23:12:19.503 [ForkJoinPool-1-worker-2] calculate 4 -> 40
23:12:19.503 [ForkJoinPool-1-worker-3] calculate 6 -> 60
23:12:19.503 [ForkJoinPool-1-worker-1] calculate 8 -> 80
23:12:19.503 [ForkJoinPool-1-worker-4] calculate 2 -> 20
23:12:20.513 [ForkJoinPool-1-worker-1] [처리 완료] [7, 8] -> sum: 150
23:12:20.513 [ForkJoinPool-1-worker-3] [처리 완료] [5, 6] -> sum: 110
23:12:20.513 [ForkJoinPool-1-worker-4] [처리 완료] [1, 2] -> sum: 30
23:12:20.513 [ForkJoinPool-1-worker-2] [처리 완료] [3, 4] -> sum: 70
23:12:20.513 [ForkJoinPool-1-worker-1] LEFT[110] + RIGHT[150] -> sum: 260
23:12:20.513 [ForkJoinPool-1-worker-2] LEFT[30] + RIGHT[70] -> sum: 100
23:12:20.513 [ForkJoinPool-1-worker-1] LEFT[100] + RIGHT[260] -> sum: 360
23:12:20.513 [     main] time: 2038ms, sum: 360
23:12:20.513 [     main] pool: java.util.concurrent.ForkJoinPool@7daf6ecc[Terminated, parallelism = 10, size = 0, active = 0, running = 0, steals = 4, tasks = 0, submissions = 0]
```

8개의 작업을 4개로 나누고 멀티 스레드를 사용해 병렬로 실행해서 전체 연산에 약 2초가 걸렸다.   
ForkJoinPool의 각 스레드는 자신이 처리하지 않는 작업을 작업 큐에 두면 다른 스레드가 작업 큐에 대기 중인 작업을 훔쳐서 대신 처리한다.   
작업 훔치기 알고리즘을 통해 작업은 worker-1, 2, 3, 4 스레드에 균등하게 분배된다.   

#### 2-4-4. Fork/Join 공용 풀(Common Pool)

자바 8에서 도입된 공용 풀은 Fork/Join 작업을 위해 자바가 제공하는 기본 스레드 풀이다.   

##### Fork/Join 공용 풀의 특징

- 시스템 전체에서 공유: 애플리케이션 내에서 단일 인스턴스로 공유되어 사용된다.
- 자동 생성: 별도로 생성하지 않아도 ```ForkJoinPool.commonPool()```을 통해 접근할 수 있다.
- 편리한 사용: 별도의 풀을 만들지 않고도 ```RecursiveTask```/```RecursiveAction```을 사용할 때 기본적으로 이 공용 풀이 사용된다.
- 병렬 스트림 사용: 자바 8의 병렬 스트림은 내부적으로 이 공용 풀을 사용한다.
- 병렬 수준 자동 설정: 시스템의 가용 프로세서 수에서 1을 뺀 값으로 병렬 수준이 설정된다.   

```ForkJoinPool```을 생성하지 않고 ```task.invoke()```를 통해 Fork/Join 공용 풀을 사용했다.   
공용 풀은 JVM이 종료될 때까지 계속 유지되므로 별도로 풀을 조욜하지 않아도 된다.   

```java
public class ForkJoinMain2 {

    public static void main(String[] args) {
        int processorCount = Runtime.getRuntime().availableProcessors();
        ForkJoinPool commonPool = ForkJoinPool.commonPool();
        log("processorCount = " + processorCount + ", commonPool = " + commonPool.getParallelism());

        List<Integer> data = IntStream.rangeClosed(1, 8).boxed().toList();

        log("[생성] " + data);
        SumTask task = new SumTask(data);
        Integer result = task.invoke();
        log("최종 결과: " + result);
    }
}
```

### 2-5. 자바 병렬 스트림