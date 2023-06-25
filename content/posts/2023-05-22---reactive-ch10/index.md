---
title: Chapter 10. Scheduler
date: "2023-05-22T23:46:37.121Z"
template: "post"
draft: false
slug: "/posts/reactive-ch10"
category: "devlog"
tags:
  - "스프링으로 시작하는 리액티브 프로그래밍"
  - "리액티브 프로그래밍"
description: "책 읽은거 정리하기, 스프링으로 시작하는 리액티브 프로그래밍 "
---

## 10. Scheduler

### 10.1 스레드의 개념 이해 
Reactor 에서 사용되는 스케줄러는 리액트 시퀀스에서 사용되는 스레드를 관리해 주는 역할을 합니다. 

#### 물리적인 스레드 
논리적인 코어라고도 불리며, 물리적인 코어를 나눈 것을 의미합니다. 

#### 논리적인 스레드 
소프트웨어적으로 생성되는 스레드를 의미하며, 이것이 바로 우리가 프로그래밍에서 사용하는 스레드입니다. 이는 프로세스 내에서 실행되는 세부 작업의 단위가 됩니다. 

#### 병렬성 vs 동시성
병렬성 : 물리적인 스레드가 실제로 동시에 실행되기 때문에 여러가지 작업을 동시에 처리함을 의미합니다.

동시성 : 동시에 진행되는 것처럼 보이는 성질입니다. 그러나 내부적으로는 무수히 많은 논리 스레드가 물리적인 스레드를 아주 빠른 속도로 번갈아 가며 사용하여 동시에 실행되는 것처럼 보입니다. (시분할)

### 10.2 Scheduler 란?
리액터의 스케줄러도 OS에서 사용되는 스케줄러의 의미와 비슷합니다. 즉, 스케줄러를 사용하여 어떤 스레드에서 무엇을 처리할지 제어합니다. Race Condition 처리 등 개발자가 직접 스레드를 제어하는 부담에서 벗어 날 수 있습니다. 

### 10.3 Scheduler 를 위한 전용 Operator
Scheduler 전용 오퍼레이터가 존재합니다.

#### subscribeOn()
> 구독이 발생한 직후 실행될 스레드를 지정하는 Operator 입니다. 
~~~java
/**
 * subscribeOn() 기본 예제
 *  - 구독 시점에 Publisher의 실행을 위한 쓰레드를 지정한다
 */
@Slf4j
public class Example10_1 {
    public static void main(String[] args) throws InterruptedException {
        Flux.fromArray(new Integer[] {1, 3, 5, 7})
                .subscribeOn(Schedulers.boundedElastic())
                .doOnNext(data -> log.info("# doOnNext: {}", data))
                .doOnSubscribe(subscription -> log.info("# doOnSubscribe"))
                .subscribe(data -> log.info("# onNext: {}", data));

        Thread.sleep(500L);
    }
}
~~~

위의 코드를 실행하여 보면 subscribeOn 에서 스케줄러를 지정했기 때문에 구독이 발생한 직후부터는 원본 Flux 의 동작을 처리하는 스레드가 바뀌게 됩니다.

#### publishOn()
> 다운스트림으로 시그널을 전송할 때 실행되는 스레드를 제어하는 역할을 하는 Operator 

~~~java
/**
 * publishOn() 기본 예제
 *  - Operator 체인에서 Downstream Operator의 실행을 위한 쓰레드를 지정한다.
 */
@Slf4j
public class Example10_2 {
    public static void main(String[] args) throws InterruptedException {
        Flux.fromArray(new Integer[] {1, 3, 5, 7})
                .doOnNext(data -> log.info("# doOnNext: {}", data))
                .doOnSubscribe(subscription -> log.info("# doOnSubscribe"))
                .publishOn(Schedulers.parallel())
                .subscribe(data -> log.info("# onNext: {}", data));

        Thread.sleep(500L);
    }
}
~~~

onNext 의 경우 7번 라인에서 publishOn Operator 을 추가했기 때문에 publishOn 을 기준으로 다운스트림의 실행 스레드가 변경되었습니다.

#### parallel()
> 라운드 로빈 방식으로 CPU 코어 개수만큼의 스레드를 병렬로 실행합니다. 여기사 코어의 개수는 물리적 코어가 아니라 논리적인 코어, 즉 물리적인 스레드의 개수를 의미합니다. 

~~~java
/**
 * parallel() 기본 사용 예제
 * - parallel()만 사용할 경우에는 병렬로 작업을 수행하지 않는다.
 * - runOn()을 사용해서 Scheduler를 할당해주어야 병렬로 작업을 수행한다.
 * - **** CPU 코어 갯수내에서 worker thread를 할당한다. ****
 */
@Slf4j
public class Example10_3 {
    public static void main(String[] args) throws InterruptedException {
        Flux.fromArray(new Integer[]{1, 3, 5, 7, 9, 11, 13, 15, 17, 19})
                .parallel(4)
                .runOn(Schedulers.parallel())
                .subscribe(data -> log.info("# onNext: {}", data));

        Thread.sleep(100L);
    }
}
~~~

### 10.4 publishOn 과 subscribeOn 의 동작 이해 

#### publishOn()
> publishOn 을 통해 실행 스레드를 목적에 맞게 분리할 수 있습니다. publishOn 호출할 때 마다 스레드가 변경됩니다.

~~~java
/**
 * subscribeOn()과 publishOn()의 동작 과정 예
 *  - 두 개의 publishOn()을 사용한 경우
 *      - 다음 publishOn()을 만나기 전까지 publishOn() 아래 쪽 Operator들의 실행 쓰레드를 변경한다.
 *
 */
@Slf4j
public class Example10_7 {
    public static void main(String[] args) throws InterruptedException {
        Flux
            .fromArray(new Integer[] {1, 3, 5, 7})
            .doOnNext(data -> log.info("# doOnNext fromArray: {}", data))
            .publishOn(Schedulers.parallel())
            .filter(data -> data > 3)
            .doOnNext(data -> log.info("# doOnNext filter: {}", data))
            .publishOn(Schedulers.parallel())
            .map(data -> data * 10)
            .doOnNext(data -> log.info("# doOnNext map: {}", data))
            .subscribe(data -> log.info("# onNext: {}", data));

        Thread.sleep(500L);
    }
}
~~~

#### subscribeOn()
> subscribeOn 은 구독이 발생한 직후에 실행될 스레드를 지정하는 오퍼레이터 입니다. 

~~~java
/**
 * subscribeOn()과 publishOn()의 동작 과정 예
 *  - subscribeOn()과 publishOn()을 함께 사용한 경우
 *      - subscribeOn()은 구독 직후에 실행될 쓰레드를 지정하고, publishOn()을 만나기 전까지 쓰레드를 변경하지 않는다.
 *
 */
@Slf4j
public class Example10_8 {
    public static void main(String[] args) throws InterruptedException {
        Flux
            .fromArray(new Integer[] {1, 3, 5, 7})
            .subscribeOn(Schedulers.boundedElastic())
            .doOnNext(data -> log.info("# doOnNext fromArray: {}", data))
            .filter(data -> data > 3)
            .doOnNext(data -> log.info("# doOnNext filter: {}", data))
            .publishOn(Schedulers.parallel())
            .map(data -> data * 10)
            .doOnNext(data -> log.info("# doOnNext map: {}", data))
            .subscribe(data -> log.info("# onNext: {}", data));

        Thread.sleep(500L);
    }
}
~~~

### 10.5 Scheduler 의 종류  

#### 1. Schedulers.immediate()

별도의 스레드를 추가적으로 생성하지 않고, 현재 스레드에서 작업을 처리하고자 할 때 사용할 수 있습니다. 

#### 2. Schedulers.single()
스레드 하나만 생성해서 스케줄러가 제거되기 전까지 재사용하는 방식 

#### 3. Schedulers.newSingle()
호출할 때마다 매번 새로운 스레드 하나를 생성 

#### 4. Schedulers.boundedElastic()
스레드 풀을 생성한 후, 그 안에서 정해진 수만큼의 스레드를 사용하여 작업을 처리하고 작업이 종료된 스레드는 반납하여 재사용하는 방식

#### 5. Schedulers.parallel()
CPU 코어 수 만큼의 스레드를 생성합니다.

#### 6. Schedulers.fromExecutorService()
기존에 이미 사용하고 있는 ExecutorService 가 있다면 이로부터 스케줄러를 생성하는 방식, 다만 Reactor 에서는 권장하지 않습니다.

#### 7. Schedulers.newXXXX()
위에서 설명한 스케쥴러 메서드들은 Reactor 에서 제공하는 디폴트 Scheduler 인터페이스를 사용하지만, 필요하다면 Schedulers.newSingle(), Schedulers.newBoundedElastic(), Schedulers.newParallel() 메서드를 사용해서 새로운 Scheduler 인스턴스를 생성할 수 있다. 