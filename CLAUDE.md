# Project 프로젝트 개요
게시판 REST API — Spring Boot 백엔드 학습용 프로젝트.
Java 21 + Spring Boot 4.1 + Spring Data JPA + PostgreSQL.
운영 배포 목적이 아닌 학습이 목표 — 동작하는 코드보다 "왜 이렇게 하는지" 이해가 우선.

## Tech Stack 기술 스택
- Language: Java 21
- Framework: Spring Boot 4.1 (Spring Web MVC, Validation)
- ORM: Spring Data JPA (Hibernate)
- DB: PostgreSQL (로컬은 Docker Compose로 실행)
- Test: JUnit 5 + AssertJ + Mockito (`spring-boot-starter-test`), DB 통합 테스트는 Testcontainers
- Build: Gradle (Groovy DSL, `build.gradle`)
- Lombok 사용

## Architecture 아키텍처
- Controller → Service → Repository (3-layer)
- Controller: 요청/응답 변환과 검증(`@Valid`)만 담당, 비즈니스 로직은 Service로 위임
- Service: 비즈니스 로직과 트랜잭션 경계(`@Transactional`) 담당, 조회 전용 서비스 메서드에는 `@Transactional(readOnly = true)` 사용
- Repository: Spring Data JPA 사용
  - 단순 조회는 Query Method 우선 (`findByTitleContaining` 등)
  - 필요한 경우 JPQL `@Query` 사용
- Entity를 API 응답으로 직접 반환하지 않음 — 반드시 DTO로 변환
- 예외는 커스텀 예외 + `@RestControllerAdvice`에서 일괄 처리

## Commands 명령어
- DB 실행: `docker compose up -d`
- Build: `./gradlew build`
- Test: `./gradlew test`
- Test single: `./gradlew test --tests "*.PostServiceTest"`
- Run: `./gradlew bootRun --args='--spring.profiles.active=local'`

## Conventions 코드 컨벤션
- 패키지: 도메인별 구성 (`post/`, `comment/`, `member/` 아래에 controller·service·repository·dto·entity)
- DTO: `*Request`, `*Response` 접미사, Java `record`로 작성
- API 응답 형식: 성공 시 DTO 그대로, 실패 시 공통 `ErrorResponse`
- REST URL: 복수형 명사 (`/api/posts`, `/api/posts/{id}/comments`)
- Entity: `@Setter` 금지, `@NoArgsConstructor(access = PROTECTED)`, 상태 변경은 의미 있는 메서드로 (`post.update(...)`)
- 연관관계는 기본 `FetchType.LAZY`
- 의존성 주입은 생성자 주입만 (`@RequiredArgsConstructor` + `final` 필드, `@Autowired` 금지)
- 테스트 파일: `*Test.java`, 메서드명은 한글 허용 (`게시글_작성_성공()`)
- 날짜/시간: `java.time` (`LocalDateTime`), 생성·수정 시각은 JPA Auditing

## Learning 학습 방식
- 한 번에 완성본을 만들지 말고 기능 단위로 작게 진행
- 새로운 Spring 개념/어노테이션이 처음 등장하면 짧게 설명
- 코드 작성 전 설계(엔티티, API 스펙)를 먼저 합의
- 가능하면 테스트를 먼저 작성 (TDD)
- 라이브러리/설정 문법이 불확실하면 Context7로 최신 문서 확인 (Spring Boot 4는 3.x와 달라진 부분이 많음)

## Commit 커밋 메시지
- 형식: `<type>: <요약>` (Conventional Commits)
- type: feat, fix, refactor, test, docs, chore
- 요약은 한글, 50자 이내, 마침표 없음
- 본문(선택): 무엇을 왜 바꿨는지
- 예: `feat: 게시글 작성 API 추가`, `docs: CLAUDE.md 커밋 규칙 추가`

## DO NOT 금지
- Entity를 Controller에서 직접 반환하지 않기
- `ddl-auto: create`/`update`를 local 외 프로파일에서 쓰지 않기
- DB 비밀번호 등 민감 정보는 설정 파일에 직접 쓰지 않고 환경변수로 관리 (`${DB_URL}`, `${DB_USERNAME}`, `${DB_PASSWORD}`)
- 설정 파일(`application*.yml`)은 커밋하고, `.env`·secret 파일만 `.gitignore`
- 승인 없이 의존성 추가하지 않기
- 한 번만 쓰이는 로직을 위해 유틸 클래스 만들지 않기
- 자명한 코드에 주석 달지 않기
- 와일드카드 import 쓰지 않기

## Workflows
- 새 기능: `superpowers:brainstorming` → `superpowers:test-driven-development`
- 버그: `superpowers:systematic-debugging`
- 코드 탐색·리팩터링: Java LSP(`jdtls-lsp`)로 정의 이동, 참조 찾기, 타입 확인 — grep보다 우선
- 라이브러리/설정 확인: Context7 MCP로 Spring Boot 4 공식 문서 조회
- 커밋: 기능 단위로 `/commit` (`commit-commands` 플러그인), 위 Commit 규칙에 맞춰 작성
- 보안 점검: 로그인/권한 기능 추가 후 `/security-review`
