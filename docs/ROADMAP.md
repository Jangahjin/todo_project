# ROADMAP — Todo List 풀스택 프로젝트

> **문서 목적**: [PRD.md](./PRD.md)를 기준으로 개발 진행 순서를 마일스톤(M0~M9) 단위로 정리한 로드맵입니다.
> 각 마일스톤은 **목표 → Task → DoD → 의존성** 순으로 구성되며, Task는 **며칠 내 완료 가능한 독립 작업 단위**입니다.
> API 계약은 [API_SPEC.md](./API_SPEC.md), 코딩 컨벤션·불변 규칙은 [CLAUDE.md](../CLAUDE.md), 미해결 기술 위험은 [PRD_VALIDATION.md](./PRD_VALIDATION.md)를 따릅니다.

---

## 0. 개요

### 0.1 마일스톤 요약

| # | 마일스톤 | 영역 | 예상 소요 | 선행 |
|---|----------|------|-----------|------|
| M0 | 프로젝트 초기화 | 공통 | 1일 (약 0.6일 완료) | - |
| M1 | 백엔드 도메인 & 공통 기반 | BE | **2일** | M0 |
| M2 | 인증(JWT) & CORS | BE | **2일** | M1 |
| M3 | OAuth2 소셜 로그인 | BE | **2일** | M2 |
| M4 | Todo API (CRUD·페이지네이션·Soft Delete) | BE | **2일** | M2 |
| M5 | 프론트엔드 기반 세팅 | FE | **1.5일** | M0 |
| M6 | 인증 화면 | FE | **2일** | M2, M5 (소셜 검증만 M3) |
| M7 | Todo 화면 (Tiptap·목록) | FE | **2.5일** | M4, M5 |
| M8 | 통합 & QA | 공통 | **1.5일** | M3~M7 |
| M9 | AWS 배포 | 인프라 | 1.5일 | M8 |

**합계: 직렬 기준 약 18일 / BE·FE 병렬 기준 약 12.5일**

> ⚠️ **기존 추정(총 12일)을 상향 조정했다.** 근거:
> - **M1 (1일 → 2일)**: PRD 8.4의 JSONB 매핑 실측, PRD 8.3의 Soft Delete 3경로 검증, **백엔드 테스트 인프라 신규 구축**이 모두 M1에 들어간다. [PRD_VALIDATION.md](./PRD_VALIDATION.md) Critical #2가 "M1에서 0.5일 추가 소요"를 예상한다.
> - **M2 (1.5 → 2일)**: `CorsConfig`·`JwtAuthenticationEntryPoint`·Spring Security 7 람다 DSL 적응 + MockMvc 통합 테스트 작성이 추가됐다.
> - **M3 (1.5 → 2일)**: OAuth2 인가 요청 저장소(쿠키 기반) 구현이 정형 작업이 아니다 (PRD_VALIDATION Major #6, 0.5일 추가 예상).
> - **M4 (1.5 → 2일)**, **M5 (1 → 1.5일, Playwright 구축 포함)**, **M6 (1.5 → 2일)**, **M7 (2 → 2.5일, Tiptap 미설치 상태에서 시작)**, **M8 (1 → 1.5일, E2E 자동화 스크립트 작성)**.

> M4는 M2 완료 후 M3와 **병렬 진행 가능**하며, M5는 M0 이후 백엔드와 **병렬 진행 가능**하다.

### 0.2 의존 관계 (흐름)

```
M0 ─┬─► M1 ─► M2 ─┬─► M3 ─┐
    │              │       ┊ (소셜 로그인 실동작 검증만)
    │              │       ┊
    │              └─► M4 ─┼───────────┐
    │                      ┊           │
    └─► M5 ─┬─► M6 ◄╌╌╌╌╌╌╌┘           ├─► M8 ─► M9
            │                          │
            └─► M7 ◄───────────────────┘
```

- **실선**: 코드 의존. 선행 마일스톤 없이는 구현을 시작할 수 없다.
- **점선(╌)**: 검증 의존. M6의 **구현**은 M3 없이 가능하지만(콜백 페이지는 API_SPEC 3.5 계약만으로 작성·테스트 가능), **실제 Google/Kakao 로그인 동작 확인**은 M3 완료 후에만 가능하다. 0.4의 M6 DoD 분리 규칙 참조.

### 0.3 진행 원칙

- **저장소는 3개다** — 루트(문서) · `todo-backend` · `todo-frontend`. 커밋은 **변경이 일어난 저장소에서** 한다 (Task 004).
- 각 마일스톤 완료 시 **Git 커밋 + 태그**(`m0-init`, `m1-domain` …)로 이력을 남긴다 (부록 B).
  - 태그는 **그 마일스톤에서 실제 변경이 있었던 저장소마다** 붙인다. 예: M1(백엔드 전용) → `todo-backend`에만 `m1-domain`. M0(공통) → 세 저장소 모두.
- 마일스톤은 **DoD(완료 기준)를 모두 충족**해야 다음으로 넘어간다.
- 브랜치 전략: `main`(안정) ← `develop`(통합) ← `feature/*`(Task 단위 작업 브랜치).
  - **문서 저장소는 `main` 단일 브랜치**로 운영한다 (통합·안정화할 빌드가 없음).
- **각 Task 완료 후 해당 테스트를 실행해 통과를 확인하고, 다음 Task로 넘어가기 전에 중단하고 지시를 기다린다.**
- 작업 완료 시 항상 백엔드 `./mvnw test`, 프론트엔드 `npm run lint` + `npm run build`가 통과해야 한다 (CLAUDE.md 2장).

### 0.4 Task 표기 규칙

- Task 번호는 **M0부터 연속**(`Task 001` …)이며, 마일스톤 안에서 재사용하지 않는다.
- 상태 표기:
  - `✅ 완료` — 구현·검증 모두 끝남
  - `🔥 우선순위` — 지금 착수할 작업
  - (표기 없음) — 대기
- 각 Task는 **구현 체크리스트**와, 검증이 필요한 경우 **테스트 체크리스트**를 함께 가진다.
- **별도 `/tasks/*.md` 파일은 만들지 않는다.** Task 상세는 이 문서 한 곳에서 관리한다 (진행 상태 이중 관리 방지 — PRD 11장의 "체크리스트는 ROADMAP 한 곳" 원칙).

### 0.5 테스트 전략 ⚠️ 신규

**현재 테스트 도구는 사실상 없다.** 백엔드는 Spring Boot 4의 `*-test` 스타터만 있고 테스트 DB 전략이 미정이며, 프론트엔드는 **테스트 도구가 하나도 설치되어 있지 않다**(Playwright/Jest/Vitest/Testing Library 전무). 아래와 같이 도입 시점을 확정한다.

| 영역 | 도구 | 도입 마일스톤 | 대상 |
|------|------|--------------|------|
| 백엔드 (API·비즈니스 로직) | **JUnit 5 + Spring Boot Test (MockMvc)** | **M1 (Task 007)** | 엔티티/Repository, Service, Controller 통합 테스트 |
| 백엔드 테스트 DB | **로컬 PostgreSQL 테스트 스키마 `todolistdb_test`** ✅ 확정 | **M1 (Task 007)** | JSONB·Soft Delete·스키마 폴딩 실측 |
| 프론트엔드 (사용자 플로우) | **Playwright** | **M5 (Task 023)** | 인증·Todo E2E 시나리오 |

**백엔드 테스트 DB — Testcontainers도 H2도 쓰지 않는다.** ✅ 확정 (PRD 1.3 · 13.1)

| 후보 | 판정 | 이유 |
|------|------|------|
| Testcontainers(PostgreSQL) | ❌ 제외 | Docker 실행 환경이 전제인데 **개발 머신에 Docker를 두지 않기로 확정**했다 |
| H2 | ❌ 제외 | `content`가 JSONB이고 검증 대상이 PostgreSQL 고유 동작(JSONB 왕복·식별자 소문자 폴딩·`@SQLRestriction`)이다. H2는 셋 중 무엇도 재현하지 못해 **통과해도 아무것도 보증하지 못한다** |
| **로컬 `todolistdb_test` 스키마** | ✅ **채택** | 실제 PostgreSQL이라 위 셋을 그대로 검증한다. Docker 불필요 |

**⚠️ 테스트 스키마는 개발 스키마와 반드시 분리한다.** 테스트는 데이터를 삽입·삭제·Soft Delete 처리하므로, 같은 스키마를 쓰면 실행할 때마다 개발 데이터가 오염된다.
분리는 **이미 `src/test/resources/application.properties`에 반영되어 있다** — `DB_URL` 환경변수를 상속하지 않고 `currentSchema=todolistdb_test`를 직접 지정하며, `hibernate.hbm2ddl.create_namespaces=true`로 스키마 자동 생성을 시도한다.

> **CI 승격 경로**: M9 이후 CI를 도입하면 그 환경에는 Docker가 있으므로 Testcontainers로 승격할 수 있다. 지금 결정은 **로컬 개발 환경 한정**이다.

**프론트엔드 — 단위 테스트는 MVP 범위 밖.**
화면 수가 적고(6개) 검증 가치가 사용자 플로우에 집중되어 있으므로, Jest/Vitest/Testing Library는 도입하지 않고 **Playwright E2E 하나로 통일**한다. 라이브러리 혼용을 피한다는 CLAUDE.md 4장 원칙과도 일치한다.

**E2E 실행 전제**: Playwright 시나리오는 **백엔드가 기동된 상태**를 요구한다(모킹하지 않는다 — 통합 리스크 축소가 목적). 실행 순서는 `백엔드 dev 기동 → playwright.config.ts의 webServer가 next dev 기동 → 테스트`다.

> ✅ **PRD 1.3 기술 스택 표에 테스트 도구 3행이 반영되었다** (테스트 BE / 테스트 DB / 테스트 FE). 스택 사실의 단일 출처는 PRD 1.3이다.

### 0.6 현재 진행 현황 스냅샷 (2026-08-21 실측)

| 항목 | 상태 |
|------|------|
| `todo-backend` 골격 (Spring Boot 4.1.0 / JDK 21 / pom 의존성) | ✅ 완료 |
| `todo-frontend` 골격 (Next.js 16.3.1 / React 19.2.8 / Tailwind 4 / shadcn) | ✅ 완료 |
| `application.properties` 3파일 분리 + 전 시크릿 `${ENV}` 처리 | ✅ 완료 |
| Kakao OAuth2 `provider`/`registration` 블록 | ✅ 완료 |
| `src/test/resources/application.properties` (테스트 전용 더미) | ✅ 완료 |
| 루트 `.gitignore` | ✅ 완료 |
| `./mvnw test` 통과 | ✅ 확인됨 (단, JDK 21 명시 필요) |
| **Git 저장소 구성** | ✅ **3-저장소 구성 완료** — 루트(문서, `main`) · `todo-backend`(`main`+`develop`, 커밋 3) · `todo-frontend`(`main`+`develop`, 커밋 3) |
| 루트용 GitHub 저장소 | ❌ **미생성** — `gh` CLI 없어 웹에서 직접 생성 필요 |
| 세 저장소 push | ❌ 미실행 |
| **`JAVA_HOME`** | ⚠️ `zulu17`을 가리킴 → **JDK 21(`C:\Program Files\Java\jdk-21.0.11`) 전환 필요** |
| 루트 `README.md` | ❌ 없음 |
| PostgreSQL `TodoListDB` 스키마 물리명 확인 | ⚠️ 미확인 (`\dn` 실측 필요) |
| 백엔드 도메인 코드 | ❌ 없음 (`TodoBackendApplication`, `TodoBackendApplicationTests`만 존재) |
| 프론트 라이브러리 (React Query·Framer Motion·RHF·Zod·Tiptap) | ❌ 미설치 |
| **프론트 테스트 도구** | ❌ **전부 미설치** (Playwright는 M5 Task 023) |
| Docker | ❌ 미설치 — **설치하지 않기로 확정.** 테스트는 로컬 `todolistdb_test` 스키마 사용 |
| 테스트 스키마 분리 설정 | ✅ 완료 (`src/test/resources/application.properties`) |
| `todolistdb_test` 스키마 **실제 생성 여부** | ⚠️ **미검증** — 엔티티가 없어 Hibernate가 생성할 네임스페이스가 없다. M1 Task 009에서 확인 |

---

## M0. 프로젝트 초기화 🏗️

**목표**: 모노레포 골격과 개발 환경, 형상관리 기반을 준비한다.

### Task 001: 백엔드·프론트엔드 프로젝트 골격 구성 ✅ 완료

**영역**: 공통

- [x] `todo-project/{todo-backend, todo-frontend}` 모노레포 배치
- [x] `todo-backend`: Spring Boot **4.1.0** (Maven, JDK 21) — `spring-boot-starter-webmvc`, `data-jpa`, `security`, `security-oauth2-client`, `validation`, PostgreSQL Driver, jjwt 0.12.6, Lombok
  - ⚠️ Spring Boot 4에서 스타터가 개명·분리되었다: `web` → **`webmvc`**, `oauth2-client` → **`security-oauth2-client`**, `test` → 기술별 **`*-test`** (pom.xml 반영 완료)
- [x] `todo-frontend`: Next.js **16.3.1** (App Router, TS) — Tailwind 4(CSS-first), shadcn CLI 4.18.0(style `radix-nova`, baseColor `neutral`), lucide-react
  - React Query·Framer Motion·React Hook Form·Zod는 **M5**, Tiptap은 **M7**에서 설치한다 (PRD 1.3 설치 상태 표)
- [x] 루트 `.gitignore` — 시크릿·빌드 산출물·IDE·로그 규칙 포함

### Task 002: 백엔드 설정 파일 분리 및 시크릿 환경변수화 ✅ 완료

**영역**: BE

- [x] `application.properties`(공통) / `application-dev.properties`(update) / `application-prod.properties`(validate) 3분할
- [x] 전 시크릿을 `${ENV}` 플레이스홀더로 치환 — 평문 비밀번호 제거 (CLAUDE.md 불변 규칙 9, PRD_VALIDATION Critical #1 해소)
- [x] Kakao OAuth2 `provider` 블록 + `registration`의 무기본값 항목 5종 등록 (PRD 12.2)
  - 누락 시 **M3가 아니라 기동 시점에** `Provider ID must be specified` / `authorizationGrantType cannot be null`로 실패한다
- [x] `src/test/resources/application.properties` 생성 — 테스트 전용 더미 값
  - ⚠️ 이 파일은 main 쪽 properties를 **완전히 가린다(shadow)**. 테스트에 필요한 설정을 자립적으로 모두 적어야 한다

### Task 003: JDK 21 개발 환경 고정 🔥 우선순위

**영역**: 공통 | **선행**: Task 001

현재 `JAVA_HOME`이 `C:\SpringBootProject\zulu17`을 가리켜 `java -version`이 **17**을 보고한다. JDK 21은 `C:\Program Files\Java\jdk-21.0.11`에 설치되어 있다.

- [ ] `JAVA_HOME`을 `C:\Program Files\Java\jdk-21.0.11`로 전환 (사용자 환경변수, 영구 적용)
- [ ] 새 셸에서 `java -version`이 **21**을 보고하는지 확인
- [ ] `./mvnw -v`가 Java 21을 사용하는지 확인
- [ ] 전환 후 `./mvnw clean test` 재실행 — 17로 되돌아가면 `release version 21 not supported`로 컴파일 자체가 실패한다

### Task 004: Git 저장소 구성 및 초기 커밋 ✅ 완료

**영역**: 공통 | **선행**: Task 001

**이 프로젝트는 모노레포가 아니라 세 개의 독립 저장소로 관리한다.** 백엔드·프론트엔드의 GitHub 저장소가 애초에 분리되어 있어, 루트를 문서 전용 저장소로 두는 구성을 택했다.

| 저장소 | 담당 | GitHub | 브랜치 |
|--------|------|--------|--------|
| 루트 `todo-project/` | `docs/` · `CLAUDE.md` · `.claude/` | (신규 생성 필요) | `main` |
| `todo-backend/` | 백엔드 코드 | `kdongjin/todo-backend` | `main` + `develop` |
| `todo-frontend/` | 프론트 코드 | `kdongjin/todo-frontend` | `main` + `develop` |

> ⚠️ **루트 `.gitignore`에서 하위 두 폴더를 반드시 제외한다.** 제외하지 않으면 git이 이들을 **embedded repository**로 인식해 내용 대신 커밋 해시만 기록하고(gitlink), clone 시 빈 디렉토리로 나온다.
> **문서 저장소는 `develop`을 두지 않는다** — 통합·안정화할 빌드가 없어 순수 오버헤드다.

- [x] 루트 `.gitignore`에 `todo-backend/`·`todo-frontend/`·`.metadata/` 제외 추가
- [x] `todo-backend`: `.gitignore`에 시크릿 규칙 보강 → `main` 브랜치 → **커밋 3개** → `develop` 생성
  - `chore: Spring Boot 4 프로젝트 골격 추가`
  - `chore: 설정 파일 프로파일 분리 및 시크릿 환경변수화`
  - `test: 테스트 전용 설정 및 스키마 분리`
- [x] `todo-frontend`: **커밋 2개** → `develop` 생성
  - `chore: shadcn/ui 초기화 및 디자인 토큰 설정`
  - `chore: VS Code Tailwind 4 린트 설정 추가`
- [x] 루트: `git init` → `main` → **커밋 1개** (`docs: PRD·ROADMAP·API_SPEC 및 프로젝트 지침 추가`)
- [x] **커밋 전 시크릿 스캔** — 세 저장소 모두 평문 시크릿 0건 확인 (불변 규칙 9)
- [x] embedded repository 경고 없음 확인
- [ ] **루트용 GitHub 저장소 생성** (사용자 작업 — `gh` CLI 미설치)
- [ ] 세 저장소 push — 코드 저장소는 `main`·`develop` 둘 다

### Task 005: PostgreSQL `TodoListDB` 스키마 준비 및 연결 검증 ✅ 완료

**영역**: 공통 | **선행**: Task 003

- [x] **따옴표 없이** 스키마 생성: `CREATE SCHEMA IF NOT EXISTS TodoListDB;` (PRD 8.1)
  - ❌ `CREATE SCHEMA "TodoListDB"` 금지 — 인용 식별자로 만들면 이름이 대문자로 고정되고, 따옴표 없이 전달되는 `default_schema`/`currentSchema`가 `todolistdb`를 찾다가 런타임에 실패한다
- [x] `\dn`으로 **실제 생성된 물리 스키마 이름이 `todolistdb`(소문자)** 임을 확인
- [x] **테스트 전용 스키마도 함께 생성**: `CREATE SCHEMA IF NOT EXISTS todolistdb_test;` (0.5 테스트 전략)
  - Hibernate의 `create_namespaces=true`가 자동 생성을 시도하지만, **수동으로 만들어 두면 M1에서 변수 하나가 준다**
  - `\dn` 결과에 `todolistdb`와 `todolistdb_test`가 **모두** 보여야 한다
- [x] 시스템 환경변수 주입 확인 — `DB_URL`, `DB_USERNAME`, `DB_PASSWORD`, `JWT_SECRET`, `APP_FRONTEND_URL` (PRD 12.1)
  - `OAUTH_*` 4종은 M3 착수 전까지 더미 값으로 두어도 무방하다
  - 실측(2026-08-22): `DB_PASSWORD`·`JWT_SECRET`은 `~/.bashrc`에서 주입, `DB_URL`·`DB_USERNAME`·`APP_FRONTEND_URL`은 dev 프로파일 로컬 기본값으로 커버됨
- [x] `./mvnw spring-boot:run -Dspring-boot.run.profiles=dev`로 기동해 DataSource 연결 성공 확인
  - 실측: `Default catalog/schema: postgres/todolistdb` + `Started TodoBackendApplication` 로그로 확인

### Task 006: README 초안 작성 ✅ 완료

**영역**: 공통 | **선행**: Task 004

- [x] 루트 `README.md` 생성 (**한국어**) — 프로젝트 소개, 모노레포 구조, 사전 요구사항(JDK 21 / Node / PostgreSQL)
- [x] 백엔드·프론트엔드 실행 명령어 (CLAUDE.md 2장과 일치)
- [x] **필요 환경변수 목록** — 값은 적지 않고 이름과 용도만 (PRD 12장 참조 링크)
- [x] 문서 안내 — `docs/PRD.md` / `docs/API_SPEC.md` / `docs/ROADMAP.md` 역할 구분
- [x] M8(Task 035)에서 실제 실행 절차 검증 후 최종본으로 다듬는다 — README 하단에 초안 표시 및 안내 문구 포함

**산출물**: 빌드 가능한 두 프로젝트 골격, Git 저장소, DB 연결 성공

**DoD**
- [x] **JDK 21이 활성 상태** — `java -version`이 21을 보고한다 (`JAVA_HOME` 확인) — 실측: `JAVA_HOME=C:\SpringBootProject\zulu21`
- [x] `todo-backend`가 `dev` 프로파일로 정상 기동 (빈 앱)
- [x] **`./mvnw test` 통과** — OAuth2 `registration`/`provider` 필수 항목 누락 시 여기서 기동 실패로 드러난다 (PRD 12.2)
- [x] `todo-frontend`가 `dev` 서버로 기동
- [x] DB 연결 확인 — `./mvnw test` 통과가 연결 성공을 간접적으로 시사하나, 아래 항목으로 직접 확인한다
- [x] **`\dn`으로 실제 생성된 스키마 이름이 `todolistdb`(소문자)임을 눈으로 확인** — 대문자로 생성됐다면 여기서 잡는다
- [x] `application.properties`에 **평문 시크릿이 없음**을 확인 (전부 `${ENV}` 플레이스홀더)
- [x] **`git init` 완료 · `main`/`develop` 브랜치 및 초기 커밋 존재**
- [x] 루트 `README.md` 초안 존재

**의존성**: 없음

---

## M1. 백엔드 도메인 & 공통 기반 🧱

**목표**: 테스트 인프라·엔티티·공통 응답·예외·Soft Delete 기반을 마련하고, **PRD가 M1 실측으로 미룬 기술 불확실성 2건을 확정**한다.

### Task 007: 백엔드 테스트 인프라 구축 ✅ 완료

**영역**: BE | **선행**: Task 003, 005

M1 이후 모든 DoD가 "테스트로 확인"을 요구하는데, **테스트 DB 전략이 아직 없다.** 이 Task를 M1의 첫 작업으로 둔다.

- [x] ~~테스트 DB 전략 확정~~ — ✅ **완료. 로컬 `todolistdb_test` 스키마로 확정** (0.5 · PRD 13.1)
  - Testcontainers 제외(Docker 미도입 확정), H2 제외(JSONB·식별자 폴딩 재현 불가)
- [x] ~~테스트 properties에 스키마 분리 반영~~ — ✅ **완료** (`src/test/resources/application.properties`)
  - `DB_URL`을 상속하지 않고 `currentSchema=todolistdb_test` 직접 지정, `hbm2ddl.create_namespaces=true` 설정
  - ⚠️ 테스트 properties는 main을 shadow하므로 **필요한 설정을 전부 자립적으로** 적는다 (Kakao `provider`/`registration` 블록 포함)
- [x] **`todolistdb_test` 스키마가 실제로 생성되는지 확인** — 실측: Hikari 연결 로그에 `Default catalog/schema: postgres/todolistdb_test` 확인됨 (스키마는 Task 005에서 수동 생성됨)
- [x] 테스트 베이스 클래스 마련 — `com.example.support.IntegrationTestSupport` (`@SpringBootTest` + `@AutoConfigureMockMvc` + `@Transactional`)
  - ⚠️ Spring Boot 4에서 `AutoConfigureMockMvc`의 패키지가 `org.springframework.boot.webmvc.test.autoconfigure`로 변경됨 (기존 `org.springframework.boot.test.autoconfigure.web.servlet`는 존재하지 않음)
  - 데이터 격리는 `@Transactional` 롤백 방식 채택 — 테스트 메서드 종료 시 자동 롤백
- [x] 패키지 구조 확정: 도메인 엔티티가 아직 없어(Task 008·009 예정) `domain`/`auth`/`todo` 하위 패키지는 실제 코드 추가 시점에 생성한다. 공통 테스트 인프라는 `com.example.support`에 둔다

**테스트 체크리스트 (JUnit 5 + Spring Boot Test)**
- [x] 기존 `TodoBackendApplicationTests.contextLoads()`가 새 전략에서 통과
- [x] 테스트 두 개를 연속 실행해도 데이터가 서로 오염되지 않음 — `com.example.support.TestIsolationVerificationTest`로 scratch 테이블 insert/count 검증(`@Transactional` 롤백으로 두 번째 테스트가 첫 번째 테스트의 행을 보지 못함, 테이블 자체도 DDL 롤백으로 남지 않음을 확인)

### Task 008: BaseEntity·JPA Auditing 및 Soft Delete 기반 구축

**영역**: BE | **선행**: Task 007

- [ ] `common/entity/BaseEntity.java` — `createdAt`, `updatedAt`, `deletedAt` (PRD 8.2 공통 컬럼)
- [ ] **`config/JpaAuditingConfig.java` — `@EnableJpaAuditing`** (PRD 2.1)
  - ⚠️ 이 설정이 없으면 `@CreatedDate`/`@LastModifiedDate`가 **조용히 동작하지 않아** `created_at`/`updated_at`이 `null`로 저장되고, `NOT NULL` 제약에서 터진다
  - `@EntityListeners(AuditingEntityListener.class)`를 `BaseEntity`에 부착
- [ ] Soft Delete: `@SQLDelete(sql = "UPDATE ... SET deleted_at = now() WHERE id = ?")` + `@SQLRestriction("deleted_at IS NULL")`
  - ⚠️ `@SQLDelete`의 UPDATE 문은 **네이티브 SQL**이라 `default_schema`가 자동 적용되지 않을 수 있다. 실행 SQL 로그로 스키마 접두사를 확인한다 (PRD 8.3)

**테스트 체크리스트 (JUnit 5)**
- [ ] 엔티티 저장 시 `createdAt`/`updatedAt`이 **자동으로 채워진다** (Auditing 동작 확인)
- [ ] 수정 시 `updatedAt`만 갱신된다
- [ ] `delete()` 호출 후 물리 행이 남아 있고 `deleted_at`이 채워진다

### Task 009: User·Todo 엔티티 및 Repository 구현

**영역**: BE | **선행**: Task 008

- [ ] `domain/user/`: `User`(BIGINT PK, `email` UNIQUE NOT NULL, `password` NULL 허용, `name`, `provider`, `providerId`), `AuthProvider` enum(LOCAL/GOOGLE/KAKAO), `UserRepository`
- [ ] `domain/todo/`: `Todo`(BIGINT PK, `user_id` FK NOT NULL, `title` VARCHAR(255) NOT NULL, `content` JSONB, `status`, `dueDate`), `TodoStatus` enum(TODO/DONE), `TodoRepository`
- [ ] **`Todo.content` JSONB 매핑** — `String` 필드 + `@JdbcTypeCode(SqlTypes.JSON)` + `@Column(columnDefinition = "jsonb")` (PRD 8.4)
- [ ] **연관관계는 `LAZY` 명시** — `@ManyToOne(fetch = FetchType.LAZY)`. JPA 기본값이 EAGER라 그대로 두면 목록 조회에서 N+1이 발생한다 (PRD_VALIDATION Minor #10)
- [ ] 인덱스: `(user_id, deleted_at)`, `(user_id, status, deleted_at)` (PRD 8.2)
  - 기본 정렬이 `created_at DESC`이므로 `(user_id, deleted_at, created_at DESC)` 형태도 함께 검토 (PRD_VALIDATION Minor #15)
- [ ] 단건 조회용 파생 쿼리 선언: `findByIdAndUser_IdAndDeletedAtIsNull(...)` — 소유권 검증과 Soft Delete 필터를 한 번에 처리 (PRD 8.3)

**테스트 체크리스트 (JUnit 5 + Spring Boot Test)** — *PRD 13.3의 미해결 가정 2건을 여기서 확정한다*
- [ ] **`Todo` 엔티티 포함 상태에서 애플리케이션이 기동된다** — Jackson 3 × Hibernate FormatMapper 문제 실측 (PRD 8.4 / PRD_VALIDATION Critical #2)
  - 실패 시: `spring.jpa.properties.hibernate.type.json_format_mapper`에 `tools.jackson` 기반 매퍼를 직접 구현해 등록한다 (`application.properties`에 주석 자리 마련됨)
- [ ] **Tiptap JSON이 손실 없이 저장·조회 왕복된다** (`{"type":"doc","content":[...]}` 원문 비교)
- [ ] **Soft Delete 3경로 모두 확인** (PRD 8.3 — `@SQLRestriction` 사각지대 검증)
  - [ ] 단건 조회(`findById` 계열) 경로에서 삭제분이 조회되지 않음
  - [ ] count 쿼리 — `totalElements`에 삭제분이 포함되지 않음
  - [ ] `keyword` 검색 경로에서 삭제분이 노출되지 않음
- [ ] `users.email` UNIQUE 제약 위반이 예외로 드러남
- [ ] 생성된 DDL이 `todolistdb` 스키마에 올라감 (스키마 폴딩 정합성)

### Task 010: 공통 응답·예외 처리 기반 구축

**영역**: BE | **선행**: Task 007

- [ ] `common/dto/ApiResponse.java` — `success`, `data`, `message`, `errorCode` (API_SPEC 1.1)
  - ⚠️ `data`는 성공 시 `T`, 실패 시 `null`, **단 검증 실패(`COMMON_001`)에 한해 필드 에러 맵**을 담는다 (API_SPEC 1.3)
- [ ] `common/dto/PageResponse.java` — `content`, `page`, `size`, `totalElements`, `totalPages`, `hasNext` (API_SPEC 1.2)
- [ ] `common/exception/ErrorCode.java` enum — `COMMON_001/002/500`, `AUTH_001~007`, `TODO_001/002` (API_SPEC 2장 전수 반영)
- [ ] `common/exception/CustomException.java`, `GlobalExceptionHandler.java`(`@RestControllerAdvice`)
- [ ] `MethodArgumentNotValidException` 핸들러 — 필드 에러를 `Map<String,String>`으로 변환해 `COMMON_001`로 응답

**테스트 체크리스트 (Spring Boot Test + MockMvc)**
- [ ] 성공 응답이 `{success:true, data:..., message:null, errorCode:null}` 형태
- [ ] `CustomException` 발생 시 매핑된 HTTP 상태코드와 `errorCode`가 응답됨
- [ ] `@Valid` 실패 시 `400` + `COMMON_001` + `data`에 **필드명→메시지 맵**

**DoD**
- [ ] 백엔드 테스트 인프라가 동작하고 전략이 문서화됨
- [ ] `users`, `todos` 테이블이 PRD 8.2 스키마대로 생성/검증됨 (ID는 BIGINT, 공통 컬럼 3종 포함)
- [ ] **JPA Auditing이 동작해 `created_at`/`updated_at`이 자동 기록됨**
- [ ] **`Todo` 엔티티 포함 상태에서 애플리케이션이 기동됨** (PRD 8.4 실측 완료)
- [ ] **Tiptap JSON이 손실 없이 저장·조회 왕복됨**
- [ ] Soft Delete **3경로(단건·count·keyword) 모두** 테스트로 확인됨 (PRD 8.3)
- [ ] 공통 응답/예외 포맷이 API_SPEC 1장·2장과 일치함
- [ ] **PRD 13.3의 미해결 가정 2건(JSONB 매핑 / `@SQLRestriction` 범위)에 결론이 기록됨**

**의존성**: M0

---

## M2. 인증 (JWT) & CORS 🔐

**목표**: 이메일 회원가입·로그인, JWT 발급/검증 체계, **그리고 프론트 연동에 필요한 CORS 기반**을 완성한다.

### Task 011: JWT 토큰 발급·검증 컴포넌트 구현

**영역**: BE | **선행**: Task 009

- [ ] `auth/jwt/JwtTokenProvider.java` — 토큰 생성·검증·클레임 추출 (jjwt 0.12.6 API)
- [ ] **만료 24시간 고정** (`jwt.expiration=86400000`, 불변 규칙 4). **Refresh Token은 만들지 않는다**
- [ ] 서명 키는 `${JWT_SECRET}` 환경변수에서만 읽는다 (불변 규칙 9)
- [ ] `auth/jwt/JwtAuthenticationFilter.java` — `Authorization: Bearer {token}` 파싱 후 `SecurityContext` 설정
- [ ] 토큰 상태별 `ErrorCode` 구분: 없음/형식오류 `AUTH_003`, 만료 `AUTH_004`, 서명 불일치 `AUTH_005`

**테스트 체크리스트 (JUnit 5)**
- [ ] 발급 토큰의 만료가 **정확히 24시간 뒤**
- [ ] 만료된 토큰이 `AUTH_004`로 판별됨
- [ ] 서명이 다른 토큰이 `AUTH_005`로 판별됨

### Task 012: SecurityConfig·CorsConfig 및 인증 진입점 구성

**영역**: BE | **선행**: Task 011

- [ ] `config/SecurityConfig.java` — 필터체인 구성
  - ⚠️ **Spring Security 7(Boot 4 동봉)은 람다 DSL만 지원**한다. `authorizeRequests()`/`antMatchers()`는 제거되었으므로 `authorizeHttpRequests { requestMatchers(...) }`를 쓴다
  - 무상태 JWT API이므로 **`csrf(csrf -> csrf.disable())`을 명시**한다
  - 인증 예외 경로: `/api/auth/signup`, `/api/auth/login`, `/oauth2/**`, `/login/oauth2/**` — 그 외 전부 인증 필요 (PRD 10장)
- [ ] `auth/jwt/JwtAuthenticationEntryPoint.java` — **필터 단계 401을 `ApiResponse` JSON으로 직접 직렬화**
  - ⚠️ 필터는 DispatcherServlet 바깥이라 `GlobalExceptionHandler`(`@RestControllerAdvice`)에 **도달하지 않는다**. 이게 없으면 "모든 응답은 `ApiResponse`로 감싼다"는 계약이 인증 실패에서만 깨진다 (API_SPEC 2.2 / PRD_VALIDATION Major #5)
- [ ] **`config/CorsConfig.java` (PRD 2.1)** — ⚠️ 기존 로드맵에서 어느 마일스톤에도 없던 항목
  - `allowedOrigins`: `${app.frontend-url}` (dev 기본 `http://localhost:3000`)
  - `allowedMethods`: `GET, POST, PUT, PATCH, DELETE, OPTIONS` — **`PATCH` 누락 시 상태 변경 API(TODO-05)만 조용히 실패**한다
  - `allowedHeaders`: `Authorization`, `Content-Type`
  - `allowCredentials`: **`false`** — 인증을 Bearer 헤더로 전달하므로 쿠키가 필요 없다. `true`로 두면 `allowedOrigins`에 와일드카드를 못 쓰는 제약만 늘어난다
  - `SecurityConfig`에서 `.cors(Customizer.withDefaults())`로 연결 (연결하지 않으면 설정이 무시된다)
  - 운영 도메인 갱신은 **M9(Task 039)** 에서 수행한다
- [ ] `PasswordEncoder` 빈 — **BCrypt** (불변 규칙 3)

**테스트 체크리스트 (Spring Boot Test + MockMvc)**
- [ ] 토큰 없이 보호 API 호출 시 **401 + `ApiResponse` 포맷 + `AUTH_003`**
- [ ] 만료 토큰 → `AUTH_004`, 위조 토큰 → `AUTH_005` (모두 `ApiResponse` 래핑)
- [ ] `/api/auth/signup`, `/api/auth/login`은 토큰 없이 접근 가능
- [ ] `Origin: http://localhost:3000`의 preflight(`OPTIONS`)가 `PATCH`를 허용 헤더에 포함해 응답

### Task 013: 이메일 회원가입·로그인·내 정보 API 구현

**영역**: BE | **선행**: Task 012

- [ ] `auth/AuthController.java` + `auth/dto/`(`SignupRequest`, `LoginRequest`, `TokenResponse`, `UserResponse`)
- [ ] `POST /api/auth/signup` (201) — **이메일 형식(`@Email`) + 중복 검증**, **비밀번호 `@Size(min=6)`**, BCrypt 저장
  - 불변 규칙 1·2: **username 필드를 만들지 않는다**, **6자 이상 외 복잡도 규칙을 추가하지 않는다**
  - 이메일 중복 → `409` + `AUTH_002`
- [ ] `POST /api/auth/login` (200) — 자격 검증 후 `accessToken` + `expiresIn: 86400000` 반환
  - ⚠️ **"이메일 없음"과 "비밀번호 불일치"를 구분하지 않고 모두 `AUTH_001`** — 가입 이메일 열거 방지 (API_SPEC 2.2)
- [ ] `GET /api/auth/me` (200) — 인증 주체 정보 반환. `provider`는 **최초 가입 수단**을 뜻한다 (PRD 13.1)
- [ ] `domain/user/UserService.java` — Controller에 비즈니스 로직을 두지 않는다 (CLAUDE.md 4장)

**테스트 체크리스트 (Spring Boot Test + MockMvc)**
- [ ] 정상 회원가입 → **201**, DB의 `password`가 **BCrypt 해시**(평문 아님)
- [ ] 잘못된 이메일 형식 / 5자 비밀번호 → **400 + `COMMON_001`**, `data`에 필드 에러 맵
- [ ] 중복 이메일 → **409 + `AUTH_002`**
- [ ] 로그인 성공 → 토큰 발급 · 그 토큰으로 `GET /api/auth/me` **200**
- [ ] 없는 이메일 / 틀린 비밀번호 → **둘 다 401 + `AUTH_001`** (응답이 서로 구별되지 않음)

**DoD**
- [ ] 잘못된 이메일/6자 미만 비밀번호가 거부됨
- [ ] 로그인 시 24h 만료 토큰 발급, 보호 API 접근 가능
- [ ] 만료/위조 토큰이 거부됨
- [ ] **인증 실패(401) 응답도 `ApiResponse` 포맷이며 `AUTH_003`/`AUTH_004`/`AUTH_005` 에러코드를 담는다** (API_SPEC 2.2)
- [ ] **CORS 설정이 존재하고 `http://localhost:3000` preflight가 `PATCH` 포함으로 통과한다**
- [ ] API_SPEC 3.1~3.3의 요청/응답 바디와 실제 응답이 일치함

**의존성**: M1

---

## M3. OAuth2 소셜 로그인 🌐

**목표**: 소셜 로그인(Google + Kakao)으로 가입/로그인하고, JWT를 프래그먼트로 전달한다.

### Task 014: OAuth2 사용자 정보 파싱 계층 구현

**영역**: BE | **선행**: Task 013

- [ ] `auth/oauth2/OAuth2UserInfo.java`(공통 인터페이스), `GoogleOAuth2UserInfo`, `KakaoOAuth2UserInfo`
  - ⚠️ **Kakao 응답은 중첩 구조**(`kakao_account.email`, `kakao_account.profile.nickname`)라 Google과 같은 파서를 쓸 수 없다 (PRD 12.2)
- [ ] `auth/oauth2/CustomOAuth2UserService.java` — 사용자 조회·생성
- [ ] **소셜 계정 연동 정책** (PRD 13.1) — 신규 이메일은 자동 가입(`password`=NULL), 기존 이메일은 해당 계정으로 로그인시키되 **`provider`를 덮어쓰지 않는다**(최초 가입 수단 유지)
- [ ] **이메일 미제공 케이스 처리 정책 확정** — 카카오 `account_email`이 선택 동의면 이메일이 내려오지 않는다. 콘솔에서 필수 동의로 설정하거나, 부재 시 `AUTH_007`로 실패 처리한다

**테스트 체크리스트 (JUnit 5)**
- [ ] Google 응답 샘플 → `email`/`name`/`providerId` 정확 추출
- [ ] **Kakao 중첩 응답 샘플** → `kakao_account.email`, `kakao_account.profile.nickname` 정확 추출
- [ ] 이메일 없는 Kakao 응답 → 확정한 정책대로 동작(예외 또는 대체값)
- [ ] 신규 이메일 → 자동 가입, `provider`가 `GOOGLE`/`KAKAO`, `password`가 `NULL`
- [ ] **기존 LOCAL 계정과 같은 이메일 → `provider`가 `LOCAL`로 유지되고 덮어써지지 않음**

### Task 015: OAuth2 인가 요청 저장소·성공 핸들러 구현

**영역**: BE | **선행**: Task 014

- [ ] ⚠️ **OAuth2 인가 요청 저장소 정책** — JWT는 무상태(`STATELESS`)지만 Spring Security의 OAuth2 로그인은 기본적으로 **HttpSession에 state/PKCE를 보관**한다. 전역 `SessionCreationPolicy.STATELESS`를 그대로 두면 콜백에서 **`authorization_request_not_found`** 가 발생한다 (PRD 10장 / PRD_VALIDATION Major #6)
  - **쿠키 기반 `AuthorizationRequestRepository`를 구현**하거나, `/oauth2/**`·`/login/oauth2/**` 경로에 한해 세션 생성을 허용한다
  - 쿠키 방식을 택하면 `HttpOnly` + `SameSite=Lax` + 짧은 만료(수 분)로 설정한다. OAuth2 왕복은 톱레벨 내비게이션이라 `Lax`로 충분하다
- [ ] `auth/oauth2/OAuth2SuccessHandler.java` — JWT 발급 후 프론트로 리다이렉트
- [ ] **토큰은 URL 프래그먼트로 전달**: `{APP_FRONTEND_URL}/oauth2/callback#token={accessToken}` (불변 규칙 12 / API_SPEC 3.5)
  - ❌ 쿼리스트링 금지 — `Referer` 헤더·브라우저 히스토리·프록시/CDN 액세스 로그에 24시간 유효 토큰이 남는다
- [ ] 실패 리다이렉트: `{APP_FRONTEND_URL}/oauth2/callback#error=AUTH_007`
- [ ] Google/Kakao 프로바이더 설정 확인 — 이미 `application.properties`에 반영됨 (Task 002)
- [ ] 개발자 콘솔에 Redirect URI 등록: `http://localhost:8080/login/oauth2/code/{google|kakao}`

**테스트 체크리스트 (Spring Boot Test + 수동 검증)**
- [ ] (자동) `OAuth2SuccessHandler`가 만드는 리다이렉트 URL이 **`#token=`(프래그먼트) 형식**이며 쿼리스트링에 토큰이 없음
- [ ] (자동) 실패 경로가 `#error=AUTH_007`을 생성
- [ ] (자동) 발급된 토큰으로 보호 API 접근 성공
- [ ] (수동) 실제 Google 로그인 → 신규 가입 → 재로그인
- [ ] (수동) 실제 Kakao 로그인 → 중첩 응답 파싱 확인
- [ ] (수동) **`authorization_request_not_found` 없이 콜백 완료** (세션 정책 검증)

> 외부 제공자 로그인은 자동화 테스트로 커버할 수 없다. **핸들러/파서 단위는 자동 테스트로, 실제 제공자 왕복은 수동 체크리스트로** 나눈다.

**DoD**
- [ ] Google 로그인으로 신규 가입 및 재로그인 성공 (수동)
- [ ] **Kakao 로그인 성공** — 중첩 응답(`kakao_account.email`) 파싱 및 이메일 미제공 케이스 처리 확인 (수동)
- [ ] 동일 이메일 로컬 계정과 소셜 계정 연동 처리 확인 — **연동 후에도 기존 비밀번호 로그인이 계속 동작**
- [ ] **`authorization_request_not_found` 없이 콜백이 완료됨** (세션 정책 검증)
- [ ] 콜백 URL이 **프래그먼트**로 토큰을 전달함 (자동 테스트로 형식 검증)
- [ ] 콜백으로 전달된 토큰으로 보호 API 접근 성공

**의존성**: M2 · (M2 완료 후 M4와 병렬 가능)

---

## M4. Todo API 📝

**목표**: Todo CRUD와 페이지네이션·Soft Delete·소유권 검증을 완성한다.

### Task 016: Todo 생성·상세·수정 API 구현

**영역**: BE | **선행**: Task 013

- [x] `todo/TodoController.java` + `todo/dto/`(`TodoCreateRequest`, `TodoUpdateRequest`, `TodoResponse`), `domain/todo/TodoService.java`
  - 📌 `com.example.domain.todo`(엔티티·서비스)와 `com.example.todo`(컨트롤러·DTO)가 나뉘어 있어 혼동하기 쉽다. 파일 생성 시 PRD 2.1 구조를 그대로 따른다
- [x] `POST /api/todos` (201) — `title` `@NotBlank @Size(max=255)`, `content`(JSONB, 선택), `dueDate`(선택)
- [x] `GET /api/todos/{id}` (200) — **`findByIdAndUser_IdAndDeletedAtIsNull`** 로 소유권 + Soft Delete 동시 처리 (PRD 8.3)
- [x] `PUT /api/todos/{id}` (200) — **`title`·`status`는 필수**(`@NotBlank`/`@NotNull`), `content`·`dueDate`만 생략 시 `null`로 갱신 (API_SPEC 4.5)
  - ⚠️ `status`를 선택으로 두면 `NOT NULL` 제약 위반(500) 경로가 열린다
  - 📌 구현 시 이 프로젝트가 Jackson 3(`tools.jackson.databind.*`)임을 확인해 `content` 요청 필드는 `JsonNode`로 받는다(`String`으로 받으면 JSON 객체 바인딩 시 매핑 예외). 응답 직렬화는 `com.fasterxml.jackson.annotation.JsonRawValue`(Jackson 3에서도 유지되는 패키지) 사용
  - 📌 `status`는 요청 DTO에서 `String @NotBlank`로 받고 서비스에서 `TodoStatus.valueOf` 실패 시 `TODO_002`로 매핑(enum 타입으로 받으면 역직렬화 실패가 `GlobalExceptionHandler` 미처리 500으로 샌다)
- [x] `TodoResponse.content`는 저장된 JSON 문자열을 **JSON 객체 그대로** 직렬화한다 (API_SPEC 4.1). 프론트는 파싱 없이 Tiptap에 전달한다
- [x] **소유권 위반·미존재는 모두 `404` + `TODO_001`** (불변 규칙 11 — 403은 타인 리소스 존재 여부를 노출한다)
- [ ] (선택) `content` 요청 바디 크기 상한 검토 — JSONB는 사실상 무제한이다 (PRD_VALIDATION Minor #3) — 이번 범위 밖, 미착수

**테스트 체크리스트 (Spring Boot Test + MockMvc)**
- [x] 생성 → **201** + `TodoResponse`, `content`가 JSON 객체로 직렬화됨
- [x] `title` 누락/256자 → **400 + `COMMON_001`**
- [x] **타인 소유 Todo 상세 조회 → 404 + `TODO_001`** (403이 아님)
- [x] `PUT`에서 `status` 누락 → **400 + `COMMON_001`** (500이 아님)
- [x] `PUT`에서 `content`·`dueDate` 생략 → 해당 필드가 `null`로 갱신됨

### Task 017: Todo 목록 페이지네이션·필터 API 구현

**영역**: BE | **선행**: Task 016

- [ ] `GET /api/todos?page=&size=&status=&keyword=` — Spring Data `Pageable` 사용 후 **`PageResponse<TodoResponse>`로 변환**해 반환 (불변 규칙 6)
- [ ] 기본 `page=0`, `size=10`, **최대 `size=100` — 초과 시 100으로 절삭** (PRD 4.4)
- [ ] 기본 정렬 **`created_at DESC` 고정** (MVP에는 정렬 선택 UI가 없다)
- [ ] `status` 필터(`TODO`/`DONE`, 미지정 시 전체) — 잘못된 값은 `400` + `COMMON_002`
- [ ] `keyword` — **`title` 대상 부분 일치, 대소문자 무시** (`LOWER(title) LIKE`). 본문(JSONB) 검색은 MVP 제외
  - ⚠️ **네이티브 쿼리를 쓰지 않는다.** `@SQLRestriction`이 적용되지 않아 삭제분이 노출된다 (PRD 8.3)
  - 📌 선행 와일드카드라 B-tree 인덱스를 타지 못한다. `user_id` 선필터로 충분하며 `pg_trgm`+GIN은 **MVP 범위 외**
- [ ] 결과가 없어도 **404가 아니라** `content: []`, `totalElements: 0` (API_SPEC 4.2)

**테스트 체크리스트 (Spring Boot Test + MockMvc)**
- [ ] 25건 생성 후 `size=10` → `totalElements=25`, `totalPages=3`, `page=0`에서 `hasNext=true`, `page=2`에서 `hasNext=false`
- [ ] **`size=200` 요청이 100으로 절삭됨**
- [ ] **타인 Todo가 목록에 섞이지 않음**
- [ ] `keyword=JANG` 이 `장보기`가 아닌 `jangbogi`류 제목을 **대소문자 무시**로 매칭
- [ ] Soft Delete된 항목이 `content`에도 `totalElements`에도 포함되지 않음
- [ ] 빈 결과 → **200** + `content: []` (404 아님)
- [ ] 잘못된 `status` 값 → **400 + `COMMON_002`**

### Task 018: Todo 상태 변경(멱등)·Soft Delete API 구현

**영역**: BE | **선행**: Task 016

- [x] `PATCH /api/todos/{id}/status` — **클라이언트가 목표 상태를 바디로 지정**한다: `{"status":"DONE"}` (불변 규칙 13)
  - ❌ **서버 반전 토글 금지** — 재시도·더블클릭·타임아웃 재전송에서 상태가 되돌아간다
  - `status` `@NotNull`, 잘못된 값은 `400` + `TODO_002`
  - 응답은 부분 응답이 아니라 **전체 `TodoResponse`** — 프론트가 낙관적 업데이트 후 `updatedAt`까지 캐시 정합에 쓴다 (API_SPEC 4.6)
- [x] `DELETE /api/todos/{id}` (200) — **Soft Delete**. `data: null`, `message: "삭제되었습니다."` (API_SPEC 4.7)
  - 물리 삭제 금지 (불변 규칙 5)

**테스트 체크리스트 (Spring Boot Test + MockMvc)**
- [x] `{"status":"DONE"}` → 200 + 전체 `TodoResponse`, `status=DONE`, `updatedAt` 갱신
- [x] **같은 요청을 두 번 보내도 결과가 동일함(멱등성)** — 두 번째 응답도 `DONE`
- [x] `{"status":"INVALID"}` → **400 + `TODO_002`**, `status` 누락 → **400 + `COMMON_001`**
- [x] 타인 Todo 상태 변경 → **404 + `TODO_001`**
- [x] 삭제 후 DB에 행이 남아 있고 `deleted_at`이 채워짐 (물리 삭제 아님)
- [x] **삭제된 Todo를 ID로 직접 조회해도 404** (Soft Delete 누수 없음)
- [x] 삭제된 Todo를 다시 삭제 → 404

**DoD**
- [x] CRUD·상태 변경 정상 동작
- [x] **같은 상태 변경 요청을 두 번 보내도 결과가 동일함(멱등성)**
- [x] 삭제가 Soft Delete로 처리되고 목록에서 제외됨
- [ ] 페이지네이션 응답에 `totalPages`, `hasNext` 등 포함. **`size=200` 요청이 100으로 절삭됨**
- [x] 타인 Todo 접근 시 차단 — **404 Not Found** 반환 (403은 리소스 존재 여부를 노출하므로 사용하지 않음)
- [x] **삭제된 Todo를 ID로 직접 조회해도 404** (Soft Delete 누수 없음)
- [ ] API_SPEC 4.1~4.7의 요청/응답 바디와 실제 응답이 일치함

**의존성**: M2 · (M3와 병렬 가능)

---

## M5. 프론트엔드 기반 세팅 🎨

**목표**: 라이브러리 설치, 디자인 시스템, 상태관리, API 클라이언트, 페이지네이션 컴포넌트, **E2E 테스트 환경**을 준비한다.

### Task 019: 프론트 라이브러리 설치 및 PRD 1.3 표 갱신

**영역**: FE | **선행**: M0

- [x] `@tanstack/react-query`, `react-hook-form`, `zod`, `@hookform/resolvers` 설치
- [x] **Framer Motion 설치 — 패키지명 확인 필요.** 최신 배포는 `motion` 패키지로 이관되었다. 설치 전 공식 문서로 정확한 패키지명·React 19 호환 버전을 확인한다 (CLAUDE.md 5장 "추측 금지")
- [x] Tiptap은 **M7(Task 028)** 에서 설치한다 — 지금 `import` 하지 않는다
- [x] 설치 후 **PRD 1.3 표의 설치 상태를 ✅와 실제 버전으로 갱신**
- [x] `npm run lint` + `npm run build` 통과 확인

### Task 020: 디자인 토큰·테마 프로바이더 구성 (Calm Minimal)

**영역**: FE | **선행**: Task 019

**PRD 9.1의 확정값을 그대로 반영한다** (기존 로드맵에 누락되어 있던 항목).

- [x] `app/globals.css`에 Tailwind 4 **CSS-first 디자인 토큰** 정의 (`tailwind.config.js`는 사용하지 않는다)
  - **베이스: 무채색 `neutral`** (`components.json`의 baseColor와 일치) — shadcn CLI 스캐폴딩 값 그대로 유지
  - **액센트: `Indigo` 단 1개 고정** (PRD 13.1 확정) — `--primary`/`--ring`을 Tailwind 4 공식 indigo 스케일(oklch)로 교체(라이트: indigo-600, 다크: indigo-400). 그라디언트 미사용
  - **타이포그래피: `Geist`** — `next/font/google`로 이미 로딩되어 있어 재작업 없이 그대로 사용
  - **모서리: `rounded-xl`**, 그림자는 **아주 옅은 소프트 섀도우** — `--radius`를 0.625rem→0.75rem으로 조정
  - 넉넉한 여백, 콘텐츠 중앙 정렬, 카드 기반 리스트를 전제로 한 스페이싱 스케일
- [x] `providers/ThemeProvider.tsx` — **다크모드(시스템 설정 연동 + 수동 토글, 선택값 로컬 저장)** (`UI-01`)
  - `next-themes`는 **도입하지 않고 자체 구현**했다. 기존 `globals.css`가 이미 `.dark` 클래스 토글 방식이라 Context + 인라인 스크립트(FOUC 방지, Next.js 16 공식 가이드 `preventing-flash-before-hydration` 패턴을 `.dark` 클래스 방식에 맞게 적용)로 충분해 불필요한 의존성을 추가하지 않았다
- [x] `providers/QueryProvider.tsx` — React Query `QueryClient` 설정(재시도·`staleTime` 기본값)
- [x] `app/layout.tsx`에 두 프로바이더 연결
- [x] 필요한 shadcn/ui 컴포넌트 추가 (`npx shadcn@latest add button input card dialog select checkbox label dropdown-menu avatar badge separator`) — style은 `radix-nova` 유지

### Task 021: API 클라이언트·토큰 저장 유틸·공통 타입 정의

**영역**: FE | **선행**: Task 019

- [ ] `types/` — `ApiResponse<T>`, `PageResponse<T>`, `TodoResponse`, `UserResponse`, `TodoStatus`, `ErrorCode` (API_SPEC 1장·2장 기준)
  - ⚠️ **`ApiResponse.data`는 실패 시 `null`이지만 `COMMON_001`에 한해 `Record<string,string>`** 이다. `ValidationErrorResponse` 별도 타입으로 분기한다 (API_SPEC 1.1)
  - `any` 금지 (CLAUDE.md 4장)
- [ ] `lib/auth/token.ts` — **`localStorage` 사용** ✅ 확정 (PRD 13.1)
  - httpOnly 쿠키는 불가 — PRD 10장이 `Authorization: Bearer`를 전제하므로 JS가 토큰을 읽어야 한다
  - `sessionStorage`는 탭을 닫으면 사라져 **24h 토큰의 의미가 없어진다**. XSS 노출도는 둘이 동일하므로 사용성이 나은 쪽을 택했다
  - ⚠️ **SSR 안전성**: `localStorage`는 서버에 없다. 접근하는 코드는 클라이언트 컴포넌트이거나 `typeof window !== 'undefined'` 가드를 둔다
- [ ] `lib/api/client.ts` — fetch 래퍼. **JWT 자동 첨부**, `NEXT_PUBLIC_API_BASE_URL` 사용
  - **401 응답 시 토큰 삭제 후 로그인 페이지로 리다이렉트** (API_SPEC 6장)
  - ⚠️ **리다이렉트 루프 방지** — 로그인/회원가입 페이지에서 받은 401은 리다이렉트하지 않는다
- [ ] `lib/api/auth.ts`, `lib/api/todo.ts` — API_SPEC의 엔드포인트별 함수
  - 목록 조회 함수는 `{ page, status, keyword }`를 인자로 받는다 — **URL 쿼리가 단일 출처**이므로 훅이 URL에서 읽어 그대로 전달한다 (PRD 6.4)
- [ ] `.env.local` 생성 (`NEXT_PUBLIC_API_BASE_URL=http://localhost:8080`) — **`.gitignore` 대상임을 확인**

### Task 022: Pagination 재사용 컴포넌트 구현

**영역**: FE | **선행**: Task 020

- [ ] `components/common/Pagination.tsx` — **재사용 컴포넌트로 구현** (불변 규칙 7, `UI-02`)
- [ ] 이전/다음 버튼, 페이지 번호, **생략 표시(`...`)**, 현재 페이지 하이라이트
- [ ] shadcn/ui + lucide-react 아이콘 사용
- [ ] 접근성: `aria-current="page"`, 각 버튼에 `aria-label`, 키보드 포커스 이동
- [ ] props 설계: `page`(0-base), `totalPages`, `onPageChange` — `PageResponse`와 그대로 맞물리게 한다
- [ ] 경계 처리: `totalPages<=1`이면 렌더 생략, 첫/마지막 페이지에서 이전/다음 비활성화

### Task 023: Playwright E2E 테스트 환경 구축

**영역**: FE | **선행**: Task 019

**프론트엔드 테스트 도구가 하나도 없다.** M6·M7의 사용자 플로우 검증이 전부 이 Task에 의존한다.

- [x] `@playwright/test` 설치 + `npx playwright install chromium`
- [x] `playwright.config.ts` — `baseURL`, `webServer`(`npm run dev`), 리포터, 타임아웃
- [x] `e2e/` 디렉토리 구조와 네이밍 규칙 확정 (`auth.spec.ts`, `todo.spec.ts` …)
- [x] `package.json`에 `"test:e2e": "playwright test"` 스크립트 추가
- [x] **테스트 계정 준비 방식 확정** — 시나리오마다 랜덤 이메일로 가입하는 방식을 기본으로 한다(격리 보장, 백엔드에 시드 API를 만들지 않아도 됨)
- [x] ⚠️ **E2E는 백엔드 기동을 전제로 한다.** 실행 순서(백엔드 dev → `npm run test:e2e`)를 README에 적는다

**테스트 체크리스트 (Playwright)**
- [x] 스모크 시나리오 1건 통과 — 루트(`/`) 접속 시 렌더 오류 없음
- [ ] 백엔드 미기동 시 실패 원인이 명확히 드러남

**DoD**
- [x] 라이브러리 설치 완료 및 **PRD 1.3 설치 상태 표 갱신 완료**
- [x] 다크/라이트 토글 동작, 시스템 설정 연동
- [x] **PRD 9.1 확정값(액센트 `Indigo` / `Geist`·`Inter` / `rounded-xl` / `neutral` 베이스)이 디자인 토큰에 반영됨**
- [x] React Query·테마 프로바이더가 `app/layout.tsx`에 적용됨
- [x] `Pagination` 컴포넌트 단독 렌더 및 페이지 이동 콜백 동작, `aria-current` 적용
- [ ] API 클라이언트가 **401 응답 시 토큰 삭제 후 로그인 페이지로 리다이렉트**(루프 없이)
- [x] **Playwright가 설치되고 스모크 테스트가 통과함**
- [x] `npm run lint` + `npm run build` 통과

**의존성**: M0 · (백엔드와 병렬 가능)

---

## M6. 인증 화면 🔑

**목표**: 로그인/회원가입, 소셜 로그인 진입, 인증 가드, 공통 헤더를 완성한다.

> ⚠️ **Next.js 16 주의**: 동적 `params`/`searchParams`는 **Promise**이며(Async Request APIs 파괴적 변경), `useSearchParams()`는 가장 가까운 `<Suspense>` 경계까지 CSR로 전환된다. Suspense 없이 쓰면 프리렌더 단계에서 문제가 된다. 착수 전 `todo-frontend/node_modules/next/dist/docs/`의 해당 가이드를 확인한다.

### Task 024: 로그인·회원가입 화면 구현

**영역**: FE | **선행**: Task 021

- [ ] `app/(auth)/login/page.tsx`, `app/(auth)/signup/page.tsx` — 중앙 정렬 카드 폼 (PRD 9.2)
- [ ] `components/auth/LoginForm.tsx`, `SignupForm.tsx` — React Hook Form + Zod
- [ ] 클라이언트 검증: **이메일 형식**, **비밀번호 6자 이상**, 비밀번호 확인 일치
  - ⚠️ **6자 이상 외의 복잡도 규칙(대문자·특수문자 등)을 추가하지 않는다** (불변 규칙 2)
- [ ] 서버 검증 실패(`COMMON_001`) 응답의 **`data` 맵을 RHF 필드 에러로 매핑** (API_SPEC 1.3)
- [ ] `AUTH_002`(이메일 중복), `AUTH_001`(로그인 실패) 메시지 처리
- [ ] 회원가입 성공 → 로그인 페이지 / 로그인 성공 → 토큰 저장 후 Todo 목록 (PRD 3장 여정)
- [ ] 인증 상태로 로그인/회원가입 접근 시 Todo 목록으로 리다이렉트

**테스트 체크리스트 (Playwright)**
- [ ] 랜덤 이메일 회원가입 → 로그인 페이지 이동 → 로그인 → Todo 목록 진입
- [ ] 5자 비밀번호 입력 시 **제출 전 클라이언트 에러 메시지** 표시
- [ ] 중복 이메일 가입 시도 시 이메일 필드에 에러 표시
- [ ] 틀린 비밀번호 로그인 시 에러 메시지 표시 + 페이지 유지

### Task 025: 소셜 로그인 버튼 및 OAuth2 콜백 페이지 구현

**영역**: FE | **선행**: Task 024

- [ ] `components/auth/SocialLoginButtons.tsx` — Google/Kakao 버튼 → `{API_BASE}/oauth2/authorization/{provider}` 로 **전체 페이지 이동**(fetch 아님)
- [ ] `app/oauth2/callback/page.tsx` — **프래그먼트 처리** (API_SPEC 3.5)
  1. `window.location.hash`에서 `token` 또는 `error` 추출
  2. 토큰 저장
  3. **`history.replaceState(null, '', '/oauth2/callback')` 로 URL에서 프래그먼트 제거**
  4. Todo 목록으로 이동
  - ⚠️ 프래그먼트는 서버로 전송되지 않으므로 **클라이언트 컴포넌트**여야 한다
  - ⚠️ **이 페이지에서는 외부 리소스(폰트·이미지·분석 스크립트)를 로드하지 않는다** (PRD 6.3)
- [ ] `#error=AUTH_007` 수신 시 로그인 페이지로 이동 + 안내 메시지
- [ ] 로딩 스피너 표시

**테스트 체크리스트 (Playwright — M3 없이도 검증 가능)**
- [ ] `/oauth2/callback#token=<유효토큰>` 직접 접속 → 토큰 저장 → Todo 목록 이동
- [ ] **이동 후 주소창에 토큰이 남아 있지 않음** (`location.hash`가 비어 있음)
- [ ] `/oauth2/callback#error=AUTH_007` → 로그인 페이지 + 에러 메시지
- [ ] 소셜 버튼 클릭 시 `/oauth2/authorization/google` 로 내비게이션이 시작됨

> 위 시나리오는 **백엔드 OAuth2(M3) 없이도** 프래그먼트 계약만으로 검증된다. 실제 제공자 왕복은 M8(Task 033) 수동 검증에서 확인한다.

### Task 026: 인증 가드 및 루트 진입 라우팅 구현

**영역**: FE | **선행**: Task 021

- [ ] `app/(main)/layout.tsx` — **인증 가드**. 미인증 시 로그인 페이지로 리다이렉트
- [ ] `app/page.tsx` — 토큰 유무로 Todo 목록 / 로그인 분기 (PRD 3장)
- [ ] ⚠️ **비인증 콘텐츠 플래시 방지** — 토큰이 브라우저 저장소에 있어 서버가 판단할 수 없다. 판정 전까지 로딩 게이트를 렌더한다
  - Next 16의 `proxy.ts`(구 middleware)는 브라우저 저장소를 볼 수 없으므로 대안이 아니다
- [ ] 401 리다이렉트(Task 021)와 가드 리다이렉트가 **서로 루프를 만들지 않는지** 확인

**테스트 체크리스트 (Playwright)**
- [ ] 토큰 없이 `/todos` 접속 → 로그인 페이지로 리다이렉트
- [ ] **리다이렉트 전에 Todo 목록 콘텐츠가 번쩍이지 않음**
- [ ] 만료/위조 토큰을 저장소에 심고 `/todos` 접속 → 토큰 삭제 후 로그인 페이지 (루프 없음)

### Task 027: 공통 헤더·테마 토글 구현 (AUTH-04 / AUTH-05 / UI-01)

**영역**: FE | **선행**: Task 026

PRD 6.7의 공통 헤더. 세 기능이 여기서만 구현되므로 별도 Task로 둔다.

- [x] `components/common/Header.tsx` — 로고(→ Todo 목록), **Todo 목록 / 새 Todo** 내비게이션
- [x] **`AUTH-04` 내 정보 표시** — `GET /api/auth/me`로 이메일·이름 조회 후 사용자 메뉴에 표시 (React Query로 캐싱)
- [x] **`AUTH-05` 로그아웃** — **클라이언트에서 토큰 삭제만 수행**. 대응 API가 없다 (API_SPEC 3.6). 삭제 후 로그인 페이지로 이동하고 React Query 캐시를 비운다
- [x] `components/common/ThemeToggle.tsx` (`UI-01`) — **로그인/회원가입 등 비인증 페이지에도 노출** (PRD 6.7)
- [x] `app/(main)/layout.tsx`에 헤더 배치

**테스트 체크리스트 (Playwright)**
- [x] 로그인 후 헤더에 **가입한 이메일이 표시**됨
- [x] 로그아웃 클릭 → 로그인 페이지 이동 → 뒤로가기해도 보호 페이지에 들어가지 못함
- [x] 테마 토글이 **로그인 페이지(비인증)와 Todo 목록(인증) 양쪽에서** 동작
- [x] 테마 선택이 새로고침 후에도 유지됨

**DoD**
- [ ] 회원가입 → 로그인 → 보호 페이지 진입 흐름 성공
- [ ] 소셜 로그인 버튼이 `/oauth2/authorization/{provider}`로 내비게이션하고, **콜백 페이지가 프래그먼트 토큰을 저장한다** (합성 토큰으로 검증)
- [ ] **콜백 처리 후 주소창에 토큰이 남아 있지 않음**
- [ ] 폼 유효성/에러 메시지 표시 — 검증 실패(`COMMON_001`)의 `data` 맵이 필드 에러로 매핑됨
- [x] **공통 헤더에 사용자 이메일·이름이 표시되고(`AUTH-04`), 로그아웃이 동작한다(`AUTH-05`)**
- [x] 테마 토글이 인증/비인증 페이지 모두에서 동작
- [ ] **브라우저 콘솔에 CORS 오류가 없다** (M2 Task 012의 `CorsConfig` 검증)
- [ ] Playwright 인증 시나리오 전부 통과
- [ ] 🔗 **(M3 완료 후 확인)** 실제 Google/Kakao 로그인 → 콜백 → 로그인 상태 유지 — **M8 Task 033의 수동 체크리스트로 이관**

> **의존성 모순 해소**: 기존 DoD의 "소셜 로그인 버튼 → 콜백 토큰 저장 → 로그인 상태 유지"는 M3 없이 충족할 수 없었다. **계약 기반으로 검증 가능한 부분(프래그먼트 파싱·저장·URL 정리)은 M6 DoD에 남기고, 실제 제공자 왕복은 M8 수동 검증으로 옮긴다.** 따라서 M6의 필수 선행은 `M2, M5`로 유지된다.

**의존성**: M2, M5 (구현·자동 검증) · M3 (실제 소셜 로그인 동작 확인 — M8에서 수행)

---

## M7. Todo 화면 ✅

**목표**: Tiptap 기반 작성/편집과 목록(페이지네이션·필터) UI를 완성한다.

### Task 028: Tiptap 설치 및 에디터 래퍼 구현

**영역**: FE | **선행**: Task 020

- [ ] **Tiptap 설치** (`@tiptap/react`, `@tiptap/starter-kit` 등) — **React 19 호환 버전을 공식 문서에서 먼저 확인**한다 (CLAUDE.md 5장)
- [ ] 설치 후 **PRD 1.3 표의 설치 상태를 ✅와 실제 버전으로 갱신**
- [ ] `components/editor/TiptapEditor.tsx` — 래퍼 컴포넌트 (`"use client"`)
  - **값 형식은 Tiptap JSON 문서 객체**다. 서버는 이를 불투명 데이터로 저장하므로 프론트가 파싱 없이 그대로 주고받는다 (PRD 8.4 / API_SPEC 4.1)
  - `value` / `onChange` 인터페이스로 RHF와 연결
- [ ] Calm Minimal 컨셉에 맞는 최소 툴바 (lucide-react 아이콘)
- [ ] SSR 주의 — 에디터는 클라이언트에서만 마운트한다

### Task 029: Todo 목록 화면 구현 (필터·페이지네이션)

**영역**: FE | **선행**: Task 022, Task 027

- [ ] `app/(main)/todos/page.tsx` — 상단 필터 + 카드 리스트 + 하단 페이지네이션 (PRD 9.2)
- [ ] `hooks/useTodos.ts` — React Query 훅. 쿼리 키에 `page`/`size`/`status`/`keyword` 포함
- [ ] `components/todo/TodoFilter.tsx` — 상태 필터(전체/TODO/DONE) + 제목 검색(디바운스)
- [ ] `components/todo/TodoList.tsx`, `TodoItem.tsx` — 제목·상태·마감일 표시, 클릭 시 상세로 이동
- [ ] **`components/common/Pagination.tsx` 재사용** (새로 만들지 않는다 — 불변 규칙 7)
- [ ] 필터/페이지 상태를 URL 쿼리와 동기화 — ⚠️ `useSearchParams()`는 **`<Suspense>` 래핑 필수** (Next 16)
- [ ] **로딩 / 빈 상태 / 에러 UI** — 빈 결과는 `content: []`이지 404가 아니다 (API_SPEC 4.2)
- [ ] "새 Todo" 버튼 → 작성 페이지

**테스트 체크리스트 (Playwright)**
- [ ] Todo 12건 생성 후 목록에 **10건만 표시**되고 2페이지가 존재
- [ ] 2페이지 이동 시 나머지 2건 표시, `aria-current`가 현재 페이지에 부여됨
- [ ] 상태 필터 `DONE` 선택 시 완료 항목만 표시
- [ ] 검색어 입력 시 매칭 항목만 표시, **대소문자를 구분하지 않음**
- [ ] Todo가 없는 계정에서 **빈 상태 UI**가 렌더됨 (에러 아님)

### Task 030: Todo 작성/상세·편집 화면 구현

**영역**: FE | **선행**: Task 028, Task 029

- [ ] `app/(main)/todos/new/page.tsx` — 제목 인풋(필수, 255자) + `TiptapEditor` + 마감일 피커 + 저장/취소
- [ ] `app/(main)/todos/[id]/page.tsx` — 기존 값 로드 후 편집 폼
  - ⚠️ Next 16에서 **`params`는 Promise**다 (Async Request APIs 파괴적 변경)
- [ ] `components/todo/TodoForm.tsx` — 작성/편집 공용, RHF + Zod
- [ ] **`PUT` 요청 시 `title`·`status`를 항상 포함**한다 — 생략하면 서버가 400을 반환한다 (API_SPEC 4.5)
- [ ] **404 처리** — 없거나 타인 소유(`TODO_001`)면 Todo 목록으로 이동 + 안내 메시지 (PRD 3장 리다이렉트 규칙)
- [ ] 저장/취소 후 목록으로 이동, React Query 캐시 무효화

**테스트 체크리스트 (Playwright)**
- [ ] 제목 + Tiptap 본문(굵게/목록 포함) 작성 → 저장 → 목록에 표시
- [ ] 상세 진입 시 **저장한 본문 서식이 그대로 복원**됨
- [ ] 수정 후 저장 → 목록에 변경 내용 반영
- [ ] 제목 미입력 시 저장 차단
- [ ] 존재하지 않는 `/todos/999999` 접속 → 목록으로 이동 + 안내 메시지

### Task 031: 상태 토글·삭제 및 Framer Motion 인터랙션

**영역**: FE | **선행**: Task 029

- [ ] 체크박스 상태 토글 — **UI는 토글이지만 요청은 목표 상태를 명시**한다: `PATCH /api/todos/{id}/status` 바디 `{"status": <현재값의 반대>}` (불변 규칙 13)
  - 응답의 전체 `TodoResponse`로 캐시를 정합시킨다 (`updatedAt` 포함)
- [ ] 삭제 버튼 — 확인 후 `DELETE` (Soft Delete), 목록 갱신
- [ ] **Framer Motion 마이크로 인터랙션** (PRD 9.1 — 절제된 모션)
  - 리스트 진입 fade/slide, 체크 토글 스프링 애니메이션, 페이지 전환
  - ⚠️ 애니메이션 라이브러리는 Framer Motion **하나만** 사용한다 (CLAUDE.md 4장)
- [ ] `prefers-reduced-motion` 존중

**테스트 체크리스트 (Playwright)**
- [ ] 체크박스 토글 → 상태 변경 후 목록 갱신 (페이지 유지)
- [ ] **같은 항목을 빠르게 두 번 클릭해도 최종 상태가 의도대로** (멱등 요청 확인)
- [ ] 삭제 → 목록에서 사라지고 **새로고침 후에도 사라진 상태 유지**
- [ ] 삭제된 항목이 `totalElements` 감소에 반영됨

**DoD**
- [ ] 목록 페이지네이션·필터 동작 (`Pagination` 재사용 컴포넌트 사용)
- [ ] Tiptap으로 본문 작성/수정 및 저장/불러오기 — **서식이 손실 없이 왕복**
- [ ] 상태 토글·삭제(Soft Delete) 반영, 상태 변경 요청이 **멱등 방식**
- [ ] 로딩/빈 상태/에러 UI 처리
- [ ] **PRD 1.3의 Tiptap 설치 상태 표 갱신 완료**
- [ ] Playwright Todo 시나리오 전부 통과
- [ ] `npm run lint` + `npm run build` 통과

**의존성**: M4, M5

---

## M8. 통합 & QA 🔍

**목표**: 전체 시나리오를 자동/수동으로 점검하고 문서를 정리한다.

### Task 032: 전체 사용자 플로우 E2E 자동화 (Playwright)

**영역**: 공통 | **선행**: M6, M7

- [x] `e2e/full-flow.spec.ts` — **가입 → 로그인 → Todo 생성 → 목록 확인 → 수정 → 상태 토글 → 삭제 → 로그아웃** 단일 시나리오
- [x] 계정 격리 시나리오 — **A 계정이 만든 Todo를 B 계정에서 볼 수 없고, ID 직접 접근 시 목록으로 리다이렉트**된다 (소유권 검증의 프론트 관점 확인) — `e2e/account-isolation.spec.ts`
- [x] 세션 만료 시나리오 — 만료 토큰으로 API 호출 → 토큰 삭제 → 로그인 페이지 — `e2e/session-expiry.spec.ts`
- [x] 실행 절차 정리: 백엔드 `dev` 기동(`./mvnw spring-boot:run -Dspring-boot.run.profiles=dev`, `DB_PASSWORD`·`JWT_SECRET` 등 환경변수 필요) → `todo-frontend`에서 `npm run test:e2e`. 위 3개 스펙은 기존 스펙과 달리 mock을 쓰지 않고 실 백엔드와 통신한다

**테스트 체크리스트 (Playwright)**
- [x] 전체 플로우 시나리오 그린 — 통과 (버그 수정 후 재검증, 아래 참조)
- [x] 계정 격리 시나리오 그린 — 통과 (일시적 dev 모드 컴파일 지연으로 최초 1회 타임아웃 후 재실행 시 통과, 앱 결함 아님)
- [x] 세션 만료 시나리오 그린 — 통과

**검증 결과 요약 (2026-08-24)**

백엔드를 `dev` 프로파일로 기동(`todolistdb` 스키마 연결 확인)하고 프론트 dev 서버를 별도로 미리 기동한 뒤(느린 파일시스템으로 인한 Playwright `webServer` 60초 타임아웃 회피) 3개 스펙을 실행했다. 최초 실행에서 3개 모두 실패했으나, `account-isolation`·`session-expiry`는 dev 모드 첫 컴파일 지연/3-worker 동시 실행 경합으로 인한 일시적 타임아웃임을 단일 worker 재실행으로 확인(재실행 시 통과, 앱·테스트 결함 아님). `full-flow.spec.ts`는 재현 가능한 실제 테스트 코드 결함으로 판명되어 수정했다(아래 참조). 수정 후 3개 스펙을 순차 재실행해 **3 passed**(25.8초)를 확인했고 `npm run lint`도 통과했다.

**중간에 발견하고 수정한 버그 — `full-flow.spec.ts`의 경쟁 상태(race condition)로 인한 테스트 실패**

- **증상**: Todo 제목을 수정하는 단계에서 `저장` 버튼을 찾지 못해 타임아웃. 재현율 100%(동일 조건 재실행 시 항상 실패).
- **원인**: 목록에서 Todo 제목 링크를 클릭한 직후, `/todos/{id}` 상세 페이지로의 네비게이션 완료를 기다리지 않고 바로 `page.getByLabel("제목")`으로 새 제목을 입력했다. 이 로케이터가 `exact` 옵션 없이 부분일치라 목록 페이지의 `aria-label="제목 검색"` 필터 인풋(`TodoFilter.tsx`)과도 매칭되어, 네비게이션이 완료되기 전에 실행되면 상세 페이지 대신 목록의 필터박스를 채워버렸다. 그 결과 앱이 `/todos?keyword=...`로 되돌아가 있었고, 이후 존재하지 않는 상세 페이지의 저장 버튼을 기다리다 타임아웃했다. trace.zip의 네트워크 로그(`/todos/11` 요청 직후 `/todos?keyword=...` 요청이 바로 이어짐)로 원인을 특정했다. `account-isolation.spec.ts`는 이미 `waitForURL`을 쓰고 있어 이 문제가 없었다.
- **수정**: `todo-frontend/e2e/full-flow.spec.ts` — 제목 링크 클릭 직후 `await page.waitForURL(/\/todos\/\d+$/)`을 추가하고, `getByLabel("제목")`에 `{ exact: true }`를 추가해 필터박스와의 오매칭을 방지했다.
- **테스트**: 수정 후 `full-flow.spec.ts` 단독 재실행 통과, 3개 스펙 순차 재실행 모두 통과, `npm run lint` 통과. 동일 패턴(`getByLabel("제목")` 부분일치)을 쓰는 `account-isolation.spec.ts`·`todo-form.spec.ts`는 필터박스가 없는 `/new`·`/edit` 페이지에서만 쓰여 안전함을 확인해 손대지 않았다.

### Task 033: 소셜 로그인 수동 검증 체크리스트 수행

**영역**: 공통 | **선행**: M3, M6

**외부 제공자 로그인은 Playwright로 자동화하지 않는다.** 실제 계정·2FA·동의 화면·봇 차단이 개입해 테스트가 불안정해지고, 자격증명을 CI에 넣게 되어 불변 규칙 9와 충돌한다. **수동 체크리스트로 분리한다.**

- [x] Google 로그인 → 신규 자동 가입 → Todo 목록 진입 — 통과 (버그 수정 후 재검증, 아래 참조)
- [x] 같은 계정으로 재로그인 → 중복 가입되지 않음 — 통과 (버그 수정 후 재검증, 아래 참조)
- [ ] Kakao 로그인 → 중첩 응답 파싱 확인 → Todo 목록 진입 — 보류 (Kakao 클라이언트 자격증명 미준비로 이번 회차는 Google만 검증)
- [ ] **Kakao 이메일 미제공 케이스** — 콘솔의 동의 항목 설정 확인 및 정책대로 동작 — 보류 (위와 동일 사유)
- [x] 기존 LOCAL 계정과 같은 이메일로 소셜 로그인 → 기존 계정으로 로그인되고 **`provider`가 `LOCAL`로 유지** — 통과 (`rlaeogus0911@gmail.com`, LOCAL 가입 후 Google 연동 → `/todos` 진입, DB `provider` 컬럼 `LOCAL` 유지 확인)
- [x] 위 연동 후 **기존 비밀번호 로그인이 계속 동작** — 통과 (로그아웃 후 이메일+비밀번호 로그인 정상, 토큰 발급 확인)
- [x] 콜백 후 **주소창에 토큰이 남지 않음** — 통과 (`/oauth2/callback#token=...` → `/todos`로 정리됨, 주소창에 토큰 노출 없음)
- [x] `authorization_request_not_found`가 발생하지 않음 — 통과 (전 과정에서 쿠키 기반 AuthorizationRequestRepository 정상 동작, 관련 에러 없음)
- [ ] 실패 경로(동의 거부) → 로그인 페이지 + 에러 안내 — 보류 (테스트 계정이 이미 앱에 동의를 완료한 상태라 Google이 동의 화면을 건너뜀(`prompt=none`). 재검증하려면 myaccount.google.com/permissions에서 앱 액세스를 해제한 뒤 재시도 필요)
- [x] 검증 결과를 이 문서 또는 커밋 메시지에 기록 — 아래 참조

**검증 결과 요약 (2026-08-24, Google만 검증, Kakao·항목9는 후속 확인 필요)**

- 통과: 1(Google 신규 자동 가입), 2(재로그인 시 중복 가입 안 됨), 5(provider LOCAL 유지), 6(비밀번호 로그인 유지), 7(토큰 미노출), 8(authorization_request_not_found 없음)
- 보류: 3·4(Kakao, 자격증명 미준비), 9(동의 거부, 테스트 계정이 이미 동의 완료 상태 — myaccount.google.com/permissions에서 앱 액세스 해제 후 재검증 필요)

**중간에 발견하고 수정한 버그 — Google 로그인(OIDC) 시 신규 가입이 항상 실패하던 문제**

- **증상**: 처음 보는 이메일의 Google 계정으로 로그인하면 `OAuth2SuccessHandler.java:44`에서 `CustomException: 사용자를 찾을 수 없습니다`(`AUTH_006`)가 발생하며 500 에러. 이미 DB에 있는 이메일(LOCAL 계정에 소셜 연동 등)은 문제없이 동작해서 처음엔 원인이 드러나지 않았음.
- **원인**: Google 로그인은 `scope`에 `openid`가 포함되어 있어 Spring Security가 OIDC 로그인 경로(`OidcAuthorizationCodeAuthenticationProvider`)로 처리하는데, 기존 `SecurityConfig.java`는 `.userInfoEndpoint(u -> u.userService(customOAuth2UserService))`만 등록하고 `.oidcUserService(...)`는 등록하지 않았다. OIDC 경로에서는 Spring 기본 `OidcUserService`가 대신 쓰여 `CustomOAuth2UserService`의 신규 사용자 생성 로직이 전혀 호출되지 않았다.
- **수정**: `todo-backend/src/main/java/com/example/auth/oauth2/OAuth2UserProvisioningService.java`(신규) — 기존 `findOrCreateUser` 로직을 공용 서비스로 분리. `CustomOAuth2UserService`는 이 서비스에 위임하도록 수정(Kakao 등 순수 OAuth2 흐름 담당, 동작 변경 없음). `CustomOidcUserService.java`(신규) — `OidcUserService`를 상속해 OIDC 경로(Google)에서도 동일한 `findOrCreateUser`가 실행되도록 구현. `SecurityConfig.java` — `.oidcUserService(customOidcUserService)`를 추가 등록.
- **테스트**: 기존 `CustomOAuth2UserServiceTest`의 로직 검증 케이스는 `OAuth2UserProvisioningServiceTest`(신규)로 이관. 백엔드 전체 테스트 71건 통과(`./mvnw test`). 브라우저로 신규 Google 계정 가입(항목 1) 및 재로그인 시 중복 미생성(항목 2) 재현 검증 완료.

### Task 034: 예외·로딩·빈 상태·반응형 최종 점검

**영역**: 공통 | **선행**: Task 032

- [x] 주요 엣지 케이스 — 만료 토큰, 빈 목록, 검증 실패, 네트워크 오류, 백엔드 다운 — 통과 (아래 참조)
- [x] 모든 API 실패 응답이 `ApiResponse` 포맷인지 확인 — **특히 필터 단계 401** (M2 Task 012) — 통과 (아래 참조)
- [x] 로딩 스켈레톤/스피너, 빈 상태, 에러 UI가 모든 화면에 존재 — 통과, 결함 1건 발견 후 수정 (아래 참조)
- [ ] 반응형 확인 (모바일/태블릿/데스크톱) — 보류 (아래 참조)
- [x] 다크모드에서 대비·가독성 점검 (액센트 `Indigo` 포함) — 통과 (아래 참조)
- [ ] 접근성 스팟체크 — 페이지네이션 `aria`, 폼 라벨, 키보드 내비게이션 — 부분 보류 (아래 참조)

**검증 결과 요약 (2026-08-24)**

정적 조사 결과 `TodoList.tsx`·`TodoForm.tsx`·`TodosView.tsx`·`Pagination.tsx`·`JwtAuthenticationEntryPoint`/`GlobalExceptionHandler`가 이미 로딩·에러·빈상태·접근성·공통 응답 포맷을 견고하게 구현하고 있음을 확인했다. 미확인 컴포넌트 4종(`TodoItem.tsx`, `todos/[id]/page.tsx`, `Header.tsx`, `SocialLoginButtons.tsx`)을 추가로 읽어 조사를 마쳤다.

- **엣지 케이스·네트워크 오류**: 백엔드 프로세스를 강제 종료한 뒤 로그인을 시도해, 프론트가 크래시 없이 "요청 처리 중 오류가 발생했습니다"를 `role="alert"`로 표시함을 확인(`LoginForm.tsx`의 `ApiError` 폴백 처리). 이후 백엔드를 재기동해 정상 복구를 확인했다. 검증 실패(400)는 회원가입 폼에 잘못된 이메일·짧은 비밀번호를 입력해 "올바른 이메일 형식이 아닙니다"·"비밀번호는 6자 이상이어야 합니다"가 정확히 표시됨을 확인. 만료 토큰 엣지 케이스는 `e2e/session-expiry.spec.ts`(Task 032) 통과로 자동 커버된다.
- **401 응답 포맷**: `curl`로 실제 `JWT_SECRET`을 이용해 만료 토큰(HS256, `exp` 과거)과 다른 시크릿으로 서명한 위조 토큰을 직접 생성해 재현했다. 토큰 없음/형식 오류 → `AUTH_003`, 만료 → `AUTH_004`, 서명 불일치 → `AUTH_005`가 모두 `{success:false,data:null,message,errorCode}` `ApiResponse` 포맷으로 정확히 응답됨을 확인했다. `JwtAuthenticationEntryPoint`가 `GlobalExceptionHandler`와 별도로 필터 단계 401을 이미 `ApiResponse`로 직렬화하고 있어, 우려했던 "필터 단계 401이 포맷을 벗어날 수 있다"는 리스크는 실측으로 해소됐다.
- **로딩/에러/빈상태 결함 1건 발견 및 수정**: `todos/[id]/page.tsx`의 `deleteMutation`에 `TodoItem.tsx`와 달리 삭제 실패 시 에러 표시가 없었다. `TodoItem.tsx`의 기존 패턴(`ApiError` 분기 + `role="alert"`)을 그대로 재사용해 `deleteError` 변수와 에러 문단을 추가했다. `window.fetch`를 몽키패치해 DELETE 요청만 실패하도록 재현한 뒤 "삭제에 실패했습니다." 에러가 다이얼로그에 정상 표시됨을 확인했다. `npm run lint`·`npm run build` 모두 통과. `Header.tsx`(사용자 정보 로딩/실패 UI 없음, 아바타 이니셜 하나뿐이라 저위험)와 `SocialLoginButtons.tsx`(즉시 리다이렉트 구조라 해당없음, 실패는 `oauth2/callback#error=` 페이지가 처리)는 결함으로 보지 않았다.
- **반응형 확인 — 보류**: `resize_window` 브라우저 자동화 도구가 이 세션에서 실제 캡처 뷰포트에 반영되지 않는 한계를 새 탭 포함 2회 재현 확인했다(390×844 요청 후에도 항상 1107×538로 캡처, `window.innerWidth`로도 미반영 확인). 앱 결함이 아니라 세션 도구 제약이다. 대신 코드 검토로 대체 확인: `TodoFilter.tsx`가 `flex-col sm:flex-row` 모바일 우선 패턴을 쓰고, 주요 컨테이너(`TodosView`, `Header` 등)가 `max-w-5xl mx-auto px-4` 유동 레이아웃 + 기본 `flex-col` 구조라 별도 브레이크포인트 없이도 좁은 화면에서 자연스럽게 축소되는 구조임을 확인했다. **실제 뷰포트에서의 육안 검증은 못했으므로 후속 세션에서 재시도가 필요하다.**
- **다크모드**: `ThemeToggle` 드롭다운으로 다크 테마로 전환한 뒤 로그인·회원가입·Todo 목록(헤더·네비·필터·로딩 스피너·빈 상태·폼·Tiptap 에디터)을 순회하며 대비와 인디고 액센트·포커스 링 가독성을 확인, 전반적으로 양호했다. 전환 도중 테마 아이콘(Sun/Moon)의 SSR/CSR 하이드레이션 불일치 경고가 콘솔에 1회 포착됐으나 Fast Refresh 리빌드 직후 발생해 dev 모드 HMR 잔여효과일 가능성이 높고 재현성을 확정하지 못해 별도 기록만 남긴다(추가 조사 필요, 코드 수정은 하지 않음).
- **접근성 — 부분 보류**: `Pagination.tsx`·`LoginForm.tsx`가 네이티브 `button`/`input`/`label htmlFor`와 `aria-label`을 사용하는 구조임을 코드로 재확인했다(정적 근거로는 통과). 다만 이 브라우저 자동화 세션에서 synthetic 클릭/Tab 키 입력이 `document.activeElement`를 안정적으로 이동시키지 못하는 도구 한계를 발견해(클릭 후에도 `activeElement`가 `BODY`로 남는 현상 재현) 실제 Tab 키만으로의 라이브 내비게이션 검증은 완결하지 못했다. **후속 세션에서 재시도가 필요하다.**

### Task 035: README 및 문서 정합성 마무리

**영역**: 공통 | **선행**: Task 034

- [x] **README 완성** (한국어) — 신규 환경에서 **처음부터 재현 가능**하도록: 사전 요구사항 → 스키마 생성 → 환경변수 → 백엔드 실행 → 프론트 실행 → E2E 실행
- [x] **PRD 1.3 설치 상태 표 최종 확인** — 모든 ⏳가 ✅와 실제 버전으로 바뀌었는지
- [x] **`docs/guides/` 스택 정보 갱신** (PRD_VALIDATION Major #10) — 실측 결과 기존 3개 파일은 이미 정확했고, `forms-react-hook-form.md`의 stale 문구(react-hook-form/zod 미설치 표기)를 추가로 발견해 수정했다
  - `nextjs-16.md`: **16.3.1** — 이미 정확함(확인만)
  - `styling-guide.md`: 실제 **`radix-nova`**, `next-themes`·`prettier-plugin-tailwindcss` 미설치 상태 — 이미 정확함(확인만)
  - `component-patterns.md`: **16.3.1** — 이미 정확함(확인만)
- [x] 위험 추적(Critical/Major Issues) 상태 갱신 — 해소된 항목 표시 (Critical 4건 전부 해소, Major 11건 중 9건 해소·2건 부분 해소로 표시. "부록 C"라는 명칭 섹션은 존재하지 않아 실제 위험 추적 섹션인 Critical/Major Issues에 표시함)
- [x] `develop` → `main` 머지 및 태그 — 실측 결과 `develop`에 `main`에 없는 커밋이 없어(오히려 `develop`이 뒤처짐) 병합은 불필요했다. 사용자 승인 하에 backend·frontend 양쪽 `origin/main`에 `v0.8.0` 태그를 부여했다

**DoD**
- [x] 핵심 사용자 흐름 무결점 통과 (**Playwright 자동 시나리오 전부 그린**) — Task 032에서 3개 스펙 전부 통과 확인
- [ ] **소셜 로그인 수동 체크리스트 전 항목 통과** — Task 033에서 Google 관련 6개 항목은 통과했으나 Kakao 2개 항목(자격증명 미준비)과 동의 거부 실패 경로 1개 항목이 보류 상태로 남아 있어 미완료
- [x] 주요 엣지 케이스(만료 토큰, 빈 목록, 검증 실패) 처리 — Task 034에서 통과 확인
- [x] 모든 실패 응답이 `ApiResponse` 포맷 유지 — Task 034에서 `curl` 실측으로 AUTH_003/004/005 전부 확인
- [x] README로 신규 환경에서 재현 가능
- [x] PRD 1.3 표와 `docs/guides/`가 실제 스택과 일치

**의존성**: M3~M7

> ⚠️ **미완료 항목**: 소셜 로그인 수동 체크리스트의 Kakao 검증(자격증명 준비 필요)과 동의 거부 실패 경로 재검증(Google 계정 앱 액세스 해제 후 재시도 필요)이 남아 있다. 이 항목들은 외부 자격증명·수동 조작이 필요해 이번 세션(Task 035)에서 처리하지 못했다.

---

## M9. AWS 배포 🚀

**목표**: PRD의 배포 아키텍처로 운영 환경에 배포한다.

### Task 036: RDS(PostgreSQL) 프로비저닝 및 운영 스키마 준비

**영역**: 인프라 | **선행**: M8

- [ ] RDS PostgreSQL 인스턴스 생성, 보안그룹(EC2에서만 접근)
- [ ] **따옴표 없이** `CREATE SCHEMA IF NOT EXISTS TodoListDB;` → `\dn`으로 `todolistdb` 확인 (PRD 8.1)
- [ ] ⚠️ **`prod` 프로파일은 `ddl-auto=validate`** 다. 스키마·테이블이 미리 존재하지 않으면 기동에 실패한다
  - 초기 1회는 `dev` 설정으로 스키마를 생성하거나, M1에서 생성된 DDL을 SQL로 추출해 적용한다
- [ ] 백업/스냅샷 정책 확인

### Task 037: 백엔드 EC2 배포 및 환경변수 주입

**영역**: 인프라 | **선행**: Task 036

- [ ] EC2 인스턴스(JDK 21) 준비, `./mvnw clean package` 산출물 배포
- [ ] `--spring.profiles.active=prod` 로 기동
- [ ] **시스템 환경변수 주입** — `DB_URL`, `DB_USERNAME`, `DB_PASSWORD`, `JWT_SECRET`, `OAUTH_GOOGLE_*`, `OAUTH_KAKAO_*`, `APP_FRONTEND_URL` (PRD 12.1)
  - ⚠️ `prod`는 기본값을 두지 않는다. 환경변수가 없으면 **기동에 실패**하도록 설계되어 있다 (의도된 동작)
  - **어떤 값도 Git에 커밋하지 않는다** (불변 규칙 9)
- [ ] 보안그룹/포트, 프로세스 관리(systemd 등), 로그 확인

### Task 038: 프론트엔드 Amplify 배포

**영역**: 인프라 | **선행**: Task 037

- [ ] AWS Amplify 연결(저장소 `main` 브랜치), 빌드 설정
- [ ] 환경변수 `NEXT_PUBLIC_API_BASE_URL` = 운영 백엔드 주소
- [ ] HTTPS 인증서 및 커스텀 도메인 확인

> 📌 **S3는 MVP에서 사용하지 않는다** (PRD 1.3 · 13.1). 첨부파일이 범위 밖이고 Next.js 정적 자산은 Amplify가 자체 처리하므로 담을 것이 없다. 첨부파일을 도입하는 시점에 다시 검토한다.

### Task 039: 운영 도메인 기준 CORS·OAuth2 Redirect URI 갱신 및 보안 점검

**영역**: 인프라 | **선행**: Task 038

**배포에서 가장 자주 실패하는 지점이다.** M2에서 만든 CORS와 M3의 Redirect URI가 모두 로컬 주소로 고정되어 있다.

- [ ] **`CorsConfig`의 허용 오리진을 운영 프론트 도메인으로 갱신** — `${APP_FRONTEND_URL}` 환경변수로 주입되게 하고 하드코딩하지 않는다
- [ ] **Google/Kakao 개발자 콘솔의 Redirect URI를 운영 백엔드 주소로 추가**: `https://{운영BE}/login/oauth2/code/{provider}`
- [ ] `APP_FRONTEND_URL`이 운영 프론트 도메인인지 확인 — OAuth2 성공 리다이렉트가 여기로 간다
- [ ] **HTTPS 전 구간 확인** — HTTP 콜백은 제공자 정책상 거부될 수 있다
- [ ] 쿠키 기반 인가 요청 저장소를 쓴다면 `Secure` 속성 확인
- [ ] 최종 보안 점검 — 저장소에 시크릿 없음, 운영 로그에 토큰·비밀번호 미노출

**DoD**
- [ ] 프론트/백엔드/DB가 운영 환경에서 연동 동작
- [ ] **운영 도메인에서 브라우저 CORS 오류 없음**
- [ ] 소셜 로그인 리다이렉트가 운영 도메인에서 정상 동작
- [ ] HTTPS 및 환경변수 보안 점검 완료
- [ ] 운영 환경에서 핵심 플로우(가입→로그인→Todo CRUD) 수동 확인

**의존성**: M8

---

## 부록 A. 기능 ID ↔ 마일스톤 매핑

PRD 4장의 모든 기능 ID가 어느 마일스톤에서 구현되는지 정리한다. **누락된 기능이 없는지 확인하는 용도**다.

> PRD 11장은 개발 순서를 이 문서에 위임하므로, PRD에는 별도의 Phase 체계가 존재하지 않는다.

| 기능 ID | 기능 | 백엔드 | 프론트엔드 |
|---------|------|--------|-----------|
| `AUTH-01` | 회원가입 | M2 (Task 013) | M6 (Task 024) |
| `AUTH-02` | 로그인 | M2 (Task 013) | M6 (Task 024) |
| `AUTH-03` | 소셜 로그인 | M3 (Task 014·015) | M6 (Task 025) |
| `AUTH-04` | 내 정보 조회 | M2 (Task 013) | **M6 (Task 027 공통 헤더)** |
| `AUTH-05` | 로그아웃 | — (API 없음) | **M6 (Task 027 공통 헤더)** |
| `TODO-01` | 목록 조회 (페이지네이션) | M4 (Task 017) | M7 (Task 029) |
| `TODO-02` | 상세 조회 | M4 (Task 016) | M7 (Task 030) |
| `TODO-03` | 생성 | M4 (Task 016) | M7 (Task 030) |
| `TODO-04` | 수정 | M4 (Task 016) | M7 (Task 030) |
| `TODO-05` | 상태 변경 (멱등) | M4 (Task 018) | M7 (Task 031) |
| `TODO-06` | 삭제 (Soft Delete) | M4 (Task 018) | M7 (Task 031) |
| `UI-01` | 다크모드 토글 | — | M5(Task 020 프로바이더) + **M6(Task 027 헤더 배치)** |
| `UI-02` | 페이지네이션 컴포넌트 | — | M5 (Task 022) |

**기능 ID가 없는 필수 구현 요소** (PRD 2.1/10장에 있으나 4장 기능표에는 없는 항목)

| 요소 | 근거 | 마일스톤 |
|------|------|----------|
| `JpaAuditingConfig` | PRD 2.1 | M1 (Task 008) |
| `JwtAuthenticationEntryPoint` | PRD 10장 / API_SPEC 2.2 | M2 (Task 012) |
| **`CorsConfig`** | PRD 2.1 / 10장 | **M2 (Task 012)** · 운영 갱신 M9 (Task 039) |
| OAuth2 인가 요청 저장소 | PRD 10장 | M3 (Task 015) |
| 루트 진입 라우팅 (`app/page.tsx`) | PRD 3장 / 2.2 | M6 (Task 026) |
| 백엔드 테스트 인프라 | 0.5 테스트 전략 | M1 (Task 007) |
| Playwright E2E 환경 | 0.5 테스트 전략 | M5 (Task 023) |

## 부록 B. 권장 커밋 태그

`m0-init` · `m1-domain` · `m2-auth` · `m3-oauth2` · `m4-todo-api` · `m5-fe-base` · `m6-fe-auth` · `m7-fe-todo` · `m8-qa` · `m9-deploy`

## 부록 C. 미해결 위험 추적 (PRD_VALIDATION 연동)

[PRD_VALIDATION.md](./PRD_VALIDATION.md)에서 제기된 항목 중 **아직 코드로 해소되지 않은 위험**과, 그것을 다루는 마일스톤을 추적한다. 해소되면 상태를 갱신한다.

| # | 위험 | 확신도 | 추적 마일스톤 | 상태 |
|---|------|--------|--------------|------|
| Critical #2 | Jackson 3 × Hibernate 7 JSON FormatMapper 자동 구성 실패 | [FACT] 이슈 실재 / [UNCERTAIN] Boot 4.1 해결 여부 | **M1 Task 009** | ⏳ 실측 대기 |
| Major #1 | PostgreSQL 소문자 폴딩 — 스키마 물리명 | [FACT] | **M0 Task 005** (`\dn` 확인) | ⏳ 미확인 |
| Major #6 | `STATELESS` ↔ OAuth2 인가 요청 저장소 충돌 | [INFERENCE, 높음] | **M3 Task 015** | ⏳ |
| Major #8 | `@SQLRestriction`의 `findById`/count 적용 범위 | [UNCERTAIN] | **M1 Task 009** (3경로 테스트) | ⏳ 실측 대기 |
| Major #11 | `keyword` 검색이 인덱스로 커버되지 않음 | [FACT] | **MVP 범위 외** — 데이터 증가 시 `pg_trgm`+GIN 검토 | 📌 보류(의도적) |
| Minor #1 | Next 16 `useSearchParams` Suspense | [FACT] | M7 Task 029 (목록 URL 쿼리) · Task 030 (async `params`) | ⏳ |
| Minor #3 | `content` 크기 상한 없음 | — | M4 Task 016 (선택) | 📌 |
| Minor #11 | `com.example.domain.todo` ↔ `com.example.todo` 패키지 혼동 | — | M4 Task 016 (주의 노트) | 📌 |
| Minor #17 | Spring Security 7 람다 DSL / `csrf.disable()` | [FACT] | **M2 Task 012** | ⏳ |
| 신규 | `todolistdb_test` 스키마 자동 생성 여부 미검증 | — | **M1 Task 007·009** | ⏳ 실측 대기 |
| 지속 검토 | Framer Motion 패키지명(`motion`) · Tiptap React 19 호환 | [UNCERTAIN] | M5 Task 019 · M7 Task 028 | ⏳ |

> **해소 완료 (문서 반영)**
> - Critical #1 평문 시크릿 → M0 Task 002 ✅
> - Critical #3 쿼리스트링 토큰 → 프래그먼트 ✅ · Critical #4 PRD 1.3 설치 상태 ✅
> - Major #2~#5, #7, #9 → PRD·API_SPEC·ROADMAP 반영 ✅
> - **Major #10 `docs/guides/` 스택 정보 → 이미 갱신 완료 ✅** (16.3.1 · `radix-nova` · eslint-config 정정)
> - **Minor #2 토큰 저장 매체 → `localStorage` 확정 ✅** (PRD 13.1)
> - **Minor #9 S3 범위 → MVP에서 제거 확정 ✅** (PRD 1.3 · 4.5)
> - **Minor #10 `@ManyToOne` EAGER → PRD 8.3에 `LAZY` 규칙 명문화 ✅** (구현 검증은 M1 Task 009)
> - **Minor #16 CORS 세부 조건 → PRD 10장에 표로 명시 ✅** (구현은 M2 Task 012)

## 부록 D. 타 문서 수정 제안 (이 로드맵에서는 반영하지 않음)

이 로드맵이 전제하지만 **PRD/기타 문서에는 아직 없는** 사항이다. 해당 문서 담당 작업에서 반영한다.

### ✅ 반영 완료 (2026-08-21)

| 대상 | 반영 내용 |
|------|-----------|
| PRD 1.3 | **테스트 도구 3행 신설** (테스트 BE / 테스트 DB / 테스트 FE) + Testcontainers·H2 제외 근거 |
| PRD 1.3 · 4.5 | **S3를 MVP에서 제거** (배포 행에서 삭제, 4.5 범위 제외 목록에 추가) |
| PRD 6.4 | 목록 상태를 **URL 쿼리로 관리** + `<Suspense>` 필수 명시 |
| PRD 8.1 | **테스트 스키마 `todolistdb_test`** 분리 정책 |
| PRD 8.3 | **연관관계 `LAZY` 명시 필수** 소절 신설 |
| PRD 10장 | **CORS 세부 조건 표** (`PATCH` 포함, `allowCredentials=false`) |
| PRD 13.1 | 확정 사항 6행 추가 — 토큰 `localStorage` · 목록 URL 쿼리 · 테스트 DB · `LAZY` · S3 제외 |
| API_SPEC 6장 | 토큰 저장 매체·URL 쿼리 단일 출처·401 루프 방지 |
| `docs/guides/` 3종 | 이미 갱신 완료 (16.3.1 · `radix-nova`) |
| `src/test/resources/application.properties` | 테스트 스키마 분리 적용 |

> **PRD 6.3(OAuth2 콜백)의 `<Suspense>` 제안은 반영하지 않았다.** 콜백 페이지는 쿼리스트링이 아니라 **프래그먼트**(`#token=`)를 `window.location.hash`로 읽으므로 `useSearchParams()`를 쓰지 않고, 따라서 Suspense 경계가 필요 없다. 해당 제약은 **PRD 6.4(Todo 목록)** 에 적용되어 그쪽에 반영했다.

### 남은 제안 (없음)

현재 미반영 제안이 없다. 새로 발견되는 타 문서 수정 사항은 이 절에 추가한다.

---

### 다음 단계

**M0의 남은 세 Task(003 JDK 21 전환 · 004 Git 초기화 · 005 스키마 실측)부터** 착수한다. 각 Task 완료 시 테스트를 실행해 통과를 확인하고, 마일스톤의 DoD를 모두 충족하면 커밋/태그 후 다음으로 이동한다. 상세 요구사항은 항상 [PRD.md](./PRD.md), API 계약은 [API_SPEC.md](./API_SPEC.md)를 기준으로 한다.
