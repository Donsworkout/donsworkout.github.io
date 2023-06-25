---
title: Chapter 12. Debugging
date: "2023-05-31T23:48:37.121Z"
template: "post"
draft: false
slug: "/posts/reactive-ch12"
category: "devlog"
tags:
  - "스프링으로 시작하는 리액티브 프로그래밍"
  - "리액티브 프로그래밍"
description: "책 읽은거 정리하기, 스프링으로 시작하는 리액티브 프로그래밍"
---

## 12. Debugging

### 12.1 Reactor 에서 디버깅 방법 

일반적인 동기식, 명령형 프로그래밍 방식은 StackTrace 나 Debugger 을 걸어서 디버깅 하기가 용이하나, 비동기 방식은 그렇게 하기 어렵기 때문에 Reactor 에서는 몇가지 방법을 제공한다.

### 12.1.1 Debug Mode 

~~~java
/**
 * onOperatorDebug() Hook 메서드를 이용한 Debug mode 예
 * - 애플리케이션 전체에서 global 하게 동작한다.
 */
@Slf4j
public class Example12_1 {
    public static Map<String, String> fruits = new HashMap<>();

    static {
        fruits.put("banana", "바나나");
        fruits.put("apple", "사과");
        fruits.put("pear", "배");
        fruits.put("grape", "포도");
    }

    public static void main(String[] args) throws InterruptedException {
        Hooks.onOperatorDebug();

        Flux
                .fromArray(new String[]{"BANANAS", "APPLES", "PEARS", "MELONS"})
                .subscribeOn(Schedulers.boundedElastic())
                .publishOn(Schedulers.parallel())
                .map(String::toLowerCase)
                .map(fruit -> fruit.substring(0, fruit.length() - 1))
                .map(fruits::get)
                .map(translated -> "맛있는 " + translated)
                .subscribe(
                        log::info,
                        error -> log.error("# onError:", error));

        Thread.sleep(100L);
    }
}
~~~

- Operator 체인이 시작되기 전에 Debug 모드를 활성화하면 에러가 발생한 지점을 좀 더 명확하게 찾을 수 있습니다.

- 다만 디버그 모드를 활성화할 시 내부적으로는 애플리케이션 내에 있는 모든 stacktrace 를 캡쳐하고, 에러가 발생하면 캡쳐한 정보를 기반으로 에러가 발생한 Assembly 의 스택트레이스를 원본 스택트레이스 중간에 끼워 넣기 때문에 비용이 크다.

### 12.1.2 Checkpoint() Operator 
해당 오퍼레이터는 특정 Operator 체인 내의 스택트레이스만 캡쳐 가능 

~~~java
/**
 *  checkpoint()를 사용한 디버깅 예
 * - checkpoint()를 지정한 Operator 체인에서만 동작한다.
 */
@Slf4j
public class Example12_3 {
    public static void main(String[] args) {
        Flux
            .just(2, 4, 6, 8)
            .zipWith(Flux.just(1, 2, 3, 0), (x, y) -> x/y)
            .checkpoint()
            .map(num -> num + 2)
            .checkpoint()
            .subscribe(
                    data -> log.info("# onNext: {}", data),
                    error -> log.error("# onError:", error)
            );
    }
}
~~~


- Traceback 출력 없이 description 출력하도록 하려면 
~~~java
/**
 * checkpoint(description)을 사용한 디버깅 예
 * - description 을 추가해서 에러가 발생한 지점을 구분할 수 있다.
 * - description 을 지정할 경우 traceback 을 추가하지 않는다.
 */
@Slf4j
public class Example12_4 {
    public static void main(String[] args) {
        Flux
            .just(2, 4, 6, 8)
            .zipWith(Flux.just(1, 2, 3, 0), (x, y) -> x/y)
            .checkpoint("Example12_4.zipWith.checkpoint")
            .map(num -> num + 2)
            .checkpoint("Example12_4.map.checkpoint")
            .subscribe(
                    data -> log.info("# onNext: {}", data),
                    error -> log.error("# onError:", error)
            );
    }
}
~~~

- Traceback 및 description 모두 출력하도록 하려면 

~~~java
/**
 * checkpoint(description)을 사용한 디버깅 예
 * - description 을 추가해서 에러가 발생한 지점을 구분할 수 있다.
 * - forceStackTrace 을 true로 지정할 경우 traceback도 추가한다.
 */
@Slf4j
public class Example12_5 {
    public static void main(String[] args) {
        Flux
            .just(2, 4, 6, 8)
            .zipWith(Flux.just(1, 2, 3, 0), (x, y) -> x/y)
            .checkpoint("Example12_4.zipWith.checkpoint", true)
            .map(num -> num + 2)
            .checkpoint("Example12_4.map.checkpoint", true)
            .subscribe(
                    data -> log.info("# onNext: {}", data),
                    error -> log.error("# onError:", error)
            );
    }
}
~~~

- Operator 체인이 복잡해지면 각각의 operator 체인에 checkpoin() 를 추가하여 디버깅 한다.

~~~java
/**
 * 복잡한 단계를 거치는 Operator 체인에서 checkpoint()를 사용하는 예제
 */
@Slf4j
public class Example12_6 {
    public static void main(String[] args) {
        Flux<Integer> source = Flux.just(2, 4, 6, 8);
        Flux<Integer> other = Flux.just(1, 2, 3, 0);

        Flux<Integer> multiplySource = divide(source, other).checkpoint();
        Flux<Integer> plusSource = plus(multiplySource).checkpoint();


        plusSource.subscribe(
                data -> log.info("# onNext: {}", data),
                error -> log.error("# onError:", error)
        );
    }

    private static Flux<Integer> divide(Flux<Integer> source, Flux<Integer> other) {
        return source.zipWith(other, (x, y) -> x/y);
    }

    private static Flux<Integer> plus(Flux<Integer> source) {
        return source.map(num -> num + 2);
    }
~~~

### 12.1.3 log() operator 을 사용한 디버깅 

~~~java
/**
 * log() operator를 사용한 예제
 */
@Slf4j
public class Example12_7 {
    public static Map<String, String> fruits = new HashMap<>();

    static {
        fruits.put("banana", "바나나");
        fruits.put("apple", "사과");
        fruits.put("pear", "배");
        fruits.put("grape", "포도");
    }

    public static void main(String[] args) {
        Flux.fromArray(new String[]{"BANANAS", "APPLES", "PEARS", "MELONS"})
                .map(String::toLowerCase)
                .map(fruit -> fruit.substring(0, fruit.length() - 1))
                .log()
//                .log("Fruit.Substring", Level.FINE)
                .map(fruits::get)
                .subscribe(
                        log::info,
                        error -> log.error("# onError:", error));
    }
}
~~~