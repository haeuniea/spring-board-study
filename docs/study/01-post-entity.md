# 01. Post Entity 설계

> 게시글 CRUD 설계 섹션 1에서 직접 설계하며 정리한 내용 (2026-09-25)

## 설계 결과

| 항목 | 결정 | 이유 |
|---|---|---|
| 클래스 | `Post` (`post/entity/`) | 클래스는 객체 하나(게시글 하나)를 나타내므로 단수형 |
| 필드 | `id` / `title` / `content` / `author` / `createdAt` / `updatedAt` | 기본 필드만 (조회수 등은 필요할 때 추가) |
| 바뀌지 않는 필드 | `id`, `author`, `createdAt` | `updatable = false`, `update()`와 수정 DTO에서 제외 |
| PK 전략 | `IDENTITY` | 학습용이라 직관적인 방식, 대량 저장이 없어 batch insert 불가 단점은 영향 없음 |
| 생성 | `Post.create(title, content, author)` | 정적 팩토리 — 의도가 드러나고 필수값 누락을 컴파일 단계에서 방지 |
| 수정 | `post.update(title, content)` | 변경 감지로 `save()` 재호출 불필요 |
| 기본 생성자 | `protected` | JPA 프록시(상속)는 허용, 외부에서 빈 객체 생성은 차단 |
| 검증 | DTO + DB | DTO는 사용자에게 400 응답, DB는 최후의 방어선 |
| 시간 | JPA Auditing | 설정은 `global/config/JpaAuditingConfig`에 분리 |
| 임시 필드 | `author` | Member 도메인 구현 시 `Member` 연관관계로 교체 예정 |

### 필드 상세

| 필드 | 자바 타입 | DB 타입·제약 | 변경 | 어떻게 막나 |
|---|---|---|---|---|
| `id` | `Long` | `bigint`, PK | ❌ | PK, DB가 자동 생성 |
| `title` | `String` | `varchar(100) not null` | ✅ | — |
| `content` | `String` | `text not null` | ✅ | — |
| `author` | `String` | `varchar(50) not null` | ❌ | `updatable = false`, `update()`·수정 DTO에서 제외 |
| `createdAt` | `LocalDateTime` | `not null` | ❌ | `updatable = false` + Auditing |
| `updatedAt` | `LocalDateTime` | `not null` | ✅ (자동) | Auditing이 채움 |

---

## 1. JPA 기본

### Entity
- DB 테이블의 한 행에 대응하는 자바 클래스. `@Entity`를 붙이면 JPA가 관리한다.
- SQL을 직접 쓰지 않고 자바 객체를 다루듯 DB를 다룰 수 있다. 객체 하나 = 행 하나.

### PK 생성 전략 (`@GeneratedValue`)

| 전략 | 번호를 누가 매기나 | 특징 |
|---|---|---|
| `IDENTITY` | DB가 INSERT할 때 | 단순·직관적. `save()` 시 INSERT가 **즉시** 실행됨 |
| `SEQUENCE` | DB 시퀀스에서 미리 | 쓰기 지연 가능, 성능 최적화에 유리, 설정이 더 필요 |
| `AUTO` | Hibernate가 선택 | 동작이 숨겨짐. PostgreSQL에서는 SEQUENCE가 선택되어 번호가 50씩 건너뛸 수 있음 |

### 영속성 컨텍스트와 영속 상태
- JPA가 Entity를 **보관하고 지켜보는 임시 저장소**. 여기 들어 있는 Entity를 "영속 상태"라고 한다.
- `save()`로 저장하거나 `findById()`로 조회하면 들어간다.
- 쓰기 지연과 변경 감지가 모두 이 저장소 덕분에 가능하다. JPA 동작의 뿌리.

### 쓰기 지연
- `save()`를 불러도 SQL을 바로 보내지 않고, 모아 뒀다가 **트랜잭션이 끝날 때** 한꺼번에 보낸다.
- 예외: `IDENTITY`는 INSERT를 해야 id를 알 수 있으므로 `save()` 시점에 즉시 INSERT한다.

### 변경 감지 (dirty checking)
- 영속 상태 Entity의 값이 바뀌면, 트랜잭션이 끝날 때 JPA가 **자동으로 UPDATE**를 실행한다.
- 원리: 조회 시 처음 값(스냅샷)을 저장해 두고, 끝날 때 현재 값과 비교한다.
- 조건: `@Transactional` 안이어야 한다.
- 주의: 무심코 바꾼 값도 DB에 저장된다 → Setter를 쓰지 않는 이유 중 하나.

```java
@Transactional
public void update(Long id, ...) {
    Post post = postRepository.findById(id)...;  // ① 조회 — 스냅샷 저장
    post.update("새 제목", "새 본문");             // ② 객체 값만 변경
}                                                 // ③ 트랜잭션 종료 — 비교 후 UPDATE 자동 실행
```

### JPA 프록시와 `protected` 기본 생성자
- JPA는 조회할 때 ① 인자 없는 생성자로 빈 객체를 만들고 ② 값을 채운다 → 기본 생성자가 필요하다.
- 프록시: JPA가 성능(지연 로딩)을 위해 Entity를 **상속한 가짜 객체**를 만들 때가 있다 → 부모 생성자에 접근할 수 있어야 한다.

| 접근 수준 | 외부 `new Post()` | 프록시(상속) | 결과 |
|---|---|---|---|
| `public` | 가능 | 가능 | ❌ `create()` 규칙이 깨짐 |
| `protected` | 불가 | 가능 | ✅ |
| `private` | 불가 | 불가 | ❌ 프록시 생성 불가 |

- 참고: 자바 `protected`는 같은 패키지에서도 접근 가능하다.
- Lombok: `@NoArgsConstructor(access = AccessLevel.PROTECTED)`

### JPA Auditing
- 생성·수정 시각을 **자동으로** 채워 주는 Spring Data JPA 기능.
- 필요한 것 두 가지
  - **켜기**: `@EnableJpaAuditing` — 설정 클래스에
  - **표시**: Entity에 `@EntityListeners(AuditingEntityListener.class)`, 필드에 `@CreatedDate` / `@LastModifiedDate`

---

## 2. 자바 기본

### 기본형 vs 래퍼 타입

| | 기본형 `int`, `long` | 래퍼 타입 `Integer`, `Long` |
|---|---|---|
| `null` | 불가 (없으면 `0`) | 가능 |
| 정체 | 값 자체 | 객체 |

- 저장 전 Entity의 id는 "아직 없음"이어야 한다. `int`면 `0`이 되어 "0번 게시글"과 구분할 수 없다. JPA도 id가 `null`이면 새 객체로 판단한다.
- `Long`인 이유: `Integer`보다 범위가 넓고(약 922경) DB `bigint`와 맞는다. **모든 PK를 `Long`으로 통일**하는 게 관례.

### `LocalDateTime`
- Java 8 `java.time`의 날짜·시간 클래스. 시간대 없이 "날짜 + 시각"을 표현한다.
- `java.util.Date`를 쓰지 않는 이유: 가변 객체이고, 월이 0부터 시작하는 등 헷갈린다. `LocalDateTime`은 불변이고 직관적이다.

### 객체 생성 방법

| | 생성자 | 정적 팩토리 메서드 | 빌더 |
|---|---|---|---|
| 모양 | `new Post(a, b, c)` | `Post.create(a, b, c)` | `Post.builder().a().build()` |
| 의도 표현 | ❌ | ✅ 메서드 이름 | ✅ 필드 이름 |
| 인자 순서 실수 | 위험 | 위험 (인자가 적으면 관리 가능) | 안전 |
| 필수 값 누락 | 컴파일 에러 ✅ | 컴파일 에러 ✅ | **통과** ❌ |
| 잘 맞는 경우 | 단순한 객체 | 필드가 적고 규칙이 중요할 때 | 필드가 많고 선택값이 많을 때 |

---

## 3. 설계 원칙

### Setter를 쓰지 않는 이유
1. **규칙 보호** — `id`, `createdAt`처럼 바뀌면 안 되는 필드까지 열린다.
2. **의도 표현** — `setTitle()` + `setContent()`보다 `update()`가 의미를 드러낸다. 규칙이 추가되면 한 곳만 고치면 된다.
3. **변경 감지 사고 방지** — 무심코 바꾼 값이 DB에 저장되는 것을 막는다.

> DTO는 `record`로 만들어 처음부터 불변이라 이 규칙과 무관하다.

### "바뀌지 않음"을 보장하는 세 겹

| 보호막 | 방법 |
|---|---|
| 코드 | Setter 없음, `update()`에서 받지 않음 |
| JPA | `@Column(updatable = false)` |
| API | 수정 DTO에 해당 필드를 두지 않음 |

### DTO 검증 vs DB 제약

| | DTO (`@NotBlank`, `@Size`) | DB (`varchar(100) not null`) |
|---|---|---|
| 지키는 곳 | HTTP 요청 입구 | 저장되는 마지막 지점 |
| 목적 | 사용자에게 **400 + 친절한 메시지** | **어떤 경로로든** 잘못된 데이터 차단 |
| 없으면 | 잘못된 입력이 **500**으로 나가 서버 잘못처럼 보임 | 검증을 빠뜨린 DTO, DB 직접 입력 등을 못 막음 |

- 한 줄 요약: **DTO는 사용자를 위해, DB는 데이터를 위해.**
- 컬럼 길이를 지정하지 않으면 JPA는 `varchar(255)`를 만든다 → 길이는 어차피 정해야 하는 값.
- Entity 내부 검증은 규칙이 단순한 지금은 생략 (복잡해지면 추가).

---

## 4. Spring 구성과 테스트

### `@Configuration`
- Spring 설정을 담는 클래스라는 표시. 앱 시작 시 읽힌다.
- `global/config/JpaAuditingConfig`에 `@EnableJpaAuditing`을 둔다 (여러 도메인이 공통으로 쓰는 설정이라 `global/`).

### 슬라이스 테스트
- 앱 전체가 아니라 **특정 계층만** 잘라서 띄우는 테스트.

| 어노테이션 | 띄우는 범위 |
|---|---|
| `@WebMvcTest` | Controller 등 웹 계층만 |
| `@DataJpaTest` | Repository·JPA 데이터 계층만 |
| `@SpringBootTest` | 앱 전체 (슬라이스 아님) |

- 빠르고, 실패하면 어느 계층 문제인지 바로 알 수 있다.
- 주의: `@EnableJpaAuditing`을 `BoardApplication`이나 Controller에 붙이면 `@WebMvcTest`에서 JPA가 없어 `JPA metamodel must not be empty` 에러가 난다 → 별도 `@Configuration`으로 분리.

---

## 구현(⑨) 때 직접 확인할 것
- [ ] `IDENTITY`: `save()` 순간 SQL 로그에 INSERT가 바로 찍히는가
- [ ] 변경 감지: `save()` 없이 `update()`만으로 UPDATE가 찍히는가
- [ ] `updatable = false`: UPDATE SQL에 `author`, `created_at`이 빠져 있는가

## 복습 질문
1. `IDENTITY` 전략에서 `save()`를 부르면 INSERT는 언제 실행되나? 왜?
2. `update()`로 값을 바꾼 뒤 `save()`를 다시 부르지 않아도 DB에 반영되는 이유는?
3. 기본 생성자를 `private`으로 하면 어떤 문제가 생기나?
4. DTO 검증만 있고 DB 제약이 없으면 어떤 상황에서 문제가 되나?
5. `@EnableJpaAuditing`을 `BoardApplication`에 붙이면 어떤 테스트가 깨지나?

<details>
<summary>정답 보기 (먼저 스스로 답해 본 뒤 확인)</summary>

1. **`save()` 즉시.** `IDENTITY`는 DB가 INSERT할 때 번호를 매기므로 INSERT를 해야 id를 알 수 있다. JPA는 영속성 컨텍스트에서 Entity를 id로 관리해야 하므로 트랜잭션 종료까지 기다리지 못하고 바로 INSERT한다 (쓰기 지연의 예외).
2. **변경 감지(dirty checking).** 조회한 Entity는 영속 상태가 되고 JPA가 스냅샷을 저장해 둔다. 트랜잭션이 끝날 때 현재 값과 비교해 바뀌었으면 UPDATE를 자동 실행한다. 단, `@Transactional` 안이어야 한다.
3. **JPA 프록시를 만들 수 없다.** 프록시는 Entity를 상속해서 만드는데, 자식 클래스는 부모의 `private` 생성자를 호출할 수 없다 → 지연 로딩 등이 동작하지 않는다. 그래서 `protected`.
4. **DTO를 거치지 않거나 검증이 빠진 경로.** 예: `PostUpdateRequest`에 `@Size`를 깜빡함, 테스트·초기 데이터·관리자 기능 등 HTTP를 거치지 않는 코드, DB 툴로 직접 `INSERT`. DB 제약은 이 모든 경우를 막는 최후의 방어선이다.
5. **`@WebMvcTest`.** 웹 계층만 띄우지만 메인 클래스의 설정은 읽으므로, JPA가 없는 환경에서 Auditing 설정을 만나 `JPA metamodel must not be empty` 에러가 난다. 별도 `@Configuration`(`JpaAuditingConfig`)은 `@WebMvcTest`가 읽지 않으므로 분리한다.

</details>

## 찾아볼 키워드
`영속성 컨텍스트`, `JPA dirty checking`, `JPA IDENTITY vs SEQUENCE`, `정적 팩토리 메서드 장점` (이펙티브 자바 아이템 1), `@Builder 필수값 누락`, `Bean Validation @NotBlank @Size`, Spring Data JPA 공식 문서 "Auditing", `@WebMvcTest 슬라이스 테스트`
