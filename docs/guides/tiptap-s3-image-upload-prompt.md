# Tiptap 에디터 이미지 첨부 기능 추가 (로컬 → S3)

> 클로드 코드에 그대로 붙여넣는 프롬프트입니다.
> **2026-09-04 개정**: 이 프로젝트(`todo-backend` + `todo_frontend`)의 실제 코드를 대조해 사실 오류를 정정하고, 그대로 구현하면 동작하지 않거나 보안 결함이 되는 지점을 보강했다. §0에 실측 결과를 미리 정리해 두었으니, 실행 세션은 이 문서만 보고 바로 §2부터 진행할 수 있다.

---

## 작업 목표

할일(Todo) 작성/수정 시 Tiptap 에디터의 본문에 **이미지 파일을 첨부**할 수 있게 한다.
파일 실체는 스토리지(로컬 디렉토리 또는 S3)에 저장하고, 메타데이터는 PostgreSQL에서 관리한다.

**개발 순서: 로컬 파일 저장으로 먼저 완성하고 테스트한 뒤, S3로 전환한다.**
프론트엔드 코드는 두 방식에서 **동일하게 동작해야 한다.** 스토리지 교체는 백엔드 설정만 바꿔서 이뤄진다.

`todo-backend`와 `todo_frontend`는 별도 저장소다(폴더명은 **언더스코어** — 문서에 하이픈으로 적힌 곳이 있어도 실제 경로는 이것). 백엔드를 먼저 완성하고 프론트로 넘어간다.

⚠️ **이 기능은 `docs/PRD.md` 4.5 / 13.1이 "S3·첨부파일은 MVP 제외"로 확정해 둔 항목을 뒤집는다.** §10 마지막 절의 문서 갱신 목록을 실제 구현 착수와 함께 처리한다.

---

## 0. 실측 결과 (선행 조사 완료 — 재조사 불필요)

아래는 이 문서를 작성하기 위해 `todo-backend`·`todo_frontend`를 직접 대조한 결과다. 실행 세션은 이걸 전제로 바로 구현에 들어가도 된다. 단, **세션 시작 시점에 파일·라인이 여전히 유효한지 한 번은 확인한다** — CLAUDE.md 1.1이 지적한 대로 이 프로젝트는 문서가 실제 코드보다 앞서 나가 틀린 채로 방치된 이력이 있다.

| 항목 | 실측 |
|---|---|
| 엔티티 구조 | `Todo`(`com.example.domain.todo.Todo`), `User`(`com.example.domain.user.User`). 본문 컬럼은 **`description`이 아니라 `content`**, 타입은 `jsonb` + `@JdbcTypeCode(SqlTypes.JSON)`, Java 타입은 `String`(`Todo.java:53-55`) |
| 본문 포맷 | **HTML이 아니라 Tiptap JSON.** 프론트는 `editor.getJSON()`만 쓰고 `getHTML()` 호출은 0건(`components/editor/TiptapEditor.tsx`). `TodoCreateRequest`/`TodoUpdateRequest`는 이미 `tools.jackson.databind.JsonNode`로 `content`를 받는다(`TodoService.toJsonString`이 문자열화해 저장) |
| Soft Delete | `@SQLDelete` + `@SQLRestriction("deleted_at IS NULL")` (Hibernate 6) + 단건 조회는 `findByIdAndUser_IdAndDeletedAtIsNull` 파생 쿼리 병행 |
| DB 스키마 관리 | **Flyway/Liquibase 없음.** `ddl-auto`만 사용 — `application-dev.properties`는 `update`, `application-prod.properties`는 `validate`. 신규 테이블은 prod에서 수동 DDL 필요(`docs/guides/aws-deployment.md`의 "ddl-auto=validate 우회 전략" 참조) |
| 설정 파일 형식 | `application.properties`/`application-dev.properties`/`application-prod.properties`만 존재. **`.yml`은 프로젝트 전체에 0건** — 전환 작업 불필요 |
| 프로파일 | **`dev` / `prod`** 두 개뿐(`local` 아님). `spring.profiles.active`는 어떤 설정 파일에도 없고 실행 시 인자로 지정(`-Dspring-boot.run.profiles=dev`, `--spring.profiles.active=prod`) |
| 인증 주체 식별 | 커스텀 `UserDetails` 없음. JWT 필터가 `Long userId`를 principal로 심고(`JwtAuthenticationFilter.java:44-48`), 컨트롤러는 `@AuthenticationPrincipal Long userId`로 꺼낸다(`TodoController.java:39` 등) |
| Tiptap 에디터 | `components/editor/TiptapEditor.tsx` — `extensions: [StarterKit]` 뿐. **`@tiptap/extension-image` 미설치.** 버전은 `@tiptap/react`·`@tiptap/pm`·`@tiptap/starter-kit` 전부 `3.30.6` |
| 공통 응답 | `ApiResponse<T>`/`PageResponse<T>` 둘 다 **record**, 정적 팩토리는 `ApiResponse.success(data)`/`fail(msg, errorCode)` 등 |
| 예외 처리 | 세분화된 예외 클래스 없음. **`CustomException(ErrorCode)`** 하나 + `ErrorCode` enum(`httpStatus`, `defaultMessage`). 소유권 위반은 **404**(`ErrorCode.TODO_001`) — 불변 규칙 11 |
| 스케줄러 | `@EnableScheduling` 미등록(전체 grep 0건). 등록된 `@Configuration`은 `SecurityConfig`·`CorsConfig`·`JpaAuditingConfig` 3개뿐 |
| 의존성 | Spring Boot **4.1.1** / Java 21 / **Jackson 3**(`tools.jackson.*`). AWS SDK·commons-io 등 파일 처리 의존성 없음 |
| 프론트 API 클라이언트 | `lib/api/client.ts`의 `apiFetch`는 (a) `Content-Type: application/json` 강제, (b) 응답을 항상 `response.json()`으로 파싱 → **바이너리 PUT/GET에 재사용 불가** |
| 프론트 폴더 | `todo_frontend`(언더스코어) |
| Todo 상세 화면 | 읽기 전용 뷰가 없다. `app/(main)/todos/[id]/page.tsx`가 곧바로 편집 폼(`TodoForm`)을 렌더 — 이미지는 **에디터 안에서만** 보인다 |
| `.gitignore` | 루트 저장소가 `docs/`를 관리하므로, `todo-project/upload/`를 무시하려면 **루트 `.gitignore`에 추가**해야 한다. 하위 저장소(`todo-backend`/`todo_frontend`) `.gitignore`에 `../upload/`를 적어도 **무효**(git은 저장소 밖 경로를 무시 규칙으로 다루지 못함) |

---

## 1. DB 스키마 설계

새 테이블 `attachment`를 추가한다.

| 컬럼 | 타입 | 설명 |
|---|---|---|
| `id` | BIGSERIAL PK | |
| `todo_id` | BIGINT NULL, FK → todos(id) | **NULL 허용** (아래 이유 참고) |
| `user_id` | BIGINT NOT NULL, FK → users(id) | 업로더 (권한 검증용) |
| `storage_type` | VARCHAR(20) NOT NULL | `LOCAL` / `S3` |
| `storage_key` | VARCHAR(512) NOT NULL UNIQUE | 로컬 상대경로 또는 S3 객체 키 |
| `original_filename` | VARCHAR(255) NOT NULL | 원본 파일명 (표시용. 저장 키의 확장자 소스로 쓰지 않는다 — 아래 참고) |
| `content_type` | VARCHAR(100) NOT NULL | 서버가 검증을 통과시킨 값 |
| `file_size` | BIGINT NOT NULL | bytes |
| `status` | VARCHAR(20) NOT NULL | `TEMP` / `LINKED` |
| `created_at` | TIMESTAMP NOT NULL | |
| `deleted_at` | TIMESTAMP NULL | Soft Delete (기존 정책과 동일하게, `@SQLDelete`+`@SQLRestriction`) |

**`todo_id`를 NULL 허용으로 두는 이유**
에디터에서는 할일을 저장하기 *전에* 이미지가 먼저 업로드된다. 따라서 업로드 시점에는 `status = TEMP`, `todo_id = NULL`로 만들고, 할일 저장/수정 시 `content`(Tiptap JSON) 안에 실제로 남아 있는 첨부만 `LINKED`로 전환하며 `todo_id`를 채운다.

**`storage_type`을 두는 이유**
로컬에서 테스트한 데이터와 S3 전환 이후 데이터가 섞여도 각각 올바른 방식으로 조회하기 위함이다.

**같은 이미지가 여러 Todo 본문에 나타나는 경우**
`todo_id`가 단일 FK라 한 첨부는 한 Todo에만 속한다. 본문을 복사·붙여넣기해 같은 `attachmentId`가 두 Todo에 들어가면, 먼저 저장되는 쪽이 `LINKED`를 가져가고 나머지 쪽은 연결이 끊긴 것으로 취급한다(§6 연결 처리 알고리즘에서 "다른 Todo에 이미 LINKED된 ID"는 소유권 위반과 동일하게 거부). **MVP 정책: 복제된 본문에서 재사용된 이미지는 재업로드를 요구한다.** 다대다 연결 테이블은 범위 밖으로 남긴다.

**인덱스**: `todo_id`, `(status, created_at)` — 고아 파일 정리 배치용

**저장 키 규칙**: `todos/{userId}/{yyyy}/{MM}/{uuid}.{ext}` (로컬·S3 동일)
`{ext}`는 **원본 파일명에서 뽑지 않는다.** 클라이언트가 보낸 파일명은 신뢰할 수 없다(`shell.jpg.exe` 같은 이중 확장자, 확장자 없음 등). §6 화이트리스트를 통과한 `contentType` → 확장자 매핑 테이블(`image/jpeg→jpg`, `image/png→png`, `image/gif→gif`, `image/webp→webp`)에서 결정한다.

**마이그레이션**: 이 프로젝트는 Flyway/Liquibase를 쓰지 않는다. `attachment` 테이블은 엔티티 추가 후 `ddl-auto=update`(dev)로 자동 생성된다. **prod는 `validate`라 자동 생성되지 않으므로**, `docs/guides/aws-deployment.md`의 "Task 036 §3 테이블 생성 — ddl-auto=validate 우회 전략"과 같은 방식으로 수동 DDL을 준비한다.

---

## 2. 스토리지 추상화 (핵심)

로컬과 S3를 같은 인터페이스로 다룬다.

```java
public interface StorageService {
    StorageType getType();
    // 업로드용 URL 발급 (S3: presigned PUT / 로컬: 백엔드 업로드 엔드포인트)
    String createUploadUrl(String storageKey, String contentType);
    // 조회용 URL 발급 (S3: presigned GET / 로컬: 백엔드 조회 엔드포인트)
    String createViewUrl(String storageKey);
    // 업로드 완료 후 실제 파일 크기 확인 (아래 3장 "크기 검증" 참고)
    long verifyUploaded(String storageKey);
    void delete(String storageKey);
}
```

구현체 두 개를 만들고 `app.storage.type` 값에 따라 `@ConditionalOnProperty`로 빈을 하나만 등록한다.

- `LocalStorageService` — 로컬 디스크 저장
- `S3StorageService` — AWS S3 저장

`AttachmentService`는 이 인터페이스만 의존하고, 어떤 구현체인지 알지 못한다.

---

## 3. 업로드 흐름 (로컬·S3 공통)

```
1. 프론트 → 백엔드   POST /api/attachments/presign  { filename, contentType, fileSize }
2. 백엔드            검증 후 attachment 레코드 생성(status=TEMP) + 업로드 URL 발급
                     ← { attachmentId, uploadUrl, storageKey }
3. 프론트 → uploadUrl  PUT (파일 본문, Content-Type 헤더 포함) — 아래 "클라이언트 구현 주의" 참고
4. 프론트 → 백엔드   POST /api/attachments/{id}/complete
5. 백엔드            verifyUploaded()로 실제 크기 확인 후 확정
                     ← { attachmentId, viewUrl }
```

**`uploadUrl`이 무엇을 가리키는가**

| | uploadUrl | viewUrl |
|---|---|---|
| 로컬 | `http://localhost:8080/api/attachments/{id}/upload` | `http://localhost:8080/api/attachments/{id}/raw?token=...` |
| S3 | S3 presigned PUT URL | S3 presigned GET URL (유효기간 30분) |

프론트는 **이 URL이 어디를 가리키는지 신경 쓰지 않고** PUT을 보낸다. 그래서 스토리지를 바꿔도 프론트 코드는 그대로다.

**클라이언트 구현 주의 (중요 — §7과 연결)**
- 이 PUT은 `lib/api/client.ts`의 공용 `apiFetch`를 **거치지 않는다.** `apiFetch`는 `Content-Type: application/json`을 강제하고 응답을 항상 `response.json()`으로 파싱해 바이너리 전송에 쓸 수 없다.
- **S3 presigned PUT에는 `Authorization` 헤더를 절대 붙이지 않는다.** presigned URL은 쿼리 파라미터 자체가 서명이라, 임의 헤더(특히 `Authorization`)를 추가하면 서명 불일치로 S3가 403을 반환한다. 로컬 업로드 엔드포인트는 반대로 JWT가 필요하므로, `uploadFile` 구현은 "로컬이면 헤더 첨부, S3면 첨부 안 함"을 분기해야 한다 — **단, 이 분기는 프론트가 스토리지 종류를 안다는 뜻이 아니라, 백엔드가 `presign` 응답에 `requiresAuth: boolean` 같은 플래그를 함께 내려줘서 프론트는 그 플래그만 본다.** (S3/로컬 여부를 프론트가 직접 판단하지 않는다는 원칙 유지)
- 업로드 진행률이 필요하면 `fetch`가 아니라 `XMLHttpRequest`(`upload.onprogress`)를 쓴다. `fetch`의 업로드 스트림에는 표준 진행률 이벤트가 없다.

**크기 검증 — presigned PUT은 크기를 강제하지 못한다**
S3 presigned URL은 키와 일부 헤더만 서명하므로, 클라이언트가 요청한 `fileSize`보다 훨씬 큰 파일을 얼마든지 올릴 수 있다. 따라서 5MB 상한은 **complete 단계의 `verifyUploaded()`(S3는 `HeadObject`, 로컬은 파일 시스템 크기 조회)가 유일한 강제 지점**이다. 초과 시 파일을 즉시 삭제하고 `FILE_001`(400)로 응답한다. (더 엄격하게 막으려면 S3 presigned **POST** policy의 `content-length-range` 조건을 쓰는 대안이 있으나, 이 문서는 구현이 단순한 presigned PUT + 사후 검증을 채택한다.)

S3 버킷은 퍼블릭으로 열지 않는다.

---

## 4. 로컬 스토리지 구현

### 저장 위치
`todo-project/upload` 디렉토리를 사용한다. `todo-backend`, `todo_frontend`와 형제 레벨이며 **어느 저장소에도 커밋하지 않는다.**

```
todo-project/
├── todo-backend/
├── todo_frontend/
└── upload/          ← 여기
```

- 경로는 설정값으로 주입한다 (하드코딩 금지)
- 애플리케이션 시작 시 디렉토리가 없으면 생성한다
- **무시 규칙은 루트 저장소 `.gitignore`에 `upload/`를 추가하는 것으로 처리한다.** 루트 저장소가 `docs/`·`CLAUDE.md`를 관리하고 `upload/`가 그 바로 아래(`todo-project/upload/`)에 생기기 때문이다. `todo-backend`/`todo_frontend` 각자의 `.gitignore`에 `../upload/` 같은 상위 경로를 적는 것은 **무효**하다 — git의 `.gitignore`는 자기 저장소 트리 밖의 경로를 대상으로 하지 않는다.

### 경로 조작 방어 (필수, 심층 방어)
`storageKey`로 상위 디렉토리 탈출(`../`)이 가능하면 서버 파일이 노출된다.
저장·조회 전에 최종 경로를 정규화(`Path.normalize()`)한 뒤 **base 디렉토리 하위인지 반드시 확인**하고, 벗어나면 예외를 던진다.
이 방어는 유지하되, **E2E로 검증하려 하지 않는다** — `storageKey`는 서버가 presign 단계에서 직접 생성하고 이후 API는 `attachmentId`만 받으므로, 클라이언트가 `../`를 주입할 입력 지점이 현재 API 설계에는 없다. 검증은 §8이 아니라 **백엔드 단위 테스트**(`StorageService` 구현체를 직접 호출)로 한다.

### 업로드 엔드포인트
`PUT /api/attachments/{id}/upload`
- 요청 본문을 그대로 파일로 기록 (`RequestBody`를 스트림으로 처리, 메모리 전체 적재 금지)
- 스트림을 기록하며 누적 바이트 수를 세어 상한(5MB)을 넘으면 **즉시 중단하고 부분 파일을 삭제, 400(`FILE_001`)** 으로 응답한다. `server.tomcat.max-swallow-size`는 이 목적에 쓸 수 없다 — 그 설정은 "서버가 읽지 않고 버릴 요청 본문의 최대치"를 뜻하며 업로드 크기 제한과 무관하다.
- `attachment.status`가 `TEMP`이고 요청자가 업로더 본인인지 확인
- 이미 파일이 존재하면 409(`FILE_004`)

### 조회 엔드포인트
`GET /api/attachments/{id}/raw`
- 소유자 검증 후 `Content-Type`과 함께 파일 스트림 반환
- `Content-Disposition: inline`
- **`X-Content-Type-Options: nosniff`를 반드시 함께 내려보낸다.** `inline` 응답은 브라우저가 MIME을 스니핑할 수 있어, 이 헤더 없이는 이미지로 위장한 파일이 실행 가능한 컨텐츠로 해석될 위험이 있다.
- 브라우저 `<img>` 태그가 직접 호출하므로 **JWT 헤더를 못 실어 보낸다.** 이 프로젝트의 `SecurityConfig`는 기본적으로 `anyRequest().authenticated()`이므로, `/api/attachments/*/raw`를 **`permitAll`에 추가**하고 대신 URL의 서명 토큰으로 인가한다:
  - `viewUrl` 발급 시 단기 유효(30분) 서명 토큰을 쿼리로 붙인다: `?token=...` — S3 presigned GET과 개념이 같아 전환 시 구조 변화가 없다
  - 토큰 검증은 컨트롤러/필터에서 별도로 수행(JWT 인증 체계와는 독립적인 서명 검증)

---

## 5. 백엔드 설정 (`application.properties`)

기존 파일에 아래 키를 **추가**한다. `application.yml`은 이 프로젝트에 없으므로 전환 작업은 필요 없다.

### `application.properties` (공통)
```properties
# 파일 업로드 제한 (스트림 처리 중 애플리케이션 레벨에서 강제 — 4장 참고)
app.upload.max-file-size=5242880
app.upload.allowed-content-types=image/jpeg,image/png,image/gif,image/webp
```

### `application-dev.properties` (기존 파일에 추가)
```properties
app.storage.type=local
app.storage.local.base-dir=${LOCAL_UPLOAD_DIR:../upload}
app.storage.local.base-url=${APP_BACKEND_URL:http://localhost:8080}
app.storage.url-expiry-minutes=30
```
> `spring.profiles.active`는 여기 넣지 않는다 — 이 프로젝트는 어떤 프로파일 파일에도 이 키를 두지 않고 실행 인자(`-Dspring-boot.run.profiles=dev`)로만 지정한다. `application-local.properties`를 새로 만들지 않는다 — 루트 `.gitignore`의 `**/application-local.properties` 규칙에 걸려 애초에 커밋되지 않는다.

### `application-prod.properties` (기존 파일에 추가)
```properties
app.storage.type=s3
app.storage.s3.bucket=${AWS_S3_BUCKET}
app.storage.s3.region=${AWS_REGION:ap-northeast-2}
app.storage.url-expiry-minutes=30
```

### 자격증명 원칙
**AWS 키를 코드나 설정 파일에 절대 넣지 않는다.**
- 로컬에서 S3를 시험할 때만 환경변수 `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` 사용
- EC2 배포 시에는 IAM Role 사용 (키 없이)
- 둘 다 `DefaultCredentialsProvider`가 자동 처리하므로 코드에 분기를 두지 않는다
- `application-prod.properties`는 이미 저장소에 커밋되어 있고 값은 전부 `${ENV}` 플레이스홀더다(CLAUDE.md 1.1) — 이 원칙을 그대로 유지한다

---

## 6. 백엔드 구현 목록

### 의존성
AWS SDK v2 (`software.amazon.awssdk:s3`) — BOM으로 버전 관리. **9장(S3 전환) 단계에서 추가한다.** Spring Boot 4.1.1 / Jackson 3 조합이므로 BOM 버전이 이 조합과 호환되는지 추가 전 확인한다(CLAUDE.md 5장 "추측 금지").

### 클래스 (실제 패키지 구조 기준)
- `com.example.domain.attachment.Attachment` (엔티티), `AttachmentStatus`, `StorageType` (enum), `AttachmentRepository`
- `com.example.storage.StorageService` (인터페이스)
- `com.example.storage.LocalStorageService` — `@ConditionalOnProperty(name="app.storage.type", havingValue="local")`
- `com.example.storage.S3StorageService` — `havingValue="s3"`
- `com.example.config.S3Config` — `S3Client`, `S3Presigner` 빈 (프로파일 조건부)
- `com.example.config.SchedulingConfig` — `@EnableScheduling` (현재 프로젝트에 없음, 신규 추가)
- `com.example.domain.attachment.AttachmentService` — 검증, 상태 전환, 권한 체크
- `com.example.domain.attachment.AttachmentController`

기존 컨벤션과 동일하게 컨트롤러는 얇게 유지하고, 예외는 `CustomException(ErrorCode)` 하나만 던진다 — `ResourceNotFoundException` 같은 세분화된 예외 클래스는 이 프로젝트에 없으므로 새로 만들지 않는다.

### API
| 메서드 | 경로 | 설명 |
|---|---|---|
| POST | `/api/attachments/presign` | 업로드 URL 발급 |
| PUT | `/api/attachments/{id}/upload` | **로컬 전용** 파일 수신 |
| POST | `/api/attachments/{id}/complete` | 업로드 완료 확정 |
| POST | `/api/attachments/urls` | 조회용 URL **벌크** 발급 (`{ids: number[]}` → `{urls: {id, viewUrl}[]}`) — 본문 하나에 이미지 N개가 있으면 N번 왕복하지 않도록 §7 조회와 짝을 맞춘다 |
| GET | `/api/attachments/{id}/raw?token=...` | **로컬 전용** 파일 스트림. `permitAll` + URL 서명 토큰으로 인가 |
| DELETE | `/api/attachments/{id}` | Soft Delete (실제 파일은 즉시 삭제하지 않음 — 아래 참고) |

> `ApiResponse<T>` 래핑 원칙(API_SPEC 1.1 "모든 응답은 예외 없이 래핑")의 **예외 두 곳**: `GET .../raw`(바이너리 스트림)과 S3 presigned PUT 응답(S3가 직접 응답하므로 우리 서버를 거치지 않음)은 래핑하지 않는다. API_SPEC 갱신 시 이 예외를 명시한다.

### 검증 규칙
- 로그인 사용자만 업로드 가능
- `contentType` 화이트리스트 검증(`image/jpeg,png,gif,webp` — **SVG는 제외**: SVG는 인라인 `<script>`를 담을 수 있어 저장형 XSS 벡터가 된다)
- 클라이언트가 보낸 `contentType`은 신뢰하지 않는다 — complete 단계에서 **파일 시그니처(매직 바이트)** 를 읽어 실제 형식과 일치하는지 재확인한다
- 파일 크기 상한 초과 시 400(`FILE_001`) — 강제 지점은 4장 참고
- 조회/삭제 시 `attachment.user_id`와 요청자 일치 확인 → **불일치 404(`FILE_003`)** — 불변 규칙 11(소유권 위반은 403이 아닌 404)을 그대로 따른다. API_SPEC 2.3의 `TODO_001`과 같은 이유

### 할일 저장 시 연결 처리
`content`는 HTML이 아니라 **Tiptap JSON**(이미 `JsonNode`로 컨트롤러에 들어옴, `TodoCreateRequest`/`TodoUpdateRequest` 참고)이므로, HTML 파싱이 아니라 **JSON 노드 트리 순회**로 구현한다. `TodoService`의 생성/수정 로직에 추가:

```java
// content는 Tiptap 문서 루트({"type":"doc","content":[...]})
private Set<Long> collectAttachmentIds(JsonNode content) {
    Set<Long> ids = new HashSet<>();
    if (content == null) return ids;
    collectRecursive(content, ids);
    return ids;
}

private void collectRecursive(JsonNode node, Set<Long> ids) {
    if (node.path("type").asText("").equals("image")) {
        JsonNode attachmentId = node.path("attrs").path("attachmentId");
        if (!attachmentId.isMissingNode() && !attachmentId.isNull()) {
            ids.add(attachmentId.asLong());
        }
    }
    for (JsonNode child : node.path("content")) {
        collectRecursive(child, ids);
    }
}
```

1. 저장될 `content`(`JsonNode`)에서 위 알고리즘으로 `attachmentId` 전부 수집
2. 해당 첨부를 `LINKED` + `todo_id` 설정
3. 기존에 연결돼 있었으나 본문에서 사라진 첨부는 Soft Delete 처리
4. 다른 사용자의 첨부 ID, 또는 **이미 다른 Todo에 `LINKED`된 첨부 ID**가 섞여 있으면 거부(1장 "같은 이미지가 여러 Todo에" 정책)

### Todo가 Soft Delete될 때
`TodoService.delete`가 Todo를 soft delete하면, 연결된 `LINKED` 첨부도 함께 soft delete한다(고아로 남기지 않는다).

### 삭제 정책 — Soft Delete와 실제 파일 삭제 시점을 분리한다
DELETE 엔드포인트는 **레코드만 soft delete**한다(`deleted_at` 설정). 파일을 그 자리에서 즉시 지우면 복구가 불가능해 Soft Delete를 두는 의미가 없어진다. 실제 파일 삭제는 아래 "고아 파일 정리" 배치가 **일정 유예 기간이 지난 soft-deleted 레코드**를 대상으로 물리 삭제하는 시점에 함께 처리한다.

### 고아 파일 정리
`@Scheduled` 배치 — 하루 1회, 아래 두 조건을 모두 물리 삭제(파일 + 레코드 hard delete)한다:
- `status = TEMP`이고 생성 후 24시간 지난 레코드 (업로드했지만 할일 저장으로 이어지지 않은 것)
- `deleted_at IS NOT NULL`이고 삭제 후 유예 기간(예: 7일)이 지난 레코드 (위 "삭제 정책" 참고)

이 배치가 동작하려면 **`@EnableScheduling`을 등록해야 한다** — 현재 프로젝트에는 없다(`SecurityConfig`·`CorsConfig`·`JpaAuditingConfig` 3개만 `@Configuration`으로 등록돼 있음). `com.example.config.SchedulingConfig`를 신설하거나 애플리케이션 클래스에 애노테이션을 추가한다. (현재 EC2 단일 인스턴스 배포이므로 다중 인스턴스 중복 실행 문제는 없음 — 배포 구성이 바뀌면 재검토)

### 신규 에러코드 (API_SPEC 2장 `{도메인}_{일련번호}` 규약)
| 코드 | HTTP | 의미 |
|---|---|---|
| `FILE_001` | 400 | 파일 크기 초과 |
| `FILE_002` | 400 | 허용되지 않는 파일 형식 |
| `FILE_003` | 404 | 첨부를 찾을 수 없음 **(타인 소유인 경우 포함)** |
| `FILE_004` | 409 | 이미 업로드된 첨부 (재업로드 시도) |
| `FILE_005` | 400 | 업로드가 완료되지 않은 첨부 (`complete` 호출 전 상태로 연결 시도) |

### SecurityConfig 변경
현재 `SecurityConfig`는 `/api/auth/signup`, `/api/auth/login`, `/oauth2/**`, `/login/oauth2/**`만 `permitAll`이고 나머지는 `anyRequest().authenticated()`다. `<img src>`가 직접 부르는 `/api/attachments/*/raw`는 브라우저가 `Authorization` 헤더를 실을 수 없으므로 **이 경로를 `permitAll`에 추가**하고, 대신 4장에서 정의한 URL 서명 토큰으로 컨트롤러 내부에서 인가한다.

---

## 7. 프론트엔드 (`todo_frontend`)

### Tiptap 설정
- **`@tiptap/extension-image`를 새로 설치**해야 한다(현재 `TiptapEditor.tsx`는 `extensions: [StarterKit]`뿐이라 이미지 노드가 없음). 기존 `@tiptap/*` 계열과 버전(`3.30.6`)을 맞추고, 설치 전 `peerDependencies`가 React 19와 호환되는지 npm 레지스트리에서 직접 확인한다(Task 028이 밟았던 절차 그대로).
- 기본 `Image` extension을 확장한 커스텀 노드를 만든다. `src` 외에 **`attachmentId` 속성을 추가**한다. 값은 Tiptap **JSON**의 `attrs.attachmentId`로 저장된다(HTML `data-attachment-id` 직렬화가 아니다 — 이 프로젝트는 `getHTML()`을 쓰지 않는다).
  → 조회 URL은 만료되므로 JSON에 URL을 박아두면 안 된다. ID를 저장하고 조회 시점에 URL을 다시 받는다.
- 툴바 이미지 버튼, 붙여넣기(paste), 드래그앤드롭 세 경로 모두 지원

### 업로드 UX
1. 파일 선택 즉시 로컬 blob URL로 **임시 미리보기** 노드 삽입 (업로드 중 표시)
2. presign → PUT → complete 순차 호출. **PUT은 `apiFetch`를 쓰지 않는다** — `XMLHttpRequest` 또는 별도 `fetch` 호출로 직접 구현하고, `presign` 응답이 알려주는 대로 헤더를 구성한다(3장 "클라이언트 구현 주의" 참고: S3 presigned PUT에는 `Authorization` 헤더를 붙이지 않는다). 진행률 표시가 필요하면 `XMLHttpRequest.upload.onprogress`를 쓴다
3. 성공 시 노드의 `src`를 실제 URL로, `attachmentId`를 채워 교체
4. 실패 시 노드 제거 + 토스트로 에러 안내

### 조회
Todo를 불러올 때(즉, `TodoForm`에 `value`로 넘기기 전) 본문(Tiptap JSON)에서 `attachmentId` 목록을 모아 **`POST /api/attachments/urls`로 한 번에** 조회용 URL을 받고, 에디터에 주입하기 전에 각 이미지 노드의 `src`를 채운다. 이 프로젝트에는 별도의 읽기 전용 본문 뷰가 없다 — `app/(main)/todos/[id]/page.tsx`는 곧바로 `TodoForm`(편집 폼)을 렌더하므로, 이미지는 **에디터 안에서만** 표시된다. 나중에 읽기 전용 뷰를 추가한다면 그때 HTML 렌더링 여부와 sanitize(DOMPurify 등, 현재 미설치) 정책을 함께 정한다.

### 클라이언트 검증
업로드 전 파일 타입/크기를 미리 확인해 불필요한 요청을 막는다. (서버 검증은 그대로 유지 — 클라이언트 검증은 우회 가능하므로 신뢰하지 않는다)

### API 함수
`lib/api/attachments.ts`에 `presignUpload`, `uploadFile`, `completeUpload`, `getViewUrls`(벌크), `deleteAttachment`를 추가하고 React Query 뮤테이션으로 감싼다. 이 프로젝트는 mutation 전용 훅 파일을 따로 두지 않고 페이지/컴포넌트에 인라인으로 작성하는 패턴(`useTodos.ts`는 쿼리 훅만 있고 mutation은 각 페이지에 있음)이므로 그 컨벤션을 따른다.
`uploadFile`은 서버가 준 `uploadUrl`로 PUT을 보낼 뿐, **로컬인지 S3인지 판단하는 로직을 두지 않는다** — `presign` 응답의 플래그(3장 참고)만 본다.

---

## 8. 로컬 테스트 시나리오

구현 후 아래를 직접 확인하고 결과를 보고한다.

1. 서버 기동 시 `todo-project/upload` 디렉토리가 자동 생성되는가
2. 이미지 첨부 → `upload/todos/{userId}/...` 경로에 파일이 실제로 생기는가
3. `attachment` 테이블에 `status=TEMP`로 행이 생기는가
4. 할일 저장 후 `status=LINKED`, `todo_id`가 채워지는가 (Tiptap JSON의 `attrs.attachmentId` 기준)
5. 저장된 할일을 다시 열었을 때 이미지가 정상 표시되는가 (에디터 내부)
6. 본문에서 이미지를 지우고 저장하면 해당 첨부가 Soft Delete 되는가
7. 5MB 초과 파일, `.exe` 파일 업로드가 거부되는가 (크기는 complete 단계에서, 형식은 매직 바이트 검사로)
8. 다른 사용자의 `attachmentId`로 조회 시 **404**가 나는가 (403 아님 — 불변 규칙 11)
9. `upload/` 폴더가 (루트 저장소 기준) git status에 잡히지 않는가
10. Todo를 삭제하면 연결된 첨부도 함께 Soft Delete 되는가
11. `/api/attachments/{id}/raw?token=...`를 로그인하지 않은 브라우저 탭에서 직접 열어도(서명 토큰만으로) 이미지가 보이는가

> `storageKey` 경로 조작(`../`) 방어는 API 설계상 클라이언트가 주입할 지점이 없어 E2E로 재현할 수 없다. 이 항목은 §4가 명시한 대로 **백엔드 단위 테스트**로 검증한다.

---

## 9. S3 전환 시 추가 작업

- `S3StorageService` 구현 + AWS SDK 의존성 추가
- `application-prod.properties` 프로파일로 기동해 §8의 시나리오를 재확인 (단, 9번 경로 조작 항목은 여전히 단위 테스트 영역)
- 프론트 코드는 **변경 없어야 한다.** 수정이 필요하다면 추상화가 잘못된 것이므로 보고할 것
- S3 presigned PUT은 크기를 강제하지 못하므로, complete 단계의 `HeadObject` 크기 검증(3장)이 로컬 전환 때보다 더 중요해진다 — 로컬 테스트에서 이 경로가 실제로 초과분을 거부하는지 반드시 재확인한다

### AWS 콘솔 설정 (별도 안내 필요)
- 버킷 **CORS 설정** — presigned PUT을 위해 `PUT`, `GET`, `HEAD` 허용 / `AllowedOrigin`에 로컬·Amplify 도메인
- 퍼블릭 액세스 차단은 **켠 상태 유지**
- IAM 정책은 해당 버킷에 대한 `s3:PutObject`, `s3:GetObject`, `s3:DeleteObject`, `s3:HeadObject`만 허용 (최소 권한)

---

## 10. 진행 순서

각 단계가 끝나면 멈추고 확인받는다.

1. DB 마이그레이션(`ddl-auto`) 확인 + `Attachment` 엔티티/리포지토리
2. `StorageService` 인터페이스 + `LocalStorageService` + `AttachmentService`/`AttachmentController` + `SchedulingConfig`
3. `TodoService` 연결 로직(JSON 노드 순회) + 고아 파일 정리 배치
4. 프론트 API 함수 + 커스텀 Tiptap 이미지 노드 + 업로드 UX
5. **로컬 통합 테스트 (8장 시나리오 전체 + 경로 조작 단위 테스트)**
6. `S3StorageService` 구현 및 전환 검증
7. **문서 갱신** — 아래 목록을 실제 코드와 함께 반영한다

### 7단계에서 함께 갱신할 문서
- `docs/PRD.md` 4.5 / 13.1 — "첨부파일·S3는 MVP 제외" 결정을 뒤집는 개정
- `docs/PRD.md` 8.4 — "서버가 본문을 해석할 필요가 없다"는 근거 문장 개정(연결 로직이 이제 `content`를 순회하므로)
- `docs/PRD.md` 8.2 — `attachment` 테이블 정의 추가
- `docs/API_SPEC.md` 2장 — `FILE_xxx` 에러코드, 1.1의 `ApiResponse` 래핑 예외(§6 참고), 신규 첨부 API 절
- `docs/ROADMAP.md` — 새 마일스톤(M10) + Task 신설(현재 Task 039까지 존재)

---

## 규칙

- 코드 주석은 한글로 작성
- 기존 Soft Delete / 페이지네이션 / 예외 처리(`CustomException`+`ErrorCode`) 컨벤션을 그대로 따른다
- 자격증명은 어떤 형태로도 소스에 남기지 않는다
- 설정 파일은 `.properties` 형식만 사용한다
- 본문(`content`) 파싱은 **JSON 노드 트리 순회만** 사용한다. 정규식이나 HTML 파서로 처리하지 않는다 — 이 프로젝트에 HTML 문자열 단계가 존재하지 않는다
