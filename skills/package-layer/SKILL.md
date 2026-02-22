---
name: package-layer
description: |
  공유 패키지(pkg/) 레이어 검증 및 가이드 스킬.
  인터페이스 설계, 생성자 패턴, 함수형 옵션, 에러 처리, 외부 서비스 클라이언트, 유틸리티 패키지를 검증하고 올바른 구현을 안내합니다.
  /audit, /review, /develop 커맨드에서 자동으로 활용됩니다.
---

# Package Layer 스킬

## 1) 목적 요약

`pkg/` 디렉토리에 위치하는 공유 패키지 레이어를 검증합니다.
이 레이어는 **재사용 가능한 인프라 컴포넌트, 외부 서비스 클라이언트, 유틸리티**를 제공하며,
비즈니스 로직과 무관한 기술적 관심사만 담당합니다.

검증 대상 (60개 패키지, 6개 카테고리):

| 카테고리 | 패키지 수 | 주요 패키지 |
|----------|----------|------------|
| Core Infrastructure | 9 | config, factory, echo, jwt, migration, logger, validator, pg_lock, apikey |
| AWS/Cloud | 9 | aws, s3, sqs, ses, queue, queuefactory, mem-queue, runner, lambda |
| External API Clients | 8 | openai, gemini, paddle, sendgrid, datadog, slack, dataforseo, customsearch |
| AI/Agent | 3 | agent, agentic, question_generator |
| Utilities | 28 | collection, db, retrier, httpclient, html, scraper, time, strings 등 |
| Testing | 2 | mocks, testhelper |

## 2) 책임 범위

| 항목 | 설명 |
|------|------|
| 인터페이스 설계 검증 | 모든 서비스가 인터페이스로 정의되는지 |
| 생성자 패턴 검증 | New() 함수가 인터페이스 타입을 반환하는지 |
| 함수형 옵션 패턴 검증 | Option 패턴의 일관성 |
| 에러 처리 검증 | errors.Wrap 사용, 컨텍스트 포함 |
| 설정 의존성 검증 | config.Config 주입 패턴 |
| APM 통합 검증 | Datadog 트레이싱 적용 |
| 보안 검증 | 시크릿 하드코딩, 환경변수 직접 읽기 금지 |
| Context 전파 검증 | I/O 작업에 context.Context 사용 |

## 3) 입력/출력

### 입력
- 패키지 파일: `pkg/<package>/*.go`
- 새로운 패키지 생성 시: 패키지명과 목적

### 출력 형식
```
[심각도] 파일: <path>:<line>
  규칙: <위반된 규칙 ID>
  이슈: <구체적 설명>
  수정: <권장 수정 방법>
```

## 4) 체크리스트

### 인터페이스 설계
- [ ] 외부 서비스 클라이언트는 인터페이스로 정의
- [ ] 인터페이스는 패키지 최상위에 정의
- [ ] 구현체는 소문자 구조체 (unexported)
- [ ] 생성자 `New()` 함수가 인터페이스 타입 반환
- [ ] 인터페이스가 최소한으로 설계되었는지 (ISP)

### 생성자 패턴
- [ ] `New(cfg *config.Config, ...) Interface` 시그니처
- [ ] 필수 의존성은 파라미터로, 선택적 설정은 Option으로
- [ ] 환경 분기: `cfg.IsLocal()` 체크 (필요한 경우)

### 함수형 옵션
- [ ] `type Option func(*options)` 패턴 사용
- [ ] 옵션 구조체는 unexported
- [ ] `With<Name>()` 네이밍 컨벤션
- [ ] 합리적인 기본값 설정

### Context 전파
- [ ] 모든 I/O 작업의 첫 번째 파라미터가 `context.Context`
- [ ] `slog.InfoContext(ctx, ...)` / `slog.ErrorContext(ctx, ...)` 로깅
- [ ] Datadog span에 context 전파

### 에러 처리
- [ ] 외부 라이브러리 에러: `errors.Wrap(err, "설명")` 사용
- [ ] 에러 메시지에 작업 컨텍스트 포함 (영어)
- [ ] 도메인 에러와 인프라 에러 분리

### 보안
- [ ] 시크릿 하드코딩 없음
- [ ] `os.Getenv()` 직접 호출 없음 (config 통해서만)
- [ ] 로그에 민감 정보 출력 없음

### 테스트 가능성
- [ ] 인터페이스 기반으로 모킹 가능
- [ ] `pkg/mocks/`에 mock 인터페이스 생성 가능
- [ ] 환경별 구현체 분리 (SQS vs mem-queue)

## 5) 권장 구현 패턴

### 패턴 A: 외부 서비스 클라이언트

인터페이스 정의 → 구현체 → 생성자 → fx 등록

```go
package myservice

import (
    "context"
    "log/slog"

    "github.com/chainshiftlabs/console-api-go/pkg/config"
    "github.com/pkg/errors"
)

// 인터페이스 (exported)
type Client interface {
    DoSomething(ctx context.Context, param string) (Result, error)
    DoAnotherThing(ctx context.Context, id uint) error
}

// 구현체 (unexported)
type client struct {
    cfg    *config.Config
    apiKey string
}

// 생성자 - 인터페이스 반환
func New(cfg *config.Config) Client {
    return &client{
        cfg:    cfg,
        apiKey: cfg.MyService.APIKey,
    }
}

// 메서드 구현
func (c *client) DoSomething(ctx context.Context, param string) (Result, error) {
    slog.InfoContext(ctx, "[MyService] 작업 시작", "param", param)

    result, err := c.callAPI(ctx, param)
    if err != nil {
        slog.ErrorContext(ctx, "[MyService] API 호출 실패", "error", err)
        return Result{}, errors.Wrap(err, "failed to call myservice API")
    }

    slog.InfoContext(ctx, "[MyService] 작업 완료", "resultID", result.ID)
    return result, nil
}
```

### 패턴 B: Queue 추상화 (인터페이스 + 다중 구현)

```go
// pkg/queue/queue.go - 인터페이스
type Queue interface {
    SendMessage(ctx context.Context, message any, attributes map[string]string, options ...MessageOption) error
    SendMessageAfterCommit(ctx context.Context, message any, attributes map[string]string, options ...MessageOption) error
    ReceiveMessage(ctx context.Context, options ...MessageOption) ([]Message, error)
    DeleteMessage(ctx context.Context, receiptHandle *string, options ...MessageOption) error
}

// pkg/sqs/client.go - SQS 구현체 (프로덕션)
func New(cfg *config.Config, awsCfg *aws.Config) queue.Queue {
    return &sqsClient{...}
}

// pkg/mem-queue/queue.go - 메모리 구현체 (로컬/테스트)
func New() queue.Queue {
    return &memQueue{...}
}

// pkg/queuefactory/factory.go - 팩토리 (환경별 분기)
func New(cfg *config.Config, awsCfg *aws.Config) queue.Queue {
    if cfg.IsLocal() {
        return memqueue.New()
    }
    return sqs.New(cfg, awsCfg)
}
```

### 패턴 C: 함수형 옵션

```go
package httpclient

import "time"

// 옵션 구조체 (unexported)
type options struct {
    timeout    time.Duration
    retryCount int
    baseURL    string
}

// 옵션 함수 타입
type Option func(*options)

// With* 함수들
func WithTimeout(d time.Duration) Option {
    return func(o *options) { o.timeout = d }
}

func WithRetry(count int) Option {
    return func(o *options) { o.retryCount = count }
}

func WithBaseURL(url string) Option {
    return func(o *options) { o.baseURL = url }
}

// 생성자에서 옵션 적용
func New(cfg *config.Config, opts ...Option) *Client {
    o := &options{
        timeout:    30 * time.Second, // 합리적 기본값
        retryCount: 3,
    }
    for _, opt := range opts {
        opt(o)
    }
    // 클라이언트 생성...
}
```

### 패턴 D: 유틸리티 패키지

상태를 갖지 않는 순수 함수 패키지:

```go
package collection

// 제네릭 슬라이스 변환
func Map[T, U any](slice []T, fn func(T) U) []U {
    result := make([]U, len(slice))
    for i, v := range slice {
        result[i] = fn(v)
    }
    return result
}

// 집합 동치성 검사
func SetEqual[T comparable](a, b []T) bool {
    if len(a) != len(b) {
        return false
    }
    set := make(map[T]struct{}, len(a))
    for _, v := range a {
        set[v] = struct{}{}
    }
    for _, v := range b {
        if _, ok := set[v]; !ok {
            return false
        }
    }
    return true
}
```

### 패턴 E: 재시도 로직 (지수 백오프)

```go
package retrier

import (
    "context"
    "math"
    "math/rand"
    "time"
)

type BackoffConfig struct {
    Initial    time.Duration // 초기 대기 시간
    Max        time.Duration // 최대 대기 시간
    Multiplier float64       // 배수
    MaxRetries int           // 최대 재시도 횟수
}

func RetryWithExponentialBackoff[T any](
    ctx context.Context,
    cfg BackoffConfig,
    isRetryable func(error) bool,
    op func(context.Context) (T, error),
) (T, error) {
    var zero T
    delay := cfg.Initial

    for attempt := 0; attempt <= cfg.MaxRetries; attempt++ {
        result, err := op(ctx)
        if err == nil {
            return result, nil
        }
        if !isRetryable(err) || attempt == cfg.MaxRetries {
            return zero, err
        }

        // Full jitter
        jitter := time.Duration(rand.Int63n(int64(delay)))
        select {
        case <-ctx.Done():
            return zero, ctx.Err()
        case <-time.After(jitter):
        }

        delay = time.Duration(math.Min(
            float64(delay)*cfg.Multiplier,
            float64(cfg.Max),
        ))
    }
    return zero, nil
}
```

### 패턴 F: GORM JSONB 제네릭 타입

```go
package db

import (
    "database/sql/driver"
    "encoding/json"
)

// JSONB 제네릭 래퍼
type JSONB[T any] struct {
    V T
}

func (j JSONB[T]) Value() (driver.Value, error) {
    return json.Marshal(j.V)
}

func (j *JSONB[T]) Scan(value any) error {
    bytes, ok := value.([]byte)
    if !ok {
        return errors.New("type assertion to []byte failed")
    }
    return json.Unmarshal(bytes, &j.V)
}

// 사용 예
type Campaign struct {
    gorm.Model
    Persona  db.JSONB[*CampaignPersona]  `gorm:"type:jsonb"`
    Metadata db.JSONB[*Metadata]          `gorm:"type:jsonb"`
}
```

### 패턴 G: DB 팩토리 + 트랜잭션 관리

```go
package factory

import (
    "context"
    "gorm.io/gorm"
)

type DBFactory struct {
    db  *gorm.DB
    Key interface{}
}

// 컨텍스트에서 트랜잭션 또는 기본 DB 추출
func (f *DBFactory) GetDB(ctx context.Context) *gorm.DB {
    db, ok := ctx.Value(f.Key).(*gorm.DB)
    if !ok {
        return f.db.WithContext(ctx)
    }
    return db.WithContext(ctx)
}

// AfterCommitHook: 트랜잭션 커밋 후 실행할 작업 등록
type AfterCommitHook struct {
    hooks []func()
}

func (h *AfterCommitHook) Add(fn func()) {
    h.hooks = append(h.hooks, fn)
}

func (h *AfterCommitHook) Execute() {
    for _, hook := range h.hooks {
        hook()
    }
}
```

### 패턴 H: APM (Datadog) 통합

```go
// 외부 API 호출에 트레이싱 추가
func (c *client) CallExternalAPI(ctx context.Context, req Request) (Response, error) {
    span, ctx := tracer.StartSpanFromContext(ctx, "myservice.call",
        tracer.ResourceName("DoSomething"),
    )
    defer span.Finish()

    resp, err := c.httpClient.R().
        SetContext(ctx).
        SetBody(req).
        Post(c.baseURL + "/api/endpoint")

    if err != nil {
        span.SetTag("error", true)
        span.SetTag("error.msg", err.Error())
        return Response{}, errors.Wrap(err, "external API call failed")
    }

    return resp, nil
}
```

## 6) 안티패턴 탐지 규칙

### BLOCKING

| ID | 안티패턴 | 올바른 패턴 |
|----|---------|-----------|
| K-001 | 시크릿 하드코딩 (`apiKey := "sk-..."`) | config.Config에서 주입 |
| K-002 | `os.Getenv()` 직접 호출 | config 구조체 통해서만 접근 |
| K-003 | 로그에 민감 정보 출력 (`slog.Info("token", token)`) | 민감 정보 마스킹 또는 제외 |
| K-004 | I/O 함수에 context.Context 누락 | 첫 번째 파라미터로 필수 |

### MAJOR

| ID | 안티패턴 | 올바른 패턴 |
|----|---------|-----------|
| K-005 | 서비스가 인터페이스 없이 구체 타입만 제공 | 인터페이스 정의 + 구현체 분리 |
| K-006 | 생성자가 구체 타입 반환 (`*client`) | 인터페이스 타입 반환 (`Client`) |
| K-007 | 에러 래핑 없이 그대로 반환 | `errors.Wrap(err, "컨텍스트")` 사용 |
| K-008 | 에러 메시지에 컨텍스트 누락 | 어떤 작업에서 실패했는지 포함 |
| K-009 | 로깅에 `log.Printf` 사용 | `slog.InfoContext(ctx, ...)` 사용 |
| K-010 | 옵션 패턴에 기본값 누락 | 합리적 기본값 설정 필수 |

### MINOR

| ID | 안티패턴 | 올바른 패턴 |
|----|---------|-----------|
| K-011 | 패키지명에 하이픈 사용 (`my-service`) | 소문자 연속 (`myservice`) |
| K-012 | exported 구현체 구조체 (`type Client struct`) | unexported (`type client struct`) |
| K-013 | APM 트레이싱 미적용 (외부 호출) | tracer.StartSpanFromContext 사용 |
| K-014 | 환경별 분기 없음 (로컬 vs 프로덕션) | `cfg.IsLocal()` 체크 |

## 7) 샘플 코드 스니펫

### 새 외부 서비스 패키지 생성 (전체 예시)

```go
// pkg/myservice/myservice.go
package myservice

import (
    "context"
    "log/slog"
    "time"

    "github.com/chainshiftlabs/console-api-go/pkg/config"
    "github.com/chainshiftlabs/console-api-go/pkg/httpclient"
    "github.com/go-resty/resty/v2"
    "github.com/pkg/errors"
    "gopkg.in/DataDog/dd-trace-go.v1/ddtrace/tracer"
)

// === 인터페이스 ===

type Client interface {
    Search(ctx context.Context, query string) ([]SearchResult, error)
    GetDetail(ctx context.Context, id string) (Detail, error)
}

// === 응답 타입 ===

type SearchResult struct {
    ID    string `json:"id"`
    Title string `json:"title"`
    Score float64 `json:"score"`
}

type Detail struct {
    ID          string `json:"id"`
    Title       string `json:"title"`
    Description string `json:"description"`
}

// === 구현체 ===

type client struct {
    httpClient *resty.Client
    apiKey     string
}

func New(cfg *config.Config, httpFactory *httpclient.Factory) Client {
    return &client{
        httpClient: httpFactory.Create(
            httpclient.WithBaseURL(cfg.MyService.BaseURL),
            httpclient.WithTimeout(30 * time.Second),
        ),
        apiKey: cfg.MyService.APIKey,
    }
}

func (c *client) Search(ctx context.Context, query string) ([]SearchResult, error) {
    span, ctx := tracer.StartSpanFromContext(ctx, "myservice.search",
        tracer.ResourceName("Search"),
    )
    defer span.Finish()

    slog.InfoContext(ctx, "[MyService] 검색 시작", "query", query)

    var result []SearchResult
    resp, err := c.httpClient.R().
        SetContext(ctx).
        SetHeader("Authorization", "Bearer "+c.apiKey).
        SetQueryParam("q", query).
        SetResult(&result).
        Get("/api/v1/search")

    if err != nil {
        span.SetTag("error", true)
        return nil, errors.Wrap(err, "failed to search myservice")
    }

    if resp.IsError() {
        return nil, errors.Errorf("myservice search returned %d: %s", resp.StatusCode(), resp.String())
    }

    slog.InfoContext(ctx, "[MyService] 검색 완료", "count", len(result))
    return result, nil
}

func (c *client) GetDetail(ctx context.Context, id string) (Detail, error) {
    span, ctx := tracer.StartSpanFromContext(ctx, "myservice.get_detail",
        tracer.ResourceName("GetDetail"),
    )
    defer span.Finish()

    var detail Detail
    resp, err := c.httpClient.R().
        SetContext(ctx).
        SetHeader("Authorization", "Bearer "+c.apiKey).
        SetResult(&detail).
        Get("/api/v1/items/" + id)

    if err != nil {
        return Detail{}, errors.Wrap(err, "failed to get detail from myservice")
    }

    if resp.IsError() {
        return Detail{}, errors.Errorf("myservice get_detail returned %d", resp.StatusCode())
    }

    return detail, nil
}
```

### Config에 새 서비스 추가

```go
// pkg/config/config.go
type Config struct {
    // ... 기존 설정
    MyService MyService `envPrefix:"MY_SERVICE_"`
}

type MyService struct {
    APIKey  string `env:"API_KEY"`                                       // 시크릿
    BaseURL string `env:"BASE_URL" envDefault:"https://api.myservice.com"` // 기본값 있음
    Timeout int    `env:"TIMEOUT" envDefault:"30"`                        // 기본값 있음
}
```

### DI 등록

```go
// cmd/api/main.go
fx.Provide(
    // ... 기존 등록
    myservice.New,  // Client 인터페이스 반환
),
```

## 8) 테스트 지침

### 패키지 테스트 원칙
- 인터페이스 기반 패키지: mock으로 상위 레이어 테스트
- 유틸리티 패키지: 직접 단위 테스트
- 외부 서비스 클라이언트: 통합 테스트 + mock 서버

### 유틸리티 패키지 테스트 (table-driven)
```go
func TestSetEqual(t *testing.T) {
    tests := []struct {
        name     string
        a, b     []int
        expected bool
    }{
        {"같은 집합", []int{1, 2, 3}, []int{3, 2, 1}, true},
        {"다른 집합", []int{1, 2}, []int{1, 3}, false},
        {"빈 집합", []int{}, []int{}, true},
        {"길이 다름", []int{1}, []int{1, 2}, false},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            assert.Equal(t, tt.expected, collection.SetEqual(tt.a, tt.b))
        })
    }
}
```

### 외부 서비스 클라이언트 Mock 테스트
```go
// pkg/mocks/myservice_client.go (mockery로 자동 생성)
type MockClient struct {
    mock.Mock
}

func (m *MockClient) Search(ctx context.Context, query string) ([]myservice.SearchResult, error) {
    args := m.Called(ctx, query)
    return args.Get(0).([]myservice.SearchResult), args.Error(1)
}

// Usecase 테스트에서 사용
func TestBrandUsecase_Analyze(t *testing.T) {
    mockClient := mocks.NewMockClient(t)
    mockClient.On("Search", mock.Anything, "test query").
        Return([]myservice.SearchResult{{ID: "1", Title: "결과"}}, nil)

    uc := brand.New(mockClient)
    result, err := uc.Analyze(ctx, "test query")
    assert.NoError(t, err)
    assert.Len(t, result, 1)
}
```

### 재시도 로직 테스트
```go
func TestRetryWithExponentialBackoff(t *testing.T) {
    attempt := 0
    result, err := retrier.RetryWithExponentialBackoff(context.Background(),
        retrier.BackoffConfig{
            Initial:    10 * time.Millisecond,
            Max:        100 * time.Millisecond,
            Multiplier: 2.0,
            MaxRetries: 3,
        },
        func(err error) bool { return true }, // 항상 재시도
        func(ctx context.Context) (string, error) {
            attempt++
            if attempt < 3 {
                return "", errors.New("일시적 오류")
            }
            return "성공", nil
        },
    )
    assert.NoError(t, err)
    assert.Equal(t, "성공", result)
    assert.Equal(t, 3, attempt)
}
```

## 9) 연계 스킬/명령

| 연계 대상 | 활용 방식 |
|----------|----------|
| `/audit` | `security-auditor`가 시크릿, 환경변수 보안 검증 |
| `/audit` | `perf-auditor`가 HTTP 클라이언트 타임아웃, 재시도 설정 검증 |
| `/audit` | `arch-auditor`가 인터페이스 설계, 의존성 방향 검증 |
| `/review` | `go-code-reviewer`가 패키지 전체 컨벤션 검증 |
| `/develop` | 새 패키지 생성 시 패턴 참조 |
| `infrastructure-layer` | DI 등록, Config 추가와 연동 |
| `interface-layer` | Repository/Delivery에서 pkg 패키지 사용 패턴 |
| `domain-layer` | JSONB 타입, 에러 처리 패턴 연동 |

## 10) 메트릭/거버넌스

| 메트릭 | 설명 | 임계값 |
|--------|------|--------|
| `인터페이스 미정의 서비스 수` | 구체 타입만 제공하는 서비스 | 0건 (MAJOR) |
| `시크릿 하드코딩 수` | 코드에 시크릿 노출 | 0건 (BLOCKING) |
| `os.Getenv 직접 호출 수` | config 우회 환경변수 읽기 | 0건 (BLOCKING) |
| `context.Context 누락 수` | I/O 함수에 ctx 없는 건 수 | 0건 (MAJOR) |
| `errors.Wrap 미사용 수` | 에러 컨텍스트 없이 반환 | 모니터링 |
| `APM 트레이싱 미적용 수` | 외부 호출에 span 없는 건 수 | 모니터링 |
| `테스트 커버리지` | 유틸리티 패키지 테스트 비율 | 70% 이상 권장 |
| `Mock 가용성` | 인터페이스에 대한 mock 존재 비율 | 100% 권장 |
