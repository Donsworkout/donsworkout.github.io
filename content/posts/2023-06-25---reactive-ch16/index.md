---
title: Chapter 16. 애너테이션 기반 컨트롤러
date: "2023-06-25T23:49:37.121Z"
template: "post"
draft: false
slug: "/posts/reactive-ch16"
category: "devlog"
tags:
  - "스프링으로 시작하는 리액티브 프로그래밍"
  - "리액티브 프로그래밍"
description: "책 읽은거 정리하기, 스프링으로 시작하는 리액티브 프로그래밍 "
---

## 16. 애너테이션 기반 컨트롤러

### 16.1 Spring MVC 기반 컨트롤러

[코드 16.1]
~~~java
@RestController
@RequestMapping("/v1/mvc/books")
public class BookMvcController {
    private final BookMvcService bookMvcService;
    private final BookMvcMapper mapper;

    public BookMvcController(BookMvcService bookMvcService, BookMvcMapper mapper) {
        this.bookMvcService = bookMvcService;
        this.mapper = mapper;
    }

    @PostMapping
    public ResponseEntity postBook(@RequestBody BookDto.Post requestBody) {
        Book book = bookMvcService.createBook(mapper.bookPostToBook(requestBody));
        return ResponseEntity.ok(mapper.bookToBookResponse(book));
    }

    @PatchMapping("/{book-id}")
    public ResponseEntity patchBook(@PathVariable("book-id") long bookId,
                                    @RequestBody BookDto.Patch requestBody) {
        requestBody.setBookId(bookId);
        Book book =
                bookMvcService.updateBook(mapper.bookPatchToBook(requestBody));
        return ResponseEntity.ok(mapper.bookToBookResponse(book));
    }

    @GetMapping("/{book-id}")
    public ResponseEntity getBook(@PathVariable("book-id") long bookId) {
        Book book = bookMvcService.findBook(bookId);
        return ResponseEntity.ok(mapper.bookToBookResponse(book));
    }
}
~~~

### 16.2 Spring WebFlux 기반 Controller

#### 애너테이션 기반 컨트롤러 

[코드 16.2]
~~~java
@RestController
@RequestMapping("/v1/books")
public class BookController {
    private final BookService bookService;
    private final BookMapper mapper;

    public BookController(BookService bookService, BookMapper mapper) {
        this.bookService = bookService;
        this.mapper = mapper;
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Mono postBook(@RequestBody BookDto.Post requestBody) {
        Mono<Book> book =
                bookService.createBook(mapper.bookPostToBook(requestBody));

        Mono<BookDto.Response> response = mapper.bookToBookResponse(book);
        return response;
    }

    @PatchMapping("/{book-id}")
    public Mono patchBook(@PathVariable("book-id") long bookId,
                                    @RequestBody BookDto.Patch requestBody) {
        requestBody.setBookId(bookId);
        Mono<Book> book =
                bookService.updateBook(mapper.bookPatchToBook(requestBody));

        return mapper.bookToBookResponse(book);
    }

    @GetMapping("/{book-id}")
    public Mono getBook(@PathVariable("book-id") long bookId) {
        Mono<Book> book = bookService.findBook(bookId);

        return mapper.bookToBookResponse(book);
    }
}
~~~

- 코드상으로는 MVC 와 Mono 타입인 차이밖에 없음 
- 애너테이션 기반 컨트롤러는 기존 Spring MVC 구조와 별 차이 없음
- 그래도 논블로킹을 지원하는 리액티브 핸들러임 

[코드 16.3]
~~~java
@Service
public class BookService {
    public Mono<Book> createBook(Book book) {
        // not implement business logic;
        return Mono.just(book);
    }

    public Mono<Book> updateBook(Book book) {
        // not implement business logic;
        return Mono.just(book);
    }

    public Mono<Book> findBook(long bookId) {
        return Mono.just(
                    new Book(bookId,
                            "Java 고급",
                            "Advanced Java",
                            "Kevin",
                            "111-11-1111-111-1",
                            "Java 중급 프로그래밍 마스터",
                            "2022-03-22",
                            LocalDateTime.now(),
                            LocalDateTime.now())
        );
    }
}
~~~

- 서비스 코드
- 더미데이터 리턴

[코드 16.4]
~~~java
@Mapper(componentModel = "spring")
public interface BookMapper {
    Book bookPostToBook(BookDto.Post requestBody);
    Book bookPatchToBook(BookDto.Patch requestBody);
    BookDto.Response bookToResponse(Book book);
    default Mono<BookDto.Response> bookToBookResponse(Mono<Book> mono) {
        return mono.flatMap(book -> Mono.just(bookToResponse(book)));
    }
}
~~~

- DTO 클래스 객체를 Book 엔티티 클래스의 객체로 변환해 주는 BookMapper 코드

#### 코드 16.2 에서 블로킹 요소 제거한 컨트롤러 코드

[코드 16.5]
~~~java
@RestController("bookControllerV2")
@RequestMapping("/v2/books")
public class BookController {
    private final BookService bookService;
    private final BookMapper mapper;

    public BookController(BookService bookService, BookMapper mapper) {
        this.bookService = bookService;
        this.mapper = mapper;
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Mono postBook(@RequestBody Mono<BookDto.Post> requestBody) {
        Mono<Book> result = bookService.createBook(requestBody);

        return result.flatMap(book -> Mono.just(mapper.bookToResponse(book)));
    }

    @PatchMapping("/{book-id}")
    public Mono patchBook(@PathVariable("book-id") long bookId,
                                    @RequestBody Mono<BookDto.Patch> requestBody) {
        Mono<Book> result = bookService.updateBook(bookId, requestBody);
        return result.flatMap(book -> Mono.just(mapper.bookToResponse(book)));
    }

    @GetMapping("/{book-id}")
    public Mono getBook(@PathVariable("book-id") long bookId) {
        return bookService.findBook(bookId)
                .flatMap(book -> Mono.just(mapper.bookToResponse(book)));
    }
}
~~~

- 근데 16.2 도 논블로킹 아닌가..?

#### 코드 16.3 에서 블로킹 요소 제거한 서비스 코드

[코드 16.6]
~~~java
@Service("bookServiceV2")
public class BookService {
    private final BookMapper mapper;

    public BookService(BookMapper mapper) {
        this.mapper = mapper;
    }

    public Mono<Book> createBook(Mono<BookDto.Post> book) {
        // not implement business logic;
        return book.flatMap(post -> Mono.just(mapper.bookPostToBook(post)));
    }

    public Mono<Book> updateBook(final long bookId, Mono<BookDto.Patch> book) {
        // not implement business logic;
        return book.flatMap(patch -> {
            patch.setBookId(bookId);
            return Mono.just(mapper.bookPatchToBook(patch));
        });
    }

    public Mono<Book> findBook(long bookId) {
        return Mono.just(
                    new Book(bookId,
                            "Java 고급",
                            "Advanced Java",
                            "Kevin",
                            "111-11-1111-111-1",
                            "Java 중급 프로그래밍 마스터",
                            "2022-03-22",
                            LocalDateTime.now(),
                            LocalDateTime.now())
        );
    }
}
~~~ 

- 근데 16.3 도 논블로킹 아닌가..?
- 파라미터도 Mono 로 받아서 flatMap 등 오퍼레이팅 하는 것 을 알 수 있음