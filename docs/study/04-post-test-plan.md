# 04. 테스트 계획

> 게시글 CRUD 설계 섹션 4에서 정리한 내용 (2026-09-26, 테스트 기초 추가 2026-09-28)

## 0. 테스트 기초 (처음 배우는 사람용)

### 테스트란?
**"내 코드가 생각한 대로 동작하는지 확인하는 것."**
- ⑤에서 앱을 실행하고 브라우저로 404를 확인한 것도 테스트다 — **사람이 직접 하는 테스트**.
- 문제: 고칠 때마다 손으로 다시 확인해야 하고, 기능이 늘면 다 못 보고, 새 기능 때문에 **예전 기능이 망가져도** 모르고 지나간다.
- 그래서 확인 과정을 **코드로 적어 두는 것**이 테스트 코드. 한 번 적으면 버튼 한 번으로 전부 다시 확인한다.
  - 빠르게 확인할 수 있다
  - 예전 기능이 망가지면 자동으로 알려 준다 (기존 테스트가 빨간불)

### 가장 간단한 테스트 코드

```java
public class Calculator {
    public int add(int a, int b) { return a + b; }
}
```

```java
class CalculatorTest {

    @Test                                  // "이건 테스트야"
    void 더하기_성공() {                    // 무엇을 확인하는지 이름으로
        // given: 준비
        Calculator calculator = new Calculator();

        // when: 실행
        int result = calculator.add(2, 3);

        // then: 결과 확인
        assertThat(result).isEqualTo(5);   // "result는 5여야 해" — 다르면 실패
    }
}
```

| 부분 | 뜻 |
|---|---|
| `@Test` | JUnit(테스트 도구)에게 "이 메서드를 실행해서 확인해 줘" |
| **given** | 준비 |
| **when** | 확인하고 싶은 동작을 실행 |
| **then** | 결과가 기대한 값인지 확인 |
| `assertThat(값).isEqualTo(기대값)` | "이 값은 이거여야 한다". 다르면 테스트 **실패** |

```
✅ 더하기_성공   ← 통과 (초록)
❌ 더하기_성공   ← 누가 add를 a - b로 잘못 고치면 실패 (빨강)
                  "Expected: 5, but was: -1"
```

- 테스트가 실패하면 = **"코드가 망가졌다"는 신호**.

### 게시판 Service도 같은 모양

```java
@Test
void 게시글_작성_성공() {
    // given
    PostCreateRequest request = new PostCreateRequest("스프링 공부", "오늘은...", "하은");

    // when
    PostResponse response = postService.create(request);

    // then
    assertThat(response.title()).isEqualTo("스프링 공부");
}
```

- 문제: `create()` 안에서 `postRepository.save()`가 **DB에 저장**한다. 테스트마다 DB가 필요할까? → **Mock**으로 해결.

### Mock: 대본대로만 대답하는 대역 배우

```
진짜: 테스트 → PostService → 진짜 PostRepository → PostgreSQL (Docker 필요, 느림, 데이터가 쌓임)
Mock: 테스트 → PostService → 🎭 가짜 PostRepository (DB 없음, 대본대로만 대답)
```

- 진짜를 쓰면 실패했을 때 Service 문제인지 DB 문제인지 헷갈린다.
- Mock에게 **대본(각본)**을 준다.

```java
// "findById(1)이 불리면 이 게시글을 준 척해"
given(postRepository.findById(1L)).willReturn(Optional.of(post));

// "findById(999)가 불리면 없다고 해"
given(postRepository.findById(999L)).willReturn(Optional.empty());
```

- `given(...)` = "이렇게 불리면", `.willReturn(...)` = "이걸 돌려줘"
- "없는 글" 같은 상황을 DB 상태와 상관없이 **한 줄로** 만든다.

```java
@Test
void 없는_게시글_조회시_예외() {
    // given: 대본 — 999번은 없다고 해
    given(postRepository.findById(999L)).willReturn(Optional.empty());

    // when & then: 조회하면 에러가 나야 해
    assertThatThrownBy(() -> postService.getPost(999L))
            .isInstanceOf(PostNotFoundException.class);
}
```

- `assertThatThrownBy(...)` = "이걸 실행하면 **에러가 나야 해**"
- 확인하는 것: Service가 "없음"을 받았을 때 제대로 에러를 던지는가 (DB는 상관없음)

테스트 파일 준비

```java
@ExtendWith(MockitoExtension.class)   // Mockito(가짜 객체 도구) 사용
class PostServiceTest {

    @Mock
    PostRepository postRepository;    // 🎭 가짜 Repository

    @InjectMocks
    PostService postService;          // 진짜 PostService를 만들고 위 가짜를 넣어 줌
}
```

- **생성자 주입이 테스트에 좋은 이유**가 여기서 드러난다 — 생성자로 받으니 가짜를 쉽게 넣는다.

### TDD: 테스트를 먼저 쓰는 방법

```
🔴 Red      실패하는 테스트를 먼저 쓴다 (코드가 없으니 당연히 실패)
   ↓
🟢 Green    테스트를 통과할 만큼만 코드를 짠다
   ↓
🔵 Refactor 통과한 상태에서 코드를 깔끔하게 다듬는다
   ↓
 (다음 기능으로 반복)
```

예: `getPost`를 TDD로
1. 🔴 `없는_게시글_조회시_예외()`를 먼저 쓴다 → `getPost`가 없으니 빨간불
2. 🟢 `findById(...).orElseThrow(...)`를 쓴다 → 초록불
3. 🔵 조회·수정·삭제에서 반복되면 `findPost()`로 모은다 → 여전히 초록불인지 확인

왜 먼저 쓰나
- 코드를 짜기 전에 **"이 기능은 무엇을 해야 하지?"**를 먼저 생각하게 된다.
- 테스트가 있으니 다듬다가 망가뜨려도 바로 안다.

### 용어 정리

| 용어 | 한 줄 |
|---|---|
| 테스트 코드 | 확인 과정을 코드로 적어 자동으로 반복 |
| given-when-then | 준비 → 실행 → 결과 확인 |
| `assertThat` | "이 값은 이거여야 해" |
| `assertThatThrownBy` | "이걸 실행하면 에러가 나야 해" |
| Mock | 대본대로만 대답하는 가짜 객체 — DB 없이 Service만 테스트 |
| `given(...).willReturn(...)` | 가짜에게 주는 대본 |
| `@Mock` / `@InjectMocks` | 가짜 만들기 / 진짜 객체에 가짜 넣기 |
| TDD | 🔴 실패하는 테스트 먼저 → 🟢 통과 → 🔵 다듬기 |

---

## 설계 결과

### 테스트 범위: Service만 TDD, Controller는 직접 호출

| 대상 | 이번 방법 | 도구 |
|---|---|---|
| **Service** | **TDD** — 테스트를 먼저 쓰고 구현 | JUnit 5 + Mockito |
| **Controller** | 앱을 실행해 **직접 호출** | `curl` / IntelliJ HTTP Client (`.http` 파일로 저장) |
| Entity · Repository · Controller 테스트 | CRUD 완성 후 **"테스트 단계"**에서 별도 학습 | `@DataJpaTest`, `@WebMvcTest` 등 |

왜 이렇게 정했나
- 보통 입문 과정은 "CRUD 먼저 → 테스트는 나중"이다. 4계층 전부 TDD는 Spring도 처음인데 Mockito·MockMvc·Testcontainers까지 한꺼번에 배워야 해서 무겁다.
- 비즈니스 규칙("없으면 404" 등)이 모인 **Service만 TDD**로 테스트 감각을 익히고, Controller는 직접 호출해서 결과를 눈으로 본다.

### `PostServiceTest` 테스트 케이스

| 기능 | 케이스 |
|---|---|
| 생성 | 저장 후 `PostResponse`를 돌려준다 |
| 단건 조회 | 있으면 `PostResponse` / **없으면 `PostNotFoundException`** |
| 목록 조회 | `Page<PostSummaryResponse>`로 돌려준다 |
| 수정 | `Post` 객체의 제목·본문이 바뀐다 / 없으면 `PostNotFoundException` |
| 삭제 | `delete(post)`를 부른다 / 없으면 `PostNotFoundException` |

### 직접 호출로 확인할 것 (Controller)
- 생성 → `201` + `Location`
- 제목 101자 + 작성자 빈칸 → `400` + `fieldErrors` 2개
- 없는 id → `404` + `POST_NOT_FOUND`
- 삭제 → `204`, 본문 없음
- 수정 후 다시 조회 → 값이 바뀌었는가, SQL 로그에 `update post ...`가 찍히는가

### 진행 순서 (⑨)
기능 하나를 안쪽부터 바깥까지 완성하고 다음 기능으로.

```
Create: Entity → Repository → Service(TDD) → Controller → 직접 호출 확인
Read  : Repository → Service(TDD) → Controller → 직접 호출 확인
Update, Delete, List ...
```

---

## 1. 계층별 테스트 도구 (나중에 "테스트 단계"에서 쓸 것 포함)

| 대상 | 도구 | 띄우는 범위 |
|---|---|---|
| Entity | 순수 JUnit | Spring 없이 자바만 |
| Repository | `@DataJpaTest` + Testcontainers | JPA + 진짜 PostgreSQL |
| Service | JUnit + **Mockito** | Spring 없이, Repository는 가짜 ← **이번에 사용** |
| Controller | `@WebMvcTest` + `MockMvc` + `@MockitoBean` | 웹 계층만, Service는 가짜 |

- **`MockMvc`**: 서버를 진짜로 띄우지 않고 HTTP 요청을 흉내 내서 Controller를 호출한다.
- **`@MockitoBean`**: `@WebMvcTest`에서 Service 자리에 Mock을 넣는다. 옛 자료의 `@MockBean`은 **Spring Boot 4에서 제거됨** — 블로그 볼 때 주의.
- **테스트 단계에서 주의**: `@DataJpaTest`도 슬라이스 테스트라 일반 `@Configuration`을 읽지 않는다 → `JpaAuditingConfig`가 안 켜져 `createdAt`이 `null`. `@Import(JpaAuditingConfig.class)` 필요. (섹션 1에서 Auditing 설정을 분리한 대가)

---

## 2. Mock (가짜 객체)

- 진짜 대신 **"이렇게 부르면 이걸 돌려줘"라고 각본을 정해 둔 객체.**
- Service 테스트에서 Repository를 Mock으로 쓰는 이유
  - Service의 **로직만** 확인하고 싶다. 진짜 Repository면 DB가 필요하고 느리며, 실패 시 Service 문제인지 DB 문제인지 구분이 어렵다.
  - "있음", "없음" 같은 **원하는 상황을 각본으로 바로** 만들 수 있다.

```java
// given: 각본 — 999번을 찾으면 "없음"을 돌려줘
given(postRepository.findById(999L)).willReturn(Optional.empty());

// when & then: 조회하면 PostNotFoundException이 나야 한다
assertThatThrownBy(() -> postService.getPost(999L))
        .isInstanceOf(PostNotFoundException.class);
```

- Mock에는 진짜 DB가 없어서 "없는 id"가 따로 존재하지 않는다 → `findById`가 **`Optional.empty()`**를 돌려주게 각본을 줘야 한다.

## 3. given-when-then
- 테스트를 **주어진 상황 → 실행 → 결과 확인** 세 단락으로 쓴다. 거의 표준처럼 쓰인다.
- 테스트 이름은 CLAUDE.md대로 한글 메서드명: `게시글_작성_성공()`, `없는_게시글_조회시_예외()`

## 4. 수정 테스트는 무엇을 확인하나 — "각 테스트는 자기 계층의 책임만"

| 선택지 | 결과 |
|---|---|
| `save()`가 불렸는지 `verify` | ❌ 우리 Service는 `save()`를 부르지 않으므로(변경 감지) **테스트가 실패**. 통과시키려고 불필요한 `save()`를 넣으면 테스트 때문에 코드를 망가뜨리는 셈 |
| **`Post` 객체의 값이 바뀌었는지** | ✅ |

```
Service의 책임 : 게시글을 찾아서 post.update(title, content)를 부른다   ← Service 테스트가 확인
JPA의 책임     : 바뀐 값을 트랜잭션 종료 시 UPDATE로 반영한다          ← Service 테스트 범위 밖
```

- Mockito 테스트에는 JPA가 없어서 변경 감지가 일어나지 않는다. DB 반영은 **직접 호출 + SQL 로그**로 확인한다.

---

## 복습 질문
1. 이번에 Service만 TDD로 하고 Controller는 직접 호출로 확인하기로 한 이유는?
2. Service 테스트에서 `PostRepository`를 Mock으로 쓰는 이유는?
3. 없는 id 조회 테스트에서 Mock에게 줄 각본은?
4. 수정 테스트에서 `save()` 호출 여부를 `verify`하면 안 되는 이유는?
5. `@DataJpaTest`에서 `createdAt`이 `null`이 되는 이유와 해결 방법은?

<details>
<summary>정답 보기 (먼저 스스로 답해 본 뒤 확인)</summary>

1. 4계층 전부 TDD는 첫 CRUD에 너무 무겁다. 비즈니스 규칙이 모인 Service만 TDD로 테스트 감각을 익히고, Controller는 직접 호출해서 결과를 눈으로 확인한다. 나머지 테스트는 CRUD 완성 후 따로 배운다.
2. Service의 로직만 확인하려고. 진짜 Repository면 DB가 필요해 느리고 실패 원인 구분이 어렵다. Mock은 원하는 상황(있음/없음)을 각본으로 바로 만든다.
3. `given(postRepository.findById(999L)).willReturn(Optional.empty());` — Mock에는 진짜 DB가 없으니 "없음"을 직접 정해 줘야 한다.
4. 우리 Service는 변경 감지 덕분에 `save()`를 부르지 않는다 → 테스트가 실패한다. Service 테스트는 `post`의 값이 바뀌었는지(Service의 책임)만 확인하고, DB 반영(JPA의 책임)은 다른 방법으로 확인한다.
5. `@DataJpaTest`는 슬라이스 테스트라 일반 `@Configuration`인 `JpaAuditingConfig`를 읽지 않는다. `@Import(JpaAuditingConfig.class)`로 직접 불러온다.

</details>

## 찾아볼 키워드
`Mockito given willReturn verify`, `AssertJ assertThatThrownBy`, `given-when-then 테스트 패턴`, `@ExtendWith(MockitoExtension.class) @Mock @InjectMocks`, `IntelliJ HTTP Client .http`, `Spring Boot 4 @MockitoBean`
