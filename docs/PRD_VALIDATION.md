# PRD 기술 검증 결과 — Todo List 풀스택 프로젝트

> **검증 대상**: `docs/PRD.md`
> **대조 문서**: `docs/API_SPEC.md`, `docs/ROADMAP.md`, `CLAUDE.md`, `docs/guides/*`
> **대조 코드**: `todo-backend/pom.xml`, `todo-backend/src/main/resources/application.properties`, `todo-frontend/package.json`, `todo-frontend/components.json`, `todo-frontend/AGENTS.md`, 실제 디렉토리 구조
> **검증일**: 2026-08-21 · **검증 방식**: Chain of Thought 단계별 검증 + 공식 문서 확인
> **주의**: 이 문서는 검증 결과만 담습니다. **PRD.md 원본은 수정하지 않았습니다.**

---

## 🧠 CoT 검증 요약

1. **초기 관찰** — PRD는 이메일/소셜 로그인 기반 개인용 Todo 웹앱을 정의하며, 13개 기능 ID(AUTH-01~05, TODO-01~06, UI-01~02), 11개 엔드포인트, 2개 테이블로 구성된 **범위가 잘 통제된 MVP**다. 문서 간 역할 분리(PRD=무엇, ROADMAP=순서, API_SPEC=계약)가 명시적으로 선언되어 있다.
2. **가설** — "이 PRD는 표준적인 JWT+OAuth2 CRUD 앱을 구현하며, 주요 기술 도전은 (a) 최신 메이저(Spring Boot 4 / Jackson 3 / Hibernate 7 / Next.js 16)의 파괴적 변경, (b) JSONB 본문 매핑, (c) OAuth2와 무상태 JWT의 공존, (d) Soft Delete와 페이지네이션의 상호작용일 것이다."
3. **단계 검증** — 가설 (a)(b)(c)는 **실제 위험으로 확인**되었고, (d)는 부분적으로만 위험했다. 예상하지 못한 발견은 **실제 코드에 평문 DB 비밀번호가 커밋되어 있다**는 점과, **PRD 1.3의 "실제 설치된 값" 주장이 5개 항목에서 사실이 아니라는 점**이다.
4. **정합성/규칙** — 불변 규칙 7개는 **전항목 준수**. PRD ↔ API_SPEC 엔드포인트는 **11:11 완전 대응**(고아/누락 없음). 다만 공통 응답 포맷과 PUT/PATCH 시맨틱에서 **내부 모순 3건**을 발견했다.
5. **종합 판단** — 목표는 전부 달성 가능하며 재설계는 불필요하다. 다만 **Critical 4건을 수정한 뒤** 구현에 착수해야 한다.

### 기술적 확신도 분포

- **[FACT]** 약 40% — 공식 문서/실제 파일로 직접 확인
- **[INFERENCE]** 약 40% — 확인된 사실에 근거한 추론
- **[UNCERTAIN]** 약 20% — 구현 착수 시점 재확인 필요

---

## Step 0. 버전·공식 문서 확인 (실제 코드 대조)

PRD 1.3은 **"아래 버전은 실제 프로젝트에 설치된 값이다"**라고 단언한다. 전 항목을 실제 파일과 대조했다.

| PRD 1.3 주장 | 실제 파일 값 | 판정 |
|---|---|---|
| Spring Boot **4.1.0** | `pom.xml` parent `4.1.0` | ✅ [FACT] |
| JDK **21** | `<java.version>21</java.version>` | ✅ [FACT] |
| Maven / Lombok | `mvnw` 존재, `lombok` optional + annotationProcessorPaths | ✅ [FACT] |
| jjwt **0.12.6** (api/impl/jackson) | 3개 모듈 모두 `0.12.6` | ✅ [FACT] |
| 베이스 패키지 `com.example` | `groupId=com.example`, `src/main/java/com/example/TodoBackendApplication.java` | ✅ [FACT] |
| `spring-boot-starter-webmvc` 로 모듈 분리 | pom에 `spring-boot-starter-webmvc` + `-webmvc-test`/`-security-test`/`-validation-test`/`-data-jpa-test` | ✅ [FACT] — Spring Boot 4.0 마이그레이션 가이드로 교차 확인 |
| Next.js **16.3.1** | `"next": "16.3.1"`, `eslint-config-next: 16.3.1` | ✅ [FACT] |
| React **19.2.8** | `react`/`react-dom` 모두 `19.2.8` | ✅ [FACT] |
| TypeScript 5 / Tailwind CSS 4 | `typescript ^5`, `tailwindcss ^4`, `@tailwindcss/postcss ^4` | ✅ [FACT] |
| Tailwind 4 CSS-first (`tailwind.config.js` 미사용) | `components.json`의 `"tailwind.config": ""`, config 파일 부재 | ✅ [FACT] |
| shadcn/ui **4.18.0**, style `radix-nova`, baseColor `neutral` | `"shadcn": "^4.18.0"`(= **CLI 버전**), `components.json` style `radix-nova` / baseColor `neutral` | ⚠️ 부분 — Minor #14 참조 |
| radix-ui, lucide-react **1.33.0** | `radix-ui ^1.6.7`, `lucide-react ^1.33.0` | ✅ [FACT] |
| `src/` 미사용, `app/`·`components/`·`lib/` 루트 배치 | 실제 구조 일치, `tsconfig` paths `@/* → ./*` | ✅ [FACT] |
| `AGENTS.md`가 `node_modules/next/dist/docs/`를 가리킴 | `AGENTS.md` 존재, `node_modules/next/dist/docs/` 실재 | ✅ [FACT] |
| **React Query (TanStack Query)** | **package.json에 없음** | ❌ [FACT] 미설치 |
| **Framer Motion** | **package.json에 없음** | ❌ [FACT] 미설치 |
| **Tiptap** | **package.json에 없음** | ❌ [FACT] 미설치 |
| **React Hook Form + Zod** | **package.json에 없음** | ❌ [FACT] 미설치 |

**추가 확인 [FACT]**

- Spring Boot 4.0에서 `spring-boot-starter-web` → `spring-boot-starter-webmvc`, `spring-boot-starter-oauth2-client` → `spring-boot-starter-security-oauth2-client`, `spring-boot-starter-test` → 기술별 `*-test` 스타터로 분리. pom.xml이 정확히 이 규칙을 따르고 있다. → **PRD 1.3의 경고 박스는 사실이며, 프로젝트가 이미 반영 완료.**
- Spring Boot 4는 **Jackson 3(`tools.jackson`)을 기본 JSON 라이브러리로 사용**하며 Jakarta EE 11 / Servlet 6.1 기반이다.
- Spring Security 7(Boot 4 동봉)에서 `WebSecurityConfigurerAdapter` 제거, `authorizeRequests()`/`antMatchers()` 제거, **람다 DSL만 지원**, CSRF 기본 정책 강화. → 무상태 JWT API는 `csrf.disable()` 명시 필요.
- PostgreSQL은 **따옴표 없는 식별자를 소문자로 폴딩**한다("unquoted names are always folded to lower case").
- Hibernate 6.x/7.x의 JSON 매핑은 `@JdbcTypeCode(SqlTypes.JSON)`로 활성화되며 PostgreSQL 방언에서 `jsonb`에 매핑된다.
- **Jackson 3 + Hibernate 7 조합에서 JSON FormatMapper 자동 구성이 깨진다** — `com.fasterxml.jackson` → `tools.jackson` 패키지 개명 때문이며, `Could not find a FormatMapper for the JSON format` 오류가 보고되었다(Hibernate 7.1.1.Final + Jackson 3.0.0-rc9 + Spring Boot 4.0.0+).
- Next.js 16 로컬 문서 확인: `useSearchParams`는 **가장 가까운 `<Suspense>` 경계까지 클라이언트 렌더로 전환**되며 Suspense 래핑이 권장된다. 또한 16에서 **Async Request APIs가 파괴적 변경**(동적 `params`/`searchParams`는 Promise)이고 `middleware` → `proxy`로 개명되었다.
- Hibernate는 6.4부터 `@SoftDelete`를 제공하지만 기본은 boolean 지시자이며, 타임스탬프 컬럼(`deleted_at`)을 유지하려면 **`@SQLDelete` + `@SQLRestriction` 조합이 여전히 적절한 선택**이다. → PRD 8.3의 선택은 타당하다.

---

## ✅ 불변 규칙 준수 결과 (INVARIANTS)

| 규칙 | 준수 | 근거 / 비고 |
|------|------|------|
| 이메일 전용 로그인 (username 없음) | ✅ | PRD 4.1 AUTH-01 "**ID는 이메일만**", 8.2 `users`에 username 컬럼 없음, API_SPEC 3.1/3.2 요청 바디에 email만 존재 |
| 비밀번호 6자 규칙 (복잡도 추가 금지) | ✅ | PRD 4.1 "**최소 6자** (그 이상 복잡도 제약을 추가하지 않는다)", 6.2, API_SPEC `@Size(min = 6)`. 가이드(`forms-react-hook-form.md`)도 복잡도 정규식을 명시적으로 무시하도록 지시 |
| JWT 24h / Refresh Token 미도입 | ✅ | PRD 4.1 AUTH-02, 10장, 12.1 `jwt.expiration=86400000`, 13.1 "Refresh Token 미구현", 4.5 범위 제외. API_SPEC `expiresIn: 86400000` 일치 |
| Soft Delete (물리 삭제 금지) | ✅ | PRD 8.1/8.3, TODO-06, API_SPEC 4.7 "물리 삭제하지 않고 `deleted_at` 기록". 두 테이블 모두 `deleted_at` 보유 |
| 페이지네이션 필수 | ✅ | PRD 4.4(규격 절 별도 존재), TODO-01, 7.2, API_SPEC 4.2. 전체 목록 반환 표현 없음 |
| 소유권 검증 (본인 리소스만) | ✅ | PRD 7.2 서두, 10장(불일치 시 **404**), API_SPEC 4장/2.3, ROADMAP M4 DoD. **세 문서가 404 정책까지 완전 일치** |
| 스키마 `TodoListDB` / BIGINT ID | ✅ (조건부) | PRD 8.1 스키마명 고정, 8.2 `id BIGINT PK AUTO`, 공통 필드 3종 존재. **다만 PostgreSQL 대소문자 폴딩 위험 → Major #1** |
| 페이지네이션 재사용 컴포넌트 | ✅ | PRD 4.4/UI-02 `components/common/Pagination.tsx`, ROADMAP M5 DoD, guides/project-structure.md 일치 |
| 민감정보 Git 커밋 금지 | ❌ | **PRD 12.1은 올바르게 명세했으나, 실제 `application.properties`에 평문 비밀번호가 커밋되어 있음 → Critical #1** |

> **결론**: PRD 문서 자체는 불변 규칙 7개를 **모두 준수**한다. 위반은 PRD가 아니라 **실제 코드가 PRD 12.1을 아직 이행하지 않은 것**이다. [INVARIANT]

---

## 🔗 문서 정합성 결과

### PRD ↔ API_SPEC — 엔드포인트 1:1 대응 (누락/고아 **없음**)

| PRD 7장 | API_SPEC | 기능 ID | 판정 |
|---|---|---|---|
| `POST /api/auth/signup` | 3.1 (201, `COMMON_001`/`AUTH_002`) | AUTH-01 | ✅ |
| `POST /api/auth/login` | 3.2 (200, `COMMON_001`/`AUTH_001`) | AUTH-02 | ✅ |
| `GET /oauth2/authorization/{provider}` | 3.4 | AUTH-03 | ✅ |
| `GET /login/oauth2/code/{provider}` | 3.5 (+ `?error=AUTH_007` 실패 경로) | AUTH-03 | ✅ |
| `GET /api/auth/me` | 3.3 (`AUTH_003~005`/`AUTH_006`) | AUTH-04 | ✅ |
| (AUTH-05 = API 없음) | 3.6 "API 없음" 명시 | AUTH-05 | ✅ 양쪽 모두 명시 |
| `GET /api/todos` | 4.2 (쿼리 4종, `PageResponse`) | TODO-01 | ✅ |
| `GET /api/todos/{id}` | 4.3 (`TODO_001`) | TODO-02 | ✅ |
| `POST /api/todos` | 4.4 (201) | TODO-03 | ✅ |
| `PUT /api/todos/{id}` | 4.5 | TODO-04 | ⚠️ 시맨틱 모순 → Major #3 |
| `PATCH /api/todos/{id}/status` | 4.6 (바디 없음) | TODO-05 | ⚠️ 비멱등 → Major #2 |
| `DELETE /api/todos/{id}` | 4.7 | TODO-06 | ✅ |

세부 값도 일치한다: 기본 size 10 / 최대 100 절삭, 정렬 `created_at DESC` 고정, `keyword`는 `title` 한정, 소유권 위반 404, `ApiResponse`/`PageResponse` 필드 구성(6필드), `expiresIn` 86400000. **[FACT] 대조 완료.**

**불일치 항목**

1. API_SPEC 4.2는 `keyword`에 "**대소문자 무시**"를 추가하지만 PRD 7장/13.1에는 이 조건이 없다. (Minor #5)
2. API_SPEC 3.5의 실패 리다이렉트 파라미터 `?error=AUTH_007`이 PRD 6.3에는 "토큰 누락/오류 시 에러 처리"로만 기술되어 파라미터 계약이 PRD 쪽에 없다. (Minor #6)
3. PRD 7장 서두 "기본 경로: `/api`"와 같은 표에 있는 `/oauth2/...`·`/login/oauth2/...`(비 `/api` 경로)가 문장상 충돌한다. (Minor #7)

### PRD ↔ ROADMAP — 기능 ID 마일스톤 매핑

| 기능 ID | 매핑 마일스톤 | 판정 |
|---|---|---|
| AUTH-01, AUTH-02, AUTH-04 (BE) | M2 (작업·DoD 모두 명시) | ✅ |
| AUTH-03 (BE) | M3 | ✅ |
| AUTH-01/02/03 (FE 화면) | M6 | ✅ |
| **AUTH-04 (헤더 표시), AUTH-05 (로그아웃)** | **없음** | ❌ 고아 — Major #9 |
| TODO-01~06 (BE) | M4 | ✅ |
| TODO-01~06 (FE) | M7 | ✅ |
| UI-01 다크모드 | M5 | ✅ |
| UI-02 Pagination | M5 (DoD에 단독 렌더/콜백까지) | ✅ |
| **공통 헤더(PRD 6.7) 구현** | **어느 마일스톤 작업/DoD에도 없음** | ❌ 고아 — Major #9 |

의존 순서 위반은 없다. `M2 → M3/M4`, `M0 → M5`, `M2+M5 → M6`, `M4+M5 → M7`, `M3~M7 → M8 → M9`는 PRD의 데이터 의존성과 정확히 일치한다 [INFERENCE]. 병렬 가능 표기(M3∥M4, M5∥BE)도 타당하다.

**ROADMAP 측 오참조 3건** (PRD 수정 대상 아님, ROADMAP 수정 필요)

- 부록 A가 **"PRD Phase 0~9"**에 매핑하지만, PRD 11장은 Phase를 정의하지 않고 "ROADMAP을 단일 기준으로 한다"고 위임했다 → **존재하지 않는 대상에 대한 매핑표**.
- M1 작업 항목의 "**PRD 3.2 스키마 준수**"는 실제로는 **PRD 8.2**다(3장은 사용자 여정).
- M0의 의존성 목록 "Web"은 Spring Boot 4에서 `webmvc`로 개명되었다(실제 pom은 이미 올바름).

### PRD ↔ CLAUDE.md (컨벤션)

전 항목 일치한다 [FACT]: `application.properties`(YAML 아님) 사용, `ApiResponse`/`PageResponse` 래핑, `BaseEntity` + `@SQLDelete`+`@SQLRestriction`, DTO↔엔티티 분리, Bean Validation, `GlobalExceptionHandler`, Controller→Service→Repository, React Query 서버 상태, `lib/api/` 경유, lucide-react + Framer Motion 단일화, `Pagination.tsx` 재사용, Calm Minimal 디자인 컨셉, 환경변수 분리.

**단, CLAUDE.md의 문서 링크가 깨져 있다**: `[PRD.md](./PRD.md)`, `[ROADMAP.md](./ROADMAP.md)`로 **루트 기준**을 가리키지만 실제 파일은 `docs/` 하위다(루트에는 `CLAUDE.md`만 존재). 매 세션 자동 로드되는 지침 파일이므로 실질 영향이 있다 → Major #10.

### 하위 가이드 정합성 (`docs/guides/`)

| 문서 | 상태 |
|---|---|
| `project-structure.md` | ✅ PRD 2.2와 완전 일치(`src/` 미사용, 경로/컴포넌트 구성, `providers/` 위치까지) |
| `forms-react-hook-form.md` | ✅ 모범적 — 비밀번호 6자 규칙과 "Server Actions 금지, Spring Boot REST 호출" 치환 지침을 서두에 명시 |
| `nextjs-16.md` | ⚠️ "현재 설치 버전: **16.3.0**" — 실제/PRD는 **16.3.1** |
| `styling-guide.md` | ⚠️ shadcn/ui **new-york** style·`next-themes`·`prettier-plugin-tailwindcss` 전제 — 실제 `components.json`은 **radix-nova**이고 세 패키지 모두 미설치 |
| `component-patterns.md` | ⚠️ "**Next.js 15.5.3** + React 19" 전제 — 스택과 불일치 |

### 내부 정합성 (기능 ↔ 메뉴 ↔ 페이지 ↔ API)

기능 13개가 5장 메뉴 구조와 6장 페이지 상세에 **전부 등장**하며, 6장의 각 페이지가 "구현 기능"을 역참조하는 구조라 추적성이 높다. **고아 기능 없음.** 다만:

- 3장 여정과 2.2 구조에 있는 **루트 진입 페이지(`app/page.tsx`, 토큰 유무 리다이렉트)**만 6장에 상세 항목이 없다. (Minor #8)
- PRD 1.3 배포 행의 "S3(정적 자산/**첨부**)"와 4.5 "첨부파일 = 범위 제외"가 상충한다. (Minor #9)

---

## ⚙️ 백엔드 검증 (Spring Boot 4 / JPA / Security)

<thought-process>

**관찰 1 — JSONB 매핑 방식이 PRD 어디에도 없다.** PRD 8.2는 `content`를 `JSONB`로, 13.1은 "HTML TEXT는 사용하지 않음"으로 확정했지만, **JPA에서 어떻게 매핑하는지**는 2.1 구조에도 8.3에도 없다.
**추론** — Hibernate 6/7은 `@JdbcTypeCode(SqlTypes.JSON)`로 네이티브 지원하므로 별도 라이브러리(hypersistence-utils 등)는 불필요하다. 그러나 이 프로젝트는 **Spring Boot 4.1 = Jackson 3(`tools.jackson`) + Hibernate 7**이며, 이 조합에서 Hibernate의 JSON FormatMapper 자동 탐지가 실패한다는 보고가 있다.
**근거** — Spring Boot 4.0 마이그레이션 가이드(Jackson 3 기본), Hibernate 포럼 이슈(`Could not find a FormatMapper for the JSON format`, Hibernate 7.1.1 + Jackson 3.0.0-rc9 + Boot 4.0.0+), 그리고 pom.xml에 JSON 매핑 관련 요소가 전혀 없음.
**결론** — [FACT] 위험 실재. `spring.jpa.properties.hibernate.type.json_format_mapper`로 커스텀 매퍼를 등록하거나 Jackson 2를 병행하는 결정이 **PRD 8.3/12.1에 명시되어야 한다.** → Critical #2

**관찰 2 — `default_schema=TodoListDB` + `currentSchema=TodoListDB` (실제 properties 확인).**
**추론** — PostgreSQL은 따옴표 없는 식별자를 소문자로 폴딩한다. Hibernate가 `TodoListDB.todos`를 따옴표 없이 방출하면 실제로는 `todolistdb.todos`를 찾는다. 스키마를 `CREATE SCHEMA "TodoListDB"`(따옴표)로 만들었다면 **런타임에 `relation does not exist`**가 발생한다.
**근거** — PostgreSQL 공식 렉시컬 문서 [FACT].
**결론** — 규칙("스키마명 `TodoListDB` 고정")은 지키되, **물리 스키마를 따옴표 없이 생성**(→ 실제 `todolistdb`)하거나 `default_schema` 값을 인용 식별자로 명시하는 것 중 **하나를 PRD 8.1에 확정**해야 한다. → Major #1

**관찰 3 — Soft Delete + 페이지네이션.**
**추론/결론** — Spring Data 파생 쿼리·JPQL은 Hibernate가 SQL을 생성하므로 `@SQLRestriction`이 **본 쿼리와 count 쿼리 양쪽에 반영**된다 [INFERENCE, 공식 문서에 count 명시는 없음 → 통합 테스트로 확인 권장]. 반면 **네이티브 쿼리에는 적용되지 않는다** [FACT] — `keyword` 검색을 네이티브로 구현하면 삭제 데이터가 노출된다. 또한 `EntityManager.find()`(by id)에 적용되는지는 javadoc에 명시가 없다 [UNCERTAIN] → `findByIdAndUserIdAndDeletedAtIsNull` 형태의 **파생 쿼리로 방어**하는 것이 안전하다(소유권 검증과 한 번에 해결되는 이점도 있다). → Major #8

**관찰 4 — N+1 위험.** `TodoResponse`에 사용자 정보가 없고(API_SPEC 4.1), 목록이 `todos` 단일 테이블 조회다.
**결론** — [INFERENCE] `Todo.user`를 `LAZY`로 두면 N+1이 발생하지 않는 **좋은 설계**다. 다만 PRD에 fetch 전략 명시가 없어 `@ManyToOne`의 JPA 기본값(EAGER)을 그대로 쓰는 실수 여지가 있다. → Minor #10

**관찰 5 — Security.** JWT 필터 + OAuth2 로그인 공존은 표준 구성이며 Spring Security 7에서도 `SecurityFilterChain` 빈 하나로 구성 가능하다 [INFERENCE].
**단, 두 개의 함정이 PRD/ROADMAP에 없다.**
(a) `SessionCreationPolicy.STATELESS`를 설정하면 Spring Security의 기본 `HttpSessionOAuth2AuthorizationRequestRepository`가 동작하지 못해 `authorization_request_not_found`가 발생한다 → 쿠키 기반 리포지토리를 쓰거나 세션 정책을 조정해야 한다 [INFERENCE, 높은 확신].
(b) API_SPEC은 "모든 응답은 **예외 없이** `ApiResponse`로 감싼다"고 선언하지만, **필터 단계의 401은 `@RestControllerAdvice`를 타지 않는다** → `AuthenticationEntryPoint`(+`AccessDeniedHandler`)에서 `AUTH_003~005`를 직접 직렬화해야 계약이 지켜진다 [FACT, 필터 체인 구조상]. → Major #5, #6

**관찰 6 — Bean Validation.** `@Email` + `@Size(min=6)` + `@NotBlank` + `@Size(max=255)`로 PRD의 모든 검증 규칙이 표현 가능하다. Boot 4에서도 `spring-boot-starter-validation` 유지 [FACT, pom 확인]. ✅ 문제 없음.

**관찰 7 — DTO 분리·레이어 구조.** PRD 2.1은 CLAUDE.md 컨벤션을 충실히 반영한다. 다만 `com.example.domain.todo`(엔티티·서비스)와 `com.example.todo`(컨트롤러·DTO)가 **동시에 존재**해 소속 패키지가 헷갈리기 쉽다(사용자 쪽은 `domain.user` + `auth`로 이름이 달라 문제가 없다). → Minor #11

</thought-process>

## 🎨 프론트엔드 검증 (Next.js 16 / React 19 / Tailwind 4)

<thought-process>

**관찰 1 — OAuth2 콜백 페이지가 쿼리스트링에서 토큰을 읽는다(PRD 6.3).**
**추론** — 서버 컴포넌트의 `searchParams`는 Next 16에서 **Promise**이며, 클라이언트에서 `useSearchParams()`를 쓰면 **가장 가까운 `<Suspense>` 경계까지 CSR로 전환**된다. Suspense 없이 작성하면 프리렌더 단계에서 경고/빌드 실패 경로를 밟는다.
**근거** — `node_modules/next/dist/docs/01-app/03-api-reference/04-functions/use-search-params.md` [FACT] 및 `.../upgrading/version-16.md`의 "Async Request APIs (Breaking change)" [FACT].
**결론** — 구현 가능하지만 **Suspense 래핑이 필수 요건**이다. PRD 6.3에 명시 권장. → Minor #1
**추가** — 같은 이유로 `app/(main)/todos/[id]/page.tsx`의 `params`도 Promise다.

**관찰 2 — 인증 가드가 `app/(main)/layout.tsx`(클라이언트 토큰 기반)다.**
**추론** — 토큰이 브라우저 저장소에 있으므로 서버에서 판단할 수 없다 → `"use client"` + 로딩 게이트가 불가피하며, 미처리 시 **비인증 콘텐츠 플래시**가 발생한다. `app/page.tsx`(토큰 유무 리다이렉트)도 동일하다. Next 16의 `proxy.ts`(구 middleware)는 브라우저 저장소를 볼 수 없으므로 대안이 아니다 [INFERENCE].
**결론** — 설계 자체는 유효. 토큰 저장 매체(localStorage / 메모리 / 쿠키)가 PRD에 미결정인 점이 실질 리스크다. → Minor #2

**관찰 3 — Tailwind 4 CSS-first + shadcn radix-nova.** `components.json`의 `"tailwind.config": ""`가 CSS-first를 확증하며 PRD 2.2/9.1과 일치한다 [FACT]. `tw-animate-css`도 설치되어 있다. ✅
**단** `styling-guide.md`는 new-york style/next-themes 전제라 신규 작업자가 잘못된 컴포넌트를 받을 수 있다. → Major #10

**관찰 4 — Tiptap 본문 왕복.** FE가 Tiptap JSON을 그대로 보내고 API_SPEC 4.1이 `content: object`로 받는 계약이라 **FE↔BE 형식이 정확히 일치**한다 ✅. 목록 카드에 본문을 표시하지 않아 XSS 표면도 작다. 다만 **본문 크기 상한이 어디에도 없다**(JSONB는 사실상 무제한) → 요청 바디 제한 등 방어 필요. → Minor #3

**관찰 5 — 페이지네이션 컴포넌트.** `PageResponse`의 `page/size/totalPages/hasNext`만으로 이전·다음·번호·생략(`...`)·`aria-current` 구현에 충분하다 ✅ [INFERENCE].

**관찰 6 — 미설치 패키지.** React Query·Framer Motion·Tiptap·RHF·Zod가 없다. 특히 **Framer Motion은 현재 `motion` 패키지로 배포**되므로 설치 시 패키지명을 확인해야 한다 [UNCERTAIN — 설치 시점 확인]. → Critical #4

</thought-process>

## 🔄 Step 7. 데이터 플로우 추론

`체크박스 토글 → PATCH /api/todos/{id}/status → 소유권 검증 → status 반전 → 목록 invalidate → 재조회`

- 흐름 자체는 기술적으로 성립한다 ✅.
- **빈틈**: (1) 토글이 **비멱등**이라 React Query 재시도/더블클릭 시 상태가 되돌아간다. (2) 응답이 `{id, status}`뿐이라 낙관적 업데이트 후 캐시에 `updatedAt`을 반영할 수 없다. → Major #2

`Todo 수정 저장 → PUT /api/todos/{id}`

- API_SPEC 4.5의 "**생략된 필드는 `null`로 갱신된다**"와 DB의 `status NOT NULL DEFAULT 'TODO'`가 충돌한다. 클라이언트가 `status`를 빼면 NOT NULL 위반(500) 경로가 열린다. → Major #3

`검증 실패 → COMMON_001 → 필드 맵을 RHF 에러로 매핑`

- API_SPEC 1.1은 "`data`는 실패 시 `null`"이라 못박았는데 1.3은 실패 응답의 `data`에 **필드 에러 맵**을 담는다. `ApiResponse<T>` 타입 정의가 양쪽을 동시에 만족할 수 없다. → Major #4

---

## Step 8. 복잡도 및 위험도 평가

| 영역 | 난이도 | 근거 |
|---|---|---|
| 백엔드 구현 | **3/5** | CRUD·JWT는 정형이나, Boot 4/Security 7/Jackson 3/Hibernate 7 조합의 신규 함정(JSON FormatMapper, 스타터 분리, 람다 DSL, CSRF 기본값)이 겹친다 |
| 프론트엔드 구현 | **3/5** | 화면 수는 적으나 Tiptap·React Query·Framer Motion이 아직 미설치이고 Next 16 async API/Suspense 규칙 적응이 필요 |
| FE-BE 통합·인증 | **4/5** | OAuth2 리다이렉트 + 무상태 JWT + 콜백 토큰 전달 + 401 리다이렉트 루프 방지가 가장 까다롭다 |
| 배포(AWS) | **3/5** | Amplify/EC2/RDS는 정형. 운영 도메인 기준 CORS·OAuth2 Redirect URI 갱신(M9 DoD에 이미 존재)이 실패 지점 |
| 외부 의존 위험 | **7/10** | Kakao 콘솔 정책(이메일 동의 항목·client secret 사용 여부) + 최신 메이저 4종의 미성숙 지점 |

**일정 영향 추정** — ROADMAP 총 12일 추정 대비, Critical #2(JSONB FormatMapper)와 Major #6(OAuth2 세션)에서 각각 **0.5일 내외의 추가 소요**가 예상된다(M1·M3).

---

## 🔴 Critical Issues (즉시 수정)

### Issue #1 — 실제 `application.properties`에 평문 DB 비밀번호가 커밋되어 있음 [INVARIANT]

<reasoning>
**발견 과정**: PRD 12.1이 명세한 `${DB_URL}` 플레이스홀더 방식이 실제로 적용됐는지 확인하려고 `todo-backend/src/main/resources/application.properties`를 열었다.

**문제 분석** [FACT]: 실제 파일에는 `spring.datasource.password=<REDACTED>`, `spring.datasource.username=postgres`, 접속 URL이 **평문**으로 들어 있다. 또한 `spring.jpa.hibernate.ddl-auto=update`가 프로파일 파일이 아니라 **공통 파일**에 있어 PRD 12.1의 "프로파일별 지정(dev=update / prod=validate)" 규칙과 어긋난다. `application-dev.properties`/`application-prod.properties`는 아직 존재하지 않는다.

**영향도**: CLAUDE.md 규칙 9와 PRD 10장·12.1을 정면으로 위반한다. 지금 상태로 M9까지 진행하면 RDS 접속정보가 그대로 커밋되는 경로가 열린다.

**해결 방안** [ALTERNATIVE]:
1. PRD 12.1대로 `${DB_URL}/${DB_USERNAME}/${DB_PASSWORD}` 플레이스홀더로 즉시 치환하고 로컬 값은 IDE 실행 구성/시스템 환경변수로 주입.
2. 또는 `application-dev.properties`를 `.gitignore`에 넣고 `application-dev.properties.example`만 커밋.
3. `ddl-auto`를 공통 파일에서 제거하고 프로파일별 파일로 이동(dev=update / prod=validate).
4. 이미 커밋 이력이 있다면 로컬 DB 비밀번호를 교체(로컬 전용이라도 습관화).

**긴급도**: 최상. M1 착수 전.

> ✅ **해소됨 (Task 035 실측 확인)** — `application.properties`는 `spring.datasource.password=${DB_PASSWORD}` 플레이스홀더만 사용하며 평문 값이 없다. `application-dev.properties`/`application-prod.properties`도 별도로 존재한다.
</reasoning>

### Issue #2 — `content` JSONB의 JPA 매핑 방식 미명시 + Jackson 3/Hibernate 7 FormatMapper 부재

<reasoning>
**발견 과정**: PRD 8.2/13.1이 `content`를 JSONB로 확정했으나 8.3(Soft Delete 방침)에는 매핑 지침이 없고, 2.1 구조와 pom.xml에도 JSON 매핑 관련 요소가 없어 공식 문서를 확인했다.

**문제 분석**:
- [FACT] Hibernate 6/7은 `@JdbcTypeCode(SqlTypes.JSON)`로 JSONB를 네이티브 매핑하며 별도 라이브러리가 필요 없다.
- [FACT] Spring Boot 4는 **Jackson 3(`tools.jackson`)**이 기본이다.
- [FACT] Jackson 3의 패키지 개명 때문에 **Hibernate 7의 JSON FormatMapper 자동 구성이 깨진다**는 사례가 Hibernate 공식 포럼에 보고되어 있다(`Could not find a FormatMapper for the JSON format`, Hibernate 7.1.1.Final + Jackson 3.0.0-rc9 + Spring Boot 4.0.0+). 이 프로젝트는 **Spring Boot 4.1.0**이다.
- [UNCERTAIN] Spring Boot 4.1에서 이 자동 구성이 해결되었는지는 확정하지 못했다. 관련 이슈/PR(spring-boot #33870, PR #42676)이 존재하나 4.1 반영 여부는 **M1 착수 시 실측 확인 필요**.

**영향도**: `Todo.content` 매핑이 실패하면 M1(엔티티) 단계에서 애플리케이션이 기동하지 못하거나 TODO-03/04가 전부 막힌다. PRD의 핵심 차별점(Tiptap 리치 본문)이 걸려 있다.

**해결 방안** [ALTERNATIVE]:
1. **가장 단순**: `Todo.content`를 `String`(Tiptap JSON 문자열)으로 두고 컬럼만 `jsonb`로 유지 — `@JdbcTypeCode(SqlTypes.JSON)` + `String` 조합이 FormatMapper 의존을 줄이는지 실측.
2. **정공법**: `AbstractJsonFormatMapper`를 `tools.jackson` 기준으로 구현하고 `spring.jpa.properties.hibernate.type.json_format_mapper=<FQCN>`으로 등록(포럼 권장 워크어라운드).
3. **회피**: Jackson 2를 클래스패스에 병행 추가.
4. **범위 조정**: MVP 한정으로 문자열 저장 후 조회 시 파싱 — 단 PRD 13.1의 "HTML TEXT 미사용" 결정과 충돌하지 않도록 "JSON 문자열을 jsonb 컬럼에" 형태를 유지.

**권고**: PRD 8.3에 "**JSONB 매핑 방침**" 소절을 신설해 1~2안 중 하나를 확정하고, 12.1 예시에 `json_format_mapper` 항목을 추가.

**긴급도**: 최상. M1 엔티티 설계 전.

> ✅ **해소됨 (Task 035 실측 확인)** — `Todo.java`가 `@JdbcTypeCode(SqlTypes.JSON)`(1안)으로 매핑되어 있고, `application.properties`의 `json_format_mapper` 설정은 주석 처리된 채로도 정상 동작해 정공법(2안) 없이 해소되었다. `TodoController`/`TodoService`가 `content` 필드를 포함해 완성되어 있어 애플리케이션 기동·CRUD 전 과정이 동작함을 코드로 확인했다.
</reasoning>

### Issue #3 — OAuth2 JWT를 쿼리스트링으로 전달 (PRD 6.3 / 7.1, API_SPEC 3.5)

<reasoning>
**발견 과정**: PRD 6.3 "쿼리 파라미터에서 토큰 추출·저장" + API_SPEC 3.5 `{APP_FRONTEND_URL}/oauth2/callback?token={accessToken}`.

**문제 분석** [FACT/INFERENCE]: 쿼리스트링의 24시간 유효 토큰은 (a) **브라우저 히스토리**에 잔류, (b) 콜백 페이지가 외부 리소스를 로드하면 **`Referer` 헤더로 유출** 가능, (c) CDN/프록시 **액세스 로그에 기록**, (d) 사용자가 URL을 복사·공유하면 계정 탈취로 직결된다. PRD 1.2가 "안전함"을 핵심 가치로 내세우고 10장이 시크릿 관리를 강조하는 것과 균형이 맞지 않는다.

**영향도**: 소셜 로그인 경로 전체(AUTH-03). 기능은 동작하므로 **기능 결함이 아니라 보안 결함**이다.

**해결 방안** [ALTERNATIVE] — 변경 비용이 낮은 순:
1. **URL 프래그먼트**: `/oauth2/callback#token=...` — 프래그먼트는 서버로 전송되지 않아 Referer/서버 로그 노출이 사라진다. FE는 `window.location.hash`로 읽고 즉시 `history.replaceState`로 제거. **PRD 대비 변경 최소**.
2. **일회용 교환 코드**: 콜백에 짧은 TTL(예: 30초) 1회용 코드를 실어 보내고 FE가 `POST /api/auth/oauth2/exchange`로 JWT 교환. 가장 안전하나 엔드포인트 1개 추가(API_SPEC·ROADMAP M3 갱신 필요).
3. **HttpOnly Secure 쿠키**: 백엔드가 콜백 응답에 쿠키 설정. 단 PRD 10장의 `Authorization: Bearer` 전제 및 크로스도메인(Amplify↔EC2) `SameSite=None` 요건과 충돌하므로 이 프로젝트에는 부적합.
4. 어떤 안이든 **토큰 취득 즉시 URL에서 제거**하고, 콜백 페이지에 외부 리소스를 로드하지 않는다.

**권고**: 최소 1안, 여유가 있으면 2안. 어느 쪽이든 PRD 6.3/7.1과 API_SPEC 3.5를 **동시에** 갱신해야 한다.

**긴급도**: 상. M3 착수 전.

> ✅ **해소됨 (Task 035 실측 확인)** — `OAuth2SuccessHandler.java:49`가 `frontendUrl + "/oauth2/callback#token=" + URLEncoder.encode(...)`로 **URL 프래그먼트(1안)**를 사용한다. 테스트(`OAuth2SuccessHandlerTest.java:57`)도 `#token=`으로 시작함을 검증한다.
</reasoning>

### Issue #4 — PRD 1.3의 "실제 프로젝트에 설치된 값" 주장이 5개 항목에서 사실이 아님

<reasoning>
**발견 과정**: PRD 1.3 서두의 단언을 `package.json`/`pom.xml` 전 항목과 대조했다.

**문제 분석** [FACT]: 버전이 명시된 항목(Spring Boot 4.1.0, JDK 21, jjwt 0.12.6, Next 16.3.1, React 19.2.8, lucide-react 1.33.0, radix-nova/neutral, `com.example`, `src/` 미사용)은 **전부 정확**하다. 그러나 **React Query(TanStack Query), Framer Motion, Tiptap, React Hook Form, Zod는 `package.json`에 존재하지 않는다.** 이 5개는 ROADMAP M0/M5/M7에서 설치될 예정 항목인데, PRD 1.3의 문장이 "설치된 값"으로 뭉뚱그려 선언한다.

부수적으로 "**shadcn/ui 4.18.0**"은 정확히는 `shadcn` **CLI 4.18.0**이며(컴포넌트 라이브러리 버전이 아님), `dependencies`에 위치해 런타임 번들 대상으로 오인될 수 있다(`devDependencies`가 적절).

**영향도**: 에이전트/개발자가 "설치되어 있다"는 전제로 `import { useQuery } from '@tanstack/react-query'`를 작성하면 즉시 빌드 실패한다. `docs/guides/forms-react-hook-form.md`는 "아직 설치되어 있지 않습니다(M5에서 설치)"라고 **정확히** 적어두었으므로 PRD와 가이드가 서로 모순되는 상태이기도 하다.

**해결 방안**: PRD 1.3 표에 **설치 상태 열**을 추가하거나("✅ 설치됨" / "⏳ M0·M5 설치 예정"), 서두 문장을 "**버전이 명시된 항목은 실제 설치값이며, 미명시 항목은 마일스톤 진행 중 설치한다**"로 정정. `shadcn/ui 4.18.0` → `shadcn CLI 4.18.0` 표기 정정.

**긴급도**: 상. 구현 착수 전(문서 신뢰성 문제이며 수정 비용은 낮다).

> ✅ **해소됨 (Task 035 실측 확인)** — `package.json`에 React Query(`@tanstack/react-query` 5.102.0), Framer Motion 후신(`motion` 13.1.1), Tiptap(`@tiptap/react`·`@tiptap/starter-kit` 3.30.2), React Hook Form(7.86.0), Zod(4.4.3)가 전부 설치되어 있음을 확인했다. PRD 1.3 표도 Task 035에서 해당 행을 ✅로 갱신했다. `docs/guides/forms-react-hook-form.md`의 "아직 설치되어 있지 않음" 문구도 Task 035에서 정정했다.
</reasoning>

---

## 🟡 Major Issues (개발 전 개선 권장)

### Major #1 — `TodoListDB` 스키마명과 PostgreSQL 대소문자 폴딩 [INVARIANT 관련]

[FACT] PostgreSQL은 따옴표 없는 식별자를 **소문자로 폴딩**한다. Hibernate `default_schema=TodoListDB`와 JDBC `currentSchema=TodoListDB`는 따옴표 없이 전달되므로 실제 조회 대상은 `todolistdb`가 된다. 스키마를 `CREATE SCHEMA "TodoListDB"`로 만든 환경에서는 **런타임 실패**, 따옴표 없이 만든 환경(실제 이름 `todolistdb`)에서는 정상 동작한다.
**권고**: PRD 8.1에 "논리 스키마명은 `TodoListDB`, **물리 식별자는 따옴표 없이 생성되어 `todolistdb`로 폴딩됨**"을 한 줄로 확정하거나, 인용 식별자를 쓸 경우 그 사실을 명시. ROADMAP M0 DoD("DB 연결 확인")에 **테이블이 생성된 스키마 실측** 항목을 추가하면 조기에 잡힌다.

> ✅ **해소됨 (Task 035 실측 확인)** — `application-dev.properties`가 `currentSchema=todolistdb`(소문자, 따옴표 없음)를 사용하고, ROADMAP M0 DoD에 `\dn`으로 물리 스키마명이 `todolistdb`임을 확인한 체크가 되어 있다.

### Major #2 — `PATCH /api/todos/{id}/status`의 "토글"이 비멱등

[FACT] 서버가 현재 상태를 반전시키는 바디 없는 요청은 재시도할 때마다 결과가 달라진다. React Query 재시도, 네트워크 타임아웃 후 재전송, 더블클릭이 모두 **상태 되돌림**을 유발한다. 또한 API_SPEC 4.6 응답은 `{id, status}`뿐이라 다른 엔드포인트의 `TodoResponse`와 계약이 다르고, 낙관적 업데이트 후 캐시 정합(`updatedAt`)을 맞출 수 없다.
**권고** [ALTERNATIVE]: 요청 바디를 `{"status": "DONE"}`으로 바꿔 **목표 상태를 클라이언트가 지정**(멱등)하고, 응답은 전체 `TodoResponse`로 통일. UI의 "토글" 경험은 그대로 유지된다(현재 값의 반대를 보내면 된다). `TODO_002`(잘못된 상태값)가 이미 정의되어 있어 이 변경과 잘 맞는다. PRD 4.2 TODO-05·7.2·API_SPEC 4.6 동시 수정.

> ✅ **해소됨 (Task 035 실측 확인)** — `TodoStatusUpdateRequest`가 `@NotBlank String status`로 **목표 상태를 클라이언트가 지정**하는 구조다(불변 규칙 13과 일치). 토글 방식이 아니다.

### Major #3 — `PUT /todos/{id}`의 "생략 필드 null 갱신" ↔ `status NOT NULL` 충돌

API_SPEC 4.5는 "전체 교체(PUT). 생략된 필드는 `null`로 갱신된다"라고 하지만 PRD 8.2의 `status`는 `NOT NULL DEFAULT 'TODO'`이고 `title`도 `NOT NULL`이다. 클라이언트가 `status`를 생략하면 제약 위반(500)이 된다.
**권고**: "`title`·`status`는 필수(`@NotBlank`/`@NotNull`), `content`·`dueDate`만 생략 시 `null`로 갱신"으로 규칙을 정밀화하고 PUT 요청 DTO의 Bean Validation과 함께 명시.

> ✅ **해소됨 (Task 035 실측 확인)** — `TodoUpdateRequest`가 정확히 권고안대로 `title`·`status`는 `@NotBlank`(필수), `content`·`dueDate`는 제약 없음(생략 시 null 허용)으로 구현되어 있다.

### Major #4 — `ApiResponse.data`의 실패 시 타입 모순

API_SPEC 1.1은 "`data`: 실패 시 `null`", 1.3은 실패 응답 `data`에 **필드 에러 맵**을 담는다. 두 규정이 동시에 성립할 수 없어 `ApiResponse<T>`의 Java 제네릭과 TS 타입 정의가 흔들린다.
**권고**: 1.1을 "실패 시 `null`, 단 검증 실패(`COMMON_001`)에 한해 `Record<string, string>` 필드 에러 맵"으로 정정하고, TS 타입을 분기(`data: T | Record<string,string> | null` 또는 별도 `ValidationErrorResponse`). PRD 13.2 표에도 각주 추가.

> ✅ **해소됨 (Task 035 실측 확인)** — `ApiResponse.java`가 정확히 권고안대로 `fail()`은 `data=null`, `validationFail(Map<String,String>)`은 필드 에러 맵을 담는 별도 팩토리 메서드로 분리되어 있다.

### Major #5 — 필터 단계 401은 `GlobalExceptionHandler`를 거치지 않는다

[FACT] `JwtAuthenticationFilter`/`AuthenticationEntryPoint`에서 발생하는 401은 `@RestControllerAdvice`에 도달하지 않는다. 따라서 API_SPEC의 "모든 응답은 예외 없이 `ApiResponse`로 감싼다"와 `AUTH_003/004/005`(401) 계약은 **커스텀 EntryPoint 없이는 지켜지지 않는다**. 프론트의 401 처리(토큰 삭제 후 리다이렉트)는 상태코드만 보므로 동작은 하지만, 에러코드 기반 메시지 표시는 깨진다.
**권고**: PRD 2.1 구조에 `JwtAuthenticationEntryPoint`(및 `AccessDeniedHandler`)를 추가하고, ROADMAP M2 DoD에 "인증 실패 응답도 `ApiResponse` 포맷" 항목 추가.

> ✅ **해소됨 (Task 035 실측 확인)** — `com.example.auth.jwt.JwtAuthenticationEntryPoint`가 구현되어 있고 `SecurityConfig`에 배선되어 있다.

### Major #6 — 무상태(STATELESS) 정책과 OAuth2 인가 요청 저장소의 충돌

[INFERENCE, 높은 확신] PRD 10장은 무상태 JWT를 전제하지만, Spring Security의 OAuth2 로그인은 기본적으로 **HttpSession 기반 `OAuth2AuthorizationRequestRepository`**로 state/PKCE를 보관한다. `SessionCreationPolicy.STATELESS`를 설정하면 콜백에서 `authorization_request_not_found`가 발생한다.
**권고**: PRD 10장 또는 ROADMAP M3 작업 항목에 "**OAuth2 인가 요청은 쿠키 기반 저장소를 사용하거나, `/oauth2/**`·`/login/oauth2/**` 경로에 한해 세션 생성을 허용**한다"를 명시. M3 DoD의 "Google 로그인 성공"만으로는 이 함정을 사전에 드러내지 못한다.

> ✅ **해소됨 (Task 035 실측 확인)** — `HttpCookieOAuth2AuthorizationRequestRepository`가 구현되어 **쿠키 기반 저장소(1안)**로 무상태 정책과 충돌 없이 동작한다.

### Major #7 — `users.provider` 단일 컬럼 모델이 "기존 이메일 계정 연동"(AUTH-03)을 표현하지 못함

PRD 4.1 AUTH-03과 API_SPEC 3.5는 "기존 이메일 → 해당 계정에 연동"을 요구하지만, 8.2의 `users`는 `provider`(NOT NULL)·`provider_id`를 **각 1개**만 갖는다. LOCAL 계정에 Google을 연동하면 `provider`를 덮어써야 하고, 그러면 다음 질문에 답이 없다.
- 덮어쓴 뒤 기존 **비밀번호 로그인은 계속 허용**되는가? (`password`는 남아 있음)
- 같은 이메일로 Google과 Kakao를 **둘 다** 연동하면?
- `GET /api/auth/me`의 `provider`는 무엇을 반환하는가?

**권고** [ALTERNATIVE]:
1. **MVP 최소 변경**: PRD 13.1 확정 사항에 "연동 시 `provider`는 **최초 가입 수단을 유지**하고, 소셜 재로그인은 이메일 매칭으로 허용. 로컬 비밀번호가 있으면 비밀번호 로그인도 계속 유효" 같은 **정책 한 줄**을 확정.
2. **정석**: `user_auth_providers` 연결 테이블 분리(MVP 범위 초과 — 4.5로 이관 권장).

어느 쪽이든 **정책을 명문화하지 않으면 M3에서 구현자가 임의 결정**하게 된다.

> ✅ **해소됨 (Task 035 실측 확인)** — PRD.md 13.1 "소셜 계정 연동 정책(AUTH-03)" 절에 **1안(MVP 최소 변경)**이 정확히 명문화되어 있다: `provider`는 최초 가입 수단을 유지하고, 소셜 재로그인은 이메일 매칭으로 허용하며, 로컬 비밀번호가 있으면 비밀번호 로그인도 계속 유효하다.

### Major #8 — `@SQLRestriction` 적용 범위의 불확실성 (Soft Delete 누수 위험)

- [FACT] 네이티브 쿼리에는 적용되지 않는다 → `keyword` 검색이나 통계성 쿼리를 네이티브로 작성하면 삭제 데이터가 노출된다.
- [UNCERTAIN] `EntityManager.find()`/`findById()`(ID 직접 로드)에 적용되는지는 javadoc에 명시가 없다 → TODO-02/04/05/06이 `findById` 기반이면 **soft-deleted 항목이 200으로 반환될 위험**.
- [INFERENCE] count 쿼리는 파생/JPQL 경로에서 함께 필터링될 것으로 보이나, `totalElements`는 사용자 신뢰와 직결되므로 **테스트로 확정**해야 한다.
- [INFERENCE] `@SQLDelete`에 적는 UPDATE 문은 **네이티브 SQL**이라 `default_schema`가 자동 적용되지 않을 수 있다 → 스키마 한정자 필요 여부 확인(Major #1과 연동).

**권고**: 조회는 `findByIdAndUser_IdAndDeletedAtIsNull` 형태의 **파생 쿼리로 소유권 검증과 Soft Delete를 동시에** 처리(방어적이고 다른 이슈와 무관하게 안전). ROADMAP M1 DoD("Soft Delete 조회 제외 테스트")에 **① findById 경로 ② count 쿼리 ③ keyword 검색 경로** 3가지를 명시적으로 추가.

> 🟡 **부분 해소 (Task 035 실측 확인)** — `TodoRepository.findByIdAndUser_IdAndDeletedAtIsNull`이 권고안대로 구현되어 단건 조회 경로(①)는 확인했다. count/keyword 경로(②③)까지 테스트로 확정됐는지는 이번 세션에서 확인하지 못해 손대지 않았다.

### Major #9 — 공통 헤더(PRD 6.7)와 AUTH-04/AUTH-05가 ROADMAP 어느 마일스톤에도 없음

PRD 6.7은 `Header.tsx`가 AUTH-04(내 정보 표시)·AUTH-05(로그아웃)·UI-01(테마 토글)을 **여기서만** 구현한다고 못박았지만, ROADMAP M5는 다크모드/Pagination만, M6은 로그인·회원가입·소셜·인증 가드만, M7은 Todo 화면만 다룬다. **헤더·사용자 메뉴·로그아웃이 어떤 작업 목록/DoD에도 등장하지 않는다.**
또한 ROADMAP **부록 A는 존재하지 않는 "PRD Phase 0~9"에 매핑**하고 있고(PRD 11장은 Phase를 정의하지 않음), **M1의 "PRD 3.2 스키마"는 실제로 PRD 8.2**다.
**권고**: ROADMAP M6 작업/DoD에 "공통 헤더(`Header.tsx`) + 사용자 메뉴(AUTH-04) + 로그아웃(AUTH-05)"을 추가하고, 부록 A는 삭제하거나 "PRD 11장이 ROADMAP에 위임" 문구로 대체, M1의 절 번호를 8.2로 정정. (**PRD가 아니라 ROADMAP 수정 대상**)

> ✅ **해소됨 (Task 035 실측 확인)** — ROADMAP M6 Task 027이 정확히 권고안대로 "공통 헤더·테마 토글 구현 (AUTH-04 / AUTH-05 / UI-01)"이며 `Header.tsx`, 내 정보 표시, 로그아웃이 DoD에 포함·체크되어 있다. 부록 A "기능 ID ↔ 마일스톤 매핑" 표도 `AUTH-04`/`AUTH-05`를 M6 Task 027로 정확히 매핑하고 있다.

### Major #10 — 문서 링크·가이드 스택 정보의 구식화

- `CLAUDE.md`의 `[PRD.md](./PRD.md)`·`[ROADMAP.md](./ROADMAP.md)`가 **루트를 가리키지만 실제 파일은 `docs/` 하위**다(루트에 두 파일 없음). 매 세션 자동 로드되는 지침 파일이라 영향이 크다 → `./docs/PRD.md`로 정정 필요.
- `guides/styling-guide.md`: shadcn **new-york** style·`next-themes`·`prettier-plugin-tailwindcss` 전제 → 실제는 **radix-nova**이고 세 패키지 모두 미설치.
- `guides/nextjs-16.md`: "현재 설치 버전 **16.3.0**" → 실제 **16.3.1**.
- `guides/component-patterns.md`: "**Next.js 15.5.3**" 전제 → 스택 불일치.

**권고**: PRD를 단일 진실로 두고 각 가이드 상단에 "스택 사실은 PRD 1.3을 따른다"는 각주 추가 + 위 값 정정.

> ✅ **해소됨 (Task 035 실측 확인)** — `CLAUDE.md`가 `./docs/PRD.md`·`./docs/ROADMAP.md`로 올바르게 링크한다. `guides/styling-guide.md`(radix-nova·next-themes/prettier 미설치 상태 정확히 기재), `guides/nextjs-16.md`("16.3.1"), `guides/component-patterns.md`("Next.js 16.3.1 + React 19.2.8")가 전부 실제 스택과 일치함을 Task 035에서 재확인했다. 각 가이드 상단에 "스택 사실의 단일 출처는 PRD 1.3" 각주도 이미 추가되어 있다. 단, `CLAUDE.md`의 "React Query·Framer Motion·Tiptap·React Hook Form·Zod는 아직 설치되지 않았다" 문구는 이제 사실과 다르다 — 이 문구는 `docs/PRD_VALIDATION.md` 범위 밖(CLAUDE.md 자체 수정)이라 이번 Task에서는 갱신하지 않았다.

### Major #11 — `keyword` 검색이 PRD 8.2 인덱스로 커버되지 않음

[INFERENCE] API_SPEC이 요구하는 "`title` 부분 일치, 대소문자 무시"는 `LOWER(title) LIKE '%kw%'` 형태가 되며, **선행 와일드카드 때문에 B-tree 인덱스를 사용할 수 없다**. PRD 8.2의 두 인덱스는 `user_id`/`status`/`deleted_at` 조합만 커버한다. 사용자당 데이터가 적은 MVP에서는 문제가 없지만, PRD 1.2가 "확장성: 대량 데이터 처리"를 핵심 가치로 내세운 것과는 어긋난다.
**권고**: PRD 7장 쿼리 표에 "대소문자 무시"를 명시(API_SPEC과 일치)하고, 8.2에 "검색은 `user_id` 선필터 후 스캔 — 대량 데이터 시 `pg_trgm` GIN 인덱스 도입 검토(MVP 범위 외)" 각주 추가.

> 🟡 **부분 해소 (Task 035 실측 확인)** — `TodoRepository.findByUser_IdAndTitleContainingIgnoreCase`로 대소문자 무시 검색은 구현되어 API_SPEC과 일치함을 확인했다. `pg_trgm` GIN 인덱스 도입 여부는 MVP 범위 외로 남아 있어 손대지 않았다.

---

## 🟢 Minor Suggestions (선택적 개선)

1. **OAuth2 콜백 페이지에 Suspense 필수** — [FACT] `useSearchParams()`는 가장 가까운 Suspense 경계까지 CSR로 전환된다. PRD 6.3에 "`<Suspense>` 래핑" 명시 권장. `[id]/page.tsx`의 `params`가 **Promise**라는 점(Next 16 파괴적 변경)도 2.2 주석에 추가.
2. **토큰 저장 매체 미결정** — `lib/auth/token.ts`만 있고 localStorage/sessionStorage/메모리 중 무엇인지 PRD에 없다. XSS 노출도와 새로고침 유지 여부가 달라지므로 13.1 확정 사항에 한 줄 추가 권장.
3. **`content` 크기 상한 없음** — JSONB는 사실상 무제한. 요청 바디 크기 제한 또는 문자수 상한을 API_SPEC 4.4에 명시 권장.
4. **`expiresIn: 86400000`의 단위** — OAuth2 관례상 `expires_in`은 **초**다. 문서 내에서는 일관되므로 동작에 문제는 없으나, 필드명을 `expiresInMs`로 바꾸거나 각주를 다는 편이 오해를 줄인다.
5. **`keyword` 대소문자 규칙이 PRD에 없음** — API_SPEC에만 "대소문자 무시"가 있다(Major #11과 연동).
6. **OAuth2 실패 파라미터(`?error=AUTH_007`)가 PRD 6.3에 없음** — API_SPEC 3.5에만 존재.
7. **PRD 7장 "기본 경로 `/api`"와 `/oauth2/...` 경로의 문장 충돌** — "OAuth2 경로는 Spring Security 규약상 `/api` 밖에 위치한다" 각주 권장.
8. **루트 진입 페이지(`app/page.tsx`)의 6장 상세 누락** — 3장 여정·2.2 구조에는 있으나 6장에 항목이 없다.
9. **S3 용도 모순** — 1.3 "S3(정적 자산/**첨부**)" vs 4.5 "첨부파일 범위 제외". "정적 자산 전용(첨부는 MVP 제외)"으로 정정 권장.
10. **`Todo.user` fetch 전략 미명시** — `@ManyToOne`의 JPA 기본값은 EAGER다. PRD 8장 또는 CLAUDE.md에 "연관관계는 `LAZY` 기본" 한 줄을 넣으면 목록 조회 N+1을 원천 차단할 수 있다.
11. **패키지 구조 모호(`com.example.domain.todo` vs `com.example.todo`)** — 같은 도메인명이 두 패키지에 나뉘어 있어 코드 생성 시 혼동 가능. `domain/todo/` 아래에 컨트롤러·DTO까지 모으거나, 컨트롤러 패키지를 `api/todo`처럼 구분되는 이름으로 변경 권장.
12. **Kakao `client-secret`은 선택 항목** — 카카오 개발자 콘솔에서 Client Secret은 기본 "사용 안 함"이다. PRD 12.1이 `OAUTH_KAKAO_CLIENT_SECRET`을 필수 환경변수로 나열하므로, 미사용 시 처리(속성 생략 또는 인증 방식 조정)를 12.2에 명시 권장. 아울러 `scope=profile_nickname,account_email`과 `client-authentication-method=client_secret_post`가 통상적인 조합이다(PRD 12.2가 이미 "구현 시점에 콘솔과 대조"를 지시한 점은 적절).
13. **`spring-boot-devtools`가 PRD 1.3에 없음** — 실제 pom에는 있다(무해).
14. **shadcn CLI가 `dependencies`에 위치** — `devDependencies`가 적절.
15. **정렬 인덱스 미세 최적화** — 기본 정렬이 `created_at DESC`이므로 `(user_id, deleted_at, created_at DESC)`가 더 잘 맞는다.
16. **CORS 세부 조건 미명시** — 허용 메서드/헤더(`Authorization`)/`allowCredentials` 여부를 PRD 10장에 한 줄 추가 권장(Amplify↔EC2 크로스도메인).
17. **Spring Security 7 구성 주의** — 람다 DSL 전용, `authorizeHttpRequests`/`requestMatchers` 사용, 무상태 API는 `csrf.disable()` 명시. ROADMAP M2 작업 노트에 추가하면 시행착오를 줄인다.

---

## 🏁 최종 판정

- ✅ 검증 완료 (그대로 구현 가능)
- **⚠️ 조건부 통과 (수정 후 구현 가능)** ← **선택된 판정**
- 🔄 대규모 수정 필요
- ⛔ 부분 구현 가능
- ❌ 재검토 필요

**판정 근거**

**Because** [FACT] 불변 규칙 7개가 PRD 전반에서 준수되고, PRD 7장의 11개 엔드포인트가 API_SPEC과 **1:1로 완전히 대응**하며(고아·누락 0건), 소유권 위반 404 정책이 PRD·API_SPEC·ROADMAP 세 문서에서 일치하고, PRD 1.3의 **버전 명시 항목이 실제 `pom.xml`/`package.json`/`components.json`과 정확히 일치**한다.
**And** [INFERENCE] 기능 ID → 마일스톤 매핑에 순서 위반이 없고, Soft Delete·페이지네이션·JSONB·OAuth2 모두 선택한 스택에서 구현 가능한 표준 기법이며, 발견된 문제 중 재설계를 요구하는 항목은 하나도 없다.
**But** [FACT/UNCERTAIN] (1) 실제 `application.properties`에 **평문 비밀번호가 커밋**되어 PRD 12.1·CLAUDE 규칙 9를 위반하고, (2) **Spring Boot 4의 Jackson 3 × Hibernate 7 조합에서 JSON FormatMapper 자동 구성이 깨진다는 보고**가 있는데 PRD에 JSONB 매핑 방침이 전혀 없으며, (3) OAuth2 토큰을 **쿼리스트링으로 전달**해 히스토리·Referer·로그 노출 위험이 있고, (4) PRD 1.3의 "실제 설치된 값" 주장이 **5개 항목에서 사실이 아니다.**
**Therefore** PRD의 목표와 범위는 타당하고 달성 가능하므로 재작성은 불필요하며, **Critical 4건을 수정하고 Major 11건 중 최소 #1~#6을 M1/M3 착수 전에 확정하면 그대로 구현에 들어갈 수 있다.**

### 신뢰도 및 위험도

- **기술 신뢰도**: 7/10 — 설계는 견고하나 최신 메이저 조합의 미검증 지점이 남아 있다
- **구현 복잡도**: 6/10 — 기능 범위는 작지만 인증 통합이 난이도를 끌어올린다
- **외부 의존 위험**: 7/10 — Kakao 콘솔 정책 + Boot 4/Jackson 3/Hibernate 7/Next 16 생태계 미성숙
- **전체 위험도**: 6/10

### 개발 진행 권장

1. **즉시 해결 (Critical)**
   - `application.properties`를 PRD 12.1의 `${ENV}` 방식으로 치환, `application-dev/prod.properties` 분리, 평문 비밀번호 제거
   - PRD 8.3에 **JSONB 매핑 방침** 소절 신설(+ `hibernate.type.json_format_mapper` 필요 여부를 M1 착수 시 실측)
   - OAuth2 토큰 전달 방식을 **프래그먼트 또는 일회용 교환 코드**로 변경 (PRD 6.3/7.1 + API_SPEC 3.5 동시 수정)
   - PRD 1.3의 "실제 설치된 값" 문장 정정 + 미설치 5개 항목 표기(`shadcn CLI` 표기 포함)

2. **개발 전 확인 (Major + UNCERTAIN)**
   - 스키마 대소문자 정책 확정(Major #1) — M0 DoD에 실측 항목 추가
   - `PATCH /status` 멱등화 + 응답 `TodoResponse` 통일(Major #2), `PUT` 필수 필드 규칙 정밀화(Major #3), `ApiResponse.data` 타입 규정 정정(Major #4)
   - 인증 실패 응답의 `ApiResponse` 래핑(Major #5), OAuth2 인가 요청 저장소 정책(Major #6)
   - 소셜 계정 연동 정책 명문화(Major #7)
   - `@SQLRestriction`의 `findById`/count/네이티브 적용 여부를 **M1 테스트로 확정**(Major #8) — 현재 최대 [UNCERTAIN]
   - ROADMAP 보정: 공통 헤더/AUTH-04/AUTH-05 마일스톤 배치, 부록 A 제거, "PRD 3.2"→"8.2"(Major #9)
   - `CLAUDE.md` 링크를 `./docs/...`로 정정, 가이드 3종의 스택 정보 갱신(Major #10)

3. **개발 중 고려 (Minor)**
   - Suspense 래핑·async `params`, 토큰 저장 매체 확정, `content` 크기 상한, LAZY fetch 명시, 패키지 구조 정리, CORS 세부 조건, Spring Security 7 람다 DSL/CSRF

4. **지속 검토**
   - Spring Boot 4.1.x 패치에서 Hibernate JSON FormatMapper 자동 구성이 해결되는지
   - Next.js 16 마이너 업데이트의 캐싱/라우팅 변경
   - Kakao 개발자 콘솔의 이메일 동의 항목·Client Secret 정책
   - Framer Motion 패키지명(`motion`) 및 Tiptap의 React 19 호환 버전

---

## 🔍 자기 검증 루프 (reflection)

- **놓친 불변 규칙 위반/문서 충돌은?** 7개 규칙을 PRD 본문·API_SPEC·ROADMAP 3중 대조했고, 엔드포인트는 11개 전수 매칭했다. 기능 ID 13개도 메뉴·페이지·API·마일스톤 4축으로 교차했다. 남은 사각지대는 `docs/guides/`의 세부 코드 예제(전문 정독 아님, 서두·스택 선언부만 확인)다.
- **논리 비약/환각은?** "Spring Boot 4 스타터 분리"는 pom.xml 실물 + 공식 마이그레이션 가이드 이중 확인했다. "STATELESS ↔ OAuth2 세션 충돌"은 공식 문서 원문을 인용하지 못했으므로 [FACT]가 아닌 [INFERENCE]로 표기했다. "Jackson 3 FormatMapper" 역시 Hibernate 공식 포럼(사용자 보고 + 메인테이너 응답) 근거이므로 이슈 자체는 [FACT], **Spring Boot 4.1에서의 해결 여부는 [UNCERTAIN]**으로 분리했다.
- **부정 편향은?** PRD의 강점을 함께 기록했다: 문서 간 역할 분리 선언, 기능 ID 기반 추적성, 404 소유권 정책의 근거 명시, 페이지네이션 규격의 별도 절, "MVP 이후 기능" 명시적 제외, 최신 메이저에 대한 경고 박스, 확정 사항/가정의 분리(13장). 이는 일반적인 MVP PRD보다 **정합성 수준이 높은 편**이다.

---

## 참고한 공식 문서 / 출처

- [Spring Boot 4.0 Migration Guide](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Migration-Guide) — 스타터 개명/분리, Jackson 3, Jakarta EE 11
- [Spring Boot 4.0 Release Notes](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Release-Notes)
- [Modularizing Spring Boot (spring.io)](https://spring.io/blog/2025/10/28/modularizing-spring-boot/)
- [Introducing Jackson 3 support in Spring](https://spring.io/blog/2025/10/07/introducing-jackson-3-support-in-spring/)
- [Missing FormatMapper for JSON format with Jackson 3.x, Hibernate 7.x (Hibernate 포럼)](https://discourse.hibernate.org/t/missing-formatmapper-for-json-format-with-jackson-3-x-hibernate-7-x/11819)
- [Auto-configure Hibernate with a JsonFormatMapper (spring-boot #33870)](https://github.com/spring-projects/spring-boot/issues/33870)
- [SQLRestriction (Hibernate Javadocs)](https://docs.hibernate.org/stable/core/javadocs/org/hibernate/annotations/SQLRestriction.html)
- [SoftDelete (Hibernate 7.1 Javadocs)](https://docs.hibernate.org/orm/7.1/javadocs/org/hibernate/annotations/SoftDelete.html)
- [PostgreSQL — Lexical Structure (식별자 대소문자 폴딩)](https://www.postgresql.org/docs/current/sql-syntax-lexical.html)
- [Spring Security — Client Authentication Support](https://docs.spring.io/spring-security/reference/servlet/oauth2/client/client-authentication.html)
- [Spring Security 5→6→7 Migration](https://ankurm.com/spring-security-5-to-6-to-7-migration-guide/)
- 로컬 Next.js 16 공식 문서(`todo-frontend/node_modules/next/dist/docs/`) — `use-search-params.md`(Suspense/프리렌더), `upgrading/version-16.md`(Async Request APIs, middleware→proxy)
