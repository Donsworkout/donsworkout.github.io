---
title: Chapter 20. WebClient
date: "2023-06-25T23:53:37.121Z"
template: "post"
draft: false
slug: "/posts/reactive-ch20"
category: "devlog"
tags:
  - "스프링으로 시작하는 리액티브 프로그래밍"
  - "리액티브 프로그래밍"
description: "책 읽은거 정리하기, 스프링으로 시작하는 리액티브 프로그래밍 "
---

## 20. WebClient
### 20.1 WebClient란?

- 논블로킹 Http Request 를 위한 리액티브 웹 클라이언트 
- 기본 클라이언트 라이브러리는 Netty
- 블로킹도 지원하기 때문에 슬슬 RestTemplate 을 대체할 예정

### 20.2 WebClient 로 도서 정보 요청하기

[코드 20.1]
~~~java
@Slf4j
@Configuration
public class WebClientExample01 {
    @Bean
    public ApplicationRunner examplesWebClient() {

        return (ApplicationArguments arguments) -> {
            exampleWebClient01();
            exampleWebClient02();
            exampleWebClient03();
            exampleWebClient04();
        };
    }

    private void exampleWebClient01() {
        BookDto.Post requestBody = new BookDto.Post("Java 중급",
                "Intermediate Java",
                "Java 중급 프로그래밍 마스터",
                "Kevin1", "222-22-2222-222-2",
                "2022-03-22");

        WebClient webClient = WebClient.create();
        Mono<ResponseEntity<Void>> response =
                webClient
                        .post()
                        .uri("http://localhost:8080/v10/books")
                        .bodyValue(requestBody)
                        .retrieve()
                        .toEntity(Void.class);

        response.subscribe(res -> {
           log.info("response status: {}", res.getStatusCode());
           log.info("Header Location: {}", res.getHeaders().get("Location"));
        });
    }

    private void exampleWebClient02() {
        BookDto.Patch requestBody =
                new BookDto.Patch.PatchBuilder().titleKorean("Java 고급")
                .titleEnglish("Advanced Java")
                .description("Java 고급 프로그래밍 마스터")
                .author("Tom")
                .build();

        WebClient webClient = WebClient.create("http://localhost:8080");
        Mono<BookDto.Response> response =
                webClient
                        .patch()
                        .uri("http://localhost:8080/v10/books/{book-id}", 20)
                        .bodyValue(requestBody)
                        .retrieve()
                        .bodyToMono(BookDto.Response.class);

        response.subscribe(book -> {
            log.info("bookId: {}", book.getBookId());
            log.info("titleKorean: {}", book.getTitleKorean());
            log.info("titleEnglish: {}", book.getTitleEnglish());
            log.info("description: {}", book.getDescription());
            log.info("author: {}", book.getAuthor());
        });
    }

    private void exampleWebClient03() {
        Mono<BookDto.Response> response =
                WebClient
                        .create("http://localhost:8080")
                        .get()
                        .uri(uriBuilder -> uriBuilder
                                .path("/v10/books/{book-id}")
                                .build(21))
                                .retrieve()
                                .bodyToMono(BookDto.Response.class);

        response.subscribe(book -> {
            log.info("bookId: {}", book.getBookId());
            log.info("titleKorean: {}", book.getTitleKorean());
            log.info("titleEnglish: {}", book.getTitleEnglish());
            log.info("description: {}", book.getDescription());
            log.info("author: {}", book.getAuthor());
        });
    }

    private void exampleWebClient04() {
        Flux<BookDto.Response> response =
                WebClient
                        .create("http://localhost:8080")
                        .get()
                        .uri(uriBuilder -> uriBuilder
                                .path("/v10/books")
                                .queryParam("page", "1")
                                .queryParam("size", "10")
                                .build())
                        .retrieve()
                        .bodyToFlux(BookDto.Response.class);

        response
                .map(book -> book.getTitleKorean())
                .subscribe(bookName -> log.info("book name: {}", bookName));
    }
}
~~~

### 20.3 WebClient Connection Timeout 설정

[코드 20.2]
~~~java
@Slf4j
@Configuration
public class WebClientExample02 {
    @Bean
    public ApplicationRunner examplesWebClient02() {

        return (ApplicationArguments arguments) -> {
            exampleWebClient01();
        };
    }

    private void exampleWebClient01() {
        HttpClient httpClient =
                HttpClient
                        .create()
                        .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 500)
                        .responseTimeout(Duration.ofMillis(500))
                        .doOnConnected(connection ->
                            connection
                                    .addHandlerLast(
                                            new ReadTimeoutHandler(500,
                                                                TimeUnit.MILLISECONDS))
                                    .addHandlerLast(
                                            new WriteTimeoutHandler(500,
                                                                TimeUnit.MILLISECONDS)));

        Flux<BookDto.Response> response =
                WebClient
                        .builder()
                        .baseUrl("http://localhost:8080")
                        .clientConnector(new ReactorClientHttpConnector(httpClient))
                        .build()
                        .get()
                        .uri(uriBuilder -> uriBuilder
                                .path("/v10/books")
                                .queryParam("page", "1")
                                .queryParam("size", "10")
                                .build())
                        .retrieve()
                        .bodyToFlux(BookDto.Response.class);

        response
                .map(book -> book.getTitleKorean())
                .subscribe(bookName -> log.info("book name2: {}", bookName));
    }
}
~~~

### 20.4 exchangeToMono() 를 사용한 응답 디코딩
- retrieve 대신에 exchangeToMono, exchangeToFlux 메서드를 이용하면 response 를 사용자의 요구 조건에 맞게 제어 가능 

[코드 20.3]
~~~java
    private void exampleWebClient02() {
        BookDto.Post post = new BookDto.Post("Java 중급",
                "Intermediate Java",
                "Java 중급 프로그래밍 마스터",
                "Kevin1", "333-33-3333-333-3",
                "2022-03-22");
        WebClient webClient = WebClient.create();
        webClient
                .post()
                .uri("http://localhost:8080/v10/books")
                .bodyValue(post)
                .exchangeToMono(response -> {
                    if(response.statusCode().equals(HttpStatus.CREATED))
                        return response.toEntity(Void.class);
                    else
                        return response
                                .createException()
                                .flatMap(throwable -> Mono.error(throwable));
                })
                .subscribe(res -> {
                    log.info("response status2: {}", res.getStatusCode());
                    log.info("Header Location2: {}", res.getHeaders().get("Location"));
                    },
                    error -> log.error("Error happened: ", error));
    }
~~~