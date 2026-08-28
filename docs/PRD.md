# PRD — Todo List 풀스택 프로젝트

> **문서 목적**: 이 문서는 **"무엇을 만들 것인가"** 를 정의하는 제품 요구사항 명세서(Product Requirements Document)입니다.
> **"어떤 순서로 만들 것인가"** 는 [ROADMAP.md](./ROADMAP.md), **API 요청/응답 상세**는 [API_SPEC.md](./API_SPEC.md)를 기준으로 합니다.

---

## 1. 프로젝트 개요

### 1.1 서비스 정의

**목적**: 흩어진 할 일을 서식 있는 문서 형태로 한곳에 기록·관리하는 개인용 Todo 웹앱.
**타겟 사용자**: 단순 체크리스트로는 부족해 메모·링크·목록이 함께 필요한 개인 사용자.

사용자가 회원가입 후 자신의 할 일(Todo)을 Tiptap 리치 텍스트로 작성/관리한다. 이메일 기반 로그인과 소셜 로그인(OAuth2)을 지원하며, 심플하고 모던한 UI를 제공한다.

### 1.2 핵심 가치

- **간결함**: 최소한의 클릭으로 할 일을 관리
- **모던함**: 최신 디자인 트렌드를 반영한 UI/UX
- **안전함**: JWT 기반 인증, Soft Delete로 데이터 보존
- **확장성**: 페이지네이션 기반 대량 데이터 처리

### 1.3 기술 스택

**버전이 명시된 항목은 `todo-backend/pom.xml` · `todo-frontend/package.json`에서 확인한 실제 설치값**이다.
버전이 없는 항목은 **아직 설치되지 않았으며** 표의 "설치 상태" 열이 가리키는 마일스톤에서 설치한다.

| 구분 | 기술 | 설치 상태 |
|------|------|-----------|
| **Backend** | Spring Boot **4.1.0**, JDK 21, Maven, Spring Data JPA / Hibernate, Spring Security, Lombok | ✅ 설치됨 |
| **JWT** | `io.jsonwebtoken:jjwt` **0.12.6** (api / impl / jackson) | ✅ 설치됨 |
| **Frontend** | Next.js **16.3.1** (App Router), React **19.2.8**, TypeScript 5, Tailwind CSS 4 | ✅ 설치됨 |
| **UI** | shadcn **CLI 4.18.0** (style `radix-nova`, baseColor `neutral`), radix-ui, lucide-react **1.33.0** | ✅ 설치됨 |
| **상태/모션** | React Query(TanStack Query) `@tanstack/react-query` **5.102.0**, `motion` **13.1.1**(Framer Motion 후신 패키지) | ✅ 설치됨 |
| **에디터** | Tiptap (리치 텍스트 웹에디터) `@tiptap/react` **3.30.2**, `@tiptap/starter-kit` **3.30.2** | ✅ 설치됨 |
| **폼 검증** | React Hook Form **7.86.0** + Zod **4.4.3** + `@hookform/resolvers` **5.9.1** (프론트) / Bean Validation (백엔드) | ✅ 프론트·백엔드 모두 설치됨 |
| **인증** | JWT (Access Token, 24시간 만료) + OAuth2 소셜 로그인 (Google, Kakao) | ✅ 완료 (M2·M3 구현·커밋 완료) |
| **Database** | PostgreSQL (스키마명: `TodoListDB`) | ✅ 완료 (M0에서 `todolistdb` 소문자 폴딩 실측 확인) |
| **테스트 (BE)** | JUnit 5 + Spring Boot Test (MockMvc) | ✅ 완료 (M1 Task 007에서 인프라 구축 완료) |
| **테스트 DB** | **로컬 PostgreSQL 테스트 스키마 `todolistdb_test`** | ✅ 완료 (M1 Task 007에서 구축 완료) |
| **테스트 (FE)** | **Playwright** (E2E 전용) | ✅ **설치됨** (Task 023) |
| **배포** | AWS Amplify(FE), EC2(BE), RDS(DB) | ⏳ M9 |
| **형상관리** | Git, GitHub | ✅ |

> ⚠️ **미설치 라이브러리를 `import` 하지 않는다.** 위 표에서 ⏳로 표시된 프론트 라이브러리는 `package.json`에 아직 없다.
> 해당 마일스톤에서 설치한 뒤 사용하고, 설치 시 이 표의 상태를 ✅와 실제 버전으로 갱신한다.
> `shadcn 4.18.0`은 컴포넌트 라이브러리가 아니라 **CLI 버전**이다.

> 📌 **테스트 DB — Testcontainers도 H2도 쓰지 않는다.**
> - **Testcontainers 제외**: Docker 실행 환경이 전제인데, 이 프로젝트는 **개발 머신에 Docker를 두지 않기로 확정**했다.
> - **H2 제외**: `content`가 `JSONB`이고 검증 대상이 PostgreSQL 고유 동작(JSONB 왕복 · 식별자 소문자 폴딩 · `@SQLRestriction` 적용 범위)이다. H2는 셋 중 무엇도 재현하지 못해, 통과해도 아무것도 보증하지 못한다.
> - **채택**: 로컬 PostgreSQL에 **개발 스키마와 분리된 `todolistdb_test`** 를 두고 테스트를 실행한다. 스키마 분리는 이미 `src/test/resources/application.properties`에 반영되어 있다.
> - CI를 도입하는 시점(M9 이후)에 Testcontainers로 승격하는 경로는 열어 둔다.

> 📌 **S3는 MVP에서 제외한다.** 첨부파일이 범위 밖(4.5)이고 Next.js 정적 자산은 Amplify가 자체 처리하므로, MVP 단계에서 S3가 담을 것이 없다.
> 이 프로젝트는 **텍스트 기반 Todo에 집중**한다. 첨부파일을 도입하는 시점에 S3를 다시 검토한다.

> ⚠️ **버전 확인 필수**: Spring Boot 4 / Next.js 16 / React 19 / Tailwind 4는 최신 메이저 버전이다. 익숙한 예전 방식으로 코드를 작성하기 전에 **해당 버전의 공식 문서를 먼저 확인**한다.
>
> - **Spring Boot 4**: Jakarta EE 11 기반. 웹 스타터가 `spring-boot-starter-webmvc`로, 테스트 스타터가 `spring-boot-starter-*-test`로 **모듈별 분리**되었다.
> - **Next.js 16**: `todo-frontend/AGENTS.md`의 지시대로 `node_modules/next/dist/docs/` 의 해당 가이드를 먼저 읽는다.
> - **Tailwind 4**: CSS-first 설정(`app/globals.css`). `tailwind.config.js`는 사용하지 않는다.

---

## 2. 프로젝트 구조

한 폴더(`todo-project/`) 안에 백엔드와 프론트엔드를 분리하여 관리한다.

> ⚠️ **디렉토리는 한곳에 모여 있지만, Git 저장소는 3개로 분리되어 있다.**
>
> | 저장소 | 담당 | GitHub |
> |--------|------|--------|
> | 루트 `todo-project/` | `docs/` · `CLAUDE.md` · `.claude/` | 문서 전용 저장소 |
> | `todo-backend/` | 백엔드 코드 | `kdongjin/todo-backend` |
> | `todo-frontend/` | 프론트 코드 | `kdongjin/todo-frontend` |
>
> 루트 `.gitignore`가 하위 두 폴더를 제외하므로 세 저장소는 서로 간섭하지 않는다.
> **커밋은 변경이 일어난 저장소에서** 한다. 상세는 [ROADMAP.md](./ROADMAP.md) Task 004 참조.

```
todo-project/
├── README.md
├── CLAUDE.md                # Claude Code 프로젝트 지침
├── .gitignore
├── docs/
│   ├── PRD.md               # 이 문서 — 무엇을 만들 것인가
│   ├── ROADMAP.md           # 마일스톤 M0~M9 — 어떤 순서로 만들 것인가
│   ├── API_SPEC.md          # API 요청/응답 상세 명세
│   └── guides/              # 프레임워크별 개발 가이드
├── todo-backend/            # Spring Boot 애플리케이션
└── todo-frontend/           # Next.js 애플리케이션
```

### 2.1 Backend 디렉토리 구조

> 베이스 패키지는 **`com.example`** 이다. (`pom.xml`의 `groupId`와 일치)

```
todo-backend/
├── src/main/java/com/example/
│   ├── TodoBackendApplication.java
│   ├── config/
│   │   ├── SecurityConfig.java          # Spring Security + 필터체인
│   │   ├── CorsConfig.java              # CORS 설정
│   │   └── JpaAuditingConfig.java       # 생성/수정일 자동관리
│   ├── common/
│   │   ├── entity/BaseEntity.java       # createdAt, updatedAt, deletedAt(soft delete)
│   │   ├── dto/
│   │   │   ├── ApiResponse.java         # 공통 응답 래퍼
│   │   │   └── PageResponse.java        # 페이지네이션 응답 DTO
│   │   └── exception/
│   │       ├── GlobalExceptionHandler.java
│   │       ├── CustomException.java
│   │       └── ErrorCode.java           # 에러코드 enum (API_SPEC.md 참조)
│   ├── domain/
│   │   ├── user/
│   │   │   ├── User.java                # 엔티티
│   │   │   ├── AuthProvider.java        # enum (LOCAL, GOOGLE, KAKAO)
│   │   │   ├── UserRepository.java
│   │   │   └── UserService.java
│   │   └── todo/
│   │       ├── Todo.java                # 엔티티
│   │       ├── TodoStatus.java          # enum (TODO, DONE)
│   │       ├── TodoRepository.java
│   │       └── TodoService.java
│   ├── auth/
│   │   ├── AuthController.java          # 회원가입/로그인/내 정보
│   │   ├── dto/                         # SignupRequest, LoginRequest, TokenResponse ...
│   │   ├── jwt/
│   │   │   ├── JwtTokenProvider.java    # 토큰 생성/검증
│   │   │   ├── JwtAuthenticationFilter.java
│   │   │   └── JwtAuthenticationEntryPoint.java  # 401도 ApiResponse로 래핑 (10장)
│   │   └── oauth2/
│   │       ├── CustomOAuth2UserService.java
│   │       ├── OAuth2SuccessHandler.java
│   │       ├── OAuth2UserInfo.java              # 공통 인터페이스
│   │       ├── GoogleOAuth2UserInfo.java
│   │       └── KakaoOAuth2UserInfo.java         # 응답 구조가 달라 별도 구현 필요
│   └── todo/
│       ├── TodoController.java
│       └── dto/                         # TodoCreateRequest, TodoUpdateRequest, TodoResponse ...
├── src/main/resources/
│   ├── application.properties           # 공통 설정
│   ├── application-dev.properties       # 로컬 개발
│   └── application-prod.properties      # 운영(RDS)
└── pom.xml
```

### 2.2 Frontend 디렉토리 구조

> ⚠️ 이 프로젝트는 **`src/` 디렉토리를 사용하지 않는다.** `app/`, `components/`, `lib/`가 프로젝트 루트에 위치한다.
> (`components.json`의 alias가 `@/components`, `@/lib`이고 css 경로가 `app/globals.css`인 것과 일치)

```
todo-frontend/
├── app/
│   ├── layout.tsx                       # 루트 레이아웃 (Providers)
│   ├── page.tsx                         # 진입 — 토큰 유무로 리다이렉트
│   ├── globals.css                      # Tailwind 4 CSS-first 설정 + 디자인 토큰
│   ├── (auth)/
│   │   ├── login/page.tsx               # 로그인 페이지
│   │   └── signup/page.tsx              # 회원가입 페이지
│   ├── (main)/
│   │   ├── layout.tsx                   # 인증 가드 + 공통 헤더
│   │   └── todos/
│   │       ├── page.tsx                 # Todo 목록 페이지 (페이지네이션)
│   │       ├── new/page.tsx             # Todo 작성 페이지
│   │       └── [id]/page.tsx            # Todo 상세/편집 페이지
│   └── oauth2/callback/page.tsx         # OAuth2 콜백 페이지
├── components/
│   ├── ui/                              # shadcn/ui 컴포넌트
│   ├── common/
│   │   ├── Pagination.tsx               # ✅ 재사용 페이지네이션 컴포넌트
│   │   ├── Header.tsx                   # 공통 헤더 (로고·사용자 메뉴·테마 토글)
│   │   └── ThemeToggle.tsx              # 다크모드 토글
│   ├── auth/
│   │   ├── LoginForm.tsx
│   │   ├── SignupForm.tsx
│   │   └── SocialLoginButtons.tsx
│   ├── todo/
│   │   ├── TodoList.tsx
│   │   ├── TodoItem.tsx
│   │   ├── TodoForm.tsx
│   │   └── TodoFilter.tsx
│   └── editor/
│       └── TiptapEditor.tsx             # Tiptap 웹에디터 래퍼
├── lib/
│   ├── api/
│   │   ├── client.ts                    # fetch 인스턴스 (JWT 자동첨부, 401 처리)
│   │   ├── auth.ts
│   │   └── todo.ts
│   ├── auth/token.ts                    # 토큰 저장/조회/삭제
│   ├── schemas/                         # Zod 스키마 (auth, todo)
│   └── utils.ts                         # cn() 등
├── hooks/
│   ├── useAuth.ts
│   └── useTodos.ts                      # React Query 훅
├── providers/
│   ├── QueryProvider.tsx
│   └── ThemeProvider.tsx
├── types/                               # 공통 타입 정의
├── components.json                      # shadcn 설정
├── next.config.ts
├── tsconfig.json
└── package.json
```

---

## 3. 사용자 여정

```
1. 진입 (/)
   │
   ├─ 토큰 없음 ──────────────► [로그인 페이지]
   └─ 토큰 있음 ──────────────► [Todo 목록 페이지]

2. [로그인 페이지]
   ├─ 로그인 성공 ────────────► [Todo 목록 페이지]
   ├─ 로그인 실패 ────────────► 에러 메시지 표시 (페이지 유지)
   ├─ "회원가입" 클릭 ────────► [회원가입 페이지]
   └─ 소셜 로그인 클릭 ───────► OAuth2 제공자 ─► [OAuth2 콜백 페이지] ─► [Todo 목록 페이지]

3. [회원가입 페이지]
   ├─ 가입 성공 ──────────────► [로그인 페이지]
   ├─ 이메일 중복/검증 실패 ──► 에러 메시지 표시 (페이지 유지)
   └─ "로그인" 클릭 ──────────► [로그인 페이지]

4. [Todo 목록 페이지]   ※ 인증 필요 — 미인증 시 [로그인 페이지]로 리다이렉트
   ├─ "새 Todo" 클릭 ─────────► [Todo 작성 페이지]
   ├─ 항목 클릭 ──────────────► [Todo 상세/편집 페이지]
   ├─ 체크박스 토글 ──────────► 상태 변경 후 목록 갱신 (페이지 유지)
   ├─ 삭제 클릭 ──────────────► Soft Delete 후 목록 갱신 (페이지 유지)
   ├─ 필터/검색/페이지 이동 ──► 목록 재조회 (페이지 유지)
   └─ "로그아웃" 클릭 ────────► 토큰 삭제 ─► [로그인 페이지]

5. [Todo 작성 페이지]
   ├─ 저장 성공 ──────────────► [Todo 목록 페이지]
   └─ 취소 ───────────────────► [Todo 목록 페이지]

6. [Todo 상세/편집 페이지]
   ├─ 수정 저장 성공 ─────────► [Todo 목록 페이지]
   ├─ 삭제 ───────────────────► [Todo 목록 페이지]
   └─ 타인 소유/존재하지 않음 ► 404 처리 ─► [Todo 목록 페이지]
```

**공통 리다이렉트 규칙**

| 조건 | 동작 |
|------|------|
| 미인증 상태로 인증 필요 페이지 접근 | [로그인 페이지]로 리다이렉트 |
| 인증 상태로 [로그인]/[회원가입] 접근 | [Todo 목록 페이지]로 리다이렉트 |
| API가 401 반환 (토큰 만료/위조) | 토큰 삭제 후 [로그인 페이지]로 리다이렉트 |
| API가 404 반환 (타인 Todo 접근 포함) | [Todo 목록 페이지]로 이동 + 안내 메시지 |

---

## 4. 기능 요구사항

### 4.1 인증 / 회원

| ID | 기능 | 상세 | 관련 페이지 | 백엔드 API |
|----|------|------|------------|-----------|
| **AUTH-01** | 회원가입 | **ID는 이메일만** 허용. 이메일 형식 + 중복 검증. 비밀번호 **6자 이상**이면 통과. BCrypt 암호화 저장 | 회원가입 페이지 | ✅ |
| **AUTH-02** | 로그인 | 이메일 + 비밀번호 → JWT Access Token 발급 (**만료 24시간**) | 로그인 페이지 | ✅ |
| **AUTH-03** | 소셜 로그인 | **OAuth2** (Google, Kakao). 신규 사용자는 자동 가입, 기존 이메일은 계정 연동 | 로그인 페이지, OAuth2 콜백 페이지 | ✅ |
| **AUTH-04** | 내 정보 조회 | 토큰 기반 현재 사용자 정보 반환. 공통 헤더의 사용자 메뉴에 표시 | 공통 헤더 (인증 필요 페이지 전역) | ✅ |
| **AUTH-05** | 로그아웃 | **클라이언트에서 토큰 삭제만 수행**. 서버 상태 없음 → **대응 API 없음** | 공통 헤더 (인증 필요 페이지 전역) | ❌ |

**검증 규칙**

- 이메일: RFC 형식 + 중복 불가
- 비밀번호: **최소 6자** (그 이상 복잡도 제약을 추가하지 않는다)

### 4.2 Todo 관리

| ID | 기능 | 상세 | 관련 페이지 | 백엔드 API |
|----|------|------|------------|-----------|
| **TODO-01** | 목록 조회 | **페이지네이션** 적용. 본인 소유만 조회. 상태(`status`)/검색어(`keyword`) 필터 지원 | Todo 목록 페이지 | ✅ |
| **TODO-02** | 상세 조회 | 단건 조회. 본인 소유 검증 | Todo 상세/편집 페이지 | ✅ |
| **TODO-03** | 생성 | 제목 + Tiptap 본문 + 마감일(선택) | Todo 작성 페이지 | ✅ |
| **TODO-04** | 수정 | 제목/본문/마감일/상태 수정 | Todo 상세/편집 페이지 | ✅ |
| **TODO-05** | 상태 변경 | 목표 상태(`TODO`/`DONE`)를 **바디로 지정**해 변경. UI는 토글로 동작하되 요청은 **멱등** | Todo 목록 페이지, Todo 상세/편집 페이지 | ✅ |
| **TODO-06** | 삭제 | **Soft Delete** (물리 삭제 아님) | Todo 목록 페이지, Todo 상세/편집 페이지 | ✅ |

### 4.3 공통 UI

| ID | 기능 | 상세 | 관련 페이지 | 백엔드 API |
|----|------|------|------------|-----------|
| **UI-01** | 다크모드 토글 | 시스템 설정 연동 + 수동 토글. 선택값 로컬 저장 | 공통 헤더 (전역) | ❌ |
| **UI-02** | 페이지네이션 컴포넌트 | `components/common/Pagination.tsx` **재사용 컴포넌트**로 구현 | Todo 목록 페이지 | ❌ |

### 4.4 페이지네이션 규격 (필수)

- **백엔드**: Spring Data `Pageable` 사용, `PageResponse<T>` DTO로 반환.
  - 필드: `content`, `page`, `size`, `totalElements`, `totalPages`, `hasNext`
  - 기본 정렬 `created_at DESC`, 기본 size `10`, 최대 size `100`
- **프론트엔드**: `components/common/Pagination.tsx` **재사용 컴포넌트**를 별도로 구현.
  - 이전/다음, 페이지 번호, 생략 표시(`...`), 현재 페이지 하이라이트
  - shadcn/ui + lucide-react 아이콘 활용, 접근성(`aria-current`, `aria-label`) 고려

### 4.5 MVP 이후 기능 (범위 제외)

- 프로필 상세 관리, 언어 설정
- 실시간 알림, 소셜 기능(공유/협업)
- 태그·카테고리, 첨부파일, 본문 전문 검색
- **S3 등 오브젝트 스토리지** (첨부파일 도입 시 함께 검토)
- Refresh Token (명시 요청 전까지 도입하지 않음)
- 프론트엔드 단위 테스트(Jest/Vitest/Testing Library) — MVP는 **Playwright E2E 하나로 통일**한다

---

## 5. 메뉴 구조

```
📱 Todo List 내비게이션

👤 비로그인 메뉴
├── 🔑 로그인          → AUTH-02, AUTH-03    [로그인 페이지]
└── 📝 회원가입        → AUTH-01             [회원가입 페이지]

📋 로그인 후 메뉴 (공통 헤더)
├── 🏠 Todo 목록       → TODO-01, TODO-05, TODO-06, UI-02   [Todo 목록 페이지]
├── ➕ 새 Todo         → TODO-03             [Todo 작성 페이지]
├── 👤 사용자 메뉴
│   ├── 내 정보 표시   → AUTH-04
│   └── 🚪 로그아웃    → AUTH-05
└── 🌓 테마 토글       → UI-01

🔗 메뉴 미노출 경유 페이지 (직접 진입 불가)
├── OAuth2 콜백        → AUTH-03             [OAuth2 콜백 페이지]
│                        ※ OAuth2 제공자가 리다이렉트로만 진입시킴
└── Todo 상세/편집     → TODO-02, TODO-04, TODO-05, TODO-06  [Todo 상세/편집 페이지]
                         ※ Todo 목록의 항목 클릭으로 진입
```

---

## 6. 페이지별 상세 기능

### 6.1 로그인 페이지

> **구현 기능:** `AUTH-02`, `AUTH-03` | **인증:** 불필요 (인증 상태면 Todo 목록으로 리다이렉트)

| 항목 | 내용 |
|------|------|
| **역할** | 기존 사용자의 이메일/소셜 로그인 진입점 |
| **진입 조건** | 루트 진입 시 토큰 없음 / 인증 가드 리다이렉트 / 회원가입 성공 후 / 로그아웃 후 |
| **사용자 행동** | 이메일·비밀번호 입력 후 로그인, 또는 소셜 로그인 버튼 클릭 |
| **주요 기능** | • 이메일/비밀번호 폼 (클라이언트 검증: 이메일 형식, 비밀번호 6자)<br>• **로그인** 버튼 → 토큰 저장<br>• Google / Kakao 소셜 로그인 버튼<br>• 회원가입 페이지 링크 |
| **연동 API** | `POST /api/auth/login`, `GET /oauth2/authorization/{provider}` |
| **다음 이동** | 성공 → Todo 목록 페이지 / 실패 → 에러 메시지 표시 |

### 6.2 회원가입 페이지

> **구현 기능:** `AUTH-01` | **인증:** 불필요

| 항목 | 내용 |
|------|------|
| **역할** | 이메일 기반 신규 계정 생성 |
| **진입 조건** | 로그인 페이지의 "회원가입" 링크 |
| **사용자 행동** | 이메일·비밀번호·(선택)이름 입력 후 가입 |
| **주요 기능** | • 이메일 형식 검증 + 중복 안내<br>• 비밀번호 **6자 이상** 검증 (복잡도 규칙 없음)<br>• 비밀번호 확인 일치 검증<br>• **회원가입** 버튼<br>• 로그인 페이지 링크 |
| **연동 API** | `POST /api/auth/signup` |
| **다음 이동** | 성공 → 로그인 페이지 / 실패 → 필드별 에러 메시지 |

### 6.3 OAuth2 콜백 페이지

> **구현 기능:** `AUTH-03` | **인증:** 불필요 (경유 전용, 메뉴 미노출)

| 항목 | 내용 |
|------|------|
| **역할** | 백엔드가 발급한 JWT를 **URL 프래그먼트**로 수신해 저장하는 중계 페이지 |
| **진입 조건** | `OAuth2SuccessHandler`의 리다이렉트 (`/oauth2/callback#token=...`) |
| **사용자 행동** | 없음 (로딩 표시 후 자동 이동) |
| **주요 기능** | • `window.location.hash`에서 토큰 추출·저장<br>• 저장 직후 `history.replaceState`로 **URL에서 토큰 제거**<br>• 로딩 스피너 표시<br>• 토큰 누락/오류 시 에러 처리 (`#error=AUTH_007`)<br>• ⚠️ 이 페이지에서는 **외부 리소스를 로드하지 않는다** |
| **연동 API** | 없음 (토큰은 리다이렉트 URL 프래그먼트로 전달됨) |
| **다음 이동** | 성공 → Todo 목록 페이지 / 실패 → 로그인 페이지 + 에러 안내 |

### 6.4 Todo 목록 페이지

> **구현 기능:** `TODO-01`, `TODO-05`, `TODO-06`, `UI-02` | **인증:** 필요

| 항목 | 내용 |
|------|------|
| **역할** | 서비스의 홈. 본인 Todo를 페이지 단위로 조회·관리 |
| **진입 조건** | 로그인 성공 / OAuth2 콜백 완료 / 헤더의 "Todo 목록" 클릭 |
| **사용자 행동** | 목록 훑기, 상태 필터·검색, 페이지 이동, 체크 토글, 삭제, 항목 클릭 |
| **주요 기능** | • 상태 필터(전체/TODO/DONE) + 제목 검색<br>• 카드 리스트 (제목·상태·마감일)<br>• 체크박스로 상태 토글<br>• 삭제 버튼 (Soft Delete)<br>• **`Pagination` 재사용 컴포넌트**<br>• **새 Todo** 버튼<br>• 로딩 / 빈 상태 / 에러 UI |
| **연동 API** | `GET /api/todos?page=&size=&status=&keyword=`, `PATCH /api/todos/{id}/status`, `DELETE /api/todos/{id}` |
| **다음 이동** | 새 Todo → Todo 작성 페이지 / 항목 클릭 → Todo 상세·편집 페이지 |

**목록 상태는 URL 쿼리로 관리한다** ⚠️

필터·검색어·페이지 번호를 React state가 아니라 **URL 쿼리스트링에 둔다.**

```
/todos?page=1&status=DONE&keyword=장보기
```

| 이유 | 설명 |
|------|------|
| 뒤로가기 | 상세 페이지에서 돌아왔을 때 보던 페이지·필터가 유지된다 |
| 새로고침 | F5를 눌러도 1페이지로 튕기지 않는다 |
| 공유·북마크 | 특정 필터 상태를 링크로 전달할 수 있다 |
| React Query 키 | 쿼리 파라미터가 곧 캐시 키가 되어 중복 요청이 자연히 제거된다 |

> ⚠️ **Next.js 16 제약**: URL 쿼리를 읽으려면 `useSearchParams()`가 필요하고, 이 훅은 **가장 가까운 `<Suspense>` 경계까지 클라이언트 렌더링으로 전환**시킨다. Suspense 없이 사용하면 프리렌더 단계에서 문제가 된다.
> 따라서 이 페이지는 **필터·목록 영역을 `<Suspense>`로 감싸고 fallback으로 스켈레톤을 렌더**한다.
> (OAuth2 콜백 페이지 6.3은 쿼리가 아니라 **프래그먼트**를 쓰므로 이 제약이 적용되지 않는다.)

### 6.5 Todo 작성 페이지

> **구현 기능:** `TODO-03` | **인증:** 필요

| 항목 | 내용 |
|------|------|
| **역할** | 새 Todo 생성 |
| **진입 조건** | Todo 목록 페이지 또는 헤더의 "새 Todo" 클릭 |
| **사용자 행동** | 제목 입력, Tiptap으로 본문 작성, 마감일 선택 후 저장 |
| **주요 기능** | • 제목 인풋 (필수, 255자 이내)<br>• **Tiptap 에디터** 본문<br>• 마감일 피커 (선택)<br>• **저장** / 취소 버튼 |
| **연동 API** | `POST /api/todos` |
| **다음 이동** | 저장/취소 → Todo 목록 페이지 |

### 6.6 Todo 상세/편집 페이지

> **구현 기능:** `TODO-02`, `TODO-04`, `TODO-05`, `TODO-06` | **인증:** 필요 (+ 서버 측 소유권 검증)

| 항목 | 내용 |
|------|------|
| **역할** | 단건 Todo 조회 및 수정/삭제 |
| **진입 조건** | Todo 목록 페이지의 항목 클릭 (메뉴 미노출) |
| **사용자 행동** | 내용 확인, 제목·본문·마감일·상태 수정 후 저장, 또는 삭제 |
| **주요 기능** | • 기존 값 로드 후 편집 폼 렌더<br>• **Tiptap 에디터** 본문 수정<br>• 상태 토글 (TODO ↔ DONE)<br>• **저장** / 삭제(Soft Delete) / 취소 버튼 |
| **연동 API** | `GET /api/todos/{id}`, `PUT /api/todos/{id}`, `PATCH /api/todos/{id}/status`, `DELETE /api/todos/{id}` |
| **다음 이동** | 저장/삭제/취소 → Todo 목록 페이지 / 404 → Todo 목록 페이지 + 안내 |

### 6.7 공통 헤더 (전역 컴포넌트)

> **구현 기능:** `AUTH-04`, `AUTH-05`, `UI-01` | **인증:** 인증 필요 페이지 전역
>
> 독립 페이지가 아니라 `app/(main)/layout.tsx`에 포함되는 **전역 컴포넌트**이지만,
> 세 기능이 여기서만 구현되므로 정합성을 위해 별도 항목으로 명세한다.
> 테마 토글(`UI-01`)은 비인증 페이지에서도 노출된다.

| 항목 | 내용 |
|------|------|
| **역할** | 인증 상태 표시, 전역 내비게이션, 테마 전환 |
| **진입 조건** | 인증 필요 페이지 진입 시 항상 렌더 (테마 토글은 로그인/회원가입 페이지에도 노출) |
| **사용자 행동** | 메뉴 이동, 사용자 정보 확인, 로그아웃, 테마 전환 |
| **주요 기능** | • 로고 → Todo 목록으로 이동<br>• **Todo 목록** / **새 Todo** 내비게이션<br>• 사용자 메뉴에 이메일·이름 표시 (`AUTH-04`)<br>• **로그아웃** — 토큰 삭제 후 이동 (`AUTH-05`)<br>• 🌓 다크모드 토글 (`UI-01`) |
| **연동 API** | `GET /api/auth/me` (로그아웃은 대응 API 없음) |
| **다음 이동** | 로그아웃 → 로그인 페이지 / 메뉴 클릭 → 해당 페이지 |

---

## 7. API 명세

기본 경로: `/api`. 모든 응답은 `ApiResponse<T>`로 래핑하며, 목록은 `PageResponse<T>`를 담는다.
인증 필요 엔드포인트는 `Authorization: Bearer {JWT}` 헤더를 전제로 한다.
**요청/응답 바디 상세와 에러코드는 [API_SPEC.md](./API_SPEC.md)를 참조한다.**

### 7.1 인증

| Method | Endpoint | 설명 | 인증 | 기능 ID |
|--------|----------|------|------|---------|
| POST | `/api/auth/signup` | 회원가입 | ❌ | AUTH-01 |
| POST | `/api/auth/login` | 로그인 → JWT 발급 | ❌ | AUTH-02 |
| GET | `/oauth2/authorization/{provider}` | 소셜 로그인 시작 (`google` / `kakao`) | ❌ | AUTH-03 |
| GET | `/login/oauth2/code/{provider}` | 소셜 콜백 (Spring Security 내부 처리) | ❌ | AUTH-03 |
| GET | `/api/auth/me` | 내 정보 조회 | ✅ | AUTH-04 |

> **AUTH-05(로그아웃)는 대응 API가 없다.** JWT는 서버에 상태를 두지 않으므로 클라이언트가 토큰을 삭제하는 것으로 완료된다.

> OAuth2 성공 시 `OAuth2SuccessHandler`가 JWT를 생성하여 프론트엔드 콜백 URL로 리다이렉트한다.
> 토큰은 **쿼리스트링이 아니라 URL 프래그먼트**로 전달한다 — `{APP_FRONTEND_URL}/oauth2/callback#token=...`
>
> **이유**: 프래그먼트(`#` 뒤)는 브라우저가 서버로 전송하지 않으므로 `Referer` 헤더·프록시/CDN 액세스 로그에 24시간 유효 토큰이 남지 않는다. 쿼리스트링은 이 세 경로 모두에 토큰을 노출시킨다.
> 프론트는 토큰을 저장한 직후 `history.replaceState`로 URL에서 프래그먼트를 제거한 뒤 Todo 목록으로 이동한다.

### 7.2 Todo

모든 Todo 엔드포인트는 **본인 소유만 접근 가능**하며, 서버에서 소유권을 검증한다.

| Method | Endpoint | 설명 | 인증 | 기능 ID |
|--------|----------|------|------|---------|
| GET | `/api/todos?page=0&size=10&status=&keyword=` | 목록 (페이지네이션 + 필터) | ✅ | TODO-01 |
| GET | `/api/todos/{id}` | 상세 조회 | ✅ | TODO-02 |
| POST | `/api/todos` | 생성 | ✅ | TODO-03 |
| PUT | `/api/todos/{id}` | 수정 | ✅ | TODO-04 |
| PATCH | `/api/todos/{id}/status` | 상태 변경 — 바디로 목표 상태 지정 (**멱등**) | ✅ | TODO-05 |
| DELETE | `/api/todos/{id}` | Soft Delete | ✅ | TODO-06 |

**목록 조회 쿼리 파라미터**

| 파라미터 | 기본값 | 설명 |
|----------|--------|------|
| `page` | `0` | 0부터 시작하는 페이지 번호 |
| `size` | `10` | 페이지 크기 (**최대 100**, 초과 시 100으로 절삭) |
| `status` | (없음) | `TODO` / `DONE`. 미지정 시 전체 |
| `keyword` | (없음) | **`title` 대상 부분 일치 (대소문자 무시)**. 본문(JSONB) 검색은 MVP 제외 |
| 정렬 | `created_at DESC` | MVP에서는 고정 (정렬 선택 UI 없음) |

---

## 8. 데이터베이스 설계

### 8.1 스키마

- **스키마명(논리)**: `TodoListDB`
- **DBMS**: PostgreSQL
- 모든 테이블은 **Soft Delete**를 위해 `deleted_at` 컬럼을 가진다. (`NULL`이면 활성, 값이 있으면 삭제됨)

#### 스키마 식별자 대소문자 정책 ⚠️

PostgreSQL은 **따옴표 없는 식별자를 소문자로 폴딩**한다. 설정·문서에는 논리명 `TodoListDB`를 그대로 쓰지만, 실제 생성되는 물리 스키마는 `todolistdb`가 된다.

| 항목 | 값 |
|------|-----|
| 문서·설정에 표기하는 논리명 | `TodoListDB` |
| 실제 물리 식별자 | `todolistdb` (소문자 폴딩 결과) |
| **테스트 전용 스키마** | **`todolistdb_test`** — 개발 스키마와 반드시 분리 |
| 스키마 생성 방법 | `CREATE SCHEMA IF NOT EXISTS TodoListDB;` — **따옴표 없이** |

**테스트 스키마를 분리하는 이유**: 테스트는 데이터를 삽입·삭제·Soft Delete 처리한다. 개발 스키마를 공유하면 테스트를 돌릴 때마다 개발 데이터가 오염된다.
테스트 설정(`src/test/resources/application.properties`)은 `DB_URL` 환경변수를 **상속하지 않고** 테스트 스키마를 직접 지정하며, `hibernate.hbm2ddl.create_namespaces=true`로 스키마가 없으면 자동 생성한다.

```sql
-- ✅ 올바름: 따옴표 없이 생성 → 실제 이름 todolistdb
CREATE SCHEMA IF NOT EXISTS TodoListDB;

-- ❌ 금지: 인용 식별자로 생성하면 이름이 대문자로 고정되어,
--    따옴표 없이 전달되는 Hibernate default_schema / JDBC currentSchema가
--    todolistdb를 찾다가 런타임에 실패한다.
CREATE SCHEMA "TodoListDB";
```

`spring.jpa.properties.hibernate.default_schema=TodoListDB`와 JDBC URL의 `currentSchema=TodoListDB`는 모두 **따옴표 없이 전달**되므로 위 규칙과 일관되게 `todolistdb`로 해석된다.
M0의 DB 연결 확인 단계에서 `\dn` 으로 **실제 생성된 스키마 이름을 눈으로 확인**한다.

### 8.2 테이블 정의

#### `users`

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| id | BIGINT | PK, AUTO | 사용자 ID |
| email | VARCHAR(255) | UNIQUE, NOT NULL | 로그인 ID (이메일만 사용) |
| password | VARCHAR(255) | NULL 허용 | BCrypt 해시. 소셜 로그인 전용 사용자는 NULL |
| name | VARCHAR(100) | NULL 허용 | 표시 이름 |
| provider | VARCHAR(20) | NOT NULL | `LOCAL`, `GOOGLE`, `KAKAO` |
| provider_id | VARCHAR(255) | NULL 허용 | 소셜 제공자 고유 ID |
| created_at | TIMESTAMP | NOT NULL | 생성일 |
| updated_at | TIMESTAMP | NOT NULL | 수정일 |
| deleted_at | TIMESTAMP | NULL | Soft Delete 시각 |

#### `todos`

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| id | BIGINT | PK, AUTO | Todo ID |
| user_id | BIGINT | FK → users.id, NOT NULL | 소유자 |
| title | VARCHAR(255) | NOT NULL | 제목 |
| content | **JSONB** | NULL 허용 | Tiptap 본문 (Tiptap JSON 문서 구조) |
| status | VARCHAR(20) | NOT NULL, DEFAULT `TODO` | `TODO` / `DONE` |
| due_date | DATE | NULL 허용 | 마감일 |
| created_at | TIMESTAMP | NOT NULL | 생성일 |
| updated_at | TIMESTAMP | NOT NULL | 수정일 |
| deleted_at | TIMESTAMP | NULL | Soft Delete 시각 |

**인덱스**: `(user_id, deleted_at)`, `(user_id, status, deleted_at)` — 목록 조회 성능 최적화.

> **`keyword` 검색은 위 인덱스로 커버되지 않는다.** 대소문자 무시 부분 일치(`LOWER(title) LIKE '%kw%'`)는 선행 와일드카드 때문에 B-tree 인덱스를 탈 수 없다.
> MVP에서는 `user_id` 선필터로 후보를 좁힌 뒤 스캔하는 것으로 충분하다. 사용자당 데이터가 크게 늘면 `pg_trgm` 확장 + GIN 인덱스 도입을 검토한다 (**MVP 범위 외**).

### 8.3 Soft Delete 구현 방침

- JPA 엔티티에 `@SQLDelete` + `@SQLRestriction("deleted_at IS NULL")` (Hibernate 6.x 이상) 적용.
- 모든 조회 쿼리는 자동으로 삭제되지 않은 레코드만 반환한다.

**단, `@SQLRestriction`만 믿지 않는다.** 적용 범위에 다음 사각지대가 있다.

| 경로 | `@SQLRestriction` 적용 | 대응 |
|------|----------------------|------|
| JPQL / 파생 쿼리 | ✅ 적용됨 | 그대로 사용 |
| 네이티브 쿼리 | ❌ **적용 안 됨** | MVP에서는 네이티브 쿼리를 쓰지 않는다 |
| `findById()` (ID 직접 로드) | ⚠️ 불확실 | **파생 쿼리로 대체** (아래) |
| count 쿼리(`totalElements`) | ⚠️ 확인 필요 | M1에서 테스트로 검증 |

- 단건 조회는 `findById` 대신 **`findByIdAndUser_IdAndDeletedAtIsNull(...)` 형태의 파생 쿼리**를 사용한다. 소유권 검증(10장)과 Soft Delete 필터를 한 번에 처리하므로 `@SQLRestriction` 동작 여부와 무관하게 안전하다.
- `@SQLDelete`에 적는 UPDATE 문은 **네이티브 SQL**이라 `default_schema`가 자동 적용되지 않을 수 있다. M1에서 실행 SQL을 확인한다.

#### 연관관계 페치 전략 — `LAZY` 기본 ⚠️

**모든 `@ManyToOne`·`@OneToOne`에 `fetch = FetchType.LAZY`를 명시한다.**

```java
@ManyToOne(fetch = FetchType.LAZY)   // ✅ 반드시 명시
@JoinColumn(name = "user_id", nullable = false)
private User user;
```

JPA에서 `@ManyToOne`의 **기본값은 `EAGER`** 다. 명시하지 않으면 `Todo` 하나를 조회할 때마다 `User`를 함께 가져오고, **목록 조회(기본 10건)에서 사용자 조회 쿼리가 10번 더 나가는 N+1 문제**가 발생한다. 페이지네이션이 필수인 이 프로젝트에서는 기본값을 그대로 두면 안 된다.

> 소유권 검증은 `findByIdAndUser_IdAndDeletedAtIsNull` 같은 **파생 쿼리의 조건절**로 처리하므로, `User` 엔티티를 실제로 로딩할 일이 거의 없다. `LAZY`로 두어도 기능상 손해가 없다.

### 8.4 Tiptap 본문(`content`) JSONB 매핑 방침 ⚠️

`content`는 `JSONB` 컬럼이며(13.1 확정), JPA 매핑은 **Hibernate 네이티브 JSON 매핑**을 사용한다. 별도 라이브러리(hypersistence-utils 등)를 추가하지 않는다.

```java
@JdbcTypeCode(SqlTypes.JSON)
@Column(columnDefinition = "jsonb")
private String content;   // Tiptap JSON 문서 문자열
```

**필드 타입은 `String`으로 둔다.** 이유:

- Spring Boot 4는 **Jackson 3(`tools.jackson`)** 이 기본이다. Hibernate의 JSON `FormatMapper` 자동 구성은 Jackson 2를 전제하고 있어, `Map`/POJO 타입으로 매핑하면 기동 시 `Could not find a FormatMapper for the JSON format` 오류가 날 수 있다.
- `String` ↔ `jsonb`는 FormatMapper 의존이 가장 낮은 조합이다.
- Tiptap 본문은 **서버가 내용을 해석할 필요가 없는 불투명(opaque) 데이터**다. 검색 대상도 `title`뿐이므로(7장), 서버에서 JSON 트리로 다룰 이유가 없다.
- API 응답에서는 `TodoResponse`가 이 문자열을 **JSON 객체 그대로** 직렬화해 내보낸다(API_SPEC 4.1). 프론트는 파싱 없이 Tiptap에 그대로 전달한다.

**M1 착수 시 반드시 실측할 것**

- [ ] `Todo` 엔티티만 만든 상태에서 애플리케이션이 기동되는가?
- [ ] Tiptap JSON을 저장·조회했을 때 값이 손실 없이 왕복하는가?
- [ ] 기동 실패 시 → `spring.jpa.properties.hibernate.type.json_format_mapper`에 `tools.jackson` 기반 매퍼를 직접 구현해 등록한다 (`application.properties`에 주석으로 자리를 마련해 두었다).

---

## 9. UI/UX 디자인 방향

### 9.1 디자인 컨셉 — "Calm Minimal"

여백 중심의 미니멀 디자인을 기준으로 한다.

- **레이아웃**: 넉넉한 여백(whitespace), 콘텐츠 중앙 정렬, 카드 기반 리스트
- **색상**: 무채색 베이스(`neutral` — `components.json`의 baseColor와 일치) + **액센트 컬러 `Indigo` 1개 고정**. 과한 그라디언트 지양
- **타이포그래피**: `Geist` 또는 `Inter` — 가독성 높은 산세리프, 명확한 위계
- **모서리/그림자**: `rounded-xl`, 아주 옅은 소프트 섀도우 (무거운 그림자 지양)
- **다크모드**: 필수 지원 (시스템 설정 연동 + 토글, `UI-01`)
- **모션**: Framer Motion으로 **절제된 마이크로 인터랙션** (리스트 진입 fade/slide, 체크 토글 스프링 애니메이션, 페이지 전환)
- **컴포넌트**: shadcn/ui(style `radix-nova`) 기반의 일관된 디자인 시스템
- **아이콘**: lucide-react (얇은 선 스타일이 미니멀 컨셉과 조화)

> 참고 트렌드: Linear, Vercel, Raycast 계열의 **저채도 + 고대비 텍스트 + 미세한 모션** 방향. 화려함보다 **정돈된 정보 위계와 반응성**에 집중한다.

### 9.2 화면별 레이아웃 요약

각 화면의 기능 상세는 6장을 따른다.

| 화면 | 레이아웃 |
|------|----------|
| 로그인 / 회원가입 | 중앙 정렬 카드 폼 + 하단 소셜 로그인 버튼 |
| Todo 목록 | 상단 필터(상태/검색) + 카드 리스트 + 하단 페이지네이션 |
| Todo 작성/편집 | 제목 인풋 + Tiptap 에디터 + 마감일 피커 + 하단 액션 바 |
| 공통 헤더 | 로고, Todo 목록/새 Todo, 사용자 메뉴, 테마 토글 |

---

## 10. 보안 요구사항

- **비밀번호**: BCrypt 해시 저장. 평문 저장 금지
- **JWT**: Access Token만 사용, **만료 24시간**, 서명 비밀키는 환경변수 관리, 헤더 `Authorization: Bearer {token}`
- **Spring Security**: 인증 필터 체인 구성. `/api/auth/signup`, `/api/auth/login`, OAuth2 경로 외 전부 인증 필요
- **인증 실패 응답도 `ApiResponse`로 래핑**: `JwtAuthenticationFilter`에서 발생하는 401은 `GlobalExceptionHandler`(`@RestControllerAdvice`)에 **도달하지 않는다**. 필터는 DispatcherServlet 바깥에 있기 때문이다. 따라서 `JwtAuthenticationEntryPoint`를 등록해 `AUTH_003`/`AUTH_004`/`AUTH_005` 에러코드를 담은 `ApiResponse` JSON을 직접 응답한다. 이것이 없으면 "모든 응답은 `ApiResponse`로 감싼다"는 계약이 인증 실패에서만 깨진다
- **OAuth2 인가 요청 저장소**: JWT는 무상태(`STATELESS`)지만, Spring Security의 OAuth2 로그인은 기본적으로 **HttpSession에 state/PKCE를 보관**한다. 전역 `SessionCreationPolicy.STATELESS`를 적용하면 콜백에서 `authorization_request_not_found`가 발생한다. **쿠키 기반 `AuthorizationRequestRepository`를 사용**하거나, `/oauth2/**`·`/login/oauth2/**` 경로에 한해 세션 생성을 허용한다
- **OAuth2 토큰 전달**: 발급된 JWT는 **URL 프래그먼트**로 프론트에 전달한다. 쿼리스트링은 `Referer`·브라우저 히스토리·프록시 로그에 토큰을 남긴다 (7.1 참조)
- **소유권 검증**: 모든 Todo 엔드포인트(`TODO-01`~`TODO-06`)에서 `user_id`가 인증 주체와 일치하는지 서버에서 검증한다. 불일치 시 **404 Not Found**를 반환한다 (403은 타인 리소스의 존재 여부를 노출시키므로 사용하지 않는다)
- **CORS**: 프론트엔드 도메인만 허용. 세부 조건은 아래를 따른다

| 항목 | 값 | 근거 |
|------|-----|------|
| `allowedOrigins` | 개발 `http://localhost:3000` / 운영 Amplify 도메인 | 와일드카드 `*` 금지 |
| `allowedMethods` | `GET`, `POST`, `PUT`, **`PATCH`**, `DELETE`, `OPTIONS` | ⚠️ **`PATCH` 누락 주의** — `TODO-05` 상태 변경이 PATCH다. 빠뜨리면 "다른 건 되는데 체크박스만 안 되는" 증상이 난다 |
| `allowedHeaders` | `Authorization`, `Content-Type` | JWT는 `Authorization` 헤더로 전달된다 |
| `allowCredentials` | **`false`** | 쿠키가 아니라 Bearer 헤더를 쓰므로 자격증명 전송이 불필요하다. `true`로 두면 `allowedOrigins`에 `*`를 쓸 수 없게 되는 제약만 추가된다 |

> CORS 설정은 `CorsConfig.java`에 두고, **M2에서 구성**한다. M9에서 운영 도메인으로 `allowedOrigins`를 갱신한다.
- **환경변수**: DB 접속정보, JWT 시크릿, OAuth2 Client ID/Secret은 시스템 환경변수로 분리 (**Git 커밋 금지**)
- **입력 검증**: Bean Validation(`@Valid`)으로 서버 측 검증 필수

---

## 11. 개발 단계

개발 진행 순서와 완료 기준(DoD)은 **[ROADMAP.md](./ROADMAP.md)의 마일스톤 M0~M9를 단일 기준**으로 한다.

> PRD는 **"무엇을"** 만들지, ROADMAP은 **"어떤 순서로"** 만들지를 담당한다.
> 진행 상태 체크리스트는 ROADMAP.md 한 곳에서만 관리한다.

---

## 12. 환경변수 목록

### 12.1 Backend

실제 값은 시스템 환경변수로 주입하고, `application.properties`에서는 플레이스홀더로 참조한다. **민감정보를 파일에 직접 적어 커밋하지 않는다.**

```properties
# --- Database ---
spring.datasource.url=${DB_URL}
spring.datasource.username=${DB_USERNAME}
spring.datasource.password=${DB_PASSWORD}

# --- JPA / Hibernate ---
# ddl-auto는 프로파일별 properties에서 지정 (dev: update / prod: validate)
spring.jpa.properties.hibernate.default_schema=TodoListDB

# Tiptap 본문(JSONB) 매핑용. 기본 자동 구성으로 동작하면 비워 둔다.
# Jackson 3 / Hibernate 조합 문제 발생 시에만 직접 구현해 등록한다. (8.4 참조)
# spring.jpa.properties.hibernate.type.json_format_mapper=

# --- JWT ---
jwt.secret=${JWT_SECRET}
jwt.expiration=86400000

# --- OAuth2 Registration (Google) ---
spring.security.oauth2.client.registration.google.client-id=${OAUTH_GOOGLE_CLIENT_ID}
spring.security.oauth2.client.registration.google.client-secret=${OAUTH_GOOGLE_CLIENT_SECRET}

# --- OAuth2 Registration (Kakao) ---
spring.security.oauth2.client.registration.kakao.client-id=${OAUTH_KAKAO_CLIENT_ID}
spring.security.oauth2.client.registration.kakao.client-secret=${OAUTH_KAKAO_CLIENT_SECRET}

# --- App ---
app.frontend-url=${APP_FRONTEND_URL}
```

**프로파일별 `ddl-auto`**

| 프로파일 | 값 | 이유 |
|----------|-----|------|
| `dev` (`application-dev.properties`) | `update` | 로컬 개발 중 스키마 변경을 즉시 반영 |
| `prod` (`application-prod.properties`) | `validate` | 운영 DB 스키마를 코드가 변경하지 못하게 차단 |

**주입할 시스템 환경변수**: `DB_URL`, `DB_USERNAME`, `DB_PASSWORD`, `JWT_SECRET`, `OAUTH_GOOGLE_CLIENT_ID`, `OAUTH_GOOGLE_CLIENT_SECRET`, `OAUTH_KAKAO_CLIENT_ID`, `OAUTH_KAKAO_CLIENT_SECRET`, `APP_FRONTEND_URL`

### 12.2 Kakao OAuth2 — provider 수동 등록 필수 ⚠️

Google은 Spring Security의 `CommonOAuth2Provider`에 내장되어 `registration`만 설정하면 되지만,
**Kakao는 내장 프로바이더가 아니므로 `provider` 블록을 직접 등록해야 한다.**

**① `provider` 블록 — 엔드포인트 수동 등록**

```properties
spring.security.oauth2.client.provider.kakao.authorization-uri=https://kauth.kakao.com/oauth/authorize
spring.security.oauth2.client.provider.kakao.token-uri=https://kauth.kakao.com/oauth/token
spring.security.oauth2.client.provider.kakao.user-info-uri=https://kapi.kakao.com/v2/user/me
spring.security.oauth2.client.provider.kakao.user-name-attribute=id
```

**② `registration` 블록 — Google이라면 기본값으로 채워지는 항목들** ⚠️

`CommonOAuth2Provider`에 없는 프로바이더는 아래 값의 **기본값이 제공되지 않는다.** 생략하면 애플리케이션이 기동조차 하지 못한다.

```properties
spring.security.oauth2.client.registration.kakao.client-name=Kakao
spring.security.oauth2.client.registration.kakao.authorization-grant-type=authorization_code
spring.security.oauth2.client.registration.kakao.redirect-uri={baseUrl}/login/oauth2/code/{registrationId}
spring.security.oauth2.client.registration.kakao.client-authentication-method=client_secret_post
spring.security.oauth2.client.registration.kakao.scope=profile_nickname,account_email
```

> **이 항목들은 M3가 아니라 M0에서 문제가 된다.** 두 블록이 모두 없으면 기동 시점에 아래 순서로 실패한다 (실제 확인됨):
> 1. `provider` 블록 누락 → `Provider ID must be specified for client registration 'kakao'`
> 2. `authorization-grant-type` 누락 → `authorizationGrantType cannot be null`

추가 고려사항:

- **응답 구조가 다르다.** Kakao의 사용자 정보는 `kakao_account.email`, `kakao_account.profile.nickname`처럼 중첩되어 있어 Google과 동일한 파서를 쓸 수 없다 → `KakaoOAuth2UserInfo` 별도 구현 (2.1 구조 참조).
- **이메일 동의 항목**이 선택으로 설정되면 이메일이 내려오지 않을 수 있다. 카카오 개발자 콘솔에서 필수 동의로 설정하거나, 이메일 부재 시 처리 정책을 정한다.
- `scope` 값은 **카카오 개발자 콘솔의 동의 항목과 반드시 일치**해야 한다. 콘솔에서 켜지 않은 scope를 요청하면 인가 단계에서 실패한다.

> 위 두 블록은 이미 `todo-backend/src/main/resources/application.properties`에 반영되어 있으며, 테스트 설정(`src/test/resources/application.properties`)에도 더미 값으로 동일하게 필요하다.

### 12.3 Frontend (`.env.local`)

```
NEXT_PUBLIC_API_BASE_URL
```

---

## 13. 확정 사항 및 가정

### 13.1 확정된 설계 결정

| 항목 | 확정값 | 근거 |
|------|--------|------|
| OAuth2 제공자 | **Google + Kakao** | 설정 추가만으로 Naver 등 확장 가능하도록 구조화 |
| Refresh Token | **미구현** | Access Token(24h)만 요구됨. 필요 시 별도 요청으로 추가 |
| Tiptap 본문 저장 | **`JSONB`** | 구조 보존·부분 조회에 유리. HTML `TEXT`는 사용하지 않음 |
| 액센트 컬러 | **Indigo** | `neutral` 베이스와 대비가 명확하고 다크모드에서도 채도 유지 |
| 소유권 위반 응답 | **404 Not Found** | 403은 타인 리소스의 존재 여부를 노출시킴 |
| OAuth2 토큰 전달 | **URL 프래그먼트** (`#token=`) | 프래그먼트는 서버로 전송되지 않아 Referer·로그 노출이 없음 |
| 상태 변경 방식 | **바디로 목표 상태 지정** (`{"status":"DONE"}`) | 서버 반전 토글은 비멱등 — 재시도·더블클릭 시 상태가 뒤집힘 |
| 소셜 계정 연동 정책 | **최초 가입 수단을 `provider`에 유지** | 아래 별도 설명 참조 |
| `content` JPA 매핑 | **`String` + `@JdbcTypeCode(SqlTypes.JSON)`** | 서버가 본문을 해석할 필요가 없고, Jackson 3 호환 문제를 회피 (8.4) |
| 스키마 물리 식별자 | **`todolistdb`** (따옴표 없이 생성) | PostgreSQL 소문자 폴딩 (8.1) |
| **토큰 저장 매체** | **`localStorage`** | 10장이 `Authorization: Bearer`를 전제하므로 JS가 토큰을 읽어야 한다 → httpOnly 쿠키 불가. `sessionStorage`는 탭을 닫으면 사라져 24h 토큰의 의미가 없다. XSS 노출도는 두 스토리지가 동일하므로 사용성이 나은 쪽을 택한다 |
| **목록 필터·페이지 상태** | **URL 쿼리스트링** (`?page=&status=&keyword=`) | 뒤로가기·새로고침·링크 공유가 동작하고, React Query 캐시 키와 자연히 일치한다 (6.4) |
| **테스트 DB** | **로컬 PostgreSQL `todolistdb_test` 스키마** | Docker 미도입 확정 → Testcontainers 제외. H2는 JSONB·식별자 폴딩을 재현 못함 (1.3) |
| **연관관계 페치** | **`LAZY` 명시 필수** | `@ManyToOne` 기본값 EAGER는 페이지네이션 목록에서 N+1을 유발 (8.3) |
| **S3 / 오브젝트 스토리지** | **MVP에서 제외** | 첨부파일이 범위 밖(4.5)이고 정적 자산은 Amplify가 처리 → 담을 것이 없다. 텍스트 기반 Todo에 집중 |
| 목록 기본 정렬 | **`created_at DESC`** | MVP에서는 정렬 선택 UI 없이 고정 |
| 기본 / 최대 page size | **10 / 100** | 최대치 초과 요청은 100으로 절삭 |
| `keyword` 검색 대상 | **`title`만** | JSONB 본문 전문 검색은 MVP 제외 |
| 로그아웃 방식 | **클라이언트 토큰 삭제** | 서버 상태가 없는 JWT 특성상 대응 API 불필요 |
| 백엔드 베이스 패키지 | **`com.example`** | `pom.xml`의 `groupId`와 일치시킴 |
| 프론트 `src/` 디렉토리 | **사용하지 않음** | `components.json`의 alias·css 경로와 일치시킴 |

**소셜 계정 연동 정책 (AUTH-03)**

`users` 테이블은 `provider`·`provider_id`를 각 1개만 가진다. 같은 이메일로 로컬 가입과 소셜 로그인이 겹칠 때의 동작을 아래와 같이 확정한다.

| 상황 | 동작 |
|------|------|
| 신규 이메일로 소셜 로그인 | 자동 가입. `provider` = `GOOGLE`/`KAKAO`, `password`는 `NULL` |
| **기존 LOCAL 계정과 같은 이메일**로 소셜 로그인 | 기존 계정으로 로그인시킨다. **`provider`는 `LOCAL`로 유지**하고 덮어쓰지 않는다 |
| 위 연동 후 비밀번호 로그인 | **계속 허용**. `password` 해시가 남아 있으므로 유효하다 |
| 같은 이메일로 Google·Kakao 모두 사용 | 이메일 매칭으로 동일 계정에 로그인. `provider`는 최초 가입 수단 그대로 |
| `GET /api/auth/me`의 `provider` | **최초 가입 수단**을 반환한다 (현재 로그인에 사용한 수단이 아님) |

> 즉 `provider`는 "이번에 어떻게 로그인했는가"가 아니라 **"이 계정이 어떻게 처음 만들어졌는가"** 를 뜻한다.
> 다중 인증수단을 정식으로 표현하려면 `user_auth_providers` 연결 테이블이 필요하나, **MVP 범위 밖**이다 (4.5 참조).

### 13.2 공통 응답 포맷

| 래퍼 | 필드 |
|------|------|
| `ApiResponse<T>` | `success`, `data`, `message`, `errorCode` |
| `PageResponse<T>` | `content`, `page`, `size`, `totalElements`, `totalPages`, `hasNext` |

실제 JSON 예시와 에러코드 체계는 [API_SPEC.md](./API_SPEC.md)를 따른다.

### 13.3 남은 가정

- 버전 관련 파괴적 변경(Spring Boot 4 / Next.js 16 / Tailwind 4)은 **구현 착수 시점에 공식 문서로 재확인**한다.
- Kakao 이메일 동의 항목 정책은 카카오 개발자 콘솔 설정 확인 후 확정한다.
- **`content` JSONB 매핑이 Spring Boot 4.1 + Jackson 3 환경에서 자동 구성만으로 동작하는지**는 M1에서 실측 확인한다 (8.4).
- **`@SQLRestriction`이 `findById`·count 쿼리에 적용되는지**는 M1 테스트로 확정한다 (8.3). 확정 전까지는 파생 쿼리를 사용한다.
- **`todolistdb_test` 스키마가 `hbm2ddl.create_namespaces=true`로 자동 생성되는지**는 M1에서 엔티티를 추가한 뒤 확인한다. 현재는 엔티티가 없어 Hibernate가 생성할 네임스페이스가 없으므로 검증되지 않은 상태다 (8.1).
- 프론트 라이브러리의 React 19 호환성(Tiptap · Framer Motion 패키지명)은 설치 시점(M5·M7)에 확인한다.

---

### 다음 단계

이 PRD를 기준으로 [ROADMAP.md](./ROADMAP.md)의 `M0`부터 순서대로 진행한다.
