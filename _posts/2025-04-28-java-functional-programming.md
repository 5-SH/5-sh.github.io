---
layout: post
title: Java - 디폴트 메서드 / 병렬 스트림
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

자바 스트림에서 ```parallel()```을 추가하면 여러 스레드가 병렬로 heavyTask를 처리한다.   
직접 스레드나 스레드 풀을 만들 필요 없이 ```parallel()``` 호출만으로 스트림이 자동으로 병렬 처리 되었다.   
```parallel()```을 호출하면 위에서 설명한 공용 ```ForkJoinPool```을 사용하고 내부적으로 ```Spliterator```를 통해 병렬 처리 가능한 스레드 숫자와 작업의 크기 등을 고려해 작업을 자동 분할하고 병렬 처리를 한다.   
```Spliterator```를 통한 분할은 데이터 소스의 특성에 따라 최적화 되어 있다.    
따라서 개발자가 ```parallel()```을 선언하면 병렬처리 방법은 자바 스트림이 내부적으로 알아서 처리한다.   

```java
public class ParallelMain4 {

    public static void main(String[] args) {
        int processorCount = Runtime.getRuntime().availableProcessors();
        ForkJoinPool commonPool = ForkJoinPool.commonPool();
        log("processorCount = " + processorCount + ", commonPool = " + commonPool.getParallelism());

        long startTime = System.currentTimeMillis();

        int sum = IntStream.rangeClosed(1, 8)
                .parallel()
                .map(HeavyJob::heavyTask)
                .sum();

        long endTime = System.currentTimeMillis();
        log("time: " + (endTime - startTime) + "ms, sum: " + sum);
    }
}

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

```
08:04:41.652 [     main] processorCount = 16, commonPool = 15
08:04:41.670 [     main] calculate 6 -> 60
08:04:41.670 [ForkJoinPool.commonPool-worker-1] calculate 3 -> 30
08:04:41.670 [ForkJoinPool.commonPool-worker-3] calculate 4 -> 40
08:04:41.670 [ForkJoinPool.commonPool-worker-2] calculate 2 -> 20
08:04:41.670 [ForkJoinPool.commonPool-worker-5] calculate 1 -> 10
08:04:41.670 [ForkJoinPool.commonPool-worker-4] calculate 8 -> 80
08:04:41.670 [ForkJoinPool.commonPool-worker-6] calculate 7 -> 70
08:04:41.670 [ForkJoinPool.commonPool-worker-7] calculate 5 -> 50
08:04:42.688 [     main] time: 1034ms, sum: 360
```

#### 2-5-1. 병렬 스트림 사용 시 주의점 (1)

Fork/Join 공용 풀은 계산 직약적인 CPU 바운드 작업을 위해 설계 되었다.   
따라서 스레드가 주로 대기해야 하는 I/O 바운드 작업에는 적합하지 않다.   

1. 스레드 블로킹에 따른 CPU 낭비
    - ```ForkJoinPool```은 CPU 코어 수에 맞춰 제한된 개수의 스레드를 사용한다.
    - I/O 작업으로 스레드가 블로킹되면 CPU가 놀게 되어, 전체 병렬 처리 효율이 크게 떨어진다.

2. 컨텍스트 스위칭 오버헤드 증가
    - I/O 작업 때문에 스레드를 늘리면, 실제 연산보다 대기 시간이 길어지는 상황이 발생할 수 있다.
    - 스레드가 많아질 수록 컨텍스트 스위칭 비용도 증가해 오히려 성능이 떨어질 수 있다.

3. 작업 훔치기 기법 무력화
    - ```ForkJoinPool```이 제공하는 작업 훔치기 알고리즘은 CPU 바운드 작업에서 빠르게 작업 단위를 계속 처리 하도록 작업을 훔쳐서 쉬는 스레드 없이 계속 작업하도록 만들어져 있다.
    - I/O 대기 시간이 많은 작업은 스레드가 I/O로 인해 대기하고 있는 경우가 많아 작업 훔치기가 동작하기 어렵고 병렬 처리의 장점을 살리기 어렵다.   

4. 분할-정복 이점 감소
    - Fork/Join으로 작업을 작게 나눠도 I/O 병목이 발생하면 CPU 병렬화 이점이 크게 줄어든다.
    - 오히려 분할된 작업들이 I/O 대기를 반복하면서 ```fork()```, ```join()```에 따른 오버헤드만 증가할 수 있다.   

아래 예제는 여러 사용자가 동시에 서버를 호출하고 각 요청을 병렬 스트림을 사용해 몇 가지 무거운 작업을 처리한다.   
그리고 모든 요청이 동일한 공용 풀(```ForkJoinPool.commonPool```)을 공유하는 예제이다.   
공용 풀의 병렬 수준을 3으로 제한하고 각 요청은 앞선 ```heavyTask```를 1~4 범위로 처리하고 ```parallel()``` 스트림을 사용해 작업을 처리한다.    
```heavyTask()```는 1초간 스레드가 대기하는 작업이므로 I/O 바운드 작업에 가깝다.   

```java
public class ParallelMain5 {

    public static void main(String[] args) throws InterruptedException {

        // 병렬 수준을 3으로 제한
        System.setProperty("java.util.concurrent.ForkJoinPool.common.parallelism", "3");

        ExecutorService requestPool = Executors.newFixedThreadPool(100);

        int nThreads = 3; // 1, 2, 3, 10, 20
        for (int i = 1; i <= nThreads; i++) {
            String requestName = "request" + i;
            requestPool.submit(() -> logic(requestName));
            Thread.sleep(100);
        }

        requestPool.close();
    }

    private static void logic(String requestName) {
        log("[" + requestName + "] START");
        long startTime = System.currentTimeMillis();

        int sum = IntStream.rangeClosed(1, 4)
                .parallel()
                .map(i -> HeavyJob.heavyTask(i, requestName))
                .reduce(0, (a, b) -> a + b);

        long endTime = System.currentTimeMillis();
        log("[" + requestName + "] time: " + (endTime - startTime) + "ms, sum: " + sum);
    }
}
```

3개의 요청 스레드가 작업을 요청하면 request1은 약 1초, request2는 약 2초, request3는 약 3초로 불균형하게 처리된다.   
```nThread```의 숫자를 늘려서 동시 요청을 늘리면 응답시간이 확연하게 늘어난다.   
모든 병렬 스트림이 동일한 공용 풀을 공유하므로 요청이 많아질수록 병목 현상이 발생한다. 제한된 스레드 풀에 여러 요청이 경쟁하게 되어 성능이 저하된다.   
따라서 같은 작업이라도 동시에 실행되는 다른 작업의 수에 따라 처리 시간이 크게 달라지게 된다.   

```
08:35:30.290 [pool-1-thread-1] [request1] START
08:35:30.310 [ForkJoinPool.commonPool-worker-3] [request1] 1 -> 10
08:35:30.310 [ForkJoinPool.commonPool-worker-1] [request1] 2 -> 20
08:35:30.310 [ForkJoinPool.commonPool-worker-2] [request1] 4 -> 40
08:35:30.310 [pool-1-thread-1] [request1] 3 -> 30
08:35:30.390 [pool-1-thread-2] [request2] START
08:35:30.390 [pool-1-thread-2] [request2] 3 -> 30
08:35:30.491 [pool-1-thread-3] [request3] START
08:35:30.491 [pool-1-thread-3] [request3] 3 -> 30
08:35:31.323 [ForkJoinPool.commonPool-worker-1] [request2] 4 -> 40
08:35:31.323 [ForkJoinPool.commonPool-worker-3] [request3] 2 -> 20
08:35:31.323 [ForkJoinPool.commonPool-worker-2] [request2] 2 -> 20
08:35:31.324 [pool-1-thread-1] [request1] time: 1033ms, sum: 100
08:35:31.391 [pool-1-thread-2] [request2] 1 -> 10
08:35:31.493 [pool-1-thread-3] [request3] 4 -> 40
08:35:32.325 [ForkJoinPool.commonPool-worker-3] [request3] 1 -> 10
08:35:32.408 [pool-1-thread-2] [request2] time: 2018ms, sum: 100
08:35:33.326 [pool-1-thread-3] [request3] time: 2835ms, sum: 100
```

```nThread```의 값을 20으로 해서 실행을 하면 Fork/Join 공용 풀에 병목이 생겨 모든 작업이 끝나는데 약 12초가 걸린다.   
```parallel()```을 빼고 실행을 하면 20개의 요청 스레드가 작업을 처리해 약 4초만에 모든 작업이 완료된다.   
따라서 I/O 바운드의 작업은 Fork/Join 공용 풀 보다는 별도의 스레드 풀을 사용하는 것이 좋다.   

##### ※ 주의 실무에서 공용 풀은 절대 I/O 바운드 작업을 하면 안된다.

애플리케이션 전체에서 공용 풀을 공유해서 사용하기 때문에 병목이 생기는 경우 모든 요청이 다 밀리게 된다.    
예를 들어 외부 API를 호출하거나 데이터베이스 결과를 기다리는 경우에 응답이 늦게 온다면 공용 풀의 스레드가 I/O 응답을 대기하게 된다.   
그러면 나머지 모든 요청들이 공용 풀의 스레드를 기다리며 다 밀리게 되는 큰 일이 발생한다.   
따라서 공용 풀은 반드시 CPU 바운드 작업에만 사용하고 I/O 바운드 작업은 ExecutorService 등의 별도의 스레드 풀을 사용해야 한다.

### 2-5-2. 병렬 스트림 사용 시 주의점 (2)

별도의 전용 스레드 풀을 사용해서 I/O 바운드 작업을 처리한다.    
```nThread```의 값을 20으로 실행을 하면 400개의 스레드를 갖는 별도의 스레드 풀에서 I/O 바운드를 처리해 약 1초만에 작업이 끝난다.   

```java
public class ParallelMain6 {

    public static void main(String[] args) throws InterruptedException {
        ExecutorService requestPool = Executors.newFixedThreadPool(100);

        // logic 처리 전용 스레드 풀 추가
        ExecutorService logicPool = Executors.newFixedThreadPool(400);

        int nThreads = 20; // 1, 2, 3, 10, 20
        for (int i = 1; i <= nThreads; i++) {
            String requestName = "request" + i;
            requestPool.submit(() -> logic(requestName, logicPool));
            Thread.sleep(100);
        }

        requestPool.close();
        logicPool.close();
    }

    private static void logic(String requestName, ExecutorService es) {
        log("[" + requestName + "] START");
        long startTime = System.currentTimeMillis();

        List<Future<Integer>> futures = IntStream.rangeClosed(1, 4)
                .mapToObj(i -> es.submit(() -> HeavyJob.heavyTask(i, requestName)))
                .toList();

        int sum = futures.stream()
                .mapToInt(f -> {
                    try {
                        return f.get();
                    } catch (Exception e) {
                        throw new RuntimeException(e);
                    }
                }).sum();

        long endTime = System.currentTimeMillis();
        log("[" + requestName + "] time: " + (endTime - startTime) + "ms, sum: " + sum);

    }
}
```

```
08:55:46.140 [pool-1-thread-1] [request1] START
08:55:46.159 [pool-2-thread-4] [request1] 4 -> 40
08:55:46.159 [pool-2-thread-3] [request1] 3 -> 30
08:55:46.159 [pool-2-thread-2] [request1] 2 -> 20
08:55:46.159 [pool-2-thread-1] [request1] 1 -> 10
08:55:46.226 [pool-1-thread-2] [request2] START
08:55:46.226 [pool-2-thread-5] [request2] 1 -> 10
08:55:46.226 [pool-2-thread-7] [request2] 3 -> 30
08:55:46.226 [pool-2-thread-6] [request2] 2 -> 20
08:55:46.226 [pool-2-thread-8] [request2] 4 -> 40
...
08:55:48.076 [pool-1-thread-10] [request10] time: 1002ms, sum: 100
08:55:48.156 [pool-1-thread-20] [request20] START
08:55:48.157 [pool-2-thread-78] [request20] 2 -> 20
08:55:48.157 [pool-2-thread-79] [request20] 3 -> 30
08:55:48.157 [pool-2-thread-80] [request20] 4 -> 40
08:55:48.157 [pool-2-thread-77] [request20] 1 -> 10
08:55:48.207 [pool-1-thread-11] [request11] time: 1018ms, sum: 100
08:55:48.293 [pool-1-thread-12] [request12] time: 1003ms, sum: 100
08:55:48.410 [pool-1-thread-13] [request13] time: 1003ms, sum: 100
08:55:48.523 [pool-1-thread-14] [request14] time: 1016ms, sum: 100
08:55:48.623 [pool-1-thread-15] [request15] time: 1015ms, sum: 100
08:55:48.723 [pool-1-thread-16] [request16] time: 1015ms, sum: 100
08:55:48.840 [pool-1-thread-17] [request17] time: 1016ms, sum: 100
08:55:48.941 [pool-1-thread-18] [request18] time: 1001ms, sum: 100
08:55:49.043 [pool-1-thread-19] [request19] time: 1002ms, sum: 100
08:55:49.173 [pool-1-thread-20] [request20] time: 1017ms, sum: 100
```
