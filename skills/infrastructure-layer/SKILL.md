---
name: infrastructure-layer
description: |
  인프라 레이어 검증 및 가이드 스킬.
  DI 등록, Config, Migration, Queue, 미들웨어 스택을 검증하고 올바른 구현을 안내합니다.
  /audit, /review, /develop 커맨드에서 자동으로 활용됩니다.
---

# Infrastructure Layer 스킬

## 1) 목적 요약

인프라 레이어는 **프레임워크, 외부 서비스, 시스템 설정**을 담당합니다.

검증 대상:
- `cmd/api/main.go` - fx 의존성 주입 설정
- `pkg/config/` - 환경변수 기반 설정
- `migrations/` - DB 마이그레이션 SQL
- `pkg/factory/` - DB 팩토리 및 트랜잭션
- `pkg/echo/` - Echo 설정, 미들웨어
- `pkg/runner/`, `pkg/queue/` - SQS 메시지 큐

이 레이어의 핵심 원칙:
1. 설정값은 환경변수로만 주입 (하드코딩 금지)
2. DI 등록 순서는 계층 순서를 따름
3. 마이그레이션은 Up/Down 쌍으로 작성
4. 시크릿은 코드에 노출되지 않음

## 2) 책임 범위

| 항목 | 설명 |
|------|------|
| DI 등록 검증 | fx.Provide/Invoke 등록 누락, 순서 준수 |
| Config 검증 | 환경변수 태그, 기본값, prefix 계층 |
| Migration 검증 | Up/Down 쌍, 타임스탬프 네이밍, 안전한 DDL |
| Queue 검증 | Runner 생명주기, 핸들러 등록 |
| 보안 검증 | 시크릿 하드코딩, 환경변수 노출 |
| 트랜잭션 검증 | TransactionMiddleware 적용, AfterCommitHook |

## 3) 입력/출력

### 입력
- DI 파일: `cmd/api/main.go`
- Config: `pkg/config/config.go`
- Migration: `migrations/*.sql`
- 인프라 패키지: `pkg/`

### 출력 형식
```
[심각도] 파일: <path>:<line>
  규칙: <위반된 규칙 ID>
  이슈: <구체적 설명>
  수정: <권장 수정 방법>
```

## 4) 체크리스트

### DI 등록 (cmd/api/main.go)
- [ ] 새 Repository가 `fx.Provide`에 등록되었는지
- [ ] 새 Usecase가 `fx.Provide`에 등록되었는지
- [ ] 새 HTTP Delivery의 Register가 `fx.Invoke`에 등록되었는지
- [ ] 새 SQS Runner의 New가 `fx.Provide`에, Register가 `fx.Invoke`에 등록되었는지
- [ ] 등록 순서: Config → Infra → Repository → Service → Usecase → Runner → Delivery
- [ ] SQS Runner의 Start/Stop이 `serve()` 함수에 추가되었는지

### Config
- [ ] 새 설정이 Config 구조체에 추가되었는지
- [ ] `env` 태그로 환경변수명 지정
- [ ] `envPrefix` 로 계층적 prefix 사용
- [ ] `envDefault`로 합리적 기본값 설정
- [ ] 시크릿 값에 기본값 없음

### Migration
- [ ] `make goose <name> sql`로 생성 (UTC 타임스탬프)
- [ ] `-- +goose Up` / `-- +goose Down` 쌍 존재
- [ ] `-- +goose StatementBegin` / `-- +goose StatementEnd` 블록
- [ ] Down 마이그레이션이 Up을 정확히 되돌리는지
- [ ] `deleted_at` 컬럼 포함 (soft delete 지원)
- [ ] 필수 인덱스: FK 컬럼, 자주 검색하는 컬럼, `deleted_at`
- [ ] 컬럼 주석 (COMMENT ON COLUMN)

### 보안
- [ ] 코드에 시크릿 하드코딩 없음 (API 키, 비밀번호 등)
- [ ] `.env` 파일이 `.gitignore`에 포함
- [ ] 환경변수명에 민감 정보가 노출되지 않음

## 5) 권장 구현 패턴

### DI 등록 패턴 (cmd/api/main.go)
```go
app := fx.New(
    fx.WithLogger(func() fxevent.Logger {
        return fxevent.NopLogger
    }),
    fx.Provide(
        // 1. 설정 & 인프라
        config.LoadConfig,
        awspkg.NewConfig,
        factory.NewDBFactory,
        jwt.New,
        echopkg.New,

        // 2. Repository
        humanRepository.New,

        // 3. Service (있을 경우)

        // 4. Usecase
        humanUsecase.New,

        // 5. SQS Runner (있을 경우)
    ),
    fx.Invoke(
        // 1. SQS Runner 등록
        // humanRunner.Register,

        // 2. HTTP Delivery 등록
        humanHttpDelivery.Register,

        // 3. 서버 시작
        beforeDo,
        serve,
    ),
)
```

### Config 추가 패턴
```go
// 새 서비스 설정 추가
type Config struct {
    // ... 기존 설정
    NewService NewService `envPrefix:"NEW_SERVICE_"`
}

type NewService struct {
    APIKey    string `env:"API_KEY"`                    // 시크릿: 기본값 없음
    BaseURL   string `env:"BASE_URL" envDefault:"https://api.example.com"`
    Timeout   int    `env:"TIMEOUT" envDefault:"30"`
    QueueName string `env:"QUEUE_NAME"`
}
```

### Migration 패턴
```sql
-- +goose Up
-- +goose StatementBegin

CREATE TABLE humans (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL,
    name VARCHAR(255) NOT NULL,
    age INTEGER,
    status VARCHAR(50) NOT NULL DEFAULT 'ACTIVE',
    approved_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP WITH TIME ZONE
);

-- 인덱스
CREATE INDEX idx_humans_user_id ON humans(user_id);
CREATE INDEX idx_humans_status ON humans(status);
CREATE INDEX idx_humans_deleted_at ON humans(deleted_at);

-- 컬럼 주석
COMMENT ON COLUMN humans.user_id IS '사용자 ID (FK)';
COMMENT ON COLUMN humans.name IS '이름';
COMMENT ON COLUMN humans.status IS '상태 (ACTIVE, INACTIVE)';

-- +goose StatementEnd

-- +goose Down
-- +goose StatementBegin

DROP TABLE IF EXISTS humans;

-- +goose StatementEnd
```

### ALTER TABLE 마이그레이션 패턴
```sql
-- +goose Up
-- +goose StatementBegin

ALTER TABLE humans ADD COLUMN email VARCHAR(255);
CREATE INDEX idx_humans_email ON humans(email);
COMMENT ON COLUMN humans.email IS '이메일 주소';

-- +goose StatementEnd

-- +goose Down
-- +goose StatementBegin

DROP INDEX IF EXISTS idx_humans_email;
ALTER TABLE humans DROP COLUMN IF EXISTS email;

-- +goose StatementEnd
```

### serve() 함수에 Runner 추가 패턴
```go
func serve(lifecycle fx.Lifecycle, echo *echo.Echo, cfg *config.Config,
    humanRunner *humanSQS.HumanRunner,
) {
    lifecycle.Append(fx.Hook{
        OnStart: func(ctx context.Context) error {
            go func() {
                if err := echo.Start(cfg.Server.Port); err != nil {
                    slog.Error("서버 시작 실패", "error", err)
                }
            }()
            go humanRunner.Start(ctx)  // Runner 시작
            return nil
        },
        OnStop: func(ctx context.Context) error {
            humanRunner.Stop()  // Runner 중지
            return echo.Shutdown(ctx)
        },
    })
}
```

## 6) 안티패턴 탐지 규칙

### BLOCKING

| ID | 안티패턴 | 올바른 패턴 |
|----|---------|-----------|
| F-001 | 시크릿 하드코딩 (`apiKey := "sk-..."`) | 환경변수에서 로드 |
| F-002 | DI 등록 누락 (Repository/Usecase/Delivery) | fx.Provide/Invoke에 모두 등록 |
| F-003 | Migration Down 미작성 | Up과 Down 쌍 필수 |
| F-004 | Migration Down이 Up을 되돌리지 않음 | 정확한 역방향 DDL |

### MAJOR

| ID | 안티패턴 | 올바른 패턴 |
|----|---------|-----------|
| F-005 | DI 등록 순서 위반 | Config→Infra→Repo→Usecase→Delivery |
| F-006 | FK 컬럼 인덱스 누락 | FK에 인덱스 필수 |
| F-007 | `deleted_at` 컬럼/인덱스 누락 | soft delete 지원 필수 |
| F-008 | SQS Runner Start/Stop 누락 | serve() 함수에 추가 |
| F-009 | Config에 `envPrefix` 미사용 (flat 구조) | 계층적 prefix 사용 |
| F-010 | 환경변수 직접 읽기 (`os.Getenv`) | Config 구조체 통해서만 접근 |

### MINOR

| ID | 안티패턴 | 올바른 패턴 |
|----|---------|-----------|
| F-011 | 컬럼 주석 누락 | COMMENT ON COLUMN 추가 |
| F-012 | 시크릿 Config에 `envDefault` 설정 | 시크릿은 기본값 없이 |
| F-013 | Migration 파일명에 한국어 사용 | 영어 snake_case |

## 7) 샘플 코드 스니펫

### 복잡한 Migration (데이터 변환 포함)
```sql
-- +goose Up
-- +goose StatementBegin

-- 컬럼 추가
ALTER TABLE sources ADD COLUMN category VARCHAR(30);

-- 기존 데이터 분류
UPDATE sources
SET category = CASE
    WHEN host_url IN ('https://www.youtube.com', 'https://www.reddit.com') THEN 'SNS'
    WHEN host_url IN ('https://www.allure.com', 'https://www.vogue.com') THEN 'MAGAZINE'
    ELSE 'OTHER'
END;

-- NOT NULL 제약조건 추가
ALTER TABLE sources ALTER COLUMN category SET NOT NULL;

CREATE INDEX idx_sources_category ON sources(category);

-- +goose StatementEnd

-- +goose Down
-- +goose StatementBegin

DROP INDEX IF EXISTS idx_sources_category;
ALTER TABLE sources DROP COLUMN IF EXISTS category;

-- +goose StatementEnd
```

### Queue 메시지 전송 (Usecase에서)
```go
// 트랜잭션 커밋 후 전송 (데이터 정합성 보장)
err = u.queue.SendMessageAfterCommit(ctx, apidomain.ProcessTaskRequest{
    TaskID: task.ID,
}, map[string]string{
    runner.FuncAttribute: "TaskUsecase.ProcessTask",
})

// 즉시 전송 (트랜잭션과 무관)
err = u.queue.SendMessage(ctx, apidomain.NotifyRequest{
    UserID: user.ID,
}, map[string]string{
    runner.FuncAttribute: "NotificationUsecase.Notify",
})

// 분산 락 키와 함께 전송
err = u.queue.SendMessage(ctx, request, map[string]string{
    runner.FuncAttribute: "BrandUsecase.Classify",
    queue.LockKeyAttribute: fmt.Sprintf("brand:%d", brandID),
})
```

### AfterCommitHook 사용 (트랜잭션 후 작업)
```go
func (u *userUsecase) CreateUser(ctx context.Context, req apidomain.CreateUserRequest) (apidomain.CreateUserResponse, error) {
    userID, err := u.userRepository.Create(ctx, domain.User{
        Email: req.Email,
        Name:  req.Name,
    })
    if err != nil {
        return apidomain.CreateUserResponse{}, err
    }

    // 트랜잭션 커밋 후 실행 (이메일 발송, 외부 API 호출 등)
    if hook, ok := ctx.Value(factory.AfterCommitKey{}).(*factory.AfterCommitHook); ok {
        hook.Add(func() {
            u.emailService.SendWelcomeEmail(userID)
        })
    }

    return apidomain.CreateUserResponse{UserID: userID}, nil
}
```

## 8) 테스트 지침

### DI 테스트
DI 등록이 올바른지 확인하는 방법:

```bash
# 빌드 성공 = DI 그래프 유효
make build

# 전체 테스트 (DI 관련 에러 포함)
go test ./...
```

### Migration 테스트
```bash
# 마이그레이션 Up 실행
make goose up

# 마이그레이션 Down 실행 (롤백 검증)
make goose down

# 다시 Up (재실행 가능 검증)
make goose up
```

### Config 테스트
```go
func TestLoadConfig(t *testing.T) {
    // 필수 환경변수 설정
    t.Setenv("DATABASE_URL", "postgres://test:test@localhost/test")
    t.Setenv("JWT_SECRET", "test-secret")

    cfg := config.LoadConfig()

    assert.Equal(t, "postgres://test:test@localhost/test", cfg.Database.URL)
    assert.Equal(t, "test-secret", cfg.JWT.Secret)
    assert.Equal(t, ":8080", cfg.Server.Port) // 기본값 확인
}
```

### Queue/Runner 테스트
```go
func TestHumanRunner_Register(t *testing.T) {
    mockUsecase := mocks.NewHumanUsecase(t)

    // 로컬 환경: 메모리 큐 사용
    cfg := &config.Config{Env: "local"}
    q := memqueue.New()
    r := runner.New(cfg, q, pg_lock.NewNoop())

    humanRunner := &HumanRunner{runner: r, usecase: mockUsecase}
    Register(humanRunner)

    // 핸들러 등록 확인
    assert.NotPanics(t, func() {
        humanRunner.Start(context.Background())
        humanRunner.Stop()
    })
}
```

## 9) 연계 스킬/명령

| 연계 대상 | 활용 방식 |
|----------|----------|
| `/audit` | `arch-auditor`가 DI 등록 누락, 순서 위반 검증 |
| `/audit` | `security-auditor`가 시크릿 하드코딩, 환경변수 보안 검증 |
| `/audit` | `data-auditor`가 마이그레이션 안전성 검증 |
| `/review` | 전체 인프라 변경 사항 검증 |
| `/develop` | 새 도메인 추가 시 DI 등록 + 마이그레이션 생성 가이드 |
| `domain-layer` | 엔티티-마이그레이션 일관성 확인 |
| `interface-layer` | Delivery/Repository 등록 일관성 확인 |

## 10) 메트릭/거버넌스

| 메트릭 | 설명 | 임계값 |
|--------|------|--------|
| `DI 등록 누락 수` | Provide/Invoke에 등록 안 된 컴포넌트 | 0건 (BLOCKING) |
| `시크릿 하드코딩 수` | 코드에 시크릿이 노출된 건 수 | 0건 (BLOCKING) |
| `Migration Down 누락 수` | Down 마이그레이션이 없는 파일 | 0건 (BLOCKING) |
| `FK 인덱스 누락 수` | FK 컬럼에 인덱스 없는 건 수 | 0건 (MAJOR) |
| `deleted_at 누락 수` | soft delete 미지원 테이블 수 | 0건 (MAJOR) |
| `컬럼 주석 커버리지` | 주석이 있는 컬럼 비율 | 80% 이상 권장 |
