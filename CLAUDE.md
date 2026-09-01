# CLAUDE.md

> 이 파일은 **Claude Code가 매 세션 자동으로 읽는 프로젝트 지침서**입니다. 작업 전 반드시 이 규칙을 준수하고, 상세 요구사항은 [PRD.md](./docs/PRD.md), 진행 순서는 [ROADMAP.md](./docs/ROADMAP.md)를 기준으로 합니다.

---

## 1. 프로젝트 개요

이메일/소셜 로그인 기반 **Todo List 풀스택 웹앱**. 사용자가 Tiptap 리치 에디터로 할 일을 작성/관리한다.

- **Backend**: Spring Boot 4.x · JDK 21 · Maven · Spring Data JPA/Hibernate · Spring Security · JWT
- **Frontend**: Next.js 16(App Router) · React 19 · TypeScript · Tailwind CSS 4 · shadcn/ui · lucide-react · React Query · `motion`(Framer Motion 후신) · Tiptap · React Hook Form · Zod — **전부 설치·사용 중** (실측 2026-09-01, 아래 §1.1 참조)
- **DB**: PostgreSQL (스키마 `TodoListDB`)
- **구조**: 모노레포 `todo-project/{todo-backend, todo_frontend}` — 프론트 폴더명은 **언더스코어**(`todo_frontend`)이다. 문서·README에 하이픈(`todo-frontend`)으로 적힌 곳이 있어도 실제 경로는 이것이 맞다.

### 1.1 현재 구현 상태 (2026-09-01 기준)

- **백엔드**: Controller·Service·Repository·Spring Security(JWT+OAuth2)까지 구현 완료(M1~M4 상당). Google·Kakao 소셜 로그인, Soft Delete, 상태 변경 API, 예외 처리 공통화 전부 포함. `src/main/resources/application.properties`는 전부 `${ENV_VAR}` 플레이스홀더이며 저장소에 커밋되어 있다(Task 039에서 `.gitignore` 실수로 누락돼 있던 것을 발견해 고쳤다).
- **프론트엔드**: 인증(이메일/소셜)·Todo CRUD·상태 토글·삭제·페이지네이션·필터·Tiptap 에디터·Framer Motion 마이크로 인터랙션까지 구현 완료(M5~M7 상당). React Query `5.102.8`, `motion` `13.1.1`, Tiptap `3.30.6`, React Hook Form `7.87.0`, Zod `4.5.4` — 정확한 버전은 항상 `todo_frontend/package.json`을 우선 신뢰한다(이 문서·PRD 1.3도 시간이 지나면 patch 버전이 뒤처질 수 있다).
- **E2E(Playwright)**: `e2e/full-flow.spec.ts`·`account-isolation.spec.ts`·`session-expiry.spec.ts` 작성 완료. 실제 백엔드(PostgreSQL + 환경변수)가 없어 아직 한 번도 실행되지 않았다 — 실행 전에는 "통과"로 간주하지 않는다.
- **AWS 배포(M9)**: RDS·EC2·Amplify·CORS/Redirect URI 갱신 절차를 [docs/guides/aws-deployment.md](./docs/guides/aws-deployment.md)로 정리했으나, 실제 AWS 리소스는 아직 하나도 생성되지 않았다.
- **Git**: 3개 저장소(루트 `main` 단일, `todo-backend`/`todo_frontend` 각 `main`+`develop`) 모두 초기화·사용 중이다.
- ⚠️ **ROADMAP.md의 "완료" 표시를 무조건 신뢰하지 않는다.** 2026-09-01 세션에서 실제로는 존재하지 않는 파일·클래스를 언급하며 "완료했다"고 기록된 항목을 6건 이상 발견해 정정했다(Task 027·M7 헤딩·Task 032~035, `docs/PRD_VALIDATION.md`의 검증 주석 다수 포함). 중요한 작업을 이어가기 전에는 인용된 파일·클래스가 실제로 존재하는지 먼저 확인한다.

---

## 2. 개발 명령어

### Backend (`todo-backend/`)
```bash
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev   # 개발 서버 실행 (dev 프로파일)
./mvnw clean package                         # 빌드
./mvnw test                                  # 테스트 (테스트 스키마 todolistdb_test 사용)
```

> ⚠️ **JDK 21이 활성 상태여야 한다.** `java -version`이 17을 보고하면 `release version 21 not supported`로 실패한다.
> ⚠️ 실행 전 `DB_PASSWORD`·`JWT_SECRET` 등 환경변수가 주입되어 있어야 한다. 설정 파일에는 평문 값이 없다.

### Frontend (`todo_frontend/`)
```bash
npm run dev            # 개발 서버
npm run build          # 프로덕션 빌드
npm run lint           # 린트
npx playwright test    # E2E 테스트 (M5에서 도입 — 백엔드 기동 필요)
npx shadcn@latest add <component>   # shadcn/ui 컴포넌트 추가
```

> 작업 완료 후에는 항상 백엔드 `test`, 프론트엔드 `lint` + `build`가 통과하는지 확인한다.

---

## 3. 반드시 지킬 핵심 규칙 (Non-negotiable)

이 프로젝트의 **불변 요구사항**이다. 절대 임의로 변경하지 않는다.

1. **로그인 ID는 이메일만** 사용한다. 별도 username 필드를 만들지 않는다.
2. **비밀번호는 최소 6자**면 충분하다. 그 이상 복잡도 규칙을 추가하지 않는다.
3. **비밀번호는 BCrypt**로 해시하여 저장한다. 평문 저장 금지.
4. **JWT Access Token 만료는 24시간**. Refresh Token은 (요구 전까지) 만들지 않는다.
5. **삭제는 Soft Delete**. 물리 삭제(`DELETE`) 금지. `deleted_at`으로 처리하고 조회에서 제외한다.
6. **목록 조회는 페이지네이션 필수**. 전체 목록을 한 번에 반환하지 않는다.
7. **페이지네이션은 재사용 컴포넌트**(`components/common/Pagination.tsx`)로 구현한다.
8. **Todo는 본인 소유만 접근** 가능하다. 서버에서 소유권을 검증한다.
9. **민감정보(비밀번호·JWT 시크릿·OAuth 키)는 Git 커밋 금지**. 환경변수로만 주입한다.
10. **DB 스키마명은 `TodoListDB`** 로 고정한다. 단 PostgreSQL 소문자 폴딩 때문에 **물리 식별자는 `todolistdb`** 이다. 스키마는 반드시 **따옴표 없이** 생성한다 (PRD 8.1).
11. **소유권 위반은 404**로 응답한다. 403은 타인 리소스의 존재 여부를 노출하므로 쓰지 않는다.
12. **OAuth2 토큰은 URL 프래그먼트**(`#token=`)로 전달한다. 쿼리스트링은 Referer·히스토리·로그에 토큰을 남긴다.
13. **상태 변경 API는 멱등**해야 한다. 서버가 값을 반전시키는 토글 대신, 클라이언트가 목표 상태를 바디로 지정한다.
14. **JPA 연관관계는 `LAZY`를 명시**한다. `@ManyToOne` 기본값이 EAGER라 페이지네이션 목록에서 N+1이 발생한다.
15. **테스트는 개발 스키마를 쓰지 않는다.** 테스트는 `todolistdb_test`, 개발은 `todolistdb`. H2·Testcontainers는 쓰지 않는다.
16. **토큰은 `localStorage`에 저장**한다. 목록의 필터·페이지 상태는 **URL 쿼리스트링**을 단일 출처로 삼는다.

---

## 4. 아키텍처 & 코딩 컨벤션

### 공통
- 새 파일/기능은 [PRD.md](./docs/PRD.md)의 디렉토리 구조를 따른다.
- 커밋은 마일스톤/기능 단위로 작게. 커밋 메시지는 **한국어**, 형식은 `type: subject` (예: `feat: Todo 목록 페이지네이션 추가`).
- 브랜치: `main` ← `develop` ← `feature/*`.

#### ⚠️ 이 프로젝트는 Git 저장소가 3개다

디렉토리는 한곳에 모여 있지만 저장소는 분리되어 있다. **변경한 파일이 속한 저장소에서 커밋한다.**

| 변경 대상 | 커밋 위치 | 브랜치 |
|-----------|-----------|--------|
| `docs/` · `CLAUDE.md` · `.claude/` | **루트** `todo-project/` | `main` 단일 |
| 백엔드 코드 | `todo-backend/` | `main` + `develop` |
| 프론트 코드 | `todo_frontend/` | `main` + `develop` |

- 루트 `.gitignore`가 `todo-backend/`·`todo_frontend/`를 제외한다. **이 규칙을 지우지 않는다** — 지우면 git이 두 폴더를 embedded repository로 기록해 clone 시 빈 디렉토리가 된다.
  - ⚠️ 단, 실제 루트 `.gitignore` 파일에는 아직 `todo-frontend/`(하이픈)로 적혀 있다. 폴더명이 `todo_frontend`이므로 이 규칙이 **현재는 매치되지 않는다** — 루트를 `git init`하기 전에 반드시 언더스코어로 고쳐야 한다.
- 한 작업이 코드와 문서를 동시에 건드리면 **저장소별로 나눠 커밋**한다.
- 마일스톤 태그는 **실제 변경이 있었던 저장소에만** 붙인다.

### Backend
- **레이어**: Controller → Service → Repository. Controller에 비즈니스 로직 금지.
- **응답**: 모든 API는 공통 `ApiResponse<T>`로 래핑. 목록은 `PageResponse<T>`.
- **엔티티**: `BaseEntity` 상속(생성/수정일 + `deleted_at`). Soft Delete는 `@SQLDelete` + `@SQLRestriction("deleted_at IS NULL")`.
- **DTO ↔ 엔티티** 분리. 엔티티를 컨트롤러 응답으로 직접 노출하지 않는다.
- **검증**: 요청 DTO에 Bean Validation(`@Valid`, `@Email`, `@Size(min=6)` 등).
- **예외**: `GlobalExceptionHandler`에서 일괄 처리. 컨트롤러에서 try-catch 남발 금지.
- **설정 파일**: `application.properties` 형식 사용(YAML 아님). 값은 `${ENV}` 플레이스홀더로 주입.
  - 공통 설정은 `application.properties`, `ddl-auto`는 프로파일별 파일에서 지정한다 (`dev`: `update` / `prod`: `validate`).
  - **어떤 프로파일 파일에도 평문 시크릿을 적지 않는다.** 비민감 값에만 `${ENV:기본값}` 형태의 로컬 기본값을 허용한다.
- **인증 실패 응답**: 필터 단계 401은 `GlobalExceptionHandler`가 잡지 못한다. `JwtAuthenticationEntryPoint`로 `ApiResponse` 포맷을 직접 응답한다.
- **Soft Delete 조회**: `@SQLRestriction`에만 의존하지 말고, 단건 조회는 `findByIdAndUser_IdAndDeletedAtIsNull` 형태의 파생 쿼리로 소유권과 함께 검증한다.
- **연관관계**: `@ManyToOne(fetch = FetchType.LAZY)`를 항상 명시한다 (불변 규칙 14).
- **CORS**: `allowedMethods`에 **`PATCH`를 반드시 포함**한다(`TODO-05`가 PATCH). `allowCredentials=false` (Bearer 헤더 사용).
- **테스트**: JUnit 5 + Spring Boot Test(MockMvc). `src/test/resources/application.properties`는 main을 **shadow**하므로 필요한 설정을 전부 자립적으로 적는다.

### Frontend
- **App Router** 기준. 서버/클라이언트 컴포넌트 경계를 명확히(`"use client"`는 필요한 곳만).
- **서버 상태는 React Query**로 관리. 컴포넌트에서 직접 fetch 남발 금지.
- **API 호출은 `lib/api/` 클라이언트**를 통해서만. JWT는 클라이언트에서 자동 첨부(`localStorage` 저장).
- **목록 상태(page·status·keyword)는 URL 쿼리가 단일 출처.** React Query 키도 이 값으로 구성한다.
  - `useSearchParams()`를 쓰는 영역은 **`<Suspense>` 래핑 필수**(Next 16). 콜백 페이지는 프래그먼트를 쓰므로 해당 없음.
- **E2E 테스트는 Playwright만** 사용한다. Jest/Vitest/Testing Library는 MVP 범위 밖.
- **스타일**: Tailwind 유틸리티 우선 + shadcn/ui. 임의 인라인 스타일 지양. `cn()` 유틸로 클래스 병합.
- **디자인**: PRD의 "Calm Minimal" 컨셉(저채도 + 단일 액센트 + 다크모드 + 절제된 모션) 유지.
- **타입**: `any` 지양. 공통 타입은 `types/`에 정의.
- **아이콘은 lucide-react**, 애니메이션은 Framer Motion만 사용(라이브러리 혼용 지양).

---

## 5. 작업 시 주의사항

- **버전 확인**: Spring Boot 4 / Next.js 16 / React 19 / Tailwind 4는 최신 메이저다. 파괴적 변경이 있을 수 있으니, 익숙한 예전 방식으로 코드를 짜기 전에 **해당 버전의 최신 공식 문서를 먼저 확인**한다. (예: Tailwind 4의 CSS-first 설정, Next.js 16 캐싱/라우팅 변경, Spring Boot 4의 Jakarta EE 11)
- **추측 금지**: 라이브러리 API·설정이 불확실하면 임의로 작성하지 말고 문서를 확인한다.
- **일관성 우선**: 이미 있는 코드의 패턴/네이밍을 따른다. 새 패턴 도입 시 이유를 남긴다.
- **범위 준수**: 요청받은 마일스톤 범위만 구현한다. 확정되지 않은 기능(Refresh Token 등)을 임의 추가하지 않는다.

---

## 6. 참고 문서

모든 문서는 `docs/` 하위에 있다.

- [docs/PRD.md](./docs/PRD.md) — 상세 요구사항, DB 스키마, 디자인 방향 (**무엇을** 만들 것인가)
- [docs/API_SPEC.md](./docs/API_SPEC.md) — API 요청/응답 바디, 상태코드, 에러코드 (FE↔BE 계약)
- [docs/ROADMAP.md](./docs/ROADMAP.md) — 마일스톤(M0~M9), 의존 관계, 완료 기준(DoD) (**어떤 순서로**)
- [docs/PRD_VALIDATION.md](./docs/PRD_VALIDATION.md) — PRD 기술 검증 결과 (미해결 위험 항목 추적)
- [docs/guides/](./docs/guides/) — 프레임워크별 개발 가이드

> ⚠️ 가이드 문서와 PRD가 충돌하면 **PRD 1.3(기술 스택)을 사실로 삼는다.**
