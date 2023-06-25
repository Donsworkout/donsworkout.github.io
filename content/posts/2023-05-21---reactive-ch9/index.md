---
title: Chapter 9. Sinks
date: "2023-05-21T23:46:37.121Z"
template: "post"
draft: false
slug: "/posts/reactive-ch9"
category: "devlog"
tags:
  - "스프링으로 시작하는 리액티브 프로그래밍"
  - "리액티브 프로그래밍"
description: "책 읽은거 정리하기, 스프링으로 시작하는 리액티브 프로그래밍 "
---

## 9. Sinks

### 9.1 Sinks 란?
Publisher와 Subscriber의 기능을 모두 지니는 Processor 의 기능을 개선한 것이며, Processor 과 관련된 API 는 Reactor 3.5.0 부터 완전히 제거될 예정입니다.

Sinks 는 리액티브 스트림즈의 Signal 을 프로그래밍 방식으로 푸시할 수 있는 구조이며 Flux 또는 Mono 의 의미 체계를 가집니다.

지금까지 배운 방식은 flux 나 mono 가 onNext 와 같은 Signal 을 내부적으로 전송해 주는 방식이였지만, Sinks 를 이용하면 명시적으로 Signal 을 전송할 수 있습니다. (기존에는 generate, create 등 오퍼레이터 사용)

### 그렇다면 Sinks 를 사용하는 것과 Operator 을 사용하여 Signal 을 전송하는 방식은 어떤 차이인가

Operator 기반 방식은 싱글스레드 기반이고, Sinks 는 멀티스레드 방식으로 Signal 을 전송해도 스레드 안정성을 보장한다.

### 9.2 Sinks 종류 및 특징
Reactor 에서 Sinks 를 사용하여 시그널을 전송하는 방식은 두가지 입니다.
첫째는 Sinks.One 을 사용하는 것이고 둘째는 Sinks.Many 를 사용하는 것입니다.

#### Sinks.One
> 한 건의 데이터를 프로그래밍 방식으로 emit 하는 역할을 하기도 하고, Mono 방식으로 Subscriber 가 데이터를 소비할 수 있도록 해 주는 Sinks 의 스펙 

~~~java
/**
 * Sinks.One 예제
 *  - emit 된 데이터 중에서 단 하나의 데이터만 Subscriber에게 전달한다. 나머지 데이터는 Drop 됨.
 */
@Slf4j
public class Example9_4 {
    public static void main(String[] args) throws InterruptedException {
        Sinks.One<String> sinkOne = Sinks.one();
        Mono<String> mono = sinkOne.asMono();

        sinkOne.emitValue("Hello Reactor", FAIL_FAST);
        sinkOne.emitValue("Hi Reactor", FAIL_FAST);
        sinkOne.emitValue(null, FAIL_FAST);

        mono.subscribe(data -> log.info("# Subscriber1 {}", data));
        mono.subscribe(data -> log.info("# Subscriber2 {}", data));
    }
}
~~~

#### Sinks.Many
> 여러 건의 데이터를 여러가지 방식으로 전송하는 기능을 정의해 둔 기능 명세입니다. Sinks.One 의 경우 단순히 한건의 데이터를 emit 하는 한가지 기능만 제공하기 때문에 별도의 Spec 이 제공되지 않고 기본 스펙을 사용하지만, Sinks.many 의 경우 여러가지 기능이 정의된 ManySpec 을 리턴합니다.

- UnicastSpec 
- MulticastSpec
- MulticastReplaySpec

~~~java
/**
 * Sinks.Many 예제
 *  - unicast()통해 단 하나의 Subscriber만 데이터를 전달 받을 수 있다
 */
@Slf4j
public class Example9_8 {
    public static void main(String[] args) throws InterruptedException {
        Sinks.Many<Integer> unicastSink = Sinks.many().unicast().onBackpressureBuffer();
        Flux<Integer> fluxView = unicastSink.asFlux();

        unicastSink.emitNext(1, FAIL_FAST);
        unicastSink.emitNext(2, FAIL_FAST);


        fluxView.subscribe(data -> log.info("# Subscriber1: {}", data));

        unicastSink.emitNext(3, FAIL_FAST);

        fluxView.subscribe(data -> log.info("# Subscriber2: {}", data));
    }
}
~~~

~~~java
/**
 * Sinks.Many 예제
 *  - multicast()를 사용해서 하나 이상의 Subscriber에게 데이터를 emit하는 예제
 */
@Slf4j
public class Example9_9 {
    public static void main(String[] args) {
        Sinks.Many<Integer> multicastSink =
                Sinks.many().multicast().onBackpressureBuffer();
        Flux<Integer> fluxView = multicastSink.asFlux();

        multicastSink.emitNext(1, FAIL_FAST);
        multicastSink.emitNext(2, FAIL_FAST);

        fluxView.subscribe(data -> log.info("# Subscriber1: {}", data));
        fluxView.subscribe(data -> log.info("# Subscriber2: {}", data));

        multicastSink.emitNext(3, FAIL_FAST);
    }
}
~~~

~~~java
/**
 * Sinks.Many 예제
 *  - replay()를 사용하여 이미 emit된 데이터 중에서 특정 개수의 최신 데이터만 전달하는 예제
 */
@Slf4j
public class Example9_10 {
    public static void main(String[] args) {
        Sinks.Many<Integer> replaySink = Sinks.many().replay().limit(2);
        Flux<Integer> fluxView = replaySink.asFlux();

        replaySink.emitNext(1, FAIL_FAST);
        replaySink.emitNext(2, FAIL_FAST);
        replaySink.emitNext(3, FAIL_FAST);

        fluxView.subscribe(data -> log.info("# Subscriber1: {}", data));

        replaySink.emitNext(4, FAIL_FAST);

        fluxView.subscribe(data -> log.info("# Subscriber2: {}", data));
    }
}
~~~