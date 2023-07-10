---
title: Chapter 19. 예외 처리
date: "2023-06-25T23:52:37.121Z"
template: "post"
draft: false
slug: "/posts/reactive-ch19"
category: "devlog"
tags:
  - "스프링으로 시작하는 리액티브 프로그래밍"
  - "리액티브 프로그래밍"
description: "책 읽은거 정리하기, 스프링으로 시작하는 리액티브 프로그래밍 "
---

## 19. 예외 처리
### 19.1 onErrorResume() Operator 을 이용한 예외처리
- Reactor 은 에러 처리를 위한 다양한 Operator 을 제공
- 그 중 onErrorResume 오퍼레이터는 에러 이벤트가 발생할 시 DownStream 으로 전파하지 않고 대체 값을 emit 하거나 발생한 에러 이벤트를 래핑한 후에 다시 에러 이벤트를 발생시키는 일을 함 

[코드 19.1]
~~~java
@Slf4j
@Component("BookHandlerV9")
public class BookHandler {
    private final BookMapper mapper;
    private final BookValidator validator;
    private final BookService bookService;

    public BookHandler(BookMapper mapper, BookValidator validator, BookService bookService) {
        this.mapper = mapper;
        this.validator = validator;
        this.bookService = bookService;
    }

    public Mono<ServerResponse> createBook(ServerRequest request) {
        return request.bodyToMono(BookDto.Post.class)
                .doOnNext(post -> validator.validate(post))
                .flatMap(post -> bookService.createBook(mapper.bookPostToBook(post)))
                .flatMap(book -> ServerResponse
                        .created(URI.create("/v9/books/" + book.getBookId()))
                        .build())
                .onErrorResume(BusinessLogicException.class, error -> ServerResponse
                            .badRequest()
                            .bodyValue(new ErrorResponse(HttpStatus.BAD_REQUEST,
                                                            error.getMessage())))
                .onErrorResume(Exception.class, error ->
                        ServerResponse
                                .unprocessableEntity()
                                .bodyValue(
                                    new ErrorResponse(HttpStatus.INTERNAL_SERVER_ERROR,
                                                        error.getMessage())));
    }

    public Mono<ServerResponse> updateBook(ServerRequest request) {
        final long bookId = Long.valueOf(request.pathVariable("book-id"));
        return request
                .bodyToMono(BookDto.Patch.class)
                .doOnNext(patch -> validator.validate(patch))
                .flatMap(patch -> {
                    patch.setBookId(bookId);
                    return bookService.updateBook(mapper.bookPatchToBook(patch));
                })
                .flatMap(book -> ServerResponse.ok()
                                        .bodyValue(mapper.bookToResponse(book)))
                .onErrorResume(error -> ServerResponse
                        .badRequest()
                        .bodyValue(new ErrorResponse(HttpStatus.BAD_REQUEST,
                                error.getMessage())));
    }

    public Mono<ServerResponse> getBook(ServerRequest request) {
        long bookId = Long.valueOf(request.pathVariable("book-id"));

        return bookService.findBook(bookId)
                        .flatMap(book -> ServerResponse
                                .ok()
                                .bodyValue(mapper.bookToResponse(book)))
                        .onErrorResume(error -> ServerResponse
                                .badRequest()
                                .bodyValue(new ErrorResponse(HttpStatus.BAD_REQUEST,
                                        error.getMessage())));
    }

    public Mono<ServerResponse> getBooks(ServerRequest request) {
        Tuple2<Long, Long> pageAndSize = getPageAndSize(request);
        return bookService.findBooks(pageAndSize.getT1(), pageAndSize.getT2())
                .flatMap(books -> ServerResponse
                        .ok()
                        .bodyValue(mapper.booksToResponse(books)));
    }

    private Tuple2<Long, Long> getPageAndSize(ServerRequest request) {
        long page = request.queryParam("page").map(Long::parseLong).orElse(0L);
        long size = request.queryParam("size").map(Long::parseLong).orElse(0L);
        return Tuples.of(page, size);
    }
}
~~~

### 19.2 ErrorWebExceptionHandler를 이용한 글로벌 예외 처리
- onErrorResume() 을 반복 사용하지 않게 하기위해 사용 

[코드 19.2]
~~~java
@Order(-2)
@Configuration
public class GlobalWebExceptionHandler implements ErrorWebExceptionHandler {
    private final ObjectMapper objectMapper;

    public GlobalWebExceptionHandler(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    @Override
    public Mono<Void> handle(ServerWebExchange serverWebExchange,
                             Throwable throwable) {
        return handleException(serverWebExchange, throwable);
    }

    private Mono<Void> handleException(ServerWebExchange serverWebExchange,
                                       Throwable throwable) {
        ErrorResponse errorResponse = null;
        DataBuffer dataBuffer = null;

        DataBufferFactory bufferFactory =
                                serverWebExchange.getResponse().bufferFactory();
        serverWebExchange.getResponse().getHeaders()
                                        .setContentType(MediaType.APPLICATION_JSON);

        if (throwable instanceof BusinessLogicException) {
            BusinessLogicException ex = (BusinessLogicException) throwable;
            ExceptionCode exceptionCode = ex.getExceptionCode();
            errorResponse = ErrorResponse.of(exceptionCode.getStatus(),
                                                exceptionCode.getMessage());
            serverWebExchange.getResponse()
                        .setStatusCode(HttpStatus.valueOf(exceptionCode.getStatus()));
        } else if (throwable instanceof ResponseStatusException) {
            ResponseStatusException ex = (ResponseStatusException) throwable;
            errorResponse = ErrorResponse.of(ex.getStatus().value(), ex.getMessage());
            serverWebExchange.getResponse().setStatusCode(ex.getStatus());
        } else {
            errorResponse = ErrorResponse.of(HttpStatus.INTERNAL_SERVER_ERROR.value(),
                                                            throwable.getMessage());
            serverWebExchange.getResponse()
                                    .setStatusCode(HttpStatus.INTERNAL_SERVER_ERROR);
        }

        try {
            dataBuffer =
                    bufferFactory.wrap(objectMapper.writeValueAsBytes(errorResponse));
        } catch (JsonProcessingException e) {
            bufferFactory.wrap("".getBytes());
        }

        return serverWebExchange.getResponse().writeWith(Mono.just(dataBuffer));
    }
}
~~~

