---
title: Chapter 11. Context
date: "2023-05-31T23:46:37.121Z"
template: "post"
draft: false
slug: "/posts/reactive-ch11"
category: "devlog"
tags:
  - "스프링으로 시작하는 리액티브 프로그래밍"
  - "리액티브 프로그래밍"
description: "책 읽은거 정리하기, 스프링으로 시작하는 리액티브 프로그래밍"
---

## 11. Context

### 11.1 Context 란?

#### 일반적인 의미 
- 어떠한 상항에서 그 상황을 처리하기 위해 필요한 정보

#### In reactor 
- Reactor 구성요소간 전파되는 key, value 형태의 저장소
- 구독이 발생할 때 마다 해당 구독과 연결되어 하나의 Context 가 생김 

~~~java
/**
 * Context 기본 예제
 *  - contextWrite() Operator로 Context에 데이터 쓰기 작업을 할 수 있다.
 *  - Context.put()으로 Context에 데이터를 쓸 수 있다.
 *  - deferContextual() Operator로 Context에 데이터 읽기 작업을 할 수 있다.
 *  - Context.get()으로 Context에서 데이터를 읽을 수 있다.
 *  - transformDeferredContextual() Operator로 Operator 중간에서 Context에 데이터 읽기 작업을 할 수 있다.
 */
@Slf4j
public class Example11_1 {
    public static void main(String[] args) throws InterruptedException {
        Mono
            .deferContextual(ctx ->
                Mono
                    .just("Hello" + " " + ctx.get("firstName"))
                    .doOnNext(data -> log.info("# just doOnNext : {}", data))
            )
            .subscribeOn(Schedulers.boundedElastic())
            .publishOn(Schedulers.parallel())
            .transformDeferredContextual(
                    (mono, ctx) -> mono.map(data -> data + " " + ctx.get("lastName"))
            )
            .contextWrite(context -> context.put("lastName", "Jobs"))
            .contextWrite(context -> context.put("firstName", "Steve"))
            .subscribe(data -> log.info("# onNext: {}", data));

        Thread.sleep(100L);
    }
}
~~~

위의 예제 코드에서 `contextWrite` Opearator 가 context 에 데이터를 쓰는 부분임을 알 수 있다. 

#### Context 에 쓰인 데이터 읽기 

읽는 방식은 두가지 입니다

1. 원본 데이터 소스 레벨에서 읽는 방식 : `deferContextual`
2. Operator 체인 중간에서 익는 방식 : `transformDeferredContextual`

11-1 예제에서 두 case 모두 볼 수 있는데, `deferContextual` 같은 경우 Context 에 저장된 데이터와 원본 데이터 소스의 처리를 지연시키는 역할을 합니다. 또한 파라미터로 정의된 람다 표현식의 람다 파라미터 ctx 는 Context 타입이 아니라 ContextView 타입인데, Context 에 데이터를 읽을 때는 이를 사용해야 한다는 것을 알 수 있습니다. (쓸 때는 그냥 Context)

### 11.2 자주 사용되는 Context 관련 API 

1. `put(key, value)` : key / value 형태로 Context 에 값을 쓴다.

2. `of(key1, value2, key2, value2...)` : key / value 형태로 Context에 여러개의 값을 쓴다 

3. `putAll(contextView)` : 현재 Context 와 파라미터로 입력된 ContextView 를 merge 한다

4. `delete(key)` : Context 에서 key 에 해당하는 value 를 삭제한다

#### ContextView API

1. `get(key)` : ContextView 에서 key 에 해당하는 value 를 반환한다

2. `getOrEmpty(key)` : ContextView 에서 key 에 해당하는 value 를 Optional 로 래핑해서 반환한다

3. `getOrDefault(key, default value)` : ContextView 에서 key 에 해당하는 value 를 가져오고, 해당하는 value 가 없으면 기본값 제공 

4. `hasKey(key)` : ContextView 에서 특정 key 가 존재하는지 확인한다.

5. `isEmpty` : Context 가 비어있는지 확인한다.

6. `size` : Context 내에 있는 key/value 의 개수를 반환한다. 

### 11.3 Context 의 특징 

- Context 는 구독이 발생할 때 마다 하나의 Context 가 해당 구독에 연결된다.

~~~java
/**
 * Context의 특징 예제
 *  - Context는 각각의 구독을 통해 Reactor Sequence에 연결 되며 체인의 각 Operator는 연결된 Context에 접근할 수 있어야 한다.
 */
@Slf4j
public class Example11_5 {
    public static void main(String[] args) throws InterruptedException {
        final String key1 = "company";

        Mono<String> mono = Mono.deferContextual(ctx ->
                        Mono.just("Company: " + " " + ctx.get(key1))
                )
                .publishOn(Schedulers.parallel());


        mono.contextWrite(context -> context.put(key1, "Apple"))
                .subscribe(data -> log.info("# subscribe1 onNext: {}", data));

        mono.contextWrite(context -> context.put(key1, "Microsoft"))
                .subscribe(data -> log.info("# subscribe2 onNext: {}", data));

        Thread.sleep(100L);
    }
}
~~~

- Context 는 Operator 체인의 아래에서 위로 전파된다.
- 동일한 key 에 대해 값을 중복 저장시, Operator 체인에서 가장 위쪽에 위치한 contextWrite() 이 저장한 값으로 덮어쓴다.

~~~java
/**
 * Context의 특징 예제
 *  - Context는 Operator 체인의 아래에서부터 위로 전파된다.
 *      - 따라서 Operator 체인 상에서 Context read 메서드가 Context write 메서드 밑에 있을 경우에는 write된 값을 read할 수 없다.
 */
@Slf4j
public class Example11_6 {
    public static void main(String[] args) throws InterruptedException {
        String key1 = "company";
        String key2 = "name";

        Mono
            .deferContextual(ctx ->
                Mono.just(ctx.get(key1))
            )
            .publishOn(Schedulers.parallel())
            .contextWrite(context -> context.put(key2, "Bill"))
            .transformDeferredContextual((mono, ctx) ->
                    mono.map(data -> data + ", " + ctx.getOrDefault(key2, "Steve"))
            )
            .contextWrite(context -> context.put(key1, "Apple"))
            .subscribe(data -> log.info("# onNext: {}", data));

        Thread.sleep(100L);
    }
}
~~~

- Inner Sequence 내부에서는 외부 Context 에 저장된 데이터를 읽을 수 없다.
- Inner Sequence 외부에서는 Inner Sequence 내부 Context 에 저장된 데이터를 읽을 수 없다. (마치 프로그래밍 언어의 scope 같은 느낌)

~~~java
/**
 * Context의 특징
 *  - inner Sequence 내부에서는 외부 Context에 저장된 데이터를 읽을 수 있다.
 *  - inner Sequence 외부에서는 inner Sequence 내부 Context에 저장된 데이터를  읽을 수 없다.
 */
@Slf4j
public class Example11_7 {
    public static void main(String[] args) throws InterruptedException {
        String key1 = "company";
        Mono
            .just("Steve")
            .transformDeferredContextual((stringMono, ctx) ->
                    ctx.get("role"))
            .flatMap(name ->
                Mono.deferContextual(ctx ->
                    Mono
                        .just(ctx.get(key1) + ", " + name)
                        .transformDeferredContextual((mono, innerCtx) ->
                                mono.map(data -> data + ", " + innerCtx.get("role"))
                        )
                        .contextWrite(context -> context.put("role", "CEO"))
                )
            )
            .publishOn(Schedulers.parallel())
            .contextWrite(context -> context.put(key1, "Apple"))
            .subscribe(data -> log.info("# onNext: {}", data));

        Thread.sleep(100L);
    }
}
~~~

위의 코드에서 flatMap 내부 시퀀스에서 쓴 Context 의 role 값은 외부에서 읽지 못한다. 


#### Context 활용 예제 

~~~java
/**
 * Context 활용 예제
 *  - 직교성을 가지는 정보를 표현할 때 주로 사용된다.
 */
@Slf4j
public class Example11_8 {
    public static final String HEADER_AUTH_TOKEN = "authToken";
    public static void main(String[] args) {
        Mono<String> mono =
                postBook(Mono.just(
                        new Book("abcd-1111-3533-2809"
                                , "Reactor's Bible"
                                ,"Kevin"))
                )
                .contextWrite(Context.of(HEADER_AUTH_TOKEN, "eyJhbGciOi"));

        mono.subscribe(data -> log.info("# onNext: {}", data));

    }

    private static Mono<String> postBook(Mono<Book> book) {
        return Mono
                .zip(book,
                        Mono
                            .deferContextual(ctx ->
                                    Mono.just(ctx.get(HEADER_AUTH_TOKEN)))
                )
                .flatMap(tuple -> {
                    String response = "POST the book(" + tuple.getT1().getBookName() +
                            "," + tuple.getT1().getAuthor() + ") with token: " +
                            tuple.getT2();
                    return Mono.just(response); // HTTP POST 전송을 했다고 가정
                });
    }
}

@AllArgsConstructor
@Data
class Book {
    private String isbn;
    private String bookName;
    private String author;
}
~~~

위 코드의 main 시퀀스의 가장 아래쪽에서 contextWrite 로 토큰 정보를 저장하기 때문에 체인 어느 위치에서든 Context 에 접근할 수 있다.

이렇게 Context 는 이렇게 인증 정보 같은 직교성(독립성) 을 가지는 정보를 전송하는데 적합하다. 