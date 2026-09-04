# API_SPEC — Todo List 백엔드 API 상세 명세

> **문서 목적**: [PRD.md](./PRD.md) 7장의 엔드포인트 목록에 대한 **요청/응답 바디, 상태코드, 에러코드**를 정의한다.
> 프론트엔드는 이 문서를 계약(contract)으로 삼아 `lib/api/` 클라이언트를 구현하고, 백엔드는 이 포맷대로 응답한다.

- **Base URL**: `{NEXT_PUBLIC_API_BASE_URL}` (예: `http://localhost:8080`)
- **Content-Type**: `application/json; charset=UTF-8`
- **인증 헤더**: `Authorization: Bearer {accessToken}`

---

## 1. 공통 응답 포맷

### 1.1 `ApiResponse<T>`

모든 응답은 예외 없이 이 래퍼로 감싼다.

| 필드 | 타입 | 설명 |
|------|------|------|
| `success` | boolean | 성공 여부 |
| `data` | T \| `Record<string,string>` \| null | 성공 시 페이로드. 실패 시 `null` — **단 검증 실패(`COMMON_001`)에 한해 필드별 에러 맵**(1.3 참조) |
| `message` | string \| null | 사용자에게 보여줄 메시지 |
| `errorCode` | string \| null | 실패 시 에러코드 (2장 참조). 성공 시 `null` |

**성공 응답**

```json
{
  "success": true,
  "data": { "id": 1, "title": "장보기" },
  "message": null,
  "errorCode": null
}
```

**실패 응답**

```json
{
  "success": false,
  "data": null,
  "message": "이미 사용 중인 이메일입니다.",
  "errorCode": "AUTH_002"
}
```

> ⚠️ **`data`의 실패 시 타입은 두 가지다.** 검증 실패(`COMMON_001`)만 필드 에러 맵을 담고, 나머지 실패는 모두 `null`이다.
> 프론트 타입은 이렇게 분기한다:
>
> ```ts
> type ApiResponse<T> = {
>   success: boolean
>   data: T | null
>   message: string | null
>   errorCode: string | null
> }
> // 검증 실패 전용
> type ValidationErrorResponse = ApiResponse<Record<string, string>> & {
>   errorCode: 'COMMON_001'
> }
> ```

> ⚠️ **래핑 예외 2건 (M10 첨부파일, 7장)**: `GET /api/attachments/{id}/raw`(바이너리 이미지 스트림)과 S3 presigned PUT 응답(S3가 직접 응답하며 우리 서버를 거치지 않음)은 `ApiResponse`로 감싸지 않는다.

### 1.2 `PageResponse<T>`

목록 조회 시 `ApiResponse.data`에 담기는 페이지네이션 래퍼.

| 필드 | 타입 | 설명 |
|------|------|------|
| `content` | T[] | 현재 페이지의 항목 배열 |
| `page` | number | 현재 페이지 번호 (0부터 시작) |
| `size` | number | 페이지 크기 |
| `totalElements` | number | 전체 항목 수 (Soft Delete 제외) |
| `totalPages` | number | 전체 페이지 수 |
| `hasNext` | boolean | 다음 페이지 존재 여부 |

```json
{
  "success": true,
  "data": {
    "content": [],
    "page": 0,
    "size": 10,
    "totalElements": 0,
    "totalPages": 0,
    "hasNext": false
  },
  "message": null,
  "errorCode": null
}
```

### 1.3 검증 실패 응답 (Bean Validation)

`@Valid` 검증에 실패하면 `data`에 **필드별 에러 맵**을 담는다. 프론트는 이 맵을 React Hook Form의 필드 에러로 매핑한다.

```json
{
  "success": false,
  "data": {
    "email": "올바른 이메일 형식이 아닙니다",
    "password": "비밀번호는 최소 6자 이상이어야 합니다"
  },
  "message": "입력값을 확인해주세요.",
  "errorCode": "COMMON_001"
}
```

---

## 2. 에러코드 체계

`{도메인}_{일련번호}` 형식. 백엔드는 `common/exception/ErrorCode.java` enum으로 관리한다.

### 2.1 공통

| 코드 | HTTP | 의미 |
|------|------|------|
| `COMMON_001` | 400 | 입력값 검증 실패 |
| `COMMON_002` | 400 | 잘못된 요청 파라미터 |
| `COMMON_500` | 500 | 서버 내부 오류 |

### 2.2 인증

| 코드 | HTTP | 의미 |
|------|------|------|
| `AUTH_001` | 401 | 이메일 또는 비밀번호가 일치하지 않음 |
| `AUTH_002` | 409 | 이미 사용 중인 이메일 |
| `AUTH_003` | 401 | 토큰이 없거나 형식이 잘못됨 |
| `AUTH_004` | 401 | 만료된 토큰 |
| `AUTH_005` | 401 | 위조된 토큰 (서명 불일치) |
| `AUTH_006` | 404 | 사용자를 찾을 수 없음 |
| `AUTH_007` | 400 | 소셜 로그인 처리 실패 |

> **참고**: 로그인 실패 시 "이메일이 없음"과 "비밀번호 불일치"를 구분하지 않고 모두 `AUTH_001`로 응답한다. 가입된 이메일 목록이 열거되는 것을 막기 위함이다.

> ⚠️ **`AUTH_003`~`AUTH_005`(401)는 컨트롤러가 아니라 보안 필터 단계에서 발생한다.**
> 필터는 DispatcherServlet 바깥이라 `GlobalExceptionHandler`가 잡지 못한다. 이 계약을 지키려면 백엔드가 `JwtAuthenticationEntryPoint`를 등록해 아래 형태를 **직접 써서 응답해야 한다.**
>
> ```json
> {
>   "success": false,
>   "data": null,
>   "message": "만료된 토큰입니다.",
>   "errorCode": "AUTH_004"
> }
> ```

### 2.3 Todo

| 코드 | HTTP | 의미 |
|------|------|------|
| `TODO_001` | 404 | Todo를 찾을 수 없음 **(타인 소유인 경우 포함)** |
| `TODO_002` | 400 | 잘못된 상태값 (`TODO`/`DONE` 외) |

> **소유권 위반은 403이 아닌 404로 응답한다.** 403은 "그 리소스는 존재하지만 네 것이 아니다"를 알려주므로, ID를 순회하며 타인 리소스의 존재 여부를 열거할 수 있게 된다. PRD 10장 보안 요구사항 참조.

### 2.4 첨부파일 (M10)

| 코드 | HTTP | 의미 |
|------|------|------|
| `FILE_001` | 400 | 파일 크기가 허용 범위(5MB) 초과 |
| `FILE_002` | 400 | 허용되지 않는 파일 형식 (jpg/png/gif/webp 외, SVG 포함) |
| `FILE_003` | 404 | 첨부파일을 찾을 수 없음 **(타인 소유인 경우 포함)** |
| `FILE_004` | 409 | 이미 처리된 첨부파일 (재업로드 시도) |
| `FILE_005` | 400 | 업로드가 완료되지 않음 (`complete` 호출 전 상태로 Todo에 연결 시도) |

> 소유권 위반은 여기서도 403이 아닌 404(`FILE_003`)로 응답한다 — 2.3과 동일한 이유(PRD 10장).

---

## 3. 인증 API

### 3.1 회원가입 — `POST /api/auth/signup`

> 기능 ID `AUTH-01` · 인증 불필요

**Request**

```json
{
  "email": "user@example.com",
  "password": "abc123",
  "name": "홍길동"
}
```

| 필드 | 타입 | 필수 | 검증 |
|------|------|------|------|
| `email` | string | ✅ | `@Email`, 중복 불가 |
| `password` | string | ✅ | `@Size(min = 6)` — **6자 이상이면 통과. 추가 복잡도 규칙 없음** |
| `name` | string | ❌ | 최대 100자 |

**Response** — `201 Created`

```json
{
  "success": true,
  "data": {
    "id": 1,
    "email": "user@example.com",
    "name": "홍길동",
    "provider": "LOCAL",
    "createdAt": "2026-08-21T10:00:00"
  },
  "message": "회원가입이 완료되었습니다.",
  "errorCode": null
}
```

**에러**: `COMMON_001`(400 검증 실패), `AUTH_002`(409 이메일 중복)

---

### 3.2 로그인 — `POST /api/auth/login`

> 기능 ID `AUTH-02` · 인증 불필요

**Request**

```json
{
  "email": "user@example.com",
  "password": "abc123"
}
```

**Response** — `200 OK`

```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiJ9...",
    "tokenType": "Bearer",
    "expiresIn": 86400000,
    "user": {
      "id": 1,
      "email": "user@example.com",
      "name": "홍길동",
      "provider": "LOCAL"
    }
  },
  "message": null,
  "errorCode": null
}
```

| 필드 | 설명 |
|------|------|
| `accessToken` | JWT Access Token |
| `tokenType` | 고정값 `Bearer` |
| `expiresIn` | 만료까지 남은 밀리초. **86400000 = 24시간** |

**에러**: `COMMON_001`(400), `AUTH_001`(401 자격 불일치)

---

### 3.3 내 정보 조회 — `GET /api/auth/me`

> 기능 ID `AUTH-04` · **인증 필요**

**Request**: 바디 없음. `Authorization: Bearer {token}` 헤더 필수.

**Response** — `200 OK`

```json
{
  "success": true,
  "data": {
    "id": 1,
    "email": "user@example.com",
    "name": "홍길동",
    "provider": "LOCAL",
    "createdAt": "2026-08-21T10:00:00"
  },
  "message": null,
  "errorCode": null
}
```

**에러**: `AUTH_003`/`AUTH_004`/`AUTH_005`(401), `AUTH_006`(404)

---

### 3.4 소셜 로그인 시작 — `GET /oauth2/authorization/{provider}`

> 기능 ID `AUTH-03` · 인증 불필요 · `provider` = `google` \| `kakao`

브라우저를 이 URL로 **직접 이동**시킨다 (fetch/XHR 아님). Spring Security가 OAuth2 제공자로 리다이렉트한다.

```
window.location.href = `${API_BASE_URL}/oauth2/authorization/google`
```

### 3.5 소셜 콜백 — `GET /login/oauth2/code/{provider}`

> 기능 ID `AUTH-03` · Spring Security가 내부 처리하므로 **프론트가 직접 호출하지 않는다.**

처리 흐름:

1. 제공자가 authorization code와 함께 이 URL로 리다이렉트
2. `CustomOAuth2UserService`가 사용자 정보 조회
   - 신규 이메일 → 자동 가입 (`provider` = `GOOGLE`/`KAKAO`)
   - 기존 이메일 → 해당 계정에 연동
3. `OAuth2SuccessHandler`가 JWT 발급 후 프론트로 리다이렉트

토큰은 **쿼리스트링이 아니라 URL 프래그먼트**로 전달한다.

```
{APP_FRONTEND_URL}/oauth2/callback#token={accessToken}
```

실패 시:

```
{APP_FRONTEND_URL}/oauth2/callback#error=AUTH_007
```

> **왜 프래그먼트인가**: `#` 뒤의 값은 브라우저가 서버로 전송하지 않는다. 따라서 `Referer` 헤더, 프록시·CDN 액세스 로그, 서버 로그 어디에도 토큰이 남지 않는다. 쿼리스트링은 이 세 곳 모두에 24시간 유효 토큰을 기록한다.
>
> **프론트 처리 순서** (`app/oauth2/callback/page.tsx`):
> 1. `window.location.hash`에서 `token` 또는 `error` 추출
> 2. 토큰을 저장
> 3. `history.replaceState(null, '', '/oauth2/callback')` 로 **URL에서 프래그먼트 제거**
> 4. Todo 목록으로 이동
>
> 이 페이지에서는 외부 리소스(폰트·이미지·분석 스크립트)를 로드하지 않는다.

### 3.6 로그아웃 (AUTH-05) — **API 없음**

JWT는 서버에 상태를 두지 않으므로 별도 엔드포인트가 없다. 클라이언트가 저장된 토큰을 삭제하고 로그인 페이지로 이동하는 것으로 완료된다.

---

## 4. Todo API

모든 엔드포인트는 **인증 필요**이며, 서버가 `user_id`와 인증 주체의 일치를 검증한다. 불일치 시 `TODO_001`(404).

### 4.1 공통 `TodoResponse` 구조

| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | number | Todo ID |
| `title` | string | 제목 |
| `content` | object \| null | **Tiptap JSON 문서** (JSONB 컬럼) |
| `status` | string | `TODO` \| `DONE` |
| `dueDate` | string \| null | `YYYY-MM-DD` |
| `createdAt` | string | ISO-8601 |
| `updatedAt` | string | ISO-8601 |

`content`는 Tiptap이 생성하는 문서 객체를 **JSON 객체 그대로** 주고받는다 (문자열로 감싸지 않는다).
백엔드는 이 값을 `jsonb` 컬럼에 저장하며 내용을 해석하지 않는다 (PRD 8.4).

```json
{
  "type": "doc",
  "content": [
    {
      "type": "paragraph",
      "content": [{ "type": "text", "text": "우유와 계란 사기" }]
    }
  ]
}
```

---

### 4.2 목록 조회 — `GET /api/todos`

> 기능 ID `TODO-01`

**Query Parameters**

| 파라미터 | 타입 | 기본값 | 설명 |
|----------|------|--------|------|
| `page` | number | `0` | 0부터 시작 |
| `size` | number | `10` | **최대 100** — 초과 시 100으로 절삭 |
| `status` | string | (없음) | `TODO` \| `DONE`. 미지정 시 전체 |
| `keyword` | string | (없음) | **`title` 부분 일치** (대소문자 무시). 본문 검색 미지원 |

정렬은 `created_at DESC` 고정이다 (MVP 범위).

```
GET /api/todos?page=0&size=10&status=TODO&keyword=장보기
```

**Response** — `200 OK`

```json
{
  "success": true,
  "data": {
    "content": [
      {
        "id": 12,
        "title": "장보기",
        "content": { "type": "doc", "content": [] },
        "status": "TODO",
        "dueDate": "2026-08-25",
        "createdAt": "2026-08-21T10:00:00",
        "updatedAt": "2026-08-21T10:00:00"
      }
    ],
    "page": 0,
    "size": 10,
    "totalElements": 1,
    "totalPages": 1,
    "hasNext": false
  },
  "message": null,
  "errorCode": null
}
```

> 결과가 없으면 `content: []`, `totalElements: 0`을 반환한다. **404가 아니다.** 프론트는 빈 상태 UI를 렌더한다.

**에러**: `COMMON_002`(400 잘못된 `status` 값), `AUTH_003`~`AUTH_005`(401)

---

### 4.3 상세 조회 — `GET /api/todos/{id}`

> 기능 ID `TODO-02`

**Response** — `200 OK` — `data`에 `TodoResponse` 단건.

**에러**: `TODO_001`(404 — 없거나 타인 소유), `AUTH_003`~`AUTH_005`(401)

---

### 4.4 생성 — `POST /api/todos`

> 기능 ID `TODO-03`

**Request**

```json
{
  "title": "장보기",
  "content": { "type": "doc", "content": [] },
  "dueDate": "2026-08-25"
}
```

| 필드 | 타입 | 필수 | 검증 |
|------|------|------|------|
| `title` | string | ✅ | `@NotBlank`, `@Size(max = 255)` |
| `content` | object | ❌ | Tiptap JSON |
| `dueDate` | string | ❌ | `YYYY-MM-DD` |

`status`는 요청에 포함하지 않는다. 생성 시 항상 `TODO`로 시작한다.

**Response** — `201 Created` — 생성된 `TodoResponse`.

**에러**: `COMMON_001`(400)

---

### 4.5 수정 — `PUT /api/todos/{id}`

> 기능 ID `TODO-04`

**Request** — 전체 교체(PUT).

```json
{
  "title": "장보기 (수정)",
  "content": { "type": "doc", "content": [] },
  "dueDate": "2026-08-26",
  "status": "DONE"
}
```

| 필드 | 타입 | 필수 | 생략 시 | 검증 |
|------|------|------|---------|------|
| `title` | string | ✅ | **400 에러** | `@NotBlank`, `@Size(max = 255)` |
| `status` | string | ✅ | **400 에러** | `@NotNull`, `TODO` \| `DONE` |
| `content` | object | ❌ | `null`로 갱신 | Tiptap JSON |
| `dueDate` | string | ❌ | `null`로 갱신 | `YYYY-MM-DD` |

> ⚠️ **`title`과 `status`는 생략할 수 없다.** DB에서 두 컬럼이 `NOT NULL`이므로(PRD 8.2), "생략 시 `null` 갱신"을 적용하면 제약 위반으로 500이 난다.
> 생략 시 `null`이 되는 것은 **nullable 컬럼인 `content`·`dueDate`뿐**이다.
> 클라이언트는 PUT 전에 현재 값을 조회했을 것이므로(상세/편집 페이지), 두 필수 필드를 항상 채워 보낼 수 있다.

**Response** — `200 OK` — 수정된 `TodoResponse`.

**에러**: `COMMON_001`(400), `TODO_001`(404), `TODO_002`(400 잘못된 상태값)

---

### 4.6 상태 변경 — `PATCH /api/todos/{id}/status`

> 기능 ID `TODO-05`

**Request** — 클라이언트가 **목표 상태를 지정**한다. 서버는 현재 값을 반전시키지 않는다.

```json
{ "status": "DONE" }
```

| 필드 | 타입 | 필수 | 검증 |
|------|------|------|------|
| `status` | string | ✅ | `@NotNull`, `TODO` \| `DONE` |

> ⚠️ **바디 없는 "서버 반전 토글"을 쓰지 않는 이유 — 멱등성.**
> 반전 방식은 같은 요청을 두 번 보내면 결과가 원래대로 돌아온다. React Query 재시도, 네트워크 타임아웃 후 재전송, 사용자의 더블클릭이 모두 **의도치 않은 상태 되돌림**을 일으킨다.
> 목표 상태를 명시하면 몇 번을 보내도 결과가 같다(멱등). **UI의 토글 경험은 그대로**다 — 프론트가 현재 값의 반대를 실어 보내면 된다.

**Response** — `200 OK` — **전체 `TodoResponse`**를 반환한다 (다른 엔드포인트와 계약 통일).

```json
{
  "success": true,
  "data": {
    "id": 12,
    "title": "장보기",
    "content": { "type": "doc", "content": [] },
    "status": "DONE",
    "dueDate": "2026-08-25",
    "createdAt": "2026-08-21T10:00:00",
    "updatedAt": "2026-08-21T11:30:00"
  },
  "message": null,
  "errorCode": null
}
```

> 부분 응답(`{id, status}`)이 아니라 전체 `TodoResponse`를 돌려주는 이유는, 프론트가 낙관적 업데이트 후 **`updatedAt`까지 포함해 캐시를 정합**시킬 수 있게 하기 위함이다.

**에러**: `COMMON_001`(400 `status` 누락), `TODO_002`(400 잘못된 상태값), `TODO_001`(404)

---

### 4.7 삭제 — `DELETE /api/todos/{id}`

> 기능 ID `TODO-06` · **Soft Delete**

물리 삭제하지 않고 `deleted_at`에 삭제 시각을 기록한다. 이후 모든 조회에서 제외된다.

**Response** — `200 OK`

```json
{
  "success": true,
  "data": null,
  "message": "삭제되었습니다.",
  "errorCode": null
}
```

**에러**: `TODO_001`(404)

---

## 5. HTTP 상태코드 정리

| 코드 | 사용 상황 |
|------|-----------|
| `200 OK` | 조회·수정·삭제·토글 성공 |
| `201 Created` | 회원가입, Todo 생성 성공 |
| `400 Bad Request` | 검증 실패, 잘못된 파라미터 |
| `401 Unauthorized` | 미인증, 토큰 만료/위조, 로그인 자격 불일치 |
| `404 Not Found` | 리소스 없음 **또는 타인 소유** |
| `409 Conflict` | 이메일 중복 |
| `500 Internal Server Error` | 서버 오류 |

> **403 Forbidden은 사용하지 않는다.** 소유권 위반은 404로 처리한다 (2.3 참조).

---

## 6. 프론트엔드 연동 규칙

- 모든 API 호출은 **`lib/api/client.ts`를 경유**한다. 컴포넌트에서 `fetch`를 직접 호출하지 않는다.
- **토큰은 `localStorage`에 저장한다** (PRD 13.1 확정). 클라이언트가 JWT를 `Authorization` 헤더에 **자동 첨부**한다.
- **401 응답을 받으면** 저장된 토큰을 삭제하고 로그인 페이지로 리다이렉트한다.
  - ⚠️ 로그인·회원가입 페이지에서 받은 401은 리다이렉트하지 않는다 (무한 루프 방지).
- 서버 상태는 **React Query**로 관리한다 (`hooks/useTodos.ts`).
- **목록 조회의 `page`·`status`·`keyword`는 URL 쿼리스트링을 단일 출처로 삼는다** (PRD 6.4). React Query의 쿼리 키도 이 값들로 구성해 URL과 캐시를 일치시킨다.
  - `useSearchParams()`를 쓰므로 해당 영역은 **`<Suspense>` 래핑이 필수**다 (Next.js 16).
- 검증 실패(`COMMON_001`) 응답의 `data` 맵을 React Hook Form의 필드 에러로 매핑한다.
- **첨부파일 업로드 PUT은 `apiFetch`를 거치지 않는다** (M10) — `apiFetch`는 `Content-Type: application/json`을 강제하고 응답을 항상 JSON으로 파싱해 바이너리 전송에 쓸 수 없다. `lib/api/attachments.ts`의 `uploadFile`(`XMLHttpRequest`)을 대신 사용한다. 7장 참조.

> ⚠️ **인증·CRUD 로직을 Next.js Server Actions에서 직접 구현하지 않는다.** 비밀번호 해싱·사용자 생성·토큰 발급은 전부 Spring Boot 백엔드의 책임이다.

---

## 7. 첨부파일 API (M10)

> 기능 ID `FILE-01` (PRD 4.6) · 상세 설계·구현 순서는 [docs/guides/tiptap-s3-image-upload-prompt.md](./guides/tiptap-s3-image-upload-prompt.md) 참조.
> Todo 본문(Tiptap JSON)에 이미지를 삽입하기 위한 업로드·조회 API. 파일 실체는 스토리지(로컬 또는 S3)에, 메타데이터는 `attachment` 테이블(PRD 8.2)에 저장한다.

### 7.1 업로드 URL 발급 — `POST /api/attachments/presign`

**Request**

```json
{ "filename": "photo.png", "contentType": "image/png", "fileSize": 102400 }
```

| 필드 | 타입 | 필수 | 검증 |
|------|------|------|------|
| `filename` | string | ✅ | 표시용 원본 파일명 (저장 키의 확장자 소스로 쓰지 않음) |
| `contentType` | string | ✅ | 화이트리스트: `image/jpeg` \| `image/png` \| `image/gif` \| `image/webp` |
| `fileSize` | number | ✅ | 5MB(5242880) 이하로 선언되어야 함 — **선언값일 뿐**, 실제 강제는 업로드 단계에서 스트림으로 재확인한다(presigned PUT은 크기를 강제하지 못함) |

**Response** — `201 Created`

```json
{
  "success": true,
  "data": { "attachmentId": 1, "uploadUrl": "http://localhost:8080/api/attachments/1/upload", "requiresAuthHeader": true },
  "message": null,
  "errorCode": null
}
```

> `requiresAuthHeader` — 로컬 스토리지는 `true`(우리 서버의 인증 엔드포인트라 `Authorization` 헤더 필요), S3는 `false`(presigned URL 자체가 서명이라 임의 헤더를 추가하면 서명 불일치로 403). **프론트는 로컬/S3 여부를 직접 판단하지 않고 이 플래그만 본다.**

**에러**: `COMMON_001`(400), `FILE_001`(400 선언 크기 초과), `FILE_002`(400 형식 불허)

---

### 7.2 파일 업로드 — `PUT /api/attachments/{id}/upload` *(로컬 전용)*

요청 본문을 스트림으로 그대로 저장한다. S3 모드에서는 7.1의 `uploadUrl`이 S3 presigned PUT URL이므로 이 엔드포인트는 호출되지 않는다.

- 누적 바이트 수가 5MB를 넘으면 **즉시 중단, 부분 파일 삭제, 400(`FILE_001`)**
- 이미 업로드된 첨부(`status != TEMP`)면 `409`(`FILE_004`)

**Response** — `200 OK` (바디 없음, `ApiResponse` 아님)

**에러**: `FILE_001`(400), `FILE_003`(404 소유자 불일치), `FILE_004`(409)

---

### 7.3 업로드 완료 확정 — `POST /api/attachments/{id}/complete`

`verifyUploaded()`로 실제 파일 크기를 재확인하고(S3는 `HeadObject`, 로컬은 파일시스템 조회), 파일 시그니처(매직 바이트)로 `contentType`을 재검증한다 — 클라이언트가 보낸 값은 신뢰하지 않는다.

**Response** — `200 OK`

```json
{
  "success": true,
  "data": { "attachmentId": 1, "viewUrl": "http://localhost:8080/api/attachments/1/raw?token=..." },
  "message": null,
  "errorCode": null
}
```

**에러**: `FILE_001`(400 재확인 결과 크기 초과), `FILE_002`(400 매직 바이트 불일치), `FILE_003`(404), `FILE_005`(400 업로드 미완료)

---

### 7.4 조회 URL 벌크 발급 — `POST /api/attachments/urls`

Todo 본문 하나에 이미지가 여러 개 있어도 한 번에 조회한다 — 이미지 개수만큼 왕복하지 않기 위함이다.

**Request**

```json
{ "ids": [1, 2, 3] }
```

**Response** — `200 OK`

```json
{
  "success": true,
  "data": { "urls": [ { "attachmentId": 1, "viewUrl": "http://localhost:8080/api/attachments/1/raw?token=..." } ] },
  "message": null,
  "errorCode": null
}
```

> 존재하지 않거나 **타인 소유인 ID는 결과에서 조용히 제외**된다(에러 아님) — 벌크 조회는 일부만 유효해도 나머지를 반환하는 편이 낫다는 판단이다. 단건 API(7.2·7.3·7.5·7.6)의 소유권 위반은 여전히 404(`FILE_003`)다.

---

### 7.5 파일 조회 — `GET /api/attachments/{id}/raw?token=...`

> ⚠️ **`ApiResponse` 래핑 예외** (1.1 참조). 바이너리 이미지 스트림을 그대로 반환한다.

`<img src>`를 브라우저가 직접 호출하므로 `Authorization` 헤더를 실을 수 없다. `SecurityConfig`에서 이 경로만 `permitAll`이고, 대신 쿼리의 **단기 서명 토큰(30분 유효)** 으로 컨트롤러가 직접 인가한다 — 로그인 세션과 무관하게 토큰만으로 접근 가능하다(S3 presigned GET과 동일한 개념).

응답 헤더에 `Content-Disposition: inline`, `X-Content-Type-Options: nosniff`를 포함한다 — 후자가 없으면 브라우저의 MIME 스니핑으로 위장 파일이 실행형 콘텐츠로 해석될 위험이 있다.

**에러**: `FILE_003`(404 — 존재하지 않음 / 토큰 만료·위조)

---

### 7.6 삭제 — `DELETE /api/attachments/{id}`

**Soft Delete**만 수행한다(불변 규칙 5와 동일). 실제 파일은 유예 기간(7일) 경과 후 고아 파일 정리 배치가 물리 삭제한다.

**Response** — `200 OK`

```json
{ "success": true, "data": null, "message": "삭제되었습니다.", "errorCode": null }
```

**에러**: `FILE_003`(404)

---

### 7.7 Todo와의 연결 (4장 참조)

`POST /api/todos`·`PUT /api/todos/{id}` 처리 시 서버가 `content`(Tiptap JSON)를 **노드 트리로 순회**해 `image` 노드의 `attachmentId`를 전부 수집하고(PRD 8.4 예외 조항), 본문에 실제로 남아 있는 것만 `LINKED` + `todo_id`로 전환한다. 저장 전에 연결돼 있었으나 본문에서 사라진 첨부는 Soft Delete하고, 다른 사용자 소유이거나 이미 다른 Todo에 `LINKED`된 ID가 섞여 있으면 요청 자체를 거부한다. Todo가 Soft Delete되면 연결된 첨부도 함께 Soft Delete된다.
