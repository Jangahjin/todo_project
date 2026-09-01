# AWS 배포 런북 (M9)

이 문서는 M9(AWS 배포) 각 Task를 실제로 수행할 때 따라가는 체크리스트다. **AWS 콘솔/CLI 조작은 Claude Code가 대신 실행할 수 없으므로**, 사용자가 직접 수행하고 결과를 알려주면 그에 맞춰 `ROADMAP.md`를 갱신한다. 현재는 Task 036(RDS) 범위만 채워져 있다 — Task 037(EC2)·038(Amplify)·039(CORS/Redirect)는 해당 Task를 진행할 때 이어서 채운다.

---

## Task 036: RDS(PostgreSQL) 프로비저닝

### 1. RDS 인스턴스 생성

콘솔(RDS → 데이터베이스 생성) 또는 CLI 어느 쪽이든, 아래 값을 확정해야 한다.

| 항목 | 권장값 | 근거 |
|---|---|---|
| 엔진 | PostgreSQL (최신 안정 버전) | PRD 1.3 |
| 인스턴스 클래스 | `db.t4g.micro` (MVP 트래픽 기준, 프리티어 고려 시) | 과다 스펙 지양 |
| 스토리지 | 범용 SSD(gp3) 20GB부터, 자동 확장 켜기 | 초기 비용 절감 |
| Multi-AZ | MVP는 끔(비용) — 운영 트래픽 늘면 재검토 | — |
| 퍼블릭 액세스 | **아니오** — EC2에서만 접근 | 보안 |
| 보안 그룹 | 인바운드 5432를 **EC2 보안그룹**에서만 허용(0.0.0.0/0 금지) | CLAUDE.md 불변 규칙 9의 정신(비밀 노출 최소화)과 동일 원칙 |
| DB 이름(초기) | `postgres` (기본 DB, 스키마는 그 안에 별도 생성) | `application.properties` 기본 URL이 `.../postgres?currentSchema=todolistdb` 전제 |
| 마스터 사용자명/비밀번호 | 콘솔에서 직접 설정, **어디에도 커밋하지 않는다** | 불변 규칙 9 |

생성 완료 후 **엔드포인트 주소**를 기록해 둔다 — 이게 `DB_URL`의 호스트가 된다.

### 2. 스키마 생성 — 따옴표 없이

PRD 8.1의 대소문자 폴딩 정책 그대로다. EC2(또는 RDS에 접근 가능한 아무 머신)에서 `psql`로 접속해 실행한다.

```sql
-- ✅ 반드시 따옴표 없이 — 그래야 실제 물리 스키마명이 소문자 todolistdb가 된다
CREATE SCHEMA IF NOT EXISTS TodoListDB;

-- 확인 (물리 식별자가 todolistdb인지 눈으로 검증)
\dn
```

`\dn` 결과에 `todolistdb`(소문자)가 보여야 한다. `"TodoListDB"`(대문자, 따옴표 포함)로 보이면 잘못 생성된 것이니 `DROP SCHEMA "TodoListDB";` 후 위 명령을 다시 따옴표 없이 실행한다.

### 3. 테이블 생성 — ddl-auto=validate 우회 전략

`application-prod.properties`는 `ddl-auto=validate`로 고정되어 있다(운영 DB를 Hibernate가 임의로 바꾸지 못하게 하는 의도된 설계, CLAUDE.md 4장). 즉 **테이블이 이미 존재하지 않으면 `prod` 프로파일로는 기동 자체가 실패한다.** 최초 1회는 아래 둘 중 하나로 테이블을 먼저 만들어야 한다.

**방법 A — 임시 dev 프로파일로 1회 기동 (권장, 더 간단)**

1. EC2(또는 RDS에 접근 가능한 로컬 환경)에서 `DB_URL`을 RDS 엔드포인트로 지정하고, **일시적으로 `dev` 프로파일**(`ddl-auto=update`)로 백엔드를 1회 기동한다.
   ```bash
   DB_URL=jdbc:postgresql://<RDS엔드포인트>:5432/postgres?currentSchema=todolistdb \
   DB_USERNAME=<마스터유저> DB_PASSWORD=<마스터비번> JWT_SECRET=<아무 값> \
   ./mvnw spring-boot:run -Dspring-boot.run.profiles=dev
   ```
2. Hibernate가 엔티티(`User`, `Todo`, `BaseEntity`) 기준으로 테이블을 자동 생성하는 로그를 확인한 뒤 종료한다.
3. 이후부터는 항상 `prod` 프로파일(`ddl-auto=validate`)로 기동한다 — Task 037에서 진행.

**방법 B — DDL을 SQL로 추출해 직접 적용**

로컬 개발 DB(`todolistdb`, 이미 M1에서 `dev` 프로파일로 생성돼 있음)에서 스키마를 덤프해 운영에 그대로 적용한다.

```bash
# 로컬 dev DB에서 스키마(구조)만 덤프 — 데이터는 제외
pg_dump -h localhost -U postgres -d postgres \
  --schema=todolistdb --schema-only --no-owner --no-privileges \
  > todolistdb_schema.sql

# 운영 RDS에 적용
psql -h <RDS엔드포인트> -U <마스터유저> -d postgres -f todolistdb_schema.sql
```

이 방법은 로컬 스키마가 최신 엔티티 상태와 정확히 일치한다는 전제가 필요하다 — 방법 A가 더 안전하다.

### 4. 백업/스냅샷 정책

- **자동 백업**: RDS 콘솔에서 "자동 백업" 활성화, 보존 기간 7일 이상 권장(MVP는 7일로 시작해도 충분).
- **백업 윈도우**: 트래픽이 적은 시간대(예: KST 새벽)로 지정.
- **수동 스냅샷**: 스키마 변경(마이그레이션)이나 대규모 배포 직전에 수동 스냅샷 1회 추가 권장 — 자동 백업만으로는 "배포 직전 시점"을 정확히 못 잡을 수 있다.
- **삭제 방지**: RDS 삭제 보호(Deletion Protection) 켜기 — 실수로 인스턴스를 지우는 사고 방지.

### 5. 완료 후 확인할 것

- [ ] RDS 엔드포인트로 `psql` 접속 성공 (EC2에서만 — 외부에서는 접속 안 되는지도 확인)
- [ ] `\dn`으로 `todolistdb` 물리 스키마 확인
- [ ] 방법 A/B 중 하나로 테이블 생성 완료, `\dt todolistdb.*`로 `users`·`todos` 테이블 존재 확인
- [ ] 백업 활성화·삭제 보호 켜짐
- [ ] `DB_URL`·`DB_USERNAME`·`DB_PASSWORD` 값을 안전한 곳(비밀번호 관리자, AWS Secrets Manager 등)에 기록 — **Git에는 커밋하지 않는다**

이 항목들을 실제로 수행한 뒤 결과를 알려주면 `ROADMAP.md` Task 036을 실측 결과로 갱신한다.
