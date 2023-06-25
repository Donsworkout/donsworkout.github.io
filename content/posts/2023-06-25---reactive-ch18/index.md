---
title: Chapter 18. Spring Data R2DBC
date: "2023-06-25T23:51:37.121Z"
template: "post"
draft: false
slug: "/posts/reactive-ch18"
category: "devlog"
tags:
  - "스프링으로 시작하는 리액티브 프로그래밍"
  - "리액티브 프로그래밍"
description: "책 읽은거 정리하기, 스프링으로 시작하는 리액티브 프로그래밍 "
---

## 18. Spring Data R2DBC
### 18.1 R2DBC 란?
- RDB 에 리액티브 프로그래밍 API 를 제공하기 위한 스펙
- 드라이버 벤더가 구현하고 클라이언트가 사용하기 위한 SPI(Service Provider Interface)

R2DBC 등장으로 NoSql 뿐 아니라 RDB 에서도 완전한 논블로킹 시퀀스 구현 가능 

### 18.2 Spring Data R2DBC 란?
R2DBC 기반 Repository 를 더 쉽게 구현하게 해주는 Spring Data Famlily 프로젝트 일부

JPA 에서 제공하는 캐싱, lazy loading 등 기타 ORM 에서 가지고 있는 특징들이 제거됨 

### 18.3 Spring Data R2DBC 설정 

- 책 참조 

### 18.4 Spring Data R2DBC 에서의 도메인 엔티티 클래스 매핑 

[코드 18.2]
~~~java
@Getter
@AllArgsConstructor
@NoArgsConstructor
@Setter
public class Book {
    @Id
    private long bookId;
    private String titleKorean;
    private String titleEnglish;
    private String description;
    private String author;
    private String isbn;
    private String publishDate;

    @CreatedDate
    private LocalDateTime createdAt;

    @LastModifiedDate
    @Column("last_modified_at")
    private LocalDateTime modifiedAt;
}
~~~

- JPA 와 유사 
- @Id, @Table @CreatedDate @LastModifiedDate 등 어노테이션 지원

### 18.5 R2DBC Repositories 를 이용한 데이터 액세스 

[코드 18.3]
~~~java
@Repository("bookRepositoryV5")
public interface BookRepository extends ReactiveCrudRepository<Book, Long> {
    Mono<Book> findByIsbn(String isbn);
}
~~~

- 쿼리 리턴 타입이 Mono / Flux 
- ReactiveCrudRepository 상속

#### 서비스 클래스 구현 

[코드 18.4]
~~~java
@Slf4j
@Service("bookServiceV5")
@RequiredArgsConstructor
public class BookService {
    private final @NonNull BookRepository bookRepository;
    private final @NonNull CustomBeanUtils<Book> beanUtils;

    public Mono<Book> saveBook(Book book) {
        return verifyExistIsbn(book.getIsbn())
                .then(bookRepository.save(book));
    }

    public Mono<Book> updateBook(Book book) {
        return findVerifiedBook(book.getBookId())
                .map(findBook -> beanUtils.copyNonNullProperties(book, findBook))
                .flatMap(updatingBook -> bookRepository.save(updatingBook));
    }

    public Mono<Book> findBook(long bookId) {
        return findVerifiedBook(bookId);
    }

    public Mono<List<Book>> findBooks() {
        return bookRepository.findAll().collectList();
    }

    private Mono<Void> verifyExistIsbn(String isbn) {
        return bookRepository.findByIsbn(isbn)
                .flatMap(findBook -> {
                    if (findBook != null) {
                        return Mono.error(new BusinessLogicException(
                                                    ExceptionCode.BOOK_EXISTS));
                    }
                    return Mono.empty();
                });
    }

    private Mono<Book> findVerifiedBook(long bookId) {
        return bookRepository
                .findById(bookId)
                .switchIfEmpty(Mono.error(new BusinessLogicException(
                                                    ExceptionCode.BOOK_NOT_FOUND)));
    }
}
~~~

#### 핸들러 클래스 수정 

[코드 18.5]
~~~java
@Slf4j
@Component("BookHandlerV5")
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
                .flatMap(post -> bookService.saveBook(mapper.bookPostToBook(post)))
                .flatMap(book -> ServerResponse
                        .created(URI.create("/v5/books/" + book.getBookId()))
                        .build());
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
                                        .bodyValue(mapper.bookToResponse(book)));
    }

    public Mono<ServerResponse> getBook(ServerRequest request) {
        long bookId = Long.valueOf(request.pathVariable("book-id"));

        return bookService.findBook(bookId)
                        .flatMap(book -> ServerResponse
                                .ok()
                                .bodyValue(mapper.bookToResponse(book)));
    }

    public Mono<ServerResponse> getBooks(ServerRequest request) {
        return bookService.findBooks()
                .flatMap(books -> ServerResponse
                        .ok()
                        .bodyValue(mapper.booksToResponse(books)));
    }
}
~~~

- 영속성 레벨까지 완전히 논블로킹 시퀀스로 구현 

### 18.6 R2dbcEntityTemplate 를 이용한 데이터 액세스
- 쿼리를 직접 전달하는 jdbcTemplate 과는 다르게 Query DSL 과 유사한 방식의 쿼리 생성 메서드를 사용 

[코드 18.6]
~~~java
@Slf4j
@Service("bookServiceV6")
@RequiredArgsConstructor
public class BookService {
    private final @NonNull R2dbcEntityTemplate template;
    private final @NonNull CustomBeanUtils<Book> beanUtils;

    public Mono<Book> saveBook(Book book) {
        return verifyExistIsbn(book.getIsbn())
                .then(template.insert(book));
    }

    public Mono<Book> updateBook(Book book) {
        return findVerifiedBook(book.getBookId())
                .map(findBook -> beanUtils.copyNonNullProperties(book, findBook))
                .flatMap(updatingBook -> template.update(updatingBook));
    }

    public Mono<Book> findBook(long bookId) {
        return findVerifiedBook(bookId);
    }

    public Mono<List<Book>> findBooks() {
        return template.select(Book.class).all().collectList();
    }

    private Mono<Void> verifyExistIsbn(String isbn) {
        return template.selectOne(query(where("ISBN").is(isbn)), Book.class)
                .flatMap(findBook -> {
                    if (findBook != null) {
                        return Mono.error(new BusinessLogicException(
                                ExceptionCode.BOOK_EXISTS));
                    }
                    return Mono.empty();
                });
    }

    private Mono<Book> findVerifiedBook(long bookId) {
        return template.selectOne(query(where("BOOK_ID").is(bookId))
                                                                        , Book.class)
                .switchIfEmpty(Mono.error(new BusinessLogicException(
                                                ExceptionCode.BOOK_NOT_FOUND)));
    }
}
~~~

#### Terminating method 

- `first()`
- `one()` : 하나(Mono)만 리턴하되 결과가 하나가 아니면 Exception
- `all()`
- `count()`
- `exists()`

#### Criteria method 
- `and`
- `or`
- `greaterThan`
- `greaterThanOrEquals`
- `in`
- `is`
- `isNull`
- `isNotNull`
- `lessThan`
- `lessThanOrEquals`
- `like`
- `not`
- `notIn`

### 18.7 Spring Data R2DBC 에서의 페이지네이션 처리

#### Repository 에서의 페이지네이션
- JPA 와 유사

[코드 18.7]
~~~java
@Repository("bookRepositoryV7")
public interface BookRepository extends ReactiveCrudRepository<Book, Long> {
    Mono<Book> findByIsbn(String isbn);
    Flux<Book> findAllBy(Pageable pageable);
}
~~~

[코드 18.8]
~~~java
/**
 * 페이지네이션 적용
 */
@Slf4j
@Service("bookServiceV7")
@RequiredArgsConstructor
public class BookService {
    private final @NonNull BookRepository bookRepository;
    private final @NonNull CustomBeanUtils<Book> beanUtils;

    public Mono<Book> saveBook(Book book) {
        return verifyExistIsbn(book.getIsbn())
                .then(bookRepository.save(book));
    }

    public Mono<Book> updateBook(Book book) {
        return findVerifiedBook(book.getBookId())
                .map(findBook -> beanUtils.copyNonNullProperties(book, findBook))
                .flatMap(updatingBook -> bookRepository.save(updatingBook));
    }

    public Mono<Book> findBook(long bookId) {
        return findVerifiedBook(bookId);
    }

    public Mono<List<Book>> findBooks(@Positive int page,
                                      @Positive int size) {
        return bookRepository
                .findAllBy(PageRequest.of(page - 1, size,
                                                    Sort.by("memberId").descending()))
                .collectList();
    }

    private Mono<Void> verifyExistIsbn(String isbn) {
        return bookRepository.findByIsbn(isbn)
                .flatMap(findBook -> {
                    if (findBook != null) {
                        return Mono.error(new BusinessLogicException(
                                                    ExceptionCode.BOOK_EXISTS));
                    }
                    return Mono.empty();
                });
    }

    private Mono<Book> findVerifiedBook(long bookId) {
        return bookRepository
                .findById(bookId)
                .switchIfEmpty(Mono.error(new BusinessLogicException(
                                                    ExceptionCode.BOOK_NOT_FOUND)));
    }
}
~~~

#### R2dbcEntityTemplate 에서의 페이지네이션
- 쿼리 빌드 메서드 사용 

[코드 18.9]
~~~java
/**
 * 페이지네이션 적용
 */
@Slf4j
@Validated
@Service("bookServiceV8")
@RequiredArgsConstructor
public class BookService {
    private final @NonNull R2dbcEntityTemplate template;
    private final @NonNull CustomBeanUtils<Book> beanUtils;

    public Mono<Book> createBook(Book book) {
        return verifyExistIsbn(book.getIsbn())
                .then(template.insert(book));
    }

    public Mono<Book> updateBook(Book book) {
        return findVerifiedBook(book.getBookId())
                .map(findBook -> beanUtils.copyNonNullProperties(book, findBook))
                .flatMap(updatingBook -> template.update(updatingBook));
    }

    public Mono<Book> findBook(long bookId) {
        return findVerifiedBook(bookId);
    }

    public Mono<List<Book>> findBooks(@Positive long page, @Positive long size) {

        return template
                .select(Book.class)
                .count()
                .flatMap(total -> {
                    Tuple2<Long, Long> skipAndTake = getSkipAndTake(total, page, size);
                    return template
                            .select(Book.class)
                            .all()
                            .skip(skipAndTake.getT1())
                            .take(skipAndTake.getT2())
                            .collectSortedList((Book b1, Book b2) ->
                                    (int) (b2.getBookId() - b1.getBookId()));
                });
    }

    private Mono<Void> verifyExistIsbn(String isbn) {
        return template.selectOne(query(where("ISBN").is(isbn)), Book.class)
                .flatMap(findBook -> {
                    if (findBook != null) {
                        return Mono.error(new BusinessLogicException(
                                ExceptionCode.BOOK_EXISTS));
                    }
                    return Mono.empty();
                });
    }

    private Mono<Book> findVerifiedBook(long bookId) {
        return template.selectOne(query(where("BOOK_ID").is(bookId))
                                                                        , Book.class)
                .switchIfEmpty(Mono.error(new BusinessLogicException(
                                                ExceptionCode.BOOK_NOT_FOUND)));
    }

    private Tuple2<Long, Long> getSkipAndTake(long total, long movePage, long size) {
        long totalPages = (long) Math.ceil((double) total / size);
        long page = movePage > totalPages ? totalPages : movePage;
        long skip = total - (page * size) < 0 ? 0 : total - (page * size);
        long take = total - (page * size) < 0 ? total - ((page - 1) * size) : size;

        return Tuples.of(skip, take);
    }
}
~~~

