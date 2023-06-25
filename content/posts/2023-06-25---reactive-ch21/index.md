---
title: Chapter 21. Reactive Streaming 데이터 처리 
date: "2023-06-25T23:54:37.121Z"
template: "post"
draft: false
slug: "/posts/reactive-ch21"
category: "devlog"
tags:
  - "스프링으로 시작하는 리액티브 프로그래밍"
  - "리액티브 프로그래밍"
description: "책 읽은거 정리하기, 스프링으로 시작하는 리액티브 프로그래밍 "
---

## 21. Reactive Streaming 데이터 처리 

> Spring WebFlux 는 SSE(Server Sent Event) 를 이용해 데이터를 스트리밍 할 수 있음, SSER 는 HTTP 연결을 통해 서버로부터 전송되는 업데이트 데이터를 지속적으로 수신할 수 있는 단방향 서버 푸시 기술 

[코드 21.1]
~~~java
/**
 * 페이지네이션 적용
 */
@Slf4j
@Validated
@Service
@RequiredArgsConstructor
public class BookService {
    private final @NonNull R2dbcEntityTemplate template;

    ...
    public Flux<Book> streamingBooks() {
        return template
                .select(Book.class)
                .all()
                .delayElements(Duration.ofSeconds(2L));
    }
    ...
}
~~~

[코드 21.2]
~~~java
@Configuration
public class BookRouter {
    @Bean
    public RouterFunction<?> routeBook(BookHandler handler) {
        return route()
                .POST("/v11/books", handler::createBook)
                .PATCH("/v11/books/{book-id}", handler::updateBook)
                .GET("/v11/books", handler::getBooks)
                .GET("/v11/books/{book-id}", handler::getBook)
                .build();
    }

    @Bean
    public RouterFunction<?> routeStreamingBook(BookService bookService,
                                                BookMapper mapper) {
        return route(RequestPredicates.GET("/v11/streaming-books"),
                request -> ServerResponse
                        .ok()
                        .contentType(MediaType.TEXT_EVENT_STREAM)
                        .body(bookService
                                        .streamingBooks()
                                        .map(book -> mapper.bookToResponse(book))
                                ,
                                BookDto.Response.class));
    }
}
~~~

- contentType 이 'text/event-stream' 이여야 함

[코드 21.3]
~~~java
@Slf4j
@Configuration
public class BookWebClient {
    @Bean
    public ApplicationRunner streamingBooks() {
        return (ApplicationArguments arguments) -> {
            WebClient webClient = WebClient.create("http://localhost:8080");
            Flux<BookDto.Response> response =
                    webClient
                            .get()
                            .uri("http://localhost:8080/v11/streaming-books")
                            .retrieve()
                            .bodyToFlux(BookDto.Response.class);

            response.subscribe(book -> {
                        log.info("bookId: {}", book.getBookId());
                        log.info("titleKorean: {}", book.getTitleKorean());
                        log.info("titleEnglish: {}", book.getTitleEnglish());
                        log.info("description: {}", book.getDescription());
                        log.info("author: {}", book.getAuthor());
                        log.info("isbn: {}", book.getIsbn());
                        log.info("publishDate: {}", book.getPublishDate());
                        log.info("=======================================");
                    },
                    error -> log.error("# error happened: ", error));
        };
    }
}
~~~