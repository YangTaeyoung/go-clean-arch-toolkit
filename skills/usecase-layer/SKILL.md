---
name: usecase-layer
description: |
  유스케이스(인터랙터) 레이어 검증 및 가이드 스킬.
  CRUD 구현, DTO 변환, 에러 처리, 트랜잭션 경계, API Domain 정의를 검증하고 올바른 구현을 안내합니다.
  /audit, /review, /develop 커맨드에서 자동으로 활용됩니다.
---

# Usecase Layer 스킬

## 1) 목적 요약

`internal/api/usecase/` 및 `internal/api/domain/` 디렉토리의 유스케이스 레이어를 검증합니다.
이 레이어는 **비즈니스 로직 오케스트레이션**을 담당하며, Repository 호출과 DTO 변환을 수행합니다.

검증 대상:
- API Domain (Usecase 인터페이스, Request/Response DTO)
- Usecase 구현체 (CRUD 메서드, 에러 처리)
- DTO 변환 패턴 (인라인 변환 필수)
- 트랜잭션 경계
- QueryOption 활용 패턴

## 2) 책임 범위

| 항목 | 설명 |
|------|------|
| API Domain 검증 | Usecase 인터페이스, DTO 구조체, JSON 태그, validate 태그 |
| Usecase 구현 검증 | CRUD 메서드 패턴, 생성자 패턴 |
| DTO 변환 검증 | 인라인 변환 준수, 헬퍼 메서드 사용 금지 |
| 에러 처리 검증 | 도메인 에러 반환, gorm.ErrRecordNotFound 처리 |
| 트랜잭션 경계 검증 | Repository 위임, ImmediateUpdate 사용 적절성 |
| 의존성 주입 검증 | 생성자 함수 시그니처, 인터페이스 반환 |

## 3) 입력/출력

### 입력
- API Domain 파일: `internal/api/domain/<entity>.go`
- Usecase 파일: `internal/api/usecase/<entity>/<entity>_ucase.go`

### 출력 형식
```
[심각도] 파일: <path>:<line>
  규칙: <위반된 규칙 ID>
  이슈: <구체적 설명>
  수정: <권장 수정 방법>
```

## 4) 체크리스트

### API Domain (internal/api/domain/)
- [ ] Usecase 인터페이스 메서드 시그니처: `(ctx context.Context, request <Type>) (<Response>, error)`
- [ ] JSON 태그가 `lowerCamelCase`인지 (snake_case 금지)
- [ ] Request 복수형 필드: Go 필드명은 복수형, JSON/Query 태그는 단수형 (`IDs []uint \`query:"id"\``)
- [ ] Path Parameter: `param` 태그 + `swaggerignore:"true"` 포함
- [ ] UserID 필드: `json:"-" swaggerignore:"true"` 태그
- [ ] Create Response: 생성된 ID만 반환 (전체 엔티티 아님)
- [ ] 모든 필드에 한국어 주석 존재
- [ ] `validate:"required"` 태그 적절성

### Usecase 구현체
- [ ] 생성자 `New()` 함수가 인터페이스 타입 반환
- [ ] Create: Repository.Create → ID 반환
- [ ] Get: Repository.Get → 인라인 DTO 변환
- [ ] List: QueryOption 배열 구성 → Repository.List → 인라인 DTO 변환
- [ ] Update: Repository.Get → 필드 할당 → Repository.Update
- [ ] Delete: Repository.Get (존재 확인) → Repository.Delete

### DTO 변환
- [ ] 인라인 변환만 사용 (별도 헬퍼 함수/메서드 금지)
- [ ] List의 DTO 변환: `make([]DTO, len(entities))` + for loop
- [ ] `lo.FromPtr()` 사용하여 optional 필터 nil 체크

### 에러 처리
- [ ] Repository 에러를 그대로 반환 (불필요한 래핑 금지)
- [ ] NotFound 에러: `domain.Err<Entity>NotFound` 사용
- [ ] 비즈니스 규칙 위반: 도메인 에러 직접 반환

## 5) 권장 구현 패턴

### API Domain 패턴
```go
package domain

import "context"

type HumanUsecase interface {
    CreateHuman(ctx context.Context, request CreateHumanRequest) (CreateHumanResponse, error)
    GetHuman(ctx context.Context, request GetHumanRequest) (GetHumanResponse, error)
    ListHumans(ctx context.Context, request ListHumansRequest) (ListHumansResponse, error)
    UpdateHuman(ctx context.Context, request UpdateHumanRequest) error
    DeleteHuman(ctx context.Context, request DeleteHumanRequest) error
}

// Create Request
type CreateHumanRequest struct {
    Name   string `json:"name" validate:"required"`                   // 이름 (필수)
    Age    int    `json:"age" validate:"required"`                    // 나이 (필수)
    UserID uint   `json:"-" swaggerignore:"true" validate:"required"` // 작성자 ID
}

// Create Response - ID만 반환
type CreateHumanResponse struct {
    HumanID uint `json:"humanId"` // 생성된 ID
}

// Get Request - param 태그 사용
type GetHumanRequest struct {
    HumanID uint `param:"humanId" swaggerignore:"true" validate:"required"` // 조회 ID
}

type GetHumanResponse struct {
    Human HumanDTO `json:"human"` // 조회된 정보
}

// DTO - lowerCamelCase JSON
type HumanDTO struct {
    ID        uint      `json:"id"`        // 고유 ID
    Name      string    `json:"name"`      // 이름
    Age       int       `json:"age"`       // 나이
    CreatedAt time.Time `json:"createdAt"` // 생성일시
    UpdatedAt time.Time `json:"updatedAt"` // 수정일시
}

// List Request - 복수 필드는 Go 복수형 + JSON 단수형
type ListHumansRequest struct {
    IDs     []uint `query:"id" json:"id"`         // 조회 ID 목록
    UserIDs []uint `query:"userId" json:"userId"`  // 작성자 ID 목록
}

type ListHumansResponse struct {
    Humans []HumanDTO `json:"humans"` // 목록
}

// Update Request
type UpdateHumanRequest struct {
    ID   uint   `param:"humanId" swaggerignore:"true" validate:"required"` // 수정 ID
    Name string `json:"name,omitempty"`                                    // 수정할 이름
    Age  int    `json:"age,omitempty"`                                     // 수정할 나이
}

// Delete Request
type DeleteHumanRequest struct {
    HumanID uint `param:"humanId" swaggerignore:"true" validate:"required"` // 삭제 ID
}
```

### Usecase 생성자 패턴
```go
package human

import (
    apidomain "github.com/chainshiftlabs/console-api-go/internal/api/domain"
    "github.com/chainshiftlabs/console-api-go/internal/domain"
)

type humanUsecase struct {
    humanRepository domain.HumanRepository
}

func New(humanRepository domain.HumanRepository) apidomain.HumanUsecase {
    return &humanUsecase{humanRepository: humanRepository}
}
```

### Create 패턴
```go
func (u *humanUsecase) CreateHuman(ctx context.Context, request apidomain.CreateHumanRequest) (apidomain.CreateHumanResponse, error) {
    humanID, err := u.humanRepository.Create(ctx, domain.Human{
        Name:   request.Name,
        Age:    request.Age,
        UserID: request.UserID,
    })
    if err != nil {
        return apidomain.CreateHumanResponse{}, err
    }
    return apidomain.CreateHumanResponse{HumanID: humanID}, nil
}
```

### Get 패턴 (인라인 DTO 변환)
```go
func (u *humanUsecase) GetHuman(ctx context.Context, request apidomain.GetHumanRequest) (apidomain.GetHumanResponse, error) {
    human, err := u.humanRepository.Get(ctx, request.HumanID)
    if err != nil {
        return apidomain.GetHumanResponse{}, err
    }
    return apidomain.GetHumanResponse{
        Human: apidomain.HumanDTO{
            ID:        human.ID,
            Name:      human.Name,
            Age:       human.Age,
            CreatedAt: human.CreatedAt,
            UpdatedAt: human.UpdatedAt,
        },
    }, nil
}
```

### List 패턴 (QueryOption + 인라인 DTO)
```go
func (u *humanUsecase) ListHumans(ctx context.Context, request apidomain.ListHumansRequest) (apidomain.ListHumansResponse, error) {
    options := []domain.HumanQueryOption{
        domain.HumanQueryOptionFunc().WithIDs(request.IDs...),
        domain.HumanQueryOptionFunc().WithUserIDs(request.UserIDs...),
    }

    _, humans, err := u.humanRepository.List(ctx, options...)
    if err != nil {
        return apidomain.ListHumansResponse{}, err
    }

    dtos := make([]apidomain.HumanDTO, len(humans))
    for i, human := range humans {
        dtos[i] = apidomain.HumanDTO{
            ID:        human.ID,
            Name:      human.Name,
            Age:       human.Age,
            CreatedAt: human.CreatedAt,
            UpdatedAt: human.UpdatedAt,
        }
    }
    return apidomain.ListHumansResponse{Humans: dtos}, nil
}
```

### Update 패턴
```go
func (u *humanUsecase) UpdateHuman(ctx context.Context, request apidomain.UpdateHumanRequest) error {
    human, err := u.humanRepository.Get(ctx, request.ID)
    if err != nil {
        return err
    }
    human.Name = request.Name
    human.Age = request.Age
    return u.humanRepository.Update(ctx, human)
}
```

### Delete 패턴
```go
func (u *humanUsecase) DeleteHuman(ctx context.Context, request apidomain.DeleteHumanRequest) error {
    if _, err := u.humanRepository.Get(ctx, request.HumanID); err != nil {
        return err
    }
    return u.humanRepository.Delete(ctx, request.HumanID)
}
```

## 6) 안티패턴 탐지 규칙

### BLOCKING

| ID | 안티패턴 | 올바른 패턴 |
|----|---------|-----------|
| U-001 | DTO 변환 헬퍼 함수 사용 (`toDTO()`, `fromEntity()`) | 인라인 변환만 허용 |
| U-002 | JSON 태그에 snake_case 사용 | lowerCamelCase 필수 |
| U-003 | Create Response에 전체 엔티티 반환 | ID만 반환 |
| U-004 | Usecase에서 직접 DB 쿼리 (gorm 호출) | Repository를 통해서만 접근 |

### MAJOR

| ID | 안티패턴 | 올바른 패턴 |
|----|---------|-----------|
| U-005 | Update에서 기존 엔티티 조회 없이 직접 Update | Get → 필드 할당 → Update |
| U-006 | Delete에서 존재 확인 없이 바로 삭제 | Get → Delete |
| U-007 | path parameter에 `swaggerignore:"true"` 누락 | `param` + `swaggerignore:"true"` 필수 |
| U-008 | 에러를 불필요하게 래핑 | Repository 에러 그대로 전달 |
| U-009 | UserID 필드에 `json:"-"` 누락 | 인증 필드는 API에 노출 금지 |

### MINOR

| ID | 안티패턴 | 올바른 패턴 |
|----|---------|-----------|
| U-010 | 필드에 한국어 주석 누락 | 모든 DTO 필드에 한국어 주석 |
| U-011 | List에서 count 반환값 사용하지 않으면서 변수 할당 | `_`로 무시 |
| U-012 | Optional 필터에 `lo.FromPtr()` 미사용 | nil 체크 시 `lo.FromPtr()` 사용 |

## 7) 샘플 코드 스니펫

### Nullable 필드가 있는 Update
```go
func (u *noticeUsecase) UpdateNotice(ctx context.Context, request apidomain.UpdateNoticeRequest) error {
    notice, err := u.noticeRepository.Get(ctx, request.ID)
    if err != nil {
        return err
    }

    notice.Title = request.Title
    notice.Content = request.Content
    notice.Draft = lo.ToPtr(request.Draft)
    notice.ExpiresAt = request.ExpiresAt

    // Nullable 필드를 명시적으로 NULL로 설정해야 할 때
    var updateOpts []option.UpdateOption
    if request.ExpiresAt == nil {
        updateOpts = append(updateOpts, option.WithUpdateFieldArgs(map[string]any{
            "expires_at": nil,
        }))
    }

    return u.noticeRepository.Update(ctx, notice, updateOpts...)
}
```

### 조건부 QueryOption 구성
```go
func (u *noticeUsecase) ListNotices(ctx context.Context, request apidomain.ListNoticesRequest) (apidomain.ListNoticesResponse, error) {
    options := []domain.NoticeQueryOption{
        domain.NoticeQueryOptionFunc().WithIDs(request.IDs...),
        domain.NoticeQueryOptionFunc().WithOrders(
            domain.OrderOption{Column: "created_at", Direction: "DESC"},
        ),
    }

    // Optional 필터: nil이 아닐 때만 추가
    if request.Draft != nil {
        options = append(options, domain.NoticeQueryOptionFunc().WithDraft(lo.FromPtr(request.Draft)))
    }
    if request.Expired != nil {
        options = append(options, domain.NoticeQueryOptionFunc().WithExpired(lo.FromPtr(request.Expired)))
    }

    _, notices, err := u.noticeRepository.List(ctx, options...)
    if err != nil {
        return apidomain.ListNoticesResponse{}, err
    }

    dtos := make([]apidomain.NoticeDTO, len(notices))
    for i, notice := range notices {
        dtos[i] = apidomain.NoticeDTO{
            ID:        notice.ID,
            Title:     notice.Title,
            Content:   notice.Content,
            CreatedAt: notice.CreatedAt,
            UpdatedAt: notice.UpdatedAt,
        }
    }
    return apidomain.ListNoticesResponse{Notices: dtos}, nil
}
```

### SQS 큐 메시지 전송 (커밋 후)
```go
func (u *taskUsecase) CompleteTask(ctx context.Context, request apidomain.CompleteTaskRequest) error {
    task, err := u.taskRepository.Get(ctx, request.TaskID)
    if err != nil {
        return err
    }
    task.Status = domain.TaskStatusCompleted
    if err := u.taskRepository.Update(ctx, task); err != nil {
        return err
    }

    // 트랜잭션 커밋 후 비동기 작업 전송
    return u.statisticQueue.SendMessageAfterCommit(ctx, apidomain.AggregateStatisticRequest{
        TaskID: task.ID,
    }, map[string]string{
        runner.FuncAttribute: "StatisticUsecase.AggregateStatistic",
    })
}
```

## 8) 테스트 지침

### 테스트 대상
- 모든 Usecase 메서드 (Create, Get, List, Update, Delete)
- 비즈니스 로직이 포함된 커스텀 메서드
- 에러 처리 경로

### 테스트 도구
- `mockery`: Repository 인터페이스 모킹
- `testify`: assertion + suite

### 테스트 패턴 (table-driven)
```go
func TestHumanUsecase_CreateHuman(t *testing.T) {
    tests := []struct {
        name    string
        request apidomain.CreateHumanRequest
        mockFn  func(repo *mocks.HumanRepository)
        want    apidomain.CreateHumanResponse
        wantErr bool
    }{
        {
            name: "정상 생성",
            request: apidomain.CreateHumanRequest{
                Name: "홍길동", Age: 30, UserID: 1,
            },
            mockFn: func(repo *mocks.HumanRepository) {
                repo.On("Create", mock.Anything, mock.MatchedBy(func(h domain.Human) bool {
                    return h.Name == "홍길동" && h.Age == 30
                })).Return(uint(1), nil)
            },
            want:    apidomain.CreateHumanResponse{HumanID: 1},
            wantErr: false,
        },
        {
            name: "Repository 에러",
            request: apidomain.CreateHumanRequest{
                Name: "에러", Age: 0, UserID: 1,
            },
            mockFn: func(repo *mocks.HumanRepository) {
                repo.On("Create", mock.Anything, mock.Anything).
                    Return(uint(0), errors.New("db error"))
            },
            want:    apidomain.CreateHumanResponse{},
            wantErr: true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            repo := mocks.NewHumanRepository(t)
            tt.mockFn(repo)
            uc := human.New(repo)

            got, err := uc.CreateHuman(context.Background(), tt.request)
            if tt.wantErr {
                assert.Error(t, err)
            } else {
                assert.NoError(t, err)
                assert.Equal(t, tt.want, got)
            }
        })
    }
}
```

### Mock 범위
- Repository: 항상 mock (DB 접근 차단)
- Service: mock (외부 서비스 차단)
- Queue: mock (SQS 접근 차단)
- Config: 테스트용 값 직접 주입

## 9) 연계 스킬/명령

| 연계 대상 | 활용 방식 |
|----------|----------|
| `/audit` | `pattern-auditor`가 JSON 태그, DTO 변환 패턴 검증 |
| `/review` | `go-code-reviewer`가 CRUD 메서드 패턴 준수 검증 |
| `/develop` | 새 Usecase 생성 시 체크리스트 + 패턴 참조 |
| `domain-layer` | Repository 인터페이스 시그니처 일관성 확인 |
| `interface-layer` | Delivery → Usecase 호출 패턴 일관성 확인 |

## 10) 메트릭/거버넌스

| 메트릭 | 설명 | 임계값 |
|--------|------|--------|
| `DTO 헬퍼 함수 사용 수` | 인라인 변환 외 헬퍼 사용 | 0건 (BLOCKING) |
| `snake_case JSON 태그 수` | lowerCamelCase 위반 | 0건 (BLOCKING) |
| `swaggerignore 누락 수` | param 태그에 swaggerignore 누락 | 0건 (MAJOR) |
| `한국어 주석 누락률` | DTO 필드 중 주석 없는 비율 | 모니터링 |
| `테스트 커버리지` | Usecase 메서드 테스트 비율 | 80% 이상 권장 |
