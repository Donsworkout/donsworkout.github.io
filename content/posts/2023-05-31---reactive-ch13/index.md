---
title: Chapter 13. Testing
date: "2023-05-31T23:49:37.121Z"
template: "post"
draft: false
slug: "/posts/reactive-ch13"
category: "devlog"
tags:
  - "스프링으로 시작하는 리액티브 프로그래밍"
  - "리액티브 프로그래밍"
description: "책 읽은거 정리하기, 스프링으로 시작하는 리액티브 프로그래밍 "
---

## 13. Testing

### 13.1 StepVerifier 을 사용한 테스팅

Reactor 에서 가장 일반적인 테스트 방식은 Flux / Mono 를 Reactor 시퀀스로 정의한 후 구독 시점에 해당 operator 체인이 시나리오대로 동작 하는지를 테스트 하는 것이다.

Reactor Sequence 에서 다음에 발생할 Signal 이 무엇인지, 기대하던 데이터들이 emit 되었는지 특정 시간동안 emit 된 데이터가 있는지 등을 단계적으로 테스트할 수 있다.

1. Signal 이벤트 테스트 
~~~java
/**
 * StepVerifier 기본 예제
 */
public class ExampleTest13_1 {
    @Test
    public void sayHelloReactorTest() {
        StepVerifier
                .create(Mono.just("Hello Reactor")) // 테스트 대상 Sequence 생성
                .expectNext("Hello Reactor")    // emit 된 데이터 검증
                .expectComplete()   // onComplete Signal 검증
                .verify();          // 검증 실행.
    }
}
~~~

#### expectXXX 메서드

- `expectSubscription()` : 구독이 이루어짐을 기대
- `expectNext(T t)` : onNext Signal 을 통해 전달되는 값이 파라미터로 전달된 값과 같음을 기대
- `expectComplete()` : onComplete Signal 이 전송되기를 기대
- `expectError()` : onError Signal 이 전송되기를 기대 
- `expectNextCount(long count)` : 구독시점 또는 이전 expectNext 를 통해 기댓값이 평가된 데이터 이후부터 emit 된 수를 기대한다.
- `expectNoEvent(Duration duration)` : 주어진 시간동안 signal 이 발생하지 않았음을 기대한다.
- `expectAccessibleContext()` : 구독 시점 이후에 Context 가 전파되었음을 기대한다.
- `expectNextSequence(Iterable <? extends T> iterable)` : emit된 데이터들이 파라미터로 전달된 iterable 의 요소와 매치됨을 기대한다. 

#### verifyXXXX() 메서드
- `verify()` : 검증을 트리거한다
- `verifyComplete` : 검증을 트리거하고, onComplete 시그널을 기대한다.
- `verfifyError` : 검증을 트리거하고, onError 시그널을 기대한다.
- `verifyTimeout(Duration duration)` : 검증을 트리거하고, 주어진 시간이 초과되어도 Publisher 가 종료되지 않음을 기대한다.

예제 Base Code
~~~java 
public class GeneralTestExample {
    public static Flux<String> sayHello() {
        return Flux
                .just("Hello", "Reactor");
    }

    public static Flux<Integer> divideByTwo(Flux<Integer> source) {
        return source
                .zipWith(Flux.just(2, 2, 2, 2, 0), (x, y) -> x/y);
    }

    public static Flux<Integer> takeNumber(Flux<Integer> source, long n) {
        return source
                .take(n);
    }
}
~~~

~~~java
/**
 * StepVerifier 활용 예제
 */
public class ExampleTest13_3 {
    @Test
    public void sayHelloTest() {
        StepVerifier
                .create(GeneralTestExample.sayHello())
                .expectSubscription()
                .as("# expect subscription")
                .expectNext("Hi")
                .as("# expect Hi")
                .expectNext("Reactor")
                .as("# expect Reactor")
                .verifyComplete();
    }
}
~~~

- as 는 기댓값 평가 단계에 대한 설명이다.

~~~java
/**
 * StepVerifier 활용 예제.
 */
public class ExampleTest13_4 {
    @Test
    public void divideByTwoTest() {
        Flux<Integer> source = Flux.just(2, 4, 6, 8, 10);
        StepVerifier
                .create(GeneralTestExample.divideByTwo(source))
                .expectSubscription()
                .expectNext(1)
                .expectNext(2)
                .expectNext(3)
                .expectNext(4)
//                .expectNext(1, 2, 3, 4)
                .expectError()
                .verify();
    }
}
~~~

- expectError 을 하고 있기 때문에 ArithmeticException 으로 테스트는 통과하게 된다.

~~~java
/**
 * StepVerifier 활용 예제
 */
public class ExampleTest13_5 {
    @Test
    public void takeNumberTest() {
        Flux<Integer> source = Flux.range(0, 1000);
        StepVerifier
                .create(GeneralTestExample.takeNumber(source, 500),
                        StepVerifierOptions.create().scenarioName("Verify from 0 to 499"))
                .expectSubscription()
                .expectNext(0)
                .expectNextCount(498)
                .expectNext(500)
                .expectComplete()
                .verify();
    }
}
~~~

- StepVerifierOptions 는 이름 그대로 StepVerifier 에 옵션을 추가하는 기능인데, 예제 코드에서는 테스트에 실패할 경우 파라미터로 입력한 테스트명을 출력한다.

#### 시간 기반 테스트 
StepVerifier 은 가상의 시간을 이용해 미래에 실행되는 리액트 시퀀스의 시간을 앞당겨 테스트할 수 있는 기능을 지원합니다.

Base Code 

~~~java
public class TimeBasedTestExample {
    public static Flux<Tuple2<String, Integer>> getCOVID19Count(Flux<Long> source) {
        return source
                .flatMap(notUse -> Flux.just(
                                Tuples.of("서울", 10),
                                Tuples.of("경기도", 5),
                                Tuples.of("강원도", 3),
                                Tuples.of("충청도", 6),
                                Tuples.of("경상도", 5),
                                Tuples.of("전라도", 8),
                                Tuples.of("인천", 2),
                                Tuples.of("대전", 1),
                                Tuples.of("대구", 2),
                                Tuples.of("부산", 3),
                                Tuples.of("제주도", 0)
                        )
                );
    }

    public static Flux<Tuple2<String, Integer>> getVoteCount(Flux<Long> source) {
        return source
                .zipWith(Flux.just(
                                Tuples.of("중구", 15400),
                                Tuples.of("서초구", 20020),
                                Tuples.of("강서구", 32040),
                                Tuples.of("강동구", 14506),
                                Tuples.of("서대문구", 35650)
                        )
                )
                .map(Tuple2::getT2);
    }
}
~~~

~~~java
/**
 * StepVerifier 활용 예제
 * - 주어진 시간을 앞당겨서 테스트 한다.
 */
public class ExampleTest13_7 {
    @Test
    public void getCOVID19CountTest() {
        StepVerifier
                .withVirtualTime(() -> TimeBasedTestExample.getCOVID19Count(
                                Flux.interval(Duration.ofHours(1)).take(1)
                        )
                )
                .expectSubscription()
                .then(() -> VirtualTimeScheduler
                                    .get()
                                    .advanceTimeBy(Duration.ofHours(1)))
                .expectNextCount(11)
                .expectComplete()
                .verify();

    }
}
~~~

- VirtualTimeScheduler 는 가상 시간을 한시간 당겨 한시간 후에 스케줄링된 시퀀스의 결과를 테스트 해 봃 수 있다.

~~~java
/**
 * StepVerifier 활용 예제
 *  -검증에 소요되는 시간을 제한한다.
 */
public class ExampleTest13_8 {
    @Test
    public void getCOVID19CountTest() {
        StepVerifier
                .create(TimeBasedTestExample.getCOVID19Count(
                                Flux.interval(Duration.ofMinutes(1)).take(1)
                        )
                )
                .expectSubscription()
                .expectNextCount(11)
                .expectComplete()
                .verify(Duration.ofSeconds(3));
    }
}
~~~

- 위의 Flux.interval 의 시간이 1분으로 설정되어 1분후에 데이터를 emit 할 것이기 때문에 3초이내 결과 평가가 끝나지 않기에 AssertionError 가 발생한다.

~~~java
/**
 * StepVerifier 활용 예제
 *  - 지정된 대기 시간동안 이벤트가 없을을 확인한다.
 */
public class ExampleTest13_9 {
    @Test
    public void getVoteCountTest() {
        StepVerifier
                .withVirtualTime(() -> TimeBasedTestExample.getVoteCount(
                                Flux.interval(Duration.ofMinutes(1))
                        )
                )
                .expectSubscription()
                .expectNoEvent(Duration.ofMinutes(1))
                .expectNoEvent(Duration.ofMinutes(1))
                .expectNoEvent(Duration.ofMinutes(1))
                .expectNoEvent(Duration.ofMinutes(1))
                .expectNoEvent(Duration.ofMinutes(1))
                .expectNextCount(5)
                .expectComplete()
                .verify();
    }
}
~~~

- expectNoEvent 메서드를 통해 1분동안 onNext Signal 이벤트가 발생하지 않을 것이라고 기대함 + 지정된 시간만큼 시간을 앞당긴다는 뜻 

BackPressure Test Base Code
~~~java
public class BackpressureTestExample {
    public static Flux<Integer> generateNumber() {
        return Flux
                .create(emitter -> {
                    for (int i = 1; i <= 100; i++) {
                        emitter.next(i);
                    }
                    emitter.complete();
                }, FluxSink.OverflowStrategy.ERROR);
    }
}
~~~

~~~java 
/**
 * StepVerifier Backpressure 테스트 예제
 */
public class ExampleTest13_11 {
    @Test
    public void generateNumberTest() {
        StepVerifier
                .create(BackpressureTestExample.generateNumber(), 1L)
                .thenConsumeWhile(num -> num >= 1)
                .verifyComplete();
    }
}
~~~

- 위의 코드는 정상적으로 시퀀스가 종료함을 기대하는 예제 
- 결과는 fail 이유는? => 1개 emit 을 기대했으나 실제로 100개의 숫자 data 가 emit 됨

~~~java
/**
 * StepVerifier Backpressure 테스트 예제
 */
public class ExampleTest13_12 {
    @Test
    public void generateNumberTest() {
        StepVerifier
                .create(BackpressureTestExample.generateNumber(), 1L)
                .thenConsumeWhile(num -> num >= 1)
                .expectError()
                .verifyThenAssertThat()
                .hasDroppedElements();

    }
}
~~~

- 오버플로우로 인해 Drop 되는 데이터가 있으므로 test passed 


#### Context 테스트 

Base Code
~~~java
public class ContextTestExample {
    public static Mono<String> getSecretMessage(Mono<String> keySource) {
        return keySource
                .zipWith(Mono.deferContextual(ctx ->
                                               Mono.just((String)ctx.get("secretKey"))))
                .filter(tp ->
                            tp.getT1().equals(
                                   new String(Base64Utils.decodeFromString(tp.getT2())))
                )
                .transformDeferredContextual(
                        (mono, ctx) -> mono.map(notUse -> ctx.get("secretMessage"))
                );
    }
}
~~~

~~~java
/**
 * StepVerifier Context 테스트 예제
 */
public class ExampleTest13_14 {
    @Test
    public void getSecretMessageTest() {
        Mono<String> source = Mono.just("hello");

        StepVerifier
                .create(
                    ContextTestExample
                        .getSecretMessage(source)
                        .contextWrite(context ->
                                        context.put("secretMessage", "Hello, Reactor"))
                        .contextWrite(context -> context.put("secretKey", "aGVsbG8="))
                )
                .expectSubscription()
                .expectAccessibleContext()
                .hasKey("secretKey")
                .hasKey("secretMessage")
                .then()
                .expectNext("Hello, Reactor")
                .expectComplete()
                .verify();
    }
}
~~~

- Context 에 해당 keys 있는지 여부를 테스트 하며 emit 된 문자열을 테스트 하고 있다. => passed 

#### Record 기반 테스트 
emit 된 데이터의 단순 기댓값만 평가하는 것이 아니라 [구체적인 조건] 으로 Assertion 해야할 때 record 기반 테스트 진행 

Base Code
~~~java
public class RecordTestExample {
    public static Flux<String> getCapitalizedCountry(Flux<String> source) {
        return source
                .map(country -> country.substring(0, 1).toUpperCase() +
                                country.substring(1));
    }
}
~~~

~~~java
/**
 * StepVerifier Record 테스트 예제
 */
public class ExampleTest13_16 {
    @Test
    public void getCountryTest() {
        StepVerifier
                .create(RecordTestExample.getCapitalizedCountry(
                        Flux.just("korea", "england", "canada", "india")))
                .expectSubscription()
                .recordWith(ArrayList::new)
                .thenConsumeWhile(country -> !country.isEmpty())
                .consumeRecordedWith(countries -> {
                    assertThat(
                            countries
                                    .stream()
                                    .allMatch(country ->
                                            Character.isUpperCase(country.charAt(0))),
                            is(true)
                    );
                })
                .expectComplete()
                .verify();
    }
}
~~~

- recordWith 으로 emit 된 데이터 레코딩 시작
- thenConsumeWhile 파라미터로 전달한 Predicate 와 일치하는 데이터는 다음 단계에서 소비할 수 있도록 함
- consumeRecordedWith 로 컬렉션에 기록된 데이터 소비하며 값들을 Assertion 한다.
- 마지막으로 expectComplete

~~~java
/**
 * StepVerifier Record 테스트 예제
 */
public class ExampleTest13_17 {
    @Test
    public void getCountryTest() {
        StepVerifier
                .create(RecordTestExample.getCapitalizedCountry(
                        Flux.just("korea", "england", "canada", "india")))
                .expectSubscription()
                .recordWith(ArrayList::new)
                .thenConsumeWhile(country -> !country.isEmpty())
                .expectRecordedMatches(countries ->
                        countries
                                .stream()
                                .allMatch(country ->
                                        Character.isUpperCase(country.charAt(0))))
                .expectComplete()
                .verify();
    }
}
~~~

- 이전 예제와 테스트 시나리오는 같음
- 이번에는 expectRecordedMatches 메서드 내에서 Predicate 를 사용함

### 13.2 TestPublisher 을 이용한 테스팅

reactor-test 모듈에서 지원하는 테스트 전용 퍼블리셔인 TestPublisher 을 이용하여 테스트를 진행할 수 있음

#### 정상 동작하는 TestPublisher 
- emit 하는 데이터가 Null 인지, 요청하는 데이터 보다 더 많은 data 를 emit 하는지 등 리액티브 스트림즈 스펙 위반여부를 사전에 체크한다는 의미

~~~java
/**
 * 정상동작 하는 TestPublisher 예제
 */
public class ExampleTest13_18 {
    @Test
    public void divideByTwoTest() {
        TestPublisher<Integer> source = TestPublisher.create();

        StepVerifier
                .create(GeneralTestExample.divideByTwo(source.flux()))
                .expectSubscription()
                .then(() -> source.emit(2, 4, 6, 8, 10))
                .expectNext(1, 2, 3, 4)
                .expectError()
                .verify();
    }
}
~~~

#### 오동작하는 TestPublisher
- 리액티브 스트림즈 사양 위반 여부를 사전에 체크하지 않아, 사양에 위반되더라도 TestPublisher 은 데이터를 emit 할 수 있음

~~~java
/**
 * 오동작 하는 TestPublisher 예제
 */
public class ExampleTest13_19 {
    @Test
    public void divideByTwoTest() {
        TestPublisher<Integer> source = TestPublisher.create();
//        TestPublisher<Integer> source =
//                TestPublisher.createNoncompliant(TestPublisher.Violation.ALLOW_NULL);

        StepVerifier
                .create(GeneralTestExample.divideByTwo(source.flux()))
                .expectSubscription()
                .then(() -> {
                    getDataSource().stream()
                            .forEach(data -> source.next(data));
                    source.complete();
                })
                .expectNext(1, 2, 3, 4, 5)
                .expectComplete()
                .verify();
    }

    private static List<Integer> getDataSource() {
        return Arrays.asList(2, 4, 6, 8, null);
    }
}
~~~

#### 오동작 하는 TestPublisher 을 생성하기 위한 위반조건

- ALLOW_NULL : 전송할 데이터가 null 이여도 널포인트 오류 발생하지 않고 다음호출 진행할 수 있도록 허용
- CLEAN_UP_TERMINATE : onComplete, onError, emit 같은 터미널 시그널을 연달아 여러 번 보낼 수 있도록 함
- DEFER_CANCELLATION : cancel 시그널을 무시하고 계속 Signal 을 emit 할 수 있도록 함
- REQUEST_OVERFLOW : 요청 개수보다 더 많은 시그널이 발생해도 IllegalStateException 이 발생하지 않고 다음 호출 진행할 수 있도록 함

### 13.3 PublisherProbe 를 사용한 테스팅 

reactor-test 모듈은 PublisherProbe 를 이용해 시퀀스의 실행 경로를 테스트 할 수 있음 

Base Code
~~~java
public class PublisherProbeTestExample {
    public static Mono<String> processTask(Mono<String> main, Mono<String> standby) {
        return main
                .flatMap(massage -> Mono.just(massage))
                .switchIfEmpty(standby);
    }

    public static Mono<String> supplyMainPower() {
        return Mono.empty();
    }

    public static Mono supplyStandbyPower() {
        return Mono.just("# supply Standby Power");
    }
}
~~~

- switchIfEmpty() Operator 은 업스트림 퍼블리셔가 데이터 emit 없이 종료되는 경우, 대체 퍼블리셔가 데이터를 emit 하게 함 (약간 서킷브레이커 느낌)

~~~java
/**
 * PublisherProbe 예제
 */
public class ExampleTest13_21 {
    @Test
    public void publisherProbeTest() {
        PublisherProbe<String> probe =
                PublisherProbe.of(PublisherProbeTestExample.supplyStandbyPower());

        StepVerifier
                .create(PublisherProbeTestExample
                        .processTask(
                                PublisherProbeTestExample.supplyMainPower(),
                                probe.mono())
                )
                .expectNextCount(1)
                .verifyComplete();

        probe.assertWasSubscribed();
        probe.assertWasRequested();
        probe.assertWasNotCancelled();
    }
}
~~~

- 먼저 PublisherProbe.of() 메서드로 퍼블리셔를 래핑하여 실행경로를 테스트 할 퍼블리셔를 지정함 
- standByPower 가 동작하면서 passed 