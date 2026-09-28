# 02. PostRepository · PostService 설계

> 게시글 CRUD 설계 섹션 2에서 정리한 내용 (2026-09-26)

## 설계 결과

### PostRepository

| 항목 | 결정 |
|---|---|
| 선언 | `PostRepository extends JpaRepository<Post, Long>` |
| 추가 메서드 | **없음** — 기본 제공 메서드로 충분 |
| 생성 | `save(post)` |
| 단건 조회 | `findById(id)` → 없으면 404 |
| 목록 조회 | `findAll(pageable)` — 최신순 정렬은 `Pageable`에 담아서 전달 |
| 수정 | `findById(id)` → 없으면 404 → `post.update(...)` → 변경 감지 |
| 삭제 | `findById(id)` → 없으면 404 → `delete(post)` |

### PostService

| 기능 | 메서드 | 트랜잭션 |
|---|---|---|
| 생성 | `create(PostCreateRequest)` → `PostResponse` | `@Transactional` |
| 단건 조회 | `getPost(Long id)` → `PostResponse` | `@Transactional(readOnly = true)` |
| 목록 조회 | `getPosts(Pageable)` → `Page<PostSummaryResponse>` | `@Transactional(readOnly = true)` |
| 수정 | `update(Long id, PostUpdateRequest)` → `PostResponse` | `@Transactional` |
| 삭제 | `delete(Long id)` → 없음 (`void`) | `@Transactional` |

- `@Transactional`은 **메서드마다 명시** (메서드만 봐도 트랜잭션 설정이 보이도록)
- 없는 게시글 → `PostNotFoundException` (404 변환은 섹션 3의 `@RestControllerAdvice`)
- "조회해서 없으면 예외" 반복 → `private Post findPost(Long id)`로 한 곳에 모음
- Service는 Entity가 아닌 **DTO를 반환**
- 의존성: `private final PostRepository` + `@RequiredArgsConstructor` (생성자 주입)

### 메서드 모양을 읽는 규칙

| 규칙 | 적용 |
|---|---|
| **"어떤 글?" → `id`** | 이미 있는 글을 다루는 `getPost`, `update`, `delete`만 `id`를 받는다 |
| **"어떤 내용으로?" → 요청 DTO** | 사용자가 내용을 보내는 `create`, `update`만 DTO를 받는다 |
| **돌려주는 것 → 응답 DTO** | 목록은 여러 개 + 페이지 정보라서 `Page<PostSummaryResponse>` |

- `create`가 `id`를 받지 않는 이유: `IDENTITY` 전략이라 **DB가 INSERT할 때** 번호를 만든다. 만드는 시점엔 아직 없다. 대신 생성 후 `PostResponse`에 담아 돌려준다.
- `update`가 둘 다 받는 이유: `id`는 **어떤 글을**, `PostUpdateRequest`는 **어떤 내용으로**.

---

## 1. Repository

### Spring Data JPA 인터페이스
- `JpaRepository`를 **상속한 인터페이스만 선언**하면 구현 클래스는 Spring이 앱 시작 시 자동으로 만든다.
- Repository는 "만드는" 게 아니라 "선언하는" 것. → "이미 있는 걸로 충분한가"를 먼저 판단한다.

### 제네릭 `JpaRepository<T, ID>`
- `T`: Entity 클래스 (`Post`) — 이름이 아니라 **타입**
- `ID`: PK의 **타입** (`Long`) — 필드 이름 `id`가 아님
- 이 덕분에 `findById(Long id)`처럼 인자 타입이 자동으로 정해진다.

### 기본 제공 메서드
`save`, `findById`, `findAll`, `findAll(Pageable)`, `delete`, `deleteById`, `count`, `existsById` 등

- **Repository에는 `update`가 없다.** 수정은 `findById` → `post.update()` → 변경 감지 (섹션 1).

### Query Method
- `findByTitle(...)`처럼 메서드 이름을 규칙대로 지으면 SQL이 자동 생성된다.
- 이번 CRUD에서는 필요 없음. 제목 검색을 추가할 때 `findByTitleContaining` 등으로 사용 예정.

### 정렬: Query Method vs `Pageable`

| | Query Method `findAllByOrderByCreatedAtDesc(Pageable)` | `findAll(Pageable)` + 정렬을 `Pageable`에 ✅ |
|---|---|---|
| 정렬 기준 | 메서드 이름에 **고정** | 호출하는 쪽이 **정해서 넘김** |
| 정렬 변경 | 메서드를 새로 만들어야 함 | `Pageable`만 바꾸면 됨 |
| Repository | 메서드가 늘어남 | 그대로 비어 있음 |

### `Pageable` / `Page`
- `Pageable`: "몇 페이지를, 몇 개씩, 어떤 순서로" — **요청**
- `Page<T>`: 결과 목록 + 전체 개수 + 전체 페이지 수 — **응답**

### `Optional`
- `findById`는 `Post`가 아니라 `Optional<Post>`를 반환한다 — 값이 없을 수 있으니까.
- 핵심은 **"없을 수 있음"을 타입으로 드러내 처리를 강제**한다는 것.

```java
Post post = postRepository.findById(999L);                 // ❌ 컴파일 에러
Post post = postRepository.findById(999L).orElseThrow(...); // ✅ 없을 때 처리를 반드시 적어야 함
```

- `null`을 반환했다면 확인을 깜빡할 때 `NullPointerException`이 터진다.

### 없는 id 삭제
- 최신 Spring Data의 `deleteById(999)`는 없는 id여도 **에러 없이 조용히 넘어간다** → 사용자는 "삭제 성공"을 받게 됨.
- 그래서 **삭제 전에 `findById`로 존재를 확인**하고, 없으면 404.

---

## 2. Service

### `@Service`와 생성자 주입
- `@Service`: 비즈니스 로직 담당 클래스라는 표시. Spring이 빈(객체)으로 만들어 관리한다.
- 생성자 주입: 필요한 객체를 생성자로 받는다. `private final` 필드 + `@RequiredArgsConstructor`.
  - `final`이라 한 번 주입되면 바뀌지 않고, 테스트에서 가짜 객체를 넣기 쉽다.
  - `@Autowired` 필드 주입은 CLAUDE.md에서 금지.

#### 왜 `@Autowired` 필드 주입은 금지인가

```java
// ❌ 필드 주입
@Service
public class PostService {
    @Autowired
    private PostRepository postRepository;
}

// ✅ 생성자 주입
@Service
@RequiredArgsConstructor
public class PostService {
    private final PostRepository postRepository;
}
```

1. **테스트하기 어렵다 (가장 큰 이유)** — Service 테스트는 Repository를 가짜(Mock)로 바꿔서 한다.
   ```java
   PostService service = new PostService(fakeRepository);  // 생성자 주입: 바로 넣음
   PostService service = new PostService();                 // 필드 주입: postRepository가 null
   service.getPost(1L);                                     // 💥 NullPointerException
   ```
   필드 주입은 Spring이 몰래 넣어 주는 방식이라, Spring 없이 `new`로 만들면 비어 있다.
2. **`final`을 쓸 수 없다** — 필드 주입은 객체 생성 *후에* 값을 채우므로 `final` 불가. 생성자 주입은 `final`로 "한 번 주입되면 안 바뀌고, 비어 있을 수 없음"을 보장한다.
3. **의존 관계가 숨겨진다** — 생성자가 "이 클래스가 동작하려면 무엇이 필요한지" 보여 주는 계약서 역할. 인자가 7~8개로 늘면 "너무 많은 일을 한다"는 설계 신호가 바로 보인다.
4. **순환 참조를 늦게 발견한다** — A↔B가 서로를 쓰는 잘못된 구조를 생성자 주입은 **앱 시작 시** 바로 에러로 알려 준다.

| | 필드 주입 `@Autowired` | 생성자 주입 |
|---|---|---|
| Spring 없이 테스트 | 어려움 (필드가 null) | `new`로 가짜를 바로 넣음 |
| `final` | 불가 | 가능 (불변 보장) |
| 필요한 의존성 | 숨겨짐 | 생성자에 드러남 |
| 설계 문제 신호 | 놓치기 쉬움 | 인자가 많으면 바로 보임 |

- Spring 공식 문서도 필수 의존성은 생성자 주입을 권장한다.

#### `@RequiredArgsConstructor`: 생성자 주입을 짧게 쓰는 도구

"생성자 주입"은 **방식**, `@RequiredArgsConstructor`는 그 생성자를 **Lombok이 대신 써 주는 도구**다.
우리 규칙(CLAUDE.md) = 생성자 주입을 `@RequiredArgsConstructor` + `final`로 쓴다.

```java
// 내가 쓰는 코드
@Service
@RequiredArgsConstructor
public class PostService {
    private final PostRepository postRepository;
}

// Lombok이 컴파일할 때 만들어 주는 코드 (눈에 안 보임)
@Service
public class PostService {
    private final PostRepository postRepository;

    public PostService(PostRepository postRepository) {
        this.postRepository = postRepository;
    }
}
```

동작 원리
1. **Lombok** — `final` 필드만 받는 생성자를 컴파일할 때 자동 생성한다. (`build.gradle`에서 Lombok이 `compileOnly` + `annotationProcessor`인 이유)
2. **Spring** — 생성자가 하나면 `@Autowired` 없이 그 생성자로 주입한다. (Spring 4.3+)

⚠️ **주의**: `final`을 빠뜨리면 생성자에 포함되지 않아 `null`로 남는다 → 에러 없이 앱이 뜨고, 호출할 때 `NullPointerException`.

| Lombok 어노테이션 | 만드는 생성자 | 쓰는 곳 |
|---|---|---|
| `@RequiredArgsConstructor` | `final` 필드만 받는 생성자 | Service, Controller (의존성 주입) |
| `@NoArgsConstructor` | 인자 없는 생성자 | Entity (`protected` 기본 생성자) |
| `@AllArgsConstructor` | 모든 필드를 받는 생성자 | 잘 안 씀 (필드가 늘면 의도치 않게 바뀜) |

### `@Transactional`
- 메서드를 **하나의 트랜잭션**으로 묶는다. 중간에 예외가 나면 전부 되돌린다(롤백).
- **변경 감지는 트랜잭션 안에서만** 동작한다.

### `readOnly = true`
- "읽기만 한다"는 표시. 변경 감지용 스냅샷을 만들지 않아 가볍고, 실수로 값을 바꿔도 DB에 반영되지 않는다.
- ⚠️ 수정 메서드에 실수로 붙이면 `post.update()`가 **에러 없이 무시된다** → 찾기 어려운 버그. 테스트로 잡는다.

### `@Transactional` 위치

| | 메서드마다 명시 ✅ | 클래스에 `readOnly` + 쓰기 메서드에 따로 |
|---|---|---|
| 장점 | 메서드만 봐도 설정이 보임 | 깜빡해도 최소한 트랜잭션은 걸림 |
| 약한 실수 | 새 메서드에 붙이는 걸 깜빡 | 쓰기 메서드에 깜빡하면 수정이 조용히 무시됨 |

- 메서드에 붙인 설정이 클래스 설정보다 **우선**한다.
- 둘 다 실무에서 쓰인다. 어느 쪽이든 **테스트가 실수를 잡아 주는 게** 핵심.

### 커스텀 예외와 책임 분리
```java
postRepository.findById(id).orElseThrow(() -> new PostNotFoundException(id));
```
- 범용 예외(`IllegalArgumentException`) 대신 **이름만 봐도 상황을 아는** 예외를 만든다.
- **Service는 HTTP를 모른다.** "게시글이 없다"는 사실만 알리고, 404로 바꾸는 건 웹 계층(`@RestControllerAdvice`)의 일.
- 그래야 같은 Service를 HTTP가 아닌 곳(배치 등)에서도 그대로 쓸 수 있다.

### 반복 제거: private 메서드
```java
private Post findPost(Long id) {
    return postRepository.findById(id).orElseThrow(() -> new PostNotFoundException(id));
}
```
- 단건 조회·수정·삭제가 모두 이걸 부른다 → "없으면 404" 규칙이 한 곳에.
- CLAUDE.md "한 번만 쓰이는 로직에 유틸 클래스 금지"와 충돌하지 않음: **세 번** 쓰이고, 클래스가 아니라 Service 안의 private 메서드.

### Service가 DTO를 반환하는 이유
```
Service (트랜잭션 O) → 반환 → 트랜잭션 끝 (open-in-view: false라 DB 연결도 반납)
Controller (트랜잭션 X) → 반환된 객체를 JSON으로 변환하며 필드를 읽음
```
1. **지연 로딩 에러**: 나중에 `Post`에 `Member` 같은 연관관계가 생기면, 트랜잭션 밖에서 연관 객체를 읽을 때 `LazyInitializationException`. (지금은 연관관계가 없어 당장은 안 터짐 → Member 도메인에서 직접 겪어 볼 것)
2. **Controller가 Entity를 알게 됨**: 실수로 `post.update()`를 부를 수 있고, Entity 구조가 바뀌면 API 응답도 같이 바뀐다.

→ **트랜잭션 안(Service)에서 DTO로 바꿔서** 내보낸다.

---

## 구현(⑨) 때 직접 확인할 것
- [ ] 수정 메서드에 `readOnly = true`를 붙이면 UPDATE가 안 나가는 것을 테스트로 확인
- [ ] 없는 id로 조회·수정·삭제 시 `PostNotFoundException`이 던져지는가

## 복습 질문
1. `JpaRepository<Post, Long>`에서 `Long`은 무엇의 타입인가?
2. Repository에 `update` 메서드가 없는데 수정은 어떻게 하나?
3. 정렬을 `Pageable`에 담는 방식이 `findAllByOrderByCreatedAtDesc` 같은 Query Method 방식보다 나은 점은?
4. 없는 id로 삭제할 때 `deleteById`만 쓰면 어떤 문제가 생기나?
5. 404 응답으로 바꾸는 일을 Service가 하지 않는 이유는?
6. 수정 메서드에 `readOnly = true`를 실수로 붙이면 어떻게 되나?
7. `create`는 왜 `id`를 받지 않나?
8. `@Autowired` 필드 주입을 쓰면 Service 단위 테스트에서 어떤 문제가 생기나?
9. `@RequiredArgsConstructor`를 붙였는데 의존성 필드에 `final`을 빠뜨리면 어떻게 되나?

<details>
<summary>정답 보기 (먼저 스스로 답해 본 뒤 확인)</summary>

1. **PK(`id` 필드)의 타입.** 첫 번째는 Entity 클래스, 두 번째는 PK 타입이다.
2. `findById`로 조회 → `post.update(...)`로 값 변경 → 트랜잭션이 끝날 때 **변경 감지**가 UPDATE를 자동 실행한다.
3. 정렬 기준을 **호출하는 쪽이 정해서** 넘기므로, 정렬을 바꿔도 Repository에 메서드를 추가할 필요가 없다. Repository가 단순하게 유지된다.
4. 없는 id여도 **에러 없이 넘어가서** 사용자가 "삭제 성공"을 받는다. 그래서 먼저 `findById`로 확인하고 없으면 404.
5. Service는 **HTTP를 몰라야** 한다. "없다"는 사실만 예외로 알리고, HTTP 응답 변환은 웹 계층의 책임. 그래야 HTTP가 아닌 곳에서도 Service를 재사용할 수 있다.
6. 변경 감지가 꺼져서 `post.update()`를 불러도 **에러 없이 DB에 반영되지 않는다.**
7. `IDENTITY` 전략이라 **DB가 INSERT할 때** id를 만든다. 만드는 시점엔 아직 없고, 사용자가 정하면 겹칠 수 있다. 생성 후 `PostResponse`에 담아 돌려준다.
8. Spring 없이 `new PostService()`로 만들면 `postRepository`가 **null**이라 가짜 Repository를 넣을 방법이 없다 (넣으려면 리플렉션 같은 우회 필요). 생성자 주입은 `new PostService(fakeRepository)`로 바로 넣을 수 있다.
9. 생성자에 포함되지 않아 주입되지 않고 `null`로 남는다. 앱은 정상적으로 뜨고, 호출할 때 `NullPointerException`이 난다.

</details>

## 찾아볼 키워드
Spring Data JPA 공식 문서 "Defining Repository Interfaces", "Query Methods", `Pageable Page PageRequest Sort`, `Java Optional orElseThrow`, `생성자 주입을 써야 하는 이유`, `@Transactional 동작 원리`, `@Transactional readOnly 효과`, `LazyInitializationException`, `open-in-view`
