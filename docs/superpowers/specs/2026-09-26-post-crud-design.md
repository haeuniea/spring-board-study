# 게시글(Post) CRUD 설계

- 작성일: 2026-09-26
- 범위: 게시글 생성·단건 조회·목록 조회·수정·삭제 REST API
- 학습 노트: `docs/study/01-post-entity.md` ~ `04-post-test-plan.md` (결정 이유와 개념 설명)

## 1. 범위

포함
- 게시글 CRUD 5개 API
- 입력값 검증, 없는 게시글 404, 공통 에러 응답
- Service 단위 테스트 (TDD)

제외 (이후 단계)
- 회원·로그인·권한 — `author`는 임시 문자열 필드, Member 도메인 구현 시 `Member` 연관관계로 교체
- 검색, 조회수, soft delete, 부분 수정(PATCH)
- Entity·Repository·Controller 테스트 — CRUD 완성 후 별도 테스트 단계

## 2. 패키지 구조

```
com.example.board
├─ BoardApplication.java
├─ global/
│   ├─ config/JpaAuditingConfig.java        @Configuration + @EnableJpaAuditing
│   ├─ dto/PageResponse.java
│   └─ exception/
│       ├─ GlobalExceptionHandler.java      @RestControllerAdvice
│       └─ ErrorResponse.java
└─ post/
    ├─ controller/PostController.java
    ├─ service/PostService.java
    ├─ repository/PostRepository.java
    ├─ entity/Post.java
    ├─ dto/
    │   ├─ PostCreateRequest.java
    │   ├─ PostUpdateRequest.java
    │   ├─ PostResponse.java
    │   └─ PostSummaryResponse.java
    └─ exception/PostNotFoundException.java
```

- `JpaAuditingConfig`를 `BoardApplication`과 분리 — `@WebMvcTest`에서 JPA 설정 충돌 방지

## 3. Entity — `Post`

| 필드 | 자바 타입 | DB 매핑 | 변경 |
|---|---|---|---|
| `id` | `Long` | `bigint` PK, `@GeneratedValue(strategy = IDENTITY)` | ❌ |
| `title` | `String` | `varchar(100) not null` | ✅ |
| `content` | `String` | `text not null` | ✅ |
| `author` | `String` | `varchar(50) not null`, `updatable = false` | ❌ |
| `createdAt` | `LocalDateTime` | `not null`, `updatable = false`, `@CreatedDate` | ❌ |
| `updatedAt` | `LocalDateTime` | `not null`, `@LastModifiedDate` | ✅ (자동) |

- 클래스: `@Entity`, `@EntityListeners(AuditingEntityListener.class)`, `@Getter`, `@NoArgsConstructor(access = PROTECTED)`, `@Setter` 없음
- 생성: `static Post create(String title, String content, String author)` — `id`·시간 필드는 받지 않음
- 수정: `void update(String title, String content)` — `author`는 받지 않음, 변경 감지로 반영 (`save()` 재호출 없음)
- Entity 내부 검증 없음 (DTO + DB 제약으로 처리)

## 4. Repository — `PostRepository`

```java
public interface PostRepository extends JpaRepository<Post, Long> { }
```

- 추가 메서드 없음. 목록 정렬은 `findAll(Pageable)`에 전달되는 `Pageable`로 지정

## 5. Service — `PostService`

| 메서드 | 반환 | 트랜잭션 | 동작 |
|---|---|---|---|
| `create(PostCreateRequest)` | `PostResponse` | `@Transactional` | `Post.create(...)` → `save` → `PostResponse.from` |
| `getPost(Long id)` | `PostResponse` | `@Transactional(readOnly = true)` | `findPost(id)` → `PostResponse.from` |
| `getPosts(Pageable)` | `Page<PostSummaryResponse>` | `@Transactional(readOnly = true)` | `findAll(pageable)` → `map(PostSummaryResponse::from)` |
| `update(Long id, PostUpdateRequest)` | `PostResponse` | `@Transactional` | `findPost(id)` → `post.update(...)` → `PostResponse.from` |
| `delete(Long id)` | `void` | `@Transactional` | `findPost(id)` → `delete(post)` |

- `private Post findPost(Long id)` — `findById(id).orElseThrow(() -> new PostNotFoundException(id))`
- `@Transactional`은 메서드마다 명시
- 의존성: `private final PostRepository` + `@RequiredArgsConstructor`
- Entity → DTO 변환은 Service 안(트랜잭션 안)에서 수행, Controller에는 DTO만 전달

## 6. API

| 기능 | 요청 | 성공 응답 |
|---|---|---|
| 생성 | `POST /api/posts` + `PostCreateRequest` | `201 Created` + `Location: /api/posts/{id}` + `PostResponse` |
| 단건 조회 | `GET /api/posts/{id}` | `200 OK` + `PostResponse` |
| 목록 조회 | `GET /api/posts?page={page}&size={size}` | `200 OK` + `PageResponse<PostSummaryResponse>` |
| 수정 | `PUT /api/posts/{id}` + `PostUpdateRequest` | `200 OK` + `PostResponse` |
| 삭제 | `DELETE /api/posts/{id}` | `204 No Content` (본문 없음) |

- Controller: `@RestController`, `@RequestMapping("/api/posts")`, 요청 DTO는 `@Valid @RequestBody`
- 목록 기본값: `@PageableDefault(size = 10, sort = "createdAt", direction = DESC)`, `page`는 0부터
- 페이지 크기 상한: `application.yml`에 `spring.data.web.pageable.max-page-size: 100`
- 알려진 한계: 클라이언트가 `sort` 파라미터로 정렬을 바꿀 수 있음 (허용)
- `Page` → `PageResponse` 변환은 Controller에서 `PageResponse.from(page)`

## 7. DTO (모두 `record`)

| DTO | 필드 | 검증 |
|---|---|---|
| `PostCreateRequest` | `title`, `content`, `author` | `title`: `@NotBlank @Size(max = 100)`, `content`: `@NotBlank`, `author`: `@NotBlank @Size(max = 50)` |
| `PostUpdateRequest` | `title`, `content` | `title`: `@NotBlank @Size(max = 100)`, `content`: `@NotBlank` |
| `PostResponse` | `id`, `title`, `content`, `author`, `createdAt`, `updatedAt` | — |
| `PostSummaryResponse` | `id`, `title`, `author`, `createdAt` | — |
| `PageResponse<T>` | `content`(`List<T>`), `page`, `size`, `totalElements`, `totalPages` | — |

- 변환: `PostResponse.from(Post)`, `PostSummaryResponse.from(Post)`, `PageResponse.from(Page<T>)` — 정적 팩토리를 DTO 안에 둠 (Entity는 DTO를 모름)
- `PostUpdateRequest`에 `author`가 없으므로 요청에 포함돼도 무시됨

## 8. 예외 처리

| 상황 | 예외 | HTTP | `code` | `message` | `fieldErrors` |
|---|---|---|---|---|---|
| 없는 게시글 | `PostNotFoundException` | 404 | `POST_NOT_FOUND` | 게시글을 찾을 수 없습니다 | `[]` |
| 검증 실패 | `MethodArgumentNotValidException` | 400 | `INVALID_INPUT` | 입력값이 올바르지 않습니다 | 필드별 `{field, reason}` |

- `PostNotFoundException extends RuntimeException`, 생성자로 `id`를 받음
- `ErrorResponse(String code, String message, List<FieldError> fieldErrors)`, `FieldError(String field, String reason)` — `record`
- `reason`은 Bean Validation 기본 메시지 사용
- 그 외 예외(숫자가 아닌 id, 잘못된 JSON 등)는 Spring 기본 처리

```json
{
  "code": "INVALID_INPUT",
  "message": "입력값이 올바르지 않습니다",
  "fieldErrors": [
    { "field": "title",  "reason": "크기가 0에서 100 사이여야 합니다" },
    { "field": "author", "reason": "공백일 수 없습니다" }
  ]
}
```

## 9. 테스트

`PostServiceTest` — JUnit 5 + Mockito (`@ExtendWith(MockitoExtension.class)`, `@Mock PostRepository`, `@InjectMocks PostService`), given-when-then, 한글 메서드명

| 기능 | 케이스 |
|---|---|
| 생성 | 저장 후 요청 값이 담긴 `PostResponse`를 반환한다 |
| 단건 조회 | 있으면 `PostResponse` 반환 / 없으면 `PostNotFoundException` |
| 목록 조회 | `Page<PostSummaryResponse>`로 변환해 반환한다 |
| 수정 | `Post`의 제목·본문이 바뀌고 작성자는 그대로다 / 없으면 `PostNotFoundException` |
| 삭제 | `delete(post)`를 호출한다 / 없으면 `PostNotFoundException` |

- "없음"은 `given(postRepository.findById(id)).willReturn(Optional.empty())`로 표현
- 수정 테스트는 `save()` 호출 여부가 아니라 `Post` 객체의 값 변경을 검증

Controller는 앱을 실행해 `src/test/http/posts.http`(IntelliJ HTTP Client)로 직접 확인
- 생성 → 201 + `Location`
- 제목 101자 + 작성자 빈칸 → 400 + `fieldErrors` 2개
- 없는 id → 404 + `POST_NOT_FOUND`
- 수정 후 재조회 → 값 변경, SQL 로그에 `update post` / UPDATE 문에 `author`·`created_at` 없음
- 삭제 → 204, 본문 없음

## 10. 구현 순서

기능 단위로 안쪽부터 바깥까지 완성한 뒤 다음 기능으로 넘어간다. 기능마다 커밋.

1. **생성** — `Post`, `JpaAuditingConfig`, `PostRepository`, `PostCreateRequest`, `PostResponse`, `PostService.create`(TDD), `PostController` 생성 API, `ErrorResponse`, `GlobalExceptionHandler`(검증 실패 400), `.http` 확인
2. **단건 조회** — `PostNotFoundException`, `findPost`, `PostService.getPost`(TDD), 조회 API, 핸들러에 404 추가
3. **목록 조회** — `PostSummaryResponse`, `PageResponse`, `PostService.getPosts`(TDD), 목록 API, `max-page-size` 설정
4. **수정** — `Post.update`, `PostUpdateRequest`, `PostService.update`(TDD), 수정 API
5. **삭제** — `PostService.delete`(TDD), 삭제 API

## 11. 구현 시 확인할 것

- `IDENTITY`: `save()` 순간 INSERT가 바로 실행되는가 (SQL 로그)
- 변경 감지: `save()` 없이 UPDATE가 실행되는가
- `updatable = false`: UPDATE 문에 `author`, `created_at`이 빠지는가
- Spring Boot 4 / Spring Data 버전에서 `@PageableDefault`, `max-page-size` 설정 키, Bean Validation 기본 메시지가 위 설계대로 동작하는가 (Context7로 확인)
