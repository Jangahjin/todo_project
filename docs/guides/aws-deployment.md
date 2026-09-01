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

---

## Task 037: 백엔드 EC2 배포 및 환경변수 주입

### 1. EC2 인스턴스 준비

| 항목 | 권장값 | 근거 |
|---|---|---|
| AMI | Amazon Linux 2023 | `dnf`로 JDK 21을 바로 설치 가능 |
| 인스턴스 타입 | `t3.micro`~`t3.small` (MVP 트래픽 기준) | 과다 스펙 지양 |
| 보안 그룹(인바운드) | `22`(SSH, 관리자 IP만) · `8080`(앱 포트, 프런트/ALB에서만 — MVP는 우선 전체 허용 후 Task 039에서 좁혀도 됨) | 최소 노출 |
| RDS 보안그룹 연동 | Task 036에서 만든 RDS 보안그룹의 인바운드 5432를 **이 EC2의 보안그룹**으로 허용 | Task 036 1절과 맞물림 |
| JDK 설치 | `sudo dnf install -y java-21-amazon-corretto` | CLAUDE.md — JDK 21 고정 |

```bash
java -version   # openjdk 21이 나와야 한다
```

### 2. 빌드 산출물 배포

로컬(또는 CI)에서 빌드한 뒤 EC2로 전송하는 방식을 권장한다 — EC2에 Maven·소스 전체를 둘 필요가 없다.

```bash
# 로컬 todo-backend/에서
./mvnw clean package -DskipTests   # target/todo-backend-0.0.1-SNAPSHOT.jar 생성

# EC2로 전송
scp -i <키페어.pem> target/todo-backend-0.0.1-SNAPSHOT.jar ec2-user@<EC2 퍼블릭IP>:/home/ec2-user/app.jar
```

### 3. 환경변수 주입 — systemd `EnvironmentFile`

**어떤 값도 Git에 커밋하지 않는다**(불변 규칙 9). EC2 로컬에만 존재하는 파일로 분리한다.

```bash
# EC2에서 — 이 파일은 EC2 로컬에만 존재, 저장소에 절대 두지 않는다
sudo tee /etc/todo-backend.env > /dev/null <<'EOF'
DB_URL=jdbc:postgresql://<RDS엔드포인트>:5432/postgres?currentSchema=todolistdb
DB_USERNAME=<마스터유저>
DB_PASSWORD=<Task 036에서 정한 비밀번호>
JWT_SECRET=<충분히 긴 랜덤 값>
OAUTH_GOOGLE_CLIENT_ID=<값>
OAUTH_GOOGLE_CLIENT_SECRET=<값>
OAUTH_KAKAO_CLIENT_ID=<값>
OAUTH_KAKAO_CLIENT_SECRET=<값>
APP_FRONTEND_URL=https://<운영 프론트 도메인>
EOF

sudo chmod 600 /etc/todo-backend.env   # ec2-user/root만 읽도록 제한
```

> 더 안전하게 하려면 평문 파일 대신 **AWS Systems Manager Parameter Store**(SecureString)에 저장하고 기동 스크립트에서 `aws ssm get-parameters`로 읽어와 이 파일을 생성하는 방식으로 바꿀 수 있다 — MVP는 위 방식으로 시작하고 필요시 전환한다.

### 4. systemd 서비스 등록 — 프로세스 관리·자동 재시작·로그

```ini
# /etc/systemd/system/todo-backend.service
[Unit]
Description=Todo Backend (Spring Boot)
After=network.target

[Service]
Type=simple
User=ec2-user
EnvironmentFile=/etc/todo-backend.env
ExecStart=/usr/bin/java -jar /home/ec2-user/app.jar --spring.profiles.active=prod
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now todo-backend
sudo systemctl status todo-backend   # active (running) 확인

# 로그 확인 (ApiResponse 포맷 에러, DB 연결 실패 등은 여기서 먼저 보인다)
sudo journalctl -u todo-backend -f
```

⚠️ **환경변수 중 하나라도 빠지면 `prod` 프로파일은 의도적으로 기동에 실패한다**(기본값 없음, CLAUDE.md 4장) — `journalctl`에 `Could not resolve placeholder` 류의 에러가 보이면 `/etc/todo-backend.env`의 누락 항목을 먼저 의심한다.

### 5. 완료 후 확인할 것

- [ ] `java -version`이 21을 보고
- [ ] `systemctl status todo-backend`가 `active (running)`
- [ ] `curl http://localhost:8080/api/auth/login`(등 아무 엔드포인트)이 `ApiResponse` 포맷 JSON으로 응답(연결 자체는 됨을 의미)
- [ ] `journalctl -u todo-backend`에 DB 연결·JWT·OAuth2 관련 에러 없음
- [ ] `/etc/todo-backend.env`가 `chmod 600`, Git 추적 대상 아님(EC2 로컬 파일이므로 애초에 저장소 밖)
- [ ] 재부팅 후에도 서비스가 자동 기동되는지(`systemctl enable` 확인)

이 항목들을 실제로 수행한 뒤 결과를 알려주면 `ROADMAP.md` Task 037을 실측 결과로 갱신한다.
