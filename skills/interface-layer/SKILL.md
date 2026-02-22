---
name: interface-layer
description: |
  인터페이스(포트/어댑터) 레이어 검증 및 가이드 스킬.
  HTTP Delivery, SQS Delivery, Repository 구현체를 검증하고 올바른 구현을 안내합니다.
  /audit, /review, /develop 커맨드에서 자동으로 활용됩니다.
---

# Interface Layer 스킬

## 1) 목적 요약

인터페이스 레이어는 외부 세계와 비즈니스 로직을 연결하는 **어댑터**입니다.

검증 대상 디렉토리:
- `internal/api/delivery/<entity>/http/` - HTTP 핸들러
- `internal/api/delivery/<entity>/sqs/` - SQS 핸들러
- `internal/repository/<entity>/postgresql/` - Repository 구현체

이 레이어의 핵심 원칙:
1. Delivery는 요청 파싱과 응답 반환만 담당 (비즈니스 로직 금지)
2. Repository 구현체는 도메인 인터페이스를 정확히 구현
3. 구체 구현이 도메인 레이어에 노출되지 않음 (인터페이스 통한 의존성 역전)

## 2) 책임 범위

| 항목 | 설명 |
|------|------|
| HTTP Delivery 검증 | Register 함수, 라우트 순서, Bind→Validate→Execute 패턴, Swagger 주석 |
| SQS Delivery 검증 | Runner 패턴, RegisterHandler, Start/Stop 생명주기 |
| Repository 구현 검증 | BaseRepository 임베딩, List 메서드, Scope 함수 |
| 미들웨어 적용 검증 | AdminOnly 적용 여부, UserID 주입 패턴 |
| HTTP 상태 코드 검증 | 201(Create), 200(Get/List), 204(Update/Delete) |

## 3) 입력/출력

### 입력
- HTTP Delivery: `internal/api/delivery/<entity>/http/<entity>_http.go`
- SQS Delivery: `internal/api/delivery/<entity>/sqs/<entity>_sqs.go`
- Repository: `internal/repository/<entity>/postgresql/<entity>_pg.go`

### 출력 형식
```
[심각도] 파일: <path>:<line>
  규칙: <위반된 규칙 ID>
  이슈: <구체적 설명>
  수정: <권장 수정 방법>
```

## 4) 체크리스트

### HTTP Delivery
- [ ] `Register(e *echo.Echo, usecase apidomain.<Entity>Usecase)` 함수 존재
- [ ] 라우트 순서: GET 목록 → GET 단건 → POST → PUT → DELETE
- [ ] Create 핸들러: Bind → GetUserID → Validate → Execute → `201 Created`
- [ ] Get 핸들러: Bind → Validate → Execute → `200 OK`
- [ ] List 핸들러: Bind → Execute → `200 OK`
- [ ] Update 핸들러: Bind → Validate → Execute → `204 NoContent`
- [ ] Delete 핸들러: Bind → Validate → Execute → `204 NoContent`
- [ ] Admin 전용 기능에 `echopkg.AdminOnly` 미들웨어 적용
- [ ] UserID 주입: `echopkg.GetUserID(c)` 사용 (Create/Update)
- [ ] Swagger 주석 존재 (Summary, Description, Tags, Param, Success, Router)
- [ ] Swagger Summary 한국어 작성
- [ ] Path parameter URL과 구조체 필드명 일치 (`:humanId` ↔ `param:"humanId"`)
- [ ] 핸들러에 비즈니스 로직 없음 (Usecase에 위임)

### SQS Delivery
- [ ] Runner 구조체 정의 (runner, dbFactory, usecase 필드)
- [ ] `New()` 생성자: Runner 생성 + SetCount 설정
- [ ] `Register()` 함수: RegisterHandler로 핸들러 등록
- [ ] `Start()` 메서드: `runner.StartWithDBFactory()` 호출
- [ ] `Stop()` 메서드: `runner.Stop()` 호출
- [ ] 핸들러에서 JSON Unmarshal → Usecase 호출 패턴

### Repository 구현체
- [ ] `domain.Repository[T]` 임베딩 (BaseRepository)
- [ ] `New(dbFactory *factory.DBFactory) domain.<Entity>Repository` 생성자
- [ ] `option.WithCustomNotFoundError()` 설정
- [ ] List 메서드: Count → Pagination/Orders → Find 순서
- [ ] Count는 필터링 Scope 적용 후 실행
- [ ] Pagination/Orders는 Count 이후 별도 Scopes로 적용
- [ ] 각 필터에 대한 `with<Field>()` Scope 함수 존재
- [ ] Scope 함수에서 빈 값 체크 (`len(ids) > 0`)

## 5) 권장 구현 패턴

### HTTP Delivery - Register 함수
```go
func Register(e *echo.Echo, usecase apidomain.HumanUsecase) {
    delivery := &humanHttpDelivery{usecase: usecase}

    humans := e.Group("/api/v1/humans")
    {
        humans.GET("", delivery.ListHumans)                              // 1. 목록
        humans.GET("/:humanId", delivery.GetHuman)                       // 2. 단건
        humans.POST("", delivery.CreateHuman, echopkg.AdminOnly)         // 3. 생성
        humans.PUT("/:humanId", delivery.UpdateHuman, echopkg.AdminOnly) // 4. 수정
        humans.DELETE("/:humanId", delivery.DeleteHuman, echopkg.AdminOnly) // 5. 삭제
    }
}
```

### HTTP Delivery - Create 핸들러
```go
// CreateHuman
// @Summary 사람 생성
// @Description 새로운 사람을 생성하는 API 입니다.
// @Tags Humans
// @Accept json
// @Produce json
// @Param request body apidomain.CreateHumanRequest true "사람 생성 요청"
// @Success 201 {object} apidomain.CreateHumanResponse
// @Security BearerAuth
// @Router /v1/humans [post]
func (d *humanHttpDelivery) CreateHuman(c echo.Context) error {
    var (
        request apidomain.CreateHumanRequest
        err     error
    )
    if err = c.Bind(&request); err != nil {
        return err
    }
    request.UserID, err = echopkg.GetUserID(c)
    if err != nil {
        return err
    }
    if err = c.Validate(&request); err != nil {
        return err
    }
    response, err := d.usecase.CreateHuman(c.Request().Context(), request)
    if err != nil {
        return err
    }
    return c.JSON(http.StatusCreated, response)
}
```

### Repository 구현체 - 생성자
```go
type humanPostgresqlRepository struct {
    domain.Repository[domain.Human]
    dbFactory *factory.DBFactory
}

func New(dbFactory *factory.DBFactory) domain.HumanRepository {
    return &humanPostgresqlRepository{
        Repository: repository.NewBaseRepository[domain.Human](dbFactory,
            option.WithCustomNotFoundError(domain.ErrHumanNotFound)),
        dbFactory: dbFactory,
    }
}
```

### Repository 구현체 - List 메서드
```go
func (repo *humanPostgresqlRepository) List(ctx context.Context, opts ...domain.HumanQueryOption) (int64, domain.Humans, error) {
    var (
        count  int64
        result domain.Humans
    )

    queryOpts := domain.HumanQueryOptionFunc()
    for _, opt := range opts {
        opt(&queryOpts)
    }

    // 1. 필터링 Scope 적용
    db := repo.dbFactory.GetDB(ctx).Model(&result).Scopes(
        withIDs(queryOpts.IDs()),
        withUserIDs(queryOpts.UserIDs()),
    )

    // 2. Count (필터 적용 후)
    if err := db.Count(&count).Error; err != nil {
        return 0, nil, errors.Wrap(err, "failed to count humans")
    }

    // 3. Pagination + Orders (Count 이후)
    db = db.Scopes(
        repository.WithOrders(queryOpts.Orders()...),
        repository.WithPagination(queryOpts.Pagination()),
    )

    // 4. Find
    if err := db.Find(&result).Error; err != nil {
        return 0, nil, errors.Wrap(err, "failed to find humans")
    }

    return count, result, nil
}
```

### Repository 구현체 - Scope 함수
```go
func withIDs(ids []uint) func(db *gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        if len(ids) > 0 {
            return db.Where("id IN ?", ids)
        }
        return db
    }
}

func withUserIDs(userIDs []uint) func(db *gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        if len(userIDs) > 0 {
            return db.Where("user_id IN ?", userIDs)
        }
        return db
    }
}

// 포인터 타입 필터
func withStatus(status *domain.HumanStatus) func(db *gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        if status != nil {
            return db.Where("status = ?", *status)
        }
        return db
    }
}
```

### SQS Delivery 패턴
```go
type HumanRunner struct {
    runner    *runner.Runner
    dbFactory *factory.DBFactory
    usecase   apidomain.HumanUsecase
}

func New(awsCfg *aws.Config, cfg *config.Config, dbFactory *factory.DBFactory,
    usecase apidomain.HumanUsecase, pglock pg_lock.PostgresLock) *HumanRunner {
    sqsClient := queuefactory.NewWithQueueName(awsCfg, cfg.AWS.SQS.Human.QueueName)
    r := runner.New(cfg, sqsClient, pglock)
    r.SetCount(cfg.AWS.SQS.Human.RunnerCount)
    return &HumanRunner{runner: r, dbFactory: dbFactory, usecase: usecase}
}

func Register(h *HumanRunner) {
    h.runner.RegisterHandler("HumanUsecase.ProcessHuman", runner.Handle(h.usecase.ProcessHuman))
}

func (h *HumanRunner) Start(ctx context.Context, options ...queue.MessageOption) {
    h.runner.StartWithDBFactory(ctx, h.dbFactory, options...)
}

func (h *HumanRunner) Stop() { h.runner.Stop() }
```

## 6) 안티패턴 탐지 규칙

### BLOCKING

| ID | 안티패턴 | 올바른 패턴 |
|----|---------|-----------|
| I-001 | 핸들러에 비즈니스 로직 존재 | Usecase에 위임 |
| I-002 | 핸들러에서 직접 Repository 호출 | Usecase를 통해서만 접근 |
| I-003 | Repository에서 HTTP 관련 코드 사용 | Repository는 DB 접근만 |
| I-004 | Scope 함수에서 빈 값 체크 누락 | `if len(ids) > 0` 필수 |

### MAJOR

| ID | 안티패턴 | 올바른 패턴 |
|----|---------|-----------|
| I-005 | 라우트 순서 위반 (POST가 GET 앞에) | GET 목록→GET 단건→POST→PUT→DELETE |
| I-006 | Create에 200 반환 | 201 StatusCreated |
| I-007 | Update/Delete에 200 반환 | 204 StatusNoContent |
| I-008 | AdminOnly 미들웨어 누락 (CUD 작업) | 생성/수정/삭제에 AdminOnly 적용 |
| I-009 | Create에서 GetUserID 미호출 | UserID 자동 주입 필수 |
| I-010 | Count 전에 Pagination 적용 | Count → Pagination 순서 |
| I-011 | BaseRepository 미임베딩 | `repository.NewBaseRepository[T]` 사용 |

### MINOR

| ID | 안티패턴 | 올바른 패턴 |
|----|---------|-----------|
| I-012 | Swagger 주석 누락 | 모든 핸들러에 Swagger 주석 |
| I-013 | Swagger Summary 영어 작성 | 한국어 작성 |
| I-014 | Path parameter 이름 불일치 | URL `:humanId` = param tag `humanId` |
| I-015 | Repository 에러 메시지 영어 미사용 | errors.Wrap 메시지는 영어 |

## 7) 샘플 코드 스니펫

### 복잡한 Scope 함수 (OR 조건)
```go
func withNameSynonymsMatches(matches []string) func(db *gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        if len(matches) == 0 {
            return db
        }
        conditions := db.Session(&gorm.Session{NewDB: true})
        for i, match := range matches {
            if i == 0 {
                conditions = conditions.Where("name ILIKE ? OR ? = ANY(SELECT lower(unnest(synonyms)))", match, strings.ToLower(match))
            } else {
                conditions = conditions.Or("name ILIKE ? OR ? = ANY(SELECT lower(unnest(synonyms)))", match, strings.ToLower(match))
            }
        }
        return db.Where(conditions)
    }
}
```

### 커서 기반 페이지네이션 Repository
```go
func withCursorPagination(cursor *domain.CursorPaginationOption) func(db *gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        if cursor == nil {
            return db
        }
        if cursor.LastCursorID > 0 {
            if cursor.IDOrderDirection == domain.DirectionDESC {
                db = db.Where("id < ?", cursor.LastCursorID)
            } else {
                db = db.Where("id > ?", cursor.LastCursorID)
            }
        }
        return db.Limit(cursor.Limit)
    }
}
```

### SQS 핸들러 + 재시도 옵션
```go
func Register(c *CampaignRunner) {
    c.runner.RegisterHandler(
        "CampaignUsecase.RecommendCompetitors",
        runner.Handle(c.usecase.RecommendCompetitors),
        runner.WithRetry(2),
        runner.WithRetryCondition(runner.DefaultRetryCondition),
        runner.WithRetryDelay(500 * time.Millisecond),
        runner.WithoutTransaction(),
    )
}
```

## 8) 테스트 지침

### HTTP Delivery 테스트
HTTP Delivery는 주로 **통합 테스트**로 검증합니다.

```go
func TestCreateHuman(t *testing.T) {
    // Echo 인스턴스 + Mock Usecase 설정
    e := echo.New()
    e.Validator = validator.NewValidator()
    mockUsecase := mocks.NewHumanUsecase(t)

    Register(e, mockUsecase)

    tests := []struct {
        name       string
        body       string
        setupMock  func()
        wantStatus int
    }{
        {
            name: "정상 생성",
            body: `{"name":"홍길동","age":30}`,
            setupMock: func() {
                mockUsecase.On("CreateHuman", mock.Anything, mock.Anything).
                    Return(apidomain.CreateHumanResponse{HumanID: 1}, nil)
            },
            wantStatus: http.StatusCreated,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            tt.setupMock()
            req := httptest.NewRequest(http.MethodPost, "/api/v1/humans", strings.NewReader(tt.body))
            req.Header.Set(echo.HeaderContentType, echo.MIMEApplicationJSON)
            rec := httptest.NewRecorder()
            e.ServeHTTP(rec, req)
            assert.Equal(t, tt.wantStatus, rec.Code)
        })
    }
}
```

### Repository 테스트
Repository는 **테스트 DB**를 사용한 통합 테스트를 권장합니다.

```go
func TestHumanRepository_List(t *testing.T) {
    db := setupTestDB(t) // 테스트 DB 설정
    repo := postgresql.New(factory.NewDBFactory(db))

    // 테스트 데이터 삽입
    repo.Create(ctx, domain.Human{Name: "홍길동", UserID: 1})
    repo.Create(ctx, domain.Human{Name: "김철수", UserID: 2})

    count, humans, err := repo.List(ctx,
        domain.HumanQueryOptionFunc().WithUserIDs(1),
    )

    assert.NoError(t, err)
    assert.Equal(t, int64(1), count)
    assert.Len(t, humans, 1)
    assert.Equal(t, "홍길동", humans[0].Name)
}
```

## 9) 연계 스킬/명령

| 연계 대상 | 활용 방식 |
|----------|----------|
| `/audit` | `arch-auditor`가 Delivery↔Usecase 의존성 방향 검증 |
| `/audit` | `security-auditor`가 AdminOnly, GetUserID 적용 검증 |
| `/audit` | `perf-auditor`가 Repository의 Count+Find, N+1 검증 |
| `/review` | `go-code-reviewer`가 전체 체크리스트 적용 |
| `/develop` | 새 Delivery/Repository 생성 시 패턴 참조 |
| `domain-layer` | Repository 인터페이스 시그니처와 구현체 일치 확인 |
| `usecase-layer` | Delivery→Usecase 호출 패턴 일관성 |
| `infrastructure-layer` | DI 등록 여부 연동 확인 |

## 10) 메트릭/거버넌스

| 메트릭 | 설명 | 임계값 |
|--------|------|--------|
| `비즈니스 로직 핸들러 침투 수` | Delivery에 비즈니스 로직이 있는 건 수 | 0건 (BLOCKING) |
| `라우트 순서 위반 수` | GET→POST→PUT→DELETE 순서 위반 | 0건 (MAJOR) |
| `AdminOnly 누락 수` | CUD에 AdminOnly 없는 건 수 | 0건 (MAJOR) |
| `HTTP 상태 코드 오류 수` | 잘못된 상태 코드 반환 | 0건 (MAJOR) |
| `Scope 빈 값 체크 누락` | len 체크 없는 Scope 수 | 0건 (BLOCKING) |
| `Count-Pagination 순서 오류` | Count 전 Pagination 적용 | 0건 (MAJOR) |
| `Swagger 주석 커버리지` | Swagger 주석이 있는 핸들러 비율 | 100% 권장 |
