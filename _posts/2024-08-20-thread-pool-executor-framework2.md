---
layout: post
title: Java 스레드8 - 스레드 풀과 Executor 프레임워크2
date: 2024-08-20 19:00:00 + 0900
categories: [java]
tags: [java, thread, multi-thread, thread pool, executor]
---
### 강의 : [김영한의 실전 자바 - 고급 1편, 멀티스레드와 동시성](https://www.inflearn.com/course/%EA%B9%80%EC%98%81%ED%95%9C%EC%9D%98-%EC%8B%A4%EC%A0%84-%EC%9E%90%EB%B0%94-%EA%B3%A0%EA%B8%89-1/dashboard)

# 13. 스레드 풀과 Executor 프레임워크2

## 13-1. ExecutorService 우아한 종료 - 소개

고객의 주문을 처리하는 서버를 운영 중이라고 가정한다.    
만약 서버 기능을 업데이트하기 위해 서버를 재시작해야 한다면,   
서버가 고객의 주문을 처리하고 있는 도중에 갑자기 재시작 되면 고객의 주문이 제대로 진행되지 못할 것이다.    
이상적인 방법은 새로운 주문 요청은 막고 이미 진행 중인 주문은 모두 완료한 다음에 서버를 재시작하는 것이 가장 좋다.    
이렇게 문제 없이 종료하는 방식을 graceful shutdown 이라고 한다.    

### 13-1-1. ExecutorService의 종료 메서드   

- 서비스 종료
  - void shutdown()
    - 새로운 작업을 받지 않고 이미 제출된 작업을 모두 완료한 후에 종료한다.
    - 논 블로킹 메서드
  - List<Runnable> shutdownNow()
    - 실행 중인 작업을 중단하고, 대기 중인 작업을 반환하며 즉시 종료한다.
    - 실행 중인 작업을 중단하기 위해 인터럽트를 발생시킨다.
    - 논 블로킹 메서드
- 서비스 상태 확인
  - boolean isShutdown()
    - 서비스가 종료 되었는지 확인한다.
  - boolean isTerminated()
    - shutdown(), shutdownNow() 호출 후, 모든 작업이 완료 되었는지 확인한다.
- 작업 완료 대기
  - boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedException
    - 서비스 종료 시 모든 작업이 완료될 때까지 대기한다. 이 때 지정된 시간 까지만 대기한다.
    - 블로킹 메서드
- close()
  - 자바 19부터 지원하는 서비스 종료 메서드이다.
  - shutdown()을 호출하고 하루를 기다려도 작업이 완료되지 않으면 shutdownNow()를 호출해 즉시 종료한다.
   
### 13-1-2. shutdown() - 처리 중인 작업이 없는 경우

- shutdown()을 호출하면 ExecutorService는 새로운 요청을 거절한다.
- 거절 시 RejectedExecutionException 예외가 발생한다.
- 스레드 풀의 자원을 정리한다.
   
### 13-1-3. shutdown() - 처리 중인 작업이 없는 경우

- shutdown()을 호출하면 ExecutorService는 새로운 요청을 거절한다.
- 스레드 풀의 스레드는 처리 중인 작업을 완료한다.
- 스레드 풀의 스레드는 큐에 남아있는 작업도 모두 꺼내서 완료한다.
- 모든 작업을 완료하면 자원을 정리한다.
   
### 13-1-4. shutdownNow() - 처리 중인 작업이 있는 경우

- shutdownNow()를 호출하면 ExecutorService는 새로운 요청을 거절한다.
- 큐를 비우면서 큐에 있는 작업을 모두 꺼내서 컬렉션으로 반환한다.
  - List<Runnable> runnables = es.shutdownNow()
- 작업 중인 스레드에 인터럽트가 발생한다.
- 작업을 완료하면 자원을 정리한다.
   
## 13-2. ExecutorService 우아한 종료 - 구현

갑자기 요청이 너무 많이 들어와서 큐에 대기 중인 작업이 너무 많아 작업 완료가 어렵거나, 작업이 너무 오래 걸리거나 또는 버그가 발생해서 특정 작업이 끝나지 않을 수 있다.    
이렇게 되면 서비스가 너무 늦게 종료 되거나 종료되지 않는 문제가 발생할 수 있다.   
   
이럴 때는 보통 우아하게 종료하는 시간을 정한다. 예를 들어 60초 까지는 기다렸다가 60초가 지나면 shutdownNow()를 호출해서 강제로 작업들을 종료한다.    
close()의 경우 이렇게 구현되어 있지만 하루를 기다리는 단점이 있다.    
ExecutorService 공식 API 문서에는 shutdown()을 통해 우하한 종료를 시도하고 n초간 종료되지 않으면 shutdownNow()를 통해 강제 종료하는 방식을 제안한다.    
아래 코드에서는 10초 동안 작업이 종료되길 기다린다.   

```java
public class RunnableTask implements Runnable {
    private final String name;
    private int sleepMs = 1000;

    public RunnableTask(String name) {
        this.name = name;
    }

    public RunnableTask(String name, int sleepMs) {
        this.name = name;
        this.sleepMs = sleepMs;
    }

    @Override
    public void run() {
        log(name + " 시작");
        sleep(sleepMs);
        log(name + " 완료");
    }
}

public class ExecutorShutdownMain {

    public static void main(String[] args) throws InterruptedException {
        ExecutorService es = Executors.newFixedThreadPool(2);
        es.execute(new RunnableTask("taskA"));
        es.execute(new RunnableTask("taskB"));
        es.execute(new RunnableTask("taskD"));
        es.execute(new RunnableTask("longTask", 100_000));
        printState(es);
        log("== shutdown 시작 ==");
        shutdownAndAwaitTermination(es);
        log("== shutdown 완료 ==");
        printState(es);
    }

    static void shutdownAndAwaitTermination(ExecutorService es) {
        es.shutdown();
        try {
            log("서비스 정상 종료 시도");
            if (!es.awaitTermination(10, TimeUnit.SECONDS)) {
                log("서비스 정상 종료 실패 -> 강제 종료 시도");
                es.shutdownNow();
                if (!es.awaitTermination(10, TimeUnit.SECONDS)) {
                    log("서비스가 종료되지 않았습니다.");
                }
            }
        } catch (InterruptedException ex) {
            es.shutdownNow();
        }
    }
}
```

### 13-2-1. 서비스 종료

```java
es.shutdown()   
```

- 새로운 작업을 받지 않는다. 처리 중이거나 큐에 이미 대기 중인 작업은 처리한 후에 스레드를 종료한다.
- shutdown()은 논 블로킹 메서드라서 main 스레드가 기다리지 않고 다음 코드를 바로 호출한다.
   
```java
if (!es.awaitTermination(10, TimeUnit.SECONDS)) { ... }   
```

- 블로킹 메서드라서 main 스레드가 서비스 종료를 10초간 기다린다.
- 10초 안에 모든 작업이 완료되면 true를 반환한다.
- 코드에선 longTask가 10초가 지나도 완료되지 않기 때문에 false를 반환한다.
   
### 13-2-2. 서비스 정상 종료 실패 → 강제 종료 시도

```java
es.shutdownNow();   
if (!es.awaitTermination(10, TimeUnit.SECONDS)) { ... }    
```

- 정상 종료가 10초 이상 걸려서 shutdownNow()를 통해 강제 종료에 들어간다.
- shutdown()과 마찬가지로 블로킹 메서드가 아니다.
- 강제 종료를 하면 작업 중인 스레드에 인터럽트가 발생하고 로그에서 확인할 수 있다.
   
### 13-2-3. 서비스 종료 실패

마지막 강제 종료인 es.shutdownNow()를 호출한 다음 10초를 기다린다.    
왜냐하면 shutdownNow()가 작업 중인 스레드에 인터럽트를 호출하는 것은 맞지만,    
인터럽트를 호출 하더라도 인터럽트 이후에 자원을 정리하는 작업을 수행하거나 해서 시간이 걸릴 수 있다.     
이런 시간을 기다려 주는 것이다.   

<br/>

극단적이지만 while(true) { ... } 같은 코드로 인해 인터럽트를 받을 수 없을 수 있다.    
이 경우 예외가 발생하지 않고 스레드가 계속 수행 되기 때문에 자바를 강제 종료해야 제거할 수 있다.   
이런 경우를 대비해서 강제 종료 후 10초간 대기해도 작업이 완료되지 않으면 "서비스가 종료되지 않았습니다" 라고 로그를 남긴다.     
개발자는 로그를 통해 문제를 확인하고 수정할 수 있다.   

## 13-3 Executor 스레드 풀 관리 - 코드

```java
public class ExecutorUtils {
    ...
    
    public static void printState(ExecutorService executorService, String taskName) {
        if (executorService instanceof ThreadPoolExecutor poolExecutor) {
            int pool = poolExecutor.getPoolSize();
            int active = poolExecutor.getActiveCount();
            int queued = poolExecutor.getQueue().size();
            long completedTask = poolExecutor.getCompletedTaskCount();
            log(taskName + " -> [pool=" + pool + ", active=" + active + ", queuedTasks=" + queued + ", completedTasks=" + completedTask + "]");
        } else {
            log(taskName + " -> " + executorService);
        }
    }
}

public class PoolSizeMainV1 {

    public static void main(String[] args) throws InterruptedException {
        BlockingQueue<Runnable> workQueue = new ArrayBlockingQueue<>(2);
        ExecutorService es = new ThreadPoolExecutor(2, 4, 3000, TimeUnit.MILLISECONDS, workQueue);
        printState(es);

        es.execute(new RunnableTask("task1"));
        printState(es, "task1");

        es.execute(new RunnableTask("task2"));
        printState(es, "task2");

        es.execute(new RunnableTask("task3"));
        printState(es, "task3");

        es.execute(new RunnableTask("task4"));
        printState(es, "task4");

        es.execute(new RunnableTask("task5"));
        printState(es, "task5");

        es.execute(new RunnableTask("task6"));
        printState(es, "task6");

        try {
            es.execute(new RunnableTask("task7"));
        } catch(RejectedExecutionException e) {
            log("task7 실행 거절 예외 발생: " + e);
        }

        sleep(3000);
        log("== 작업 수행 완료 ==");
        printState(es);

        sleep(3000);
        log("== maximumPoolSize 대기 시간 초과 ==");
        printState(es);

        es.close();
        log("== shutdown 완료 ==");
        printState(es);
    }
}
```
   
- 작업을 보관할 블로킹 큐의 구현체로 ArrayBlockingQueue(2)를 사용했다. 최대 2개 까지 작업을 큐에 보관할 수 있다.
- corePoolSize=2, maximumPoolSize=5를 사용해서 기본 스레드는 2개, 최대 스레드는 4개로 설정했다.
  - 스레드 풀에 기본 2개의 스레드를 운영하고 요청이 많아지면 스레드 풀을 최대 4개 까지 증가시켜 사용할 수 있다.
- 3000, TimeUnit.MILLISECONDS
  - 초과 스레드가 생존할 수 있는 대기 시간을 뜻한다. 이 시간 동안 초과 스레드가 처리할 작업이 없다면 초과 스레드는 제거된다.
- 스레드 풀의 스레드가 초과 스레드 사이즈 만큼 가득 찼을 때 task7 요청이 들어오면 RejectedExecutionException 이 발생한다.
   
### 13-3-1. Executor 스레드 풀 관리

1. 작업을 요청하면 core 사이즈 만큼 스레드를 만든다.
2. core 사지으를 초과하면 큐에 작업을 넣는다.
3. 큐를 초과하면 max 사이즈 만큼 스레드를 만든다. 임시로 사용되는 초과 스레드가 생성된다.
   - 작업이 큐에 가득 차면 큐에 넣을 수 없고 초과 스레드가 바로 수행한다.
4. max 사이즈를 초과하면 요청을 거절하고 예외가 발생한다.
   

### 13-3-2. 스레드 미리 생성하기
   
응답시간이 아주 중요한 서버라면 고객의 첫 요청을 받기 전에 스레드를 스레드 풀에 미리 생성해 두고 싶을 수 있다.   
스레드를 미리 생성해두면 처음 요청에서 사용되는 스레드의 생성 시간을 줄일 수 있다.    
ThreadPoolExecutor.prestartAllCoreThreads()를 사용하면 기본 스레드를 미리 생성할 수 있다.    
참고로 ExecutorService는 이 메서드를 제공하지 않는다.   

```java
public class PrestartPoolMain {

    public static void main(String[] args) {
        ExecutorService es = Executors.newFixedThreadPool(1000);

        printState(es);
        ThreadPoolExecutor poolExecutor = (ThreadPoolExecutor) es;
        poolExecutor.prestartAllCoreThreads();
        printState(es);
    }
}
```
