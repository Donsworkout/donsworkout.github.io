---
title: Chapter 17. 함수형 엔드포인트
date: "2023-06-25T23:50:37.121Z"
template: "post"
draft: false
slug: "/posts/reactive-ch17"
category: "devlog"
tags:
  - "스프링으로 시작하는 리액티브 프로그래밍"
  - "리액티브 프로그래밍"
description: "책 읽은거 정리하기, 스프링으로 시작하는 리액티브 프로그래밍 "
---

## 17. 함수형 엔드포인트 
### 17.1 HandlerFunction 을 사용한 request 처리
함수형 기반의 핸들러

[코드 17.1]
~~~java
@FunctionalInterface
public interface HandlerFunction<T extends ServerResponse> {
  Mono<T> handle(ServerRequest request);
}
~~~

- handle 메서드 하나만 있음
- 파라미터로 ServerRequest 하나만 받고, 응답은 Mono<ServerResponse> 형태 

#### ServerRequest
- Http Request 표현

#### ServerResponse
- Http Response 표현 

[참고]  
HandlerFunction 은 RouterFunction 을 통해 요청이 라우팅된 이후에 동작함

### 17.2 request 라우팅을 위한 RouterFunction

- RouterFunction 은 요청을 해당 HandlerFunction 으로 라우팅
- RouterFunction 은 `@RequestMapping` 어노테이션과 동일한 역할 

[코드 17.2]
~~~java
@FunctionalInterface
public interface RouterFunction<T extends ServerResponse> {
  Optional<HandlerFunction<T>> route(ServerRequest request);

  ...
}
~~~

- RouterFunction 인터페이스는 route() 메서드 하나만 정의된 함수형 인터페이스
- route() 메서드에서 파라미터로 전달받은 request 에 매치되는 핸들러 함수를 리턴해줌
- 파라미터로 ServerRequest 하나만 받고, 응답은 매치되는 HandlerFunction 리턴 

[코드 17.3]
~~~java
@Configuration("bookRouterV1")
public class BookRouter {
    @Bean
    public RouterFunction<?> routeBookV1(BookHandler handler) {
        return route()
                .POST("/v1/books", handler::createBook)
                .PATCH("/v1/books/{book-id}", handler::updateBook)
                .GET("/v1/books", handler::getBooks)
                .GET("/v1/books/{book-id}", handler::getBook)
                .build();
    }
}
~~~

- 각 url 패턴에 맞는 HandlerFunction 등록

### HandlerFunction 예제 

[코드 17.4]
~~~java
@Component("bookHandlerV1")
public class BookHandler {
    private final BookMapper mapper;

    public BookHandler(BookMapper mapper) {
        this.mapper = mapper;
    }

    public Mono<ServerResponse> createBook(ServerRequest request) {
        return request.bodyToMono(BookDto.Post.class)
                .map(post -> mapper.bookPostToBook(post))
                .flatMap(book ->
                        ServerResponse
                                .created(URI.create("/v1/books/" + book.getBookId()))
                                .build());
    }

    public Mono<ServerResponse> getBook(ServerRequest request) {
        long bookId = Long.valueOf(request.pathVariable("book-id"));
        Book book =
                new Book(bookId,
                        "Java 고급",
                        "Advanced Java",
                        "Kevin",
                        "111-11-1111-111-1",
                        "Java 중급 프로그래밍 마스터",
                        "2022-03-22",
                        LocalDateTime.now(),
                        LocalDateTime.now());
        return ServerResponse
                            .ok()
                            .bodyValue(mapper.bookToResponse(book))
                            .switchIfEmpty(ServerResponse.notFound().build());
    }

    public Mono<ServerResponse> updateBook(ServerRequest request) {
        final long bookId = Long.valueOf(request.pathVariable("book-id"));
        return request
                .bodyToMono(BookDto.Patch.class)
                .map(patch -> {
                    patch.setBookId(bookId);
                    return mapper.bookPatchToBook(patch);
                })
                .flatMap(book -> ServerResponse.ok()
                        .bodyValue(mapper.bookToResponse(book)));
    }

    public Mono<ServerResponse> getBooks(ServerRequest request) {
        List<Book> books = List.of(
                new Book(1L,
                        "Java 고급",
                        "Advanced Java",
                        "Kevin",
                        "111-11-1111-111-1",
                        "Java 중급 프로그래밍 마스터",
                        "2022-03-22",
                        LocalDateTime.now(),
                        LocalDateTime.now()),
                new Book(2L,
                        "Kotlin 고급",
                        "Advanced Kotlin",
                        "Kevin",
                        "222-22-2222-222-2",
                        "Kotlin 중급 프로그래밍 마스터",
                        "2022-05-22",
                        LocalDateTime.now(),
                        LocalDateTime.now())
        );
        return ServerResponse
                .ok()
                .bodyValue(mapper.booksToResponse(books));
    }
}
~~~

### 17.3 함수형 엔드포인트에서 request body 유효성 검증 

- 유효성 검증 필요시 Spring Validator 인터페이스를 구현한 커스텀 밸리데이터를 이용해 request body 유효성 검증 가능  

[코드 17.5]  
도서 정보 저장을 위한 커스텀 밸리데이터 
~~~java
@Component("bookValidatorV2")
public class BookValidator implements Validator {
    @Override
    public boolean supports(Class<?> clazz) {
        return BookDto.Post.class.isAssignableFrom(clazz);
    }

    @Override
    public void validate(Object target, Errors errors) {
        BookDto.Post post = (BookDto.Post) target;

        ValidationUtils.rejectIfEmptyOrWhitespace(
                errors, "titleKorean", "field.required");

        ValidationUtils.rejectIfEmptyOrWhitespace(
                errors, "titleEnglish", "field.required");
    }
}
~~~

- Spring 에서 지원하는 Validator 인터페이스를 구현한 방식 

#### BookValidator 을 핸들러 함수에 적용한 예시 

[코드 17.6]
~~~java
@Slf4j
@Component("bookHandlerV2")
public class BookHandler {
    private final BookMapper mapper;
    private final BookValidator validator;

    public BookHandler(BookMapper mapper, BookValidator validator) {
        this.mapper = mapper;
        this.validator = validator;
    }

    public Mono<ServerResponse> createBook(ServerRequest request) {
        return request.bodyToMono(BookDto.Post.class)
                .doOnNext(post -> this.validate(post))
                .map(post -> mapper.bookPostToBook(post))
                .flatMap(book ->
                        ServerResponse
                                .created(URI.create("/v2/books/" + book.getBookId()))
                                .build());
    }

    public Mono<ServerResponse> patchBook(ServerRequest request) {
        final long bookId = Long.valueOf(request.pathVariable("book-id"));
        return request
                .bodyToMono(BookDto.Patch.class)
                .map(patch -> {
                    patch.setBookId(bookId);
                    return mapper.bookPatchToBook(patch);
                })
                .flatMap(book -> ServerResponse.ok()
                        .bodyValue(mapper.bookToResponse(book)));
    }

    public Mono<ServerResponse> getBook(ServerRequest request) {
        long bookId = Long.valueOf(request.pathVariable("book-id"));
        Book book =
                new Book(bookId,
                        "Java 고급",
                        "Advanced Java",
                        "Kevin",
                        "111-11-1111-111-1",
                        "Java 중급 프로그래밍 마스터",
                        "2022-03-22",
                        LocalDateTime.now(),
                        LocalDateTime.now());
        return ServerResponse
                            .ok()
                            .bodyValue(mapper.bookToResponse(book))
                            .switchIfEmpty(ServerResponse.notFound().build());
    }

    public Mono<ServerResponse> getBooks(ServerRequest request) {
        List<Book> books = List.of(
                new Book(1L,
                        "Java 고급",
                        "Advanced Java",
                        "Kevin",
                        "111-11-1111-111-1",
                        "Java 중급 프로그래밍 마스터",
                        "2022-03-22",
                        LocalDateTime.now(),
                        LocalDateTime.now()),
                new Book(2L,
                        "Kotlin 고급",
                        "Advanced Kotlin",
                        "Kevin",
                        "222-22-2222-222-2",
                        "Kotlin 중급 프로그래밍 마스터",
                        "2022-05-22",
                        LocalDateTime.now(),
                        LocalDateTime.now())
        );
        return ServerResponse
                .ok()
                .bodyValue(mapper.booksToResponse(books));
    }

    private void validate(BookDto.Post post) {
        Errors errors = new BeanPropertyBindingResult(post, BookDto.class.getName());
        validator.validate(post, errors);
        if (errors.hasErrors()) {
            log.error(errors.getAllErrors().toString());
            throw new ServerWebInputException(errors.toString());
        }
    }
}
~~~
- operator 체인 내에서 유효성 검증 

위의 코드에서는 커스텀 밸리데이터를 이용하여 유효성을 검증했으나, 사실 이는 좋지 않은 방법임 (비즈니스 로직이 들어가 있음)

따라서 스프링에서 제공하는 표준 Bean validation 인터페이스로 유효성 검증하는것을 추천 

#### Spring Validator 인터페이스 사용 

[코드 17.7]
~~~java
@Slf4j
@Component("bookValidatorV3")
public class BookValidator<T> {
    private final Validator validator;

    public BookValidator(@Qualifier("springValidator") Validator validator) {
        this.validator = validator;
    }

    public void validate(T body) {
        Errors errors =
                new BeanPropertyBindingResult(body, body.getClass().getName());

        this.validator.validate(body, errors);

        if (!errors.getAllErrors().isEmpty()) {
            onValidationErrors(errors);
        }
    }

    private void onValidationErrors(Errors errors) {
        log.error(errors.getAllErrors().toString());
        throw new ResponseStatusException(HttpStatus.BAD_REQUEST, errors.getAllErrors()
                .toString());
    }
}
~~~

#### javax 표준 Validator 인터페이스 사용 

[코드 17.8]
~~~java
@Slf4j
@Component("bookValidatorV4")
public class BookValidator<T> {
    private final Validator validator;

    public BookValidator(@Qualifier("javaxValidator") Validator validator) {
        this.validator = validator;
    }

    public void validate(T body) {
        Set<ConstraintViolation<T>> constraintViolations = validator.validate(body);
        if (!constraintViolations.isEmpty()) {
            onValidationErrors(constraintViolations);
        }
    }

    private void onValidationErrors(Set<ConstraintViolation<T>> constraintViolations) {
        log.error(constraintViolations.toString());
        throw new ResponseStatusException(HttpStatus.BAD_REQUEST,
                                            constraintViolations.toString());
    }
}
~~~


#### javax 표준 Validator 유효성 검증이 적용된 핸들러  

[코드 17.9]
~~~java
@Slf4j
@Component("bookHandlerV4")
public class BookHandler {
    private final BookMapper mapper;
    private final BookValidator validator;

    public BookHandler(BookMapper mapper, BookValidator validator) {
        this.mapper = mapper;
        this.validator = validator;
    }

    public Mono<ServerResponse> createBook(ServerRequest request) {
        return request.bodyToMono(BookDto.Post.class)
                .doOnNext(post -> validator.validate(post))
                .map(post -> mapper.bookPostToBook(post))
                .flatMap(book -> ServerResponse
                        .created(URI.create("/v4/books/" + book.getBookId()))
                        .build());

    }

    public Mono<ServerResponse> updateBook(ServerRequest request) {
        final long bookId = Long.valueOf(request.pathVariable("book-id"));
        return request
                .bodyToMono(BookDto.Patch.class)
                .doOnNext(patch -> validator.validate(patch))
                .map(patch -> {
                    patch.setBookId(bookId);
                    return mapper.bookPatchToBook(patch);
                })
                .flatMap(book -> ServerResponse.ok()
                        .bodyValue(mapper.bookToResponse(book)));
    }

    public Mono<ServerResponse> getBook(ServerRequest request) {
        long bookId = Long.valueOf(request.pathVariable("book-id"));
        Book book =
                new Book(bookId,
                        "Java 고급",
                        "Advanced Java",
                        "Kevin",
                        "111-11-1111-111-1",
                        "Java 중급 프로그래밍 마스터",
                        "2022-03-22",
                        LocalDateTime.now(),
                        LocalDateTime.now());
        return ServerResponse
                .ok()
                .bodyValue(mapper.bookToResponse(book))
                .switchIfEmpty(ServerResponse.notFound().build());
    }

    public Mono<ServerResponse> getBooks(ServerRequest request) {
        List<Book> books = List.of(
                new Book(1L,
                        "Java 고급",
                        "Advanced Java",
                        "Kevin",
                        "111-11-1111-111-1",
                        "Java 중급 프로그래밍 마스터",
                        "2022-03-22",
                        LocalDateTime.now(),
                        LocalDateTime.now()),
                new Book(2L,
                        "Kotlin 고급",
                        "Advanced Kotlin",
                        "Kevin",
                        "222-22-2222-222-2",
                        "Kotlin 중급 프로그래밍 마스터",
                        "2022-05-22",
                        LocalDateTime.now(),
                        LocalDateTime.now())
        );
        return ServerResponse
                .ok()
                .bodyValue(mapper.booksToResponse(books));
    }
}
~~~