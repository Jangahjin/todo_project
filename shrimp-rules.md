# Development Guidelines (AI Agent Operational Rules)

> 이 문서는 AI Agent 전용 작업 규칙이다. 프로젝트 개요·기술 배경은 `CLAUDE.md`/`docs/PRD.md`를 참조하고, 여기서는 "무엇을 어떻게 수정/추가할지"만 규정한다.

## 1. 저장소 구조 — 커밋 위치 결정

- 이 워크스페이스는 **3개의 독립 Git 저장소**다: 루트 `todo-project/`, `todo-backend/`, `todo-frontend/`.
- 파일 경로로 커밋 위치를 판단한다:
  - `docs/**`, `CLAUDE.md`, `.claude/**`, `shrimp-rules.md` → 루트 저장소, `main` 브랜치.
  - `todo-backend/**` → `todo-backend` 저장소, `main`+`develop` 브랜치 정책.
  - `todo-frontend/**` → `todo-frontend` 저장소, `main`+`develop` 브랜치 정책.
- 한 작업에서 두 종류 이상의 경로를 수정했다면 **각 저장소에서 별도 커밋**을 생성한다. 하나의 커밋으로 묶지 않는다.
- **금지**: 루트 `.gitignore`에서 `todo-backend/`·`todo-frontend/` 제외 규칙을 삭제하는 것. 삭제 시 두 폴더가 embedded repository로 기록되어 clone 시 빈 디렉토리가 된다.

## 2. 참조해야 할 문서 (우선순위)

- 요구사항 상세는 `docs/PRD.md`, API 계약은 `docs/API_SPEC.md`, 작업 순서는 `docs/ROADMAP.md`를 확인한 뒤 구현한다.
- `docs/guides/`와 `docs/PRD.md`가 충돌하면 **`docs/PRD.md` 1.3(기술 스택)을 사실로 삼는다.**
- 새 기능을 추가하기 전, 해당 기능이 이미 명시적으로 설치/도입된 마일스톤인지 `docs/PRD.md` 1.3 설치 상태 표와 `docs/ROADMAP.md`에서 먼저 확인한다.

## 3. 미설치 라이브러리 — 임의 import 금지

- 다음 라이브러리는 **아직 설치되지 않았다**: React Query, Framer Motion, Tiptap, React Hook Form, Zod.
- 이들을 `import`하는 코드를 작성하기 전, `todo-frontend/package.json`의 `dependencies`/`devDependencies`에 실제로 존재하는지 먼저 확인한다.
- 없는데 필요하다면 `npm install`로 설치를 먼저 수행하고, 설치 시점이 현재 마일스톤에 해당하는지 `docs/PRD.md` 1.3을 재확인한다.

## 4. Backend 구현 규칙

### 4.1 레이어 및 파일 배치
- Controller → Service → Repository 순서를 지킨다. Controller에 비즈니스 로직을 작성하지 않는다.
- 새 엔티티 추가 시 `BaseEntity`(생성/수정일 + `deleted_at`)를 상속한다.
- Soft Delete가 필요한 엔티티는 `@SQLDelete` + `@SQLRestriction("deleted_at IS NULL")`를 함께 붙인다. 물리 `DELETE` 문을 발생시키는 코드를 작성하지 않는다.
- 단건 조회 API는 `@SQLRestriction`에만 의존하지 말고 `findByIdAndUser_IdAndDeletedAtIsNull` 형태의 파생 쿼리로 소유권+삭제여부를 동시에 검증한다.
- 엔티티를 컨트롤러 응답 DTO로 직접 반환하지 않는다. 요청/응답 DTO를 별도로 만든다.
- 응답은 항상 `ApiResponse<T>`로 감싼다. 목록 응답은 `PageResponse<T>`를 사용한다(페이지네이션 없는 전체 목록 반환 금지).
- `@ManyToOne` 연관관계에는 반드시 `fetch = FetchType.LAZY`를 명시한다. 기본값(EAGER)을 그대로 두지 않는다.
- 요청 DTO에는 Bean Validation을 적용한다: 이메일 필드는 `@Email`, 비밀번호는 `@Size(min = 6)` (그 이상 복잡도 규칙 추가 금지).
- 예외 처리는 `GlobalExceptionHandler`에서 일괄 처리한다. 컨트롤러에 try-catch를 산발적으로 추가하지 않는다.
- 필터 단계(Spring Security)에서 발생하는 401은 `GlobalExceptionHandler`가 잡지 못하므로, `JwtAuthenticationEntryPoint`에서 `ApiResponse` 포맷으로 직접 응답을 작성한다.
- CORS 설정의 `allowedMethods`에 `PATCH`를 반드시 포함한다. `allowCredentials`는 `false`로 둔다(Bearer 헤더 사용, 쿠키 미사용).
- 상태 변경 API(toggle 등)는 서버가 값을 반전시키지 않는다. 클라이언트가 목표 상태를 요청 바디에 명시하는 멱등 방식으로 구현한다.
- 리소스 소유권 위반 시 `403`이 아닌 `404`를 반환한다(존재 여부 노출 방지).

### 4.2 설정 파일
- 설정 형식은 `application.properties`만 사용한다(YAML 금지).
- 공통 설정은 `application.properties`에, `ddl-auto`는 프로파일별 파일에 지정한다: `dev` → `update`, `prod` → `validate`.
- 어떤 프로파일 파일에도 평문 시크릿(`DB_PASSWORD`, `JWT_SECRET`, OAuth 키 등)을 적지 않는다. `${ENV}` 플레이스홀더만 사용하고, 비민감 값에 한해 `${ENV:기본값}` 형태의 로컬 기본값을 허용한다.
- DB 스키마는 물리 식별자 `todolistdb`로, **따옴표 없이** 생성한다(대문자 `TodoListDB`로 따옴표를 붙여 생성하지 않는다).

### 4.3 테스트
- 테스트는 `todolistdb_test` 스키마를 사용한다. 개발 스키마(`todolistdb`)를 테스트에 재사용하지 않는다.
- H2, Testcontainers는 사용하지 않는다. JUnit 5 + Spring Boot Test(MockMvc)만 사용한다.
- `src/test/resources/application.properties`는 main 설정을 **shadow**하므로, 테스트에 필요한 설정(DB 접속 정보 등)을 이 파일에 전부 자립적으로 적는다. main의 값이 상속된다고 가정하지 않는다.
- 실행 전 JDK 21이 활성 상태인지 확인한다(`java -version`이 17을 보고하면 빌드가 실패한다).

## 5. Frontend 구현 규칙

### 5.1 컴포넌트 및 상태
- `"use client"`는 클라이언트 상호작용이 실제로 필요한 컴포넌트에만 선언한다. 서버/클라이언트 경계를 불필요하게 넓히지 않는다.
- 서버 상태(API로 가져오는 데이터)는 React Query로 관리한다. 컴포넌트 내부에서 직접 `fetch`를 호출하지 않는다 — 반드시 `lib/api/`의 클라이언트를 통해 호출한다.
- 목록 화면의 상태(`page`, `status`, `keyword` 등)는 URL 쿼리스트링을 단일 출처(source of truth)로 삼는다. React Query의 쿼리 키도 이 쿼리스트링 값으로 구성한다. 별도의 로컬 state로 페이지/필터를 이중 관리하지 않는다.
- `useSearchParams()`를 사용하는 컴포넌트는 `<Suspense>`로 감싼다(Next.js 16 요구사항). 단, OAuth 콜백 페이지는 URL 프래그먼트(`#token=`)를 쓰므로 이 규칙에서 예외다.
- 페이지네이션 UI는 항상 공용 컴포넌트 `components/common/Pagination.tsx`를 통해 구현한다. 화면마다 페이지네이션을 새로 구현하지 않는다.
- JWT는 `localStorage`에 저장하고, API 클라이언트가 요청 시 자동으로 첨부하도록 구현한다.
- OAuth2 로그인 완료 후 토큰 전달은 URL 프래그먼트(`#token=`)로만 한다. 쿼리스트링으로 토큰을 전달하지 않는다.

### 5.2 스타일 및 아이콘
- Tailwind CSS 유틸리티 클래스를 우선 사용하고 `cn()` 유틸(`lib/utils.ts`)로 클래스를 병합한다. 임의의 인라인 `style` 속성 사용을 지양한다.
- UI 컴포넌트는 `npx shadcn@latest add <component>`로 추가한다. shadcn/ui 컴포넌트를 손으로 처음부터 새로 작성하지 않는다.
- 아이콘은 `lucide-react`만 사용한다. 다른 아이콘 라이브러리를 혼용하지 않는다.
- 애니메이션은 Framer Motion만 사용한다(설치 후). 다른 애니메이션 라이브러리를 추가하지 않는다.
- 디자인은 PRD의 "Calm Minimal" 컨셉(저채도 + 단일 액센트 + 다크모드 + 절제된 모션)을 유지한다.

### 5.3 타입 및 테스트
- `any` 타입 사용을 지양한다. 여러 파일에서 공유되는 타입은 `types/`에 정의한다.
- E2E 테스트는 Playwright만 사용한다(M5부터 도입). Jest, Vitest, Testing Library는 이 프로젝트 범위 밖이므로 추가하지 않는다.

## 6. 공통 규칙 — 불변 요구사항 (임의 변경 금지)

아래 항목은 사용자가 명시적으로 요청하지 않는 한 절대 변경하지 않는다:

1. 로그인 ID는 이메일만 사용한다. `username` 필드를 추가하지 않는다.
2. 비밀번호 최소 길이는 6자다. 추가 복잡도 규칙(특수문자 필수 등)을 넣지 않는다.
3. 비밀번호는 BCrypt로 해시하여 저장한다. 평문 저장 금지.
4. JWT Access Token 만료는 24시간이다. Refresh Token은 요구되기 전까지 구현하지 않는다.
5. 삭제는 Soft Delete만 사용한다(`deleted_at`). 물리 `DELETE` 금지.
6. 목록 조회 API는 반드시 페이지네이션을 적용한다.
7. Todo는 소유자 본인만 접근 가능하도록 서버에서 검증한다.

## 7. 커밋 및 작업 범위

- 커밋 메시지는 한국어, `type: subject` 형식(예: `feat: Todo 목록 페이지네이션 추가`)을 따른다.
- 요청받은 마일스톤 범위만 구현한다. `docs/ROADMAP.md`에서 아직 도달하지 않은 마일스톤의 기능(예: Refresh Token, 미도입 라이브러리 사용)을 임의로 선행 구현하지 않는다.
- 백엔드 코드 변경 후 `./mvnw test`, 프론트엔드 코드 변경 후 `npm run lint && npm run build`가 통과하는지 확인한다.

## 8. 버전 확인 — 추측 금지

- Spring Boot 4, Next.js 16, React 19, Tailwind CSS 4는 모두 최신 메이저 버전이며 이전 버전과 파괴적 변경이 있다.
- 다음 상황에서는 익숙한 이전 버전 방식(예: Tailwind 3의 `tailwind.config.js`, Spring Boot 3 이하 설정 방식)으로 코드를 작성하기 전에 해당 라이브러리의 최신 공식 문서를 먼저 확인한다:
  - Tailwind CSS 설정 파일 작성/수정 (CSS-first 설정 방식 확인)
  - Next.js 라우팅/캐싱 관련 코드 작성
  - Spring Boot의 Jakarta EE 관련 import 작성
- 라이브러리 API나 설정 방식이 불확실하면 임의로 작성하지 말고 공식 문서 또는 `context7` MCP로 확인한다.
