---
name: domain-layer
description: |
  도메인(엔티티) 레이어 검증 및 가이드 스킬.
  엔티티 정의, Repository 인터페이스, QueryOption 패턴, Enum 패턴, 에러 정의를 검증하고 올바른 구현을 안내합니다.
  /audit, /review, /develop 커맨드에서 자동으로 활용됩니다.
---

# Domain Layer 스킬

## 1) 목적 요약

`internal/domain/` 디렉토리에 위치하는 도메인 레이어의 정합성을 검증합니다.
이 레이어는 **순수 비즈니스 엔티티와 인터페이스만** 포함하며, 외부 의존성(HTTP, DB 드라이버 등)을 가져서는 안 됩니다.

검증 대상:
- 엔티티 구조체 정의
- Repository 인터페이스
- QueryOption 패턴
- Enum(상수) 패턴
- 도메인 에러 정의
- 복수형 타입 및 유틸리티 메서드

## 2) 책임 범위

| 항목 | 설명 |
|------|------|
| 엔티티 구조 검증 | gorm.Model 임베딩, GetID(), 복수형 타입, FK 필드 규칙 |
| Repository 인터페이스 검증 | Repository[T] 임베딩, List 메서드 시그니처 |
| QueryOption 검증 | 함수형 옵션 패턴, With*/Getter 메서드 쌍 |
| Enum 검증 | String 기반 const, Name() 메서드 |
| 에러 정의 검증 | Error 구조체 사용, Code/Message/Status 패턴 |
| 의존성 방향 검증 | 외부 패키지 import 금지 (gorm, lo, pq만 허용) |

## 3) 입력/출력

### 입력
- 대상 파일 경로: `internal/domain/<entity>.go`
- 선택적으로 특정 엔티티명 지정 가능

### 출력 형식
```
[심각도] 파일: <path>:<line>
  규칙: <위반된 규칙 ID>
  이슈: <구체적 설명>
  수정: <권장 수정 방법>
```

심각도: `BLOCKING` | `MAJOR` | `MINOR` | `NIT`

## 4) 체크리스트

### 엔티티 구조체
- [ ] `gorm.Model` 임베딩 여부
- [ ] `GetID() uint` 메서드 존재 여부
- [ ] 복수형 타입 정의 여부 (예: `type Users []User`)
- [ ] FK 필드가 `uint` 타입만 사용하는지 (엔티티 임베딩 금지)
- [ ] Nullable 필드에 포인터 타입(`*T`) 사용 여부
- [ ] 엔티티명이 단수형인지

### Repository 인터페이스
- [ ] `Repository[T]` 임베딩 여부
- [ ] `List(ctx context.Context, options ...<Entity>QueryOption) (int64, <Entities>, error)` 시그니처
- [ ] 커스텀 메서드의 `context.Context` 첫 번째 파라미터 여부

### QueryOption 패턴
- [ ] `type <Entity>QueryOption = func(*<Entity>QueryOptionFields)` 정의
- [ ] `<Entity>QueryOptionFunc()` 팩토리 함수 존재
- [ ] 모든 필드에 `With<Field>()` 설정 메서드 존재
- [ ] 모든 필드에 `<Field>()` getter 메서드 존재
- [ ] `pagination` 및 `orders` 필드 포함 여부

### Enum 패턴
- [ ] `type <Name> string` 형태 정의
- [ ] `const` 블록으로 값 정의 (접두사 패턴: `<Type><Value>`)
- [ ] `Name() string` 메서드 (한국어 디스플레이) 존재 여부

### 에러 정의
- [ ] `domain.Error` 구조체 사용 (Code, Message, Status)
- [ ] 에러 코드 네이밍: `Err<Domain><Description>` 패턴
- [ ] HTTP 상태 코드 적절성

## 5) 권장 구현 패턴

### 엔티티 정의 패턴
```go
package domain

import "gorm.io/gorm"

// Human 엔티티
type Human struct {
    gorm.Model
    UserID uint    `gorm:"not null;index"`       // FK: uint 타입만
    Name   string  `gorm:"not null;size:255"`     // 필수 필드
    Age    *int    `gorm:"index"`                 // Nullable: 포인터
}

func (h Human) GetID() uint {
    return h.ID
}

type Humans []Human

func (hs Humans) IDs() []uint {
    ids := make([]uint, len(hs))
    for i, h := range hs {
        ids[i] = h.ID
    }
    return ids
}
```

### Repository 인터페이스 패턴
```go
type HumanRepository interface {
    Repository[Human]
    List(ctx context.Context, options ...HumanQueryOption) (int64, Humans, error)
}
```

### QueryOption 패턴
```go
type HumanQueryOption = func(*HumanQueryOptionFields)

type HumanQueryOptionFields struct {
    ids        []uint
    userIDs    []uint
    pagination *PaginationOption
    orders     []OrderOption
}

func HumanQueryOptionFunc() HumanQueryOptionFields {
    return HumanQueryOptionFields{}
}

func (o HumanQueryOptionFields) WithIDs(ids ...uint) HumanQueryOption {
    return func(f *HumanQueryOptionFields) { f.ids = ids }
}

func (o HumanQueryOptionFields) IDs() []uint { return o.ids }

func (o HumanQueryOptionFields) WithPagination(p *PaginationOption) HumanQueryOption {
    return func(f *HumanQueryOptionFields) { f.pagination = p }
}

func (o HumanQueryOptionFields) Pagination() *PaginationOption { return o.pagination }

func (o HumanQueryOptionFields) WithOrders(orders ...OrderOption) HumanQueryOption {
    return func(f *HumanQueryOptionFields) { f.orders = orders }
}

func (o HumanQueryOptionFields) Orders() []OrderOption { return o.orders }
```

### Enum 패턴
```go
type HumanStatus string

const (
    HumanStatusActive   HumanStatus = "ACTIVE"
    HumanStatusInactive HumanStatus = "INACTIVE"
)

func (s HumanStatus) Name() string {
    switch s {
    case HumanStatusActive:
        return "활성"
    case HumanStatusInactive:
        return "비활성"
    default:
        return ""
    }
}
```

### 에러 정의 패턴
```go
var (
    ErrHumanNotFound = &Error{
        Code:    "HUMAN_NOT_FOUND",
        Message: "사람을 찾을 수 없습니다",
        Status:  http.StatusNotFound,
    }
)
```

## 6) 안티패턴 탐지 규칙

### BLOCKING

| ID | 안티패턴 | 올바른 패턴 |
|----|---------|-----------|
| D-001 | FK 필드에 엔티티 타입 임베딩 (`Car Car`) | `CarID uint` 만 사용 |
| D-002 | `gorm.Model` 미임베딩 | 모든 엔티티에 `gorm.Model` 필수 |
| D-003 | `GetID() uint` 메서드 누락 | 모든 엔티티에 필수 구현 |
| D-004 | 외부 패키지 import (net/http, echo 등) | domain 레이어는 순수 타입만 |

### MAJOR

| ID | 안티패턴 | 올바른 패턴 |
|----|---------|-----------|
| D-005 | 복수형 타입 미정의 | `type Entities []Entity` 필수 |
| D-006 | Repository에 `Repository[T]` 미임베딩 | 기본 CRUD 포함 필수 |
| D-007 | QueryOption With/Getter 메서드 쌍 불일치 | 모든 필드에 쌍 필수 |
| D-008 | Nullable 필드에 값 타입 사용 | 포인터 타입 `*T` 사용 |

### MINOR

| ID | 안티패턴 | 올바른 패턴 |
|----|---------|-----------|
| D-009 | Enum에 `Name()` 메서드 누락 | 한국어 디스플레이 메서드 추가 |
| D-010 | 엔티티명이 복수형 | 단수형 사용 (User, not Users) |
| D-011 | QueryOption에 pagination/orders 필드 누락 | 기본 필드로 포함 |

## 7) 샘플 코드 스니펫

### 완전한 엔티티 파일 (신규 생성 시 참고)
```go
package domain

import (
    "context"
    "time"

    "github.com/samber/lo"
    "gorm.io/gorm"
)

// === 엔티티 ===

type Review struct {
    gorm.Model
    ProductID  uint          `gorm:"not null;index"`
    UserID     uint          `gorm:"not null;index"`
    Title      string        `gorm:"not null;size:255"`
    Content    string        `gorm:"type:text;not null"`
    Rating     int           `gorm:"not null"`
    Status     ReviewStatus  `gorm:"not null;size:50;default:PENDING"`
    ApprovedAt *time.Time    `gorm:"index"`
}

func (r Review) GetID() uint { return r.ID }

// === 복수형 ===

type Reviews []Review

func (rs Reviews) IDs() []uint {
    ids := make([]uint, len(rs))
    for i, r := range rs { ids[i] = r.ID }
    return ids
}

func (rs Reviews) UserIDs() []uint {
    userIDs := make([]uint, len(rs))
    for i, r := range rs { userIDs[i] = r.UserID }
    return lo.Uniq(userIDs)
}

// === Enum ===

type ReviewStatus string

const (
    ReviewStatusPending  ReviewStatus = "PENDING"
    ReviewStatusApproved ReviewStatus = "APPROVED"
    ReviewStatusRejected ReviewStatus = "REJECTED"
)

func (s ReviewStatus) Name() string {
    switch s {
    case ReviewStatusPending:  return "승인 대기"
    case ReviewStatusApproved: return "승인됨"
    case ReviewStatusRejected: return "거부됨"
    default: return ""
    }
}

// === Repository ===

type ReviewRepository interface {
    Repository[Review]
    List(ctx context.Context, options ...ReviewQueryOption) (int64, Reviews, error)
}

// === QueryOption ===

type ReviewQueryOption = func(*ReviewQueryOptionFields)

type ReviewQueryOptionFields struct {
    ids        []uint
    userIDs    []uint
    productIDs []uint
    statuses   []ReviewStatus
    pagination *PaginationOption
    orders     []OrderOption
}

func ReviewQueryOptionFunc() ReviewQueryOptionFields {
    return ReviewQueryOptionFields{}
}

func (o ReviewQueryOptionFields) WithIDs(ids ...uint) ReviewQueryOption {
    return func(f *ReviewQueryOptionFields) { f.ids = ids }
}
func (o ReviewQueryOptionFields) IDs() []uint { return o.ids }

func (o ReviewQueryOptionFields) WithUserIDs(ids ...uint) ReviewQueryOption {
    return func(f *ReviewQueryOptionFields) { f.userIDs = ids }
}
func (o ReviewQueryOptionFields) UserIDs() []uint { return o.userIDs }

func (o ReviewQueryOptionFields) WithProductIDs(ids ...uint) ReviewQueryOption {
    return func(f *ReviewQueryOptionFields) { f.productIDs = ids }
}
func (o ReviewQueryOptionFields) ProductIDs() []uint { return o.productIDs }

func (o ReviewQueryOptionFields) WithStatuses(s ...ReviewStatus) ReviewQueryOption {
    return func(f *ReviewQueryOptionFields) { f.statuses = s }
}
func (o ReviewQueryOptionFields) Statuses() []ReviewStatus { return o.statuses }

func (o ReviewQueryOptionFields) WithPagination(p *PaginationOption) ReviewQueryOption {
    return func(f *ReviewQueryOptionFields) { f.pagination = p }
}
func (o ReviewQueryOptionFields) Pagination() *PaginationOption { return o.pagination }

func (o ReviewQueryOptionFields) WithOrders(orders ...OrderOption) ReviewQueryOption {
    return func(f *ReviewQueryOptionFields) { f.orders = orders }
}
func (o ReviewQueryOptionFields) Orders() []OrderOption { return o.orders }
```

## 8) 테스트 지침

도메인 레이어는 순수 타입 정의이므로 직접적인 단위 테스트보다는 **다른 레이어의 테스트에서 간접 검증**됩니다.

테스트가 필요한 경우:
- Enum의 `Name()` 메서드 매핑 완전성
- 복수형 타입의 유틸리티 메서드 (IDs, IDMap 등)
- 비즈니스 로직이 포함된 엔티티 메서드

```go
func TestReviewStatus_Name(t *testing.T) {
    tests := []struct {
        status   domain.ReviewStatus
        expected string
    }{
        {domain.ReviewStatusPending, "승인 대기"},
        {domain.ReviewStatusApproved, "승인됨"},
        {domain.ReviewStatusRejected, "거부됨"},
        {domain.ReviewStatus("UNKNOWN"), ""},
    }

    for _, tt := range tests {
        t.Run(string(tt.status), func(t *testing.T) {
            assert.Equal(t, tt.expected, tt.status.Name())
        })
    }
}
```

## 9) 연계 스킬/명령

| 연계 대상 | 활용 방식 |
|----------|----------|
| `/audit` | `pattern-auditor` 에이전트가 이 스킬의 체크리스트로 도메인 파일 검증 |
| `/review` | `go-code-reviewer` 에이전트가 안티패턴 탐지 규칙 적용 |
| `/develop` | 새 엔티티 생성 시 권장 구현 패턴 참조 |
| `/design` | 설계 대안의 엔티티 구조 검증 |
| `usecase-layer` | Repository 인터페이스 시그니처 일관성 확인 |
| `interface-layer` | Repository 구현체가 인터페이스와 일치하는지 확인 |

## 10) 메트릭/거버넌스

| 메트릭 | 설명 | 임계값 |
|--------|------|--------|
| `GetID() 누락률` | GetID 미구현 엔티티 비율 | 0% (BLOCKING) |
| `FK 임베딩 위반 수` | 엔티티 타입을 FK로 사용한 수 | 0건 (BLOCKING) |
| `QueryOption 쌍 불일치` | With/Getter 메서드가 없는 필드 수 | 0건 (MAJOR) |
| `복수형 미정의율` | 복수형 타입 없는 엔티티 비율 | 0% (MAJOR) |
| `Enum Name() 누락률` | Name 메서드 없는 Enum 비율 | 모니터링 |
