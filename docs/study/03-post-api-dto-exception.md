# 03. API 스펙 · DTO · 예외 처리 설계

> 게시글 CRUD 설계 섹션 3에서 정리한 내용 (2026-09-26)

## 설계 결과

### API 스펙

| 기능 | 메서드와 URL | 요청 본문 | 성공 응답 |
|---|---|---|---|
| 생성 | `POST /api/posts` | `PostCreateRequest` | **201 Created** + `Location` + `PostResponse` |
| 단건 조회 | `GET /api/posts/{id}` | — | 200 OK + `PostResponse` |
| 목록 조회 | `GET /api/posts?page=0&size=10` | — | 200 OK + `PageResponse<PostSummaryResponse>` |
| 수정 | `PUT /api/posts/{id}` | `PostUpdateRequest` | 200 OK + `PostResponse` |
| 삭제 | `DELETE /api/posts/{id}` | — | **204 No Content** |

| 실패 상황 | 응답 |
|---|---|
| 없는 `id` (조회·수정·삭제) | 404 Not Found, `POST_NOT_FOUND` |
| 검증 실패 (생성·수정) | 400 Bad Request, `INVALID_INPUT` |

- 페이징 기본값: `page=0`, `size=10`, `createdAt` 내림차순 — Controller의 `@PageableDefault`로 지정
- `size` 최대 100 (`spring.data.web.pageable.max-page-size`)
- 한계: 사용자가 `?sort=title`처럼 정렬을 바꿔 보낼 수 있음 — 학습용이라 지금은 허용

### DTO (모두 `record`)

| DTO | 위치 | 필드 | 검증 |
|---|---|---|---|
| `PostCreateRequest` | `post/dto/` | `title`, `content`, `author` | `title`: `@NotBlank @Size(max=100)` / `content`: `@NotBlank` / `author`: `@NotBlank @Size(max=50)` |
| `PostUpdateRequest` | `post/dto/` | `title`, `content` | 생성과 같음. **`author` 없음** (바뀌지 않는 필드) |
| `PostResponse` | `post/dto/` | `id`, `title`, `content`, `author`, `createdAt`, `updatedAt` | — |
| `PostSummaryResponse` | `post/dto/` | `id`, `title`, `author`, `createdAt` | — (목록에선 본문 제외) |
| `PageResponse<T>` | `global/dto/` | `content`, `page`, `size`, `totalElements`, `totalPages` | — |

### 예외 처리

| 상황 | 예외 | HTTP | `code` | `fieldErrors` |
|---|---|---|---|---|
| 없는 게시글 | `PostNotFoundException` | 404 | `POST_NOT_FOUND` | `[]` |
| 입력값 검증 실패 | `MethodArgumentNotValidException` | 400 | `INVALID_INPUT` | 틀린 필드마다 하나씩 |

- `ErrorResponse(code, message, fieldErrors)` + `FieldError(field, reason)` — `global/exception/`
- `GlobalExceptionHandler` (`@RestControllerAdvice`) — `global/exception/`
- `PostNotFoundException extends RuntimeException` — `post/exception/`
- 이번 범위 밖: 숫자가 아닌 id(`/api/posts/abc`), 잘못된 JSON 형식 → Spring 기본 처리, 필요 시 추가

```json
{
  "code": "INVALID_INPUT",
  "message": "입력값이 올바르지 않습니다",
  "fieldErrors": [
    { "field": "title",  "reason": "100자 이하여야 합니다" },
    { "field": "author", "reason": "필수 입력값입니다" }
  ]
}
```

---

## 1. REST API URL 설계

### 핵심 원칙: URL = 명사(무엇), HTTP 메서드 = 동사(어떻게)

```
❌ 동사를 URL에                   ✅ REST
POST /api/createPost              POST   /api/posts
GET  /api/getPost?id=1            GET    /api/posts/1
POST /api/updatePost              PUT    /api/posts/1
POST /api/deletePost?id=1         DELETE /api/posts/1
```

| 규칙 | 예시 | 이유 |
|---|---|---|
| 명사 사용, 동사 금지 | `/posts` ⭕ `/getPosts` ❌ | 행동은 HTTP 메서드가 표현 |
| 복수형 | `/posts`, `/posts/1` | "게시글 모음"과 "그중 1번" |
| 계층은 `/` | `/posts/1/comments` | 1번 게시글의 댓글들 |
| 소문자, 단어 구분은 `-` | `/post-categories` | 대소문자 혼동 방지, `_`는 링크 밑줄에 가려짐 |
| 끝에 `/` 없음 | `/posts` ⭕ `/posts/` ❌ | 같은 자원에 주소가 두 개 생기는 것 방지 |
| 필터·정렬·페이징은 쿼리 파라미터 | `/posts?page=0&size=10` | "어떤 자원"이 아니라 "어떻게 가져올지"의 옵션 |
| 확장자 없음 | `/posts/1` ⭕ `/posts/1.json` ❌ | 형식은 `Accept` 헤더로 |

| HTTP 메서드 | 의미 | 우리 API |
|---|---|---|
| `GET` | 조회 — **서버 상태를 바꾸지 않음** | 단건·목록 조회 |
| `POST` | 생성 | 생성 |
| `PUT` | 전체 교체 | 수정 |
| `PATCH` | 부분 수정 | (사용 안 함) |
| `DELETE` | 삭제 | 삭제 |

- `GET`이 상태를 바꾸지 않는다는 약속이 중요한 이유: 브라우저·검색 엔진이 GET 링크를 마음대로 미리 불러오기도 한다 → `GET /deletePost` 같은 설계는 사고로 이어진다.

### `/api`를 붙이는 이유
- REST 규칙이 아니라 **관례**. 기능적으로는 달라지는 게 없다.
- 화면 주소(`/posts`)와 API 주소(`/api/posts`)가 겹치지 않는다.
- 로그인(Security), CORS, 프록시 규칙을 `/api/**` 패턴 하나로 걸 수 있다.
- 버전 관리로 확장하기 쉽다 (`/api/v1/posts`, `/api/v2/posts`).
- 코드에서는 Controller에 `@RequestMapping("/api/posts")` 한 번.

---

## 2. HTTP 상태 코드

| 범위 | 의미 | 예시 |
|---|---|---|
| **2xx** | 성공 | 200, 201, 204 |
| **4xx** | **요청한 쪽** 잘못 | 400 잘못된 입력, 404 없음 |
| **5xx** | **서버 쪽** 잘못 | 500 서버 에러 |

| 코드 | 의미 | 본문 | 우리 API |
|---|---|---|---|
| **200 OK** | 성공, 결과는 여기 | 있음 | 조회, 수정 |
| **201 Created** | 성공, **새 자원을 만들었음** | 보통 있음 + `Location` 헤더 | 생성 |
| **204 No Content** | 성공, **돌려줄 내용 없음** | 없음 | 삭제 |

- 코드만 보고도 무슨 일이 일어났는지 알 수 있다 → 프론트엔드가 `201`이면 상세 화면으로 이동, `204`면 본문을 읽지 않는 식으로 반응.
- `Location: /api/posts/17` — "새 글은 여기 있어요".

---

## 3. DTO

### Entity와 DTO는 어디서 쓰이나

- **Entity(`Post`)**: **DB와 대화**할 때 — Service ↔ Repository ↔ DB
- **DTO**: **사용자와 대화**할 때 — 사용자 ↔ Controller ↔ Service
- **Service가 둘을 바꿔 주는 경계**

```
조회 (GET /api/posts/1)
사용자 → Controller → Service → Repository → DB
                         │ ← Post (Entity)
                         ★ PostResponse.from(post)   Entity → DTO
사용자 ← JSON ← Controller ← PostResponse (DTO)

생성 (POST /api/posts)
사용자 → JSON → Controller → PostCreateRequest (DTO) → Service
                              ★ Post.create(request.title(), ...)   DTO → Entity
                              save(post) → Repository → DB (INSERT)
                              ★ PostResponse.from(post)             Entity → DTO
사용자 ← JSON ← Controller ← PostResponse (DTO)
```

| 구간 | 다니는 객체 |
|---|---|
| 사용자 ↔ Controller | JSON ↔ DTO |
| Controller ↔ Service | DTO |
| Service ↔ Repository ↔ DB | Entity |

왜 나누나
- Controller가 Entity를 모르게 → 트랜잭션 밖에서 Entity를 읽는 문제 방지 (섹션 2)
- 사용자가 Entity를 직접 못 만지게 → DTO로 받으면 **허용한 필드만** 들어온다 (`PostUpdateRequest`에 `author`가 없는 이유)

### `record`
- "데이터를 담아 옮기는 용도" 클래스를 짧게 쓰는 문법 (Java 16+).

```java
public record PostCreateRequest(String title, String content) {}
```

- 모든 필드가 `private final` → **불변**. Setter를 만들 수조차 없다.
- 생성자, 값 꺼내는 메서드, `equals`/`toString` 자동 생성.
- 값 꺼내기는 `getTitle()`이 아니라 **`request.title()`**.

### 변환 메서드는 어디에 사나

```
post/entity/Post.java                  → Post.create(...)               값 → Entity
post/dto/PostResponse.java             → PostResponse.from(post)        Entity → DTO
post/dto/PostSummaryResponse.java      → PostSummaryResponse.from(post) Entity → DTO
global/dto/PageResponse.java           → PageResponse.from(page)        Page → DTO
```

```java
public record PostResponse(Long id, String title, String content, String author,
                           LocalDateTime createdAt, LocalDateTime updatedAt) {

    public static PostResponse from(Post post) {
        return new PostResponse(post.getId(), post.getTitle(), post.getContent(),
                                post.getAuthor(), post.getCreatedAt(), post.getUpdatedAt());
    }
}
```

- `static`이라 `PostResponse.from(post)`처럼 클래스 이름으로 바로 호출.
- 변환 코드가 한 곳에 모인다 (생성·조회·수정 세 곳에서 사용).
- `from` = "~로부터 만든다"는 관례적인 이름.

**왜 Entity가 아니라 DTO 안에?** — 의존 방향

```
DTO    ──알고 있음──> Entity   ✅
Entity ──알고 있음──> DTO      ❌
```

- Entity는 핵심 도메인, 응답 모양(DTO)은 화면 요구에 따라 자주 바뀐다. Entity가 DTO를 알면 응답이 바뀔 때마다 Entity까지 고쳐야 한다.
- 같은 이유로 `Post.create(title, content, author)`도 DTO가 아니라 **값**을 받는다.
- 다른 방법: Service 안에서 직접 변환, 별도 Mapper 클래스/MapStruct (큰 프로젝트용).

### `@NotBlank` vs `@NotEmpty` vs `@NotNull`

| 들어온 값 | `@NotNull` | `@NotEmpty` | `@NotBlank` |
|---|---|---|---|
| `null` | ❌ 막음 | ❌ 막음 | ❌ 막음 |
| `""` | ✅ 통과 | ❌ 막음 | ❌ 막음 |
| `"   "` | ✅ 통과 | ✅ 통과 | ❌ 막음 |
| `"스프링"` | ✅ 통과 | ✅ 통과 | ✅ 통과 |

- 문자열 필수값은 `@NotBlank`. `@NotNull`은 숫자·날짜 등에.

### `PageResponse`로 감싸는 이유
- `Page`를 그대로 JSON으로 내보내면 `pageable`, `sort`(두 번), `paged`/`unpaged` 등 **내부 정보가 가득**하고, 모양이 Spring 버전에 따라 바뀔 수 있다 (Spring도 경고 로그를 남김).
- 필요한 것만 우리가 정한 모양으로:

```json
{ "content": [ ... ], "page": 0, "size": 10, "totalElements": 42, "totalPages": 5 }
```

- Entity를 직접 내보내지 않는 것과 **같은 원칙** — 내부 객체는 노출하지 않고 API 응답은 DTO로 통제한다.

---

## 4. 예외 처리

### 흐름

```
Service    게시글 없음       → PostNotFoundException           ┐
Controller @Valid 검증 실패  → MethodArgumentNotValidException ┤→ GlobalExceptionHandler → 404 / 400 + ErrorResponse
                                                              ┘   (@RestControllerAdvice)
```

### Checked vs Unchecked
- **Checked** (`Exception`): 부르는 쪽이 반드시 `try-catch`나 `throws`를 적어야 한다.
- **Unchecked** (`RuntimeException`): 적지 않아도 된다.
- `PostNotFoundException`은 **`RuntimeException`** 상속
  - 처리는 `GlobalExceptionHandler` 한 곳에서 하므로 `throws`를 줄줄이 적을 필요가 없다.
  - `@Transactional`은 기본적으로 **Unchecked 예외에서만 롤백**한다.

### `@RestControllerAdvice` + `@ExceptionHandler`

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(PostNotFoundException.class)
    ResponseEntity<ErrorResponse> handle(...) { ... }   // 404로 변환
}
```

- 모든 Controller의 예외를 **한 곳에서** HTTP 응답으로 바꾼다.
- 없으면 Controller마다 같은 `try-catch` 반복 + Controller의 "요청/응답 변환만" 책임이 깨진다.
- 섹션 2의 "404 변환은 웹 계층이 한다"의 그 웹 계층.

### 검증 실패
- `@Valid @RequestBody PostCreateRequest request`에서 검증에 걸리면, **Controller 메서드가 실행되기도 전에** Spring이 `MethodArgumentNotValidException`을 던진다 → 잡아서 400으로.

### 에러 응답 모양 선택: 코드 + 메시지 + 필드별 에러

| | 특징 |
|---|---|
| 메시지만 | 단순하지만 어느 필드가 왜 틀렸는지 모름 |
| **code + message + fieldErrors** ✅ | 틀린 칸을 콕 집어 알려 줌, code로 분기 가능 |
| `ProblemDetail` (RFC 9457) | Spring 기본 제공 표준. 필드별 에러는 확장 필요 |

- `fieldErrors` → 화면에서 해당 입력칸 아래에 각각 메시지 표시
- `code` → 프로그램이 분기 (`POST_NOT_FOUND`면 없는 글 화면, `INVALID_INPUT`이면 폼 유지)
- `message`는 사람이 읽는 문구라 **바뀔 수 있다.** 프로그램이 문구로 분기하면 문구가 바뀔 때 깨진다. `code`는 **바뀌지 않는 약속**.

---

## 구현(⑨) 때 직접 확인할 것
- [ ] 생성 응답이 201이고 `Location` 헤더가 있는가
- [ ] 삭제 응답이 204이고 본문이 없는가
- [ ] 제목 101자 + 작성자 빈칸 → 400, `fieldErrors`에 두 개가 담기는가
- [ ] 없는 id → 404, `code`가 `POST_NOT_FOUND`인가

## 복습 질문
1. `/api/deletePost?id=1` 대신 `DELETE /api/posts/1`을 쓰는 이유는?
2. 게시글 생성은 왜 `200`이 아니라 `201`로 응답하나?
3. 삭제는 왜 `204`인가?
4. `PostResponse.from(post)`는 어느 파일에 있고, 누가 부르나?
5. 변환 메서드를 Entity가 아니라 DTO 안에 두는 이유는?
6. 제목에 `@NotNull` 대신 `@NotBlank`를 쓰는 이유는?
7. `Page`를 그대로 반환하지 않고 `PageResponse`로 감싸는 이유는?
8. 컨트롤러마다 `try-catch`를 쓰지 않고 `@RestControllerAdvice`를 쓰는 이유는?
9. `PostNotFoundException`이 `RuntimeException`을 상속하는 이유는?
10. 에러 응답에 `message`가 있는데 `code`를 따로 두는 이유는?

<details>
<summary>정답 보기 (먼저 스스로 답해 본 뒤 확인)</summary>

1. URL은 **무엇(자원)**을, HTTP 메서드는 **어떻게(행동)**를 나타낸다. 같은 주소 `/api/posts/1`에 GET·PUT·DELETE로 행동만 바꾸면 주소만 봐도 무엇을 다루는지 알 수 있다.
2. 201은 "성공했고 **새 자원이 만들어졌다**"는 뜻이다. `Location` 헤더로 새 글의 주소도 알려 줄 수 있다. 200은 그냥 "성공"이라 무엇이 일어났는지 구분이 안 된다.
3. `delete()`가 돌려줄 데이터가 없다(`void`). 204는 "성공, 돌려줄 내용 없음".
4. `post/dto/PostResponse.java` 안의 `static` 메서드. Service가 트랜잭션 안에서 Entity를 DTO로 바꿀 때 부른다.
5. 의존 방향: DTO → Entity는 괜찮지만 Entity → DTO는 피한다. 응답 모양은 자주 바뀌는데, Entity가 DTO를 알면 그때마다 핵심 도메인인 Entity를 고쳐야 한다.
6. `@NotNull`은 `null`만 막고 `""`, `"   "`는 통과시킨다. 공백만 있는 제목을 막으려면 `@NotBlank`.
7. `Page`의 JSON에는 내부 정보가 가득하고 Spring 버전에 따라 모양이 바뀔 수 있다. 필요한 것만 우리가 정한 모양으로 내보내려고 감싼다 (Entity를 직접 내보내지 않는 것과 같은 원칙).
8. 예외를 **한 곳에서** 처리해서 Controller마다 같은 `try-catch`를 반복하지 않으려고. Controller는 요청/응답 변환만 한다는 책임도 지켜진다.
9. Unchecked라서 `throws`를 줄줄이 적지 않아도 되고(처리는 핸들러 한 곳에서), `@Transactional`이 기본으로 롤백하는 대상이다.
10. `code`는 프로그램이 분기할 때 쓰는 **바뀌지 않는 약속**. `message`는 사람이 읽는 문구라 바뀔 수 있어서, 문구로 분기하면 깨진다.

</details>

## 찾아볼 키워드
`REST API 설계 가이드`, `RESTful URI 규칙`, `HTTP 메서드 멱등성`, `HTTP 201 Created Location`, `Java record`, `Bean Validation @NotBlank @NotEmpty @NotNull`, `Spring Data Page JSON 직렬화 PagedModel`, `Checked vs Unchecked Exception`, `@Transactional rollback 기본 규칙`, `@RestControllerAdvice @ExceptionHandler`, `ProblemDetail RFC 9457`
