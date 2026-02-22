---
name: linear-workflow
description: |
  Linear 이슈 관리 공통 패턴 스킬.
  이슈 조회, description 업데이트, Sub Task 관리, 상태 전이의 표준 워크플로우를 제공합니다.
  /specify, /develop, /fix-review 커맨드에서 공통으로 활용됩니다.
---

# Linear Workflow 스킬

## 1) 목적 요약

Linear MCP 도구를 사용한 이슈 관리의 **표준 패턴**을 제공합니다.
모든 커맨드에서 Linear 연동 시 이 스킬의 패턴을 따릅니다.

핵심 변경: **코멘트가 아닌 이슈 description을 직접 업데이트**하는 것이 기본 패턴입니다.

## 2) 책임 범위

| 항목 | 설명 |
|------|------|
| 이슈 조회 | `/specify`, `/develop`에서 이슈 상세 및 description 확인 |
| description 업데이트 | `/specify`에서 구체화된 스펙을 이슈 본문에 직접 반영 |
| Sub Task 조회 | `/develop`에서 부모 이슈의 자식 이슈 목록 조회 |
| 상태 업데이트 | 상태 전이 (Todo → 진행 중 → 리뷰 중 → 완료) |

## 3) 입력/출력

### 입력
- Linear 이슈 ID (예: `PD-210`)

### 출력
- 이슈 상세 (title, description, state, children)
- 업데이트 확인

## 4) 체크리스트

### MCP 도구 로드
- [ ] `ToolSearch`로 `+linear` 검색하여 MCP 도구 로드
- [ ] 필요한 도구: `get_issue`, `update_issue`, `list_issues`

### 이슈 조회
- [ ] 이슈 상세 확인 (title, description)
- [ ] 부모 이슈인지 Sub Task인지 판별
- [ ] Sub Task인 경우 부모 이슈 컨텍스트도 확인

### description 업데이트
- [ ] 기존 description 내용을 보존하면서 확장
- [ ] 마크다운 형식 사용
- [ ] 요구사항 + 기술 설계 섹션 포함

### 상태 업데이트
- [ ] 적절한 상태로 전이 (아래 상태 흐름 참고)

## 5) 권장 구현 패턴

### MCP 도구 로드
```
ToolSearch: "+linear get_issue"
→ MCP 도구 로드

ToolSearch: "+linear update_issue"
→ MCP 도구 로드

ToolSearch: "+linear list_issues"
→ MCP 도구 로드
```

### 이슈 조회 패턴 (/specify, /develop에서 사용)
```
get_issue:
  issueId: "PD-210"

→ 반환값에서 확인:
  - title: 이슈 제목
  - description: 이슈 설명 (구체화된 스펙 포함 여부)
  - state: 현재 상태
  - parent: 부모 이슈 (있는 경우)
  - children: 자식 이슈 목록 (있는 경우)
```

### Sub Task 목록 조회 패턴 (/develop에서 부모 이슈 처리 시)
```
# 방법 1: get_issue로 부모 이슈의 children 확인
get_issue:
  issueId: "PD-210"
→ children 필드에서 Sub Task 목록 추출

# 방법 2: list_issues로 부모 이슈 필터
list_issues:
  teamId: "<팀 ID>"
→ 결과에서 parent가 PD-210인 이슈 필터링
```

### description 업데이트 패턴 (/specify에서 사용)
```
update_issue:
  issueId: "PD-210"
  description: |
    ## 요구사항
    - 사용자별 알림 설정을 CRUD할 수 있는 API
    - 알림 유형별 ON/OFF 설정 가능
    - 기본값은 모두 ON

    ## API 스펙
    | 메서드 | 경로 | 권한 | 설명 |
    |--------|------|------|------|
    | GET | /api/v1/notification-settings | 인증 | 설정 조회 |
    | PUT | /api/v1/notification-settings | 인증 | 설정 수정 |

    ## 데이터 구조
    | 필드 | Go 타입 | GORM 태그 | Nullable | 설명 |
    |------|---------|-----------|----------|------|
    | UserID | uint | not null;uniqueIndex | N | 사용자 ID |
    | EmailEnabled | bool | not null;default:true | N | 이메일 알림 |
    | PushEnabled | bool | not null;default:true | N | 푸시 알림 |

    ## 기술 설계
    ### 변경 파일
    - `internal/domain/notification_setting.go` (신규) - 엔티티 + Repository
    - `internal/repository/notification_setting/postgresql/notification_setting_pg.go` (신규)
    - `internal/api/domain/notification_setting.go` (신규) - Usecase + DTO
    - `internal/api/usecase/notification_setting/notification_setting_ucase.go` (신규)
    - `internal/api/delivery/notification_setting/http/notification_setting_http.go` (신규)
    - `cmd/api/main.go` (수정) - DI 등록

    ### 마이그레이션
    CREATE TABLE notification_settings (...)

    ### DI 변경
    fx.Provide에 notificationSettingRepository.New, notificationSettingUsecase.New 추가
    fx.Invoke에 notificationSettingHttpDelivery.Register 추가

    ### 테스트 전략
    - Usecase: table-driven 테스트 (mockery)
```

### 상태 업데이트 패턴
```
update_issue:
  issueId: "PD-210"
  stateId: "<상태 ID>"
```

## 6) 안티패턴 탐지 규칙

| ID | 안티패턴 | 올바른 패턴 |
|----|---------|-----------|
| L-001 | ToolSearch 없이 MCP 도구 직접 호출 | 반드시 ToolSearch로 먼저 로드 |
| L-002 | 스펙을 코멘트로 작성 | description을 직접 업데이트 |
| L-003 | 상태 업데이트 없이 작업 완료 | 각 단계에서 상태 전이 |
| L-004 | description에 마크다운 미사용 | 마크다운 형식 필수 |
| L-005 | 기존 description 내용 덮어쓰기 | 기존 내용 보존하면서 확장 |

## 7) 상태 흐름

```
Todo → In Progress → In Review → Done
              ↑            ↓
              └── Changes Requested
```

| 커맨드 | 상태 전이 |
|--------|----------|
| `/specify` 완료 | 변경 없음 (이슈 내용만 업데이트) |
| `/develop` 시작 | → In Progress |
| `/develop` PR 생성 | → In Review |
| 개발자 Approve | → Done |
| 개발자 Request Changes | → Changes Requested |
| `/fix-review` 완료 | → In Review |

## 8) 테스트 지침

Linear MCP 도구는 실제 Linear 인스턴스에 연결되므로, 테스트 시 주의:
- 실제 이슈 업데이트 전 개발자 확인 (`developer-gate` 스킬 활용)
- `update_issue`로 description 업데이트 시 기존 내용 백업 권장

## 9) 연계 스킬/명령

| 연계 대상 | 활용 방식 |
|----------|----------|
| `/specify` | 이슈 조회 + description 업데이트 |
| `/develop` | 이슈 조회 + Sub Task 조회 + 상태 업데이트 |
| `/fix-review` | 상태 업데이트 (리뷰 반영 후 In Review) |
| `developer-gate` | 이슈 업데이트 전 개발자 확인 |

## 10) 메트릭/거버넌스

| 메트릭 | 설명 |
|--------|------|
| `description 구체화율` | /specify로 description이 업데이트된 이슈 비율 |
| `상태 전이 완전성` | 모든 단계에서 상태가 올바르게 전이되었는지 |
| `Sub Task 완료율` | 부모 이슈의 Sub Task 중 완료된 비율 |
