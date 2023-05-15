---
title: Chapter 5. 리액티브 개요 
date: "2023-05-12T23:46:37.121Z"
template: "post"
draft: false
slug: "/posts/reactive-ch5"
category: "devlog"
tags:
  - "스프링으로 시작하는 리액티브 프로그래밍"
  - "리액티브 프로그래밍"
description: "책 읽은거 정리하기, 스프링으로 시작하는 리액티브 프로그래밍 "
socialImage: "./media/reactive-str.png"
---

## 5. 리액티브 개요 

### 5.1 Reactor 란?
Spring 팀의 주도하에 만들어진 리액티브 스트림즈의 구현체이며,  
Reactor Core 라이브러리는 Spring WebFlux 프레임워크에 포함되어 있습니다.

Reactor 의 특징은 아래와 같습니다. 

#### 1. Reactive Streams 
Reactor 은 리액티브 스트림즈를 구현한 구현체입니다. 

> Reactive Streams :  
> 데이터 스트림을 Non-blocking + Async 하게 처리하기 위하여 정의해 놓은 인터페이스 

#### 2. Non-Blocking  
블로킹 방식과 달리, 작업이 완료가 되지 않았어도 프로그램의 제어권을 메인루틴에게 바로 돌려줍니다.  
바로 돌려주기 때문에 메인루틴은 서브루틴이 종료되지 않아도 자신이 할 일을 진행할 수 있습니다.

#### 3. Java's Functional API 
구독자와 발행자 간 상호작용은 Java 의 함수형 프로그래밍 API 를 통하여 이루어 집니다. 

#### 4. Flux[N]
Reactor 의 Pulisher 타입은 두가지(Flux, Mono) 중 하나로 무한대의 데이터를 emit 할 수 있는 Publisher 입니다. 

#### 5. Mono[0|1]
Flux 와 마찬가지로 Publisher 의 타입 중 하나로, 괄호 안 숫자처럼 emit 하지 않거나 1개만 emit 하는 Publisher 입니다.

#### 6. Well suited for microservice 

#### 7. BackPressure ready network 
발행자로부터 처리된 데이터를 처리하는데 있어 과부하가 걸리지 않도록 배압(BackPressure) 조절을 지원합니다. 

### 5.2 Hello Reactor 코드로 보는 Reactor 의 구성요소 

~~~java
public static void main(String[] args) {
  Flux<String> sequence = Flux.just("Hello", "Reactor"); 
  sequence.map(data -> data.toLowerCase()) 
    .subscribe(data -> system.out.println(data));
}
~~~

1. publisher Flux 는 데이터 소스 (`Flux.just("Hello", "Reactor");`) 로부터 데이터를 발행합니다.
2. subscribe 메서드로 전달된 람다 표현식인 subscriber(`data -> system.out.println(data)`) 는 퍼블리셔로 부터 발행된 데이터를 처리합니다.
3. just 메서드와 map 메서드는 리액터에서 지원하는 Operator 메서드로
- just 는 데이터를 생성하여 제공하는 역할을 하고
- map 은 전달받은 데이터를 가공합니다 (소문자로 변경)