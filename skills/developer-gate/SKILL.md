---
name: developer-gate
description: |
  개발자 확인 게이트 공통 패턴 스킬.
  AskUserQuestion 기반으로 개발자의 의사결정을 받는 표준 패턴을 제공합니다.
  /specify, /develop, /fix-review 커맨드에서 공통으로 활용됩니다.
---

# Developer Gate 스킬

## 1) 목적 요약

모든 워크플로우에서 **개발자 확인이 필요한 시점**에 `AskUserQuestion` 도구를 사용하여 표준화된 의사결정 게이트를 제공합니다.

핵심 원칙:
- 자동화와 인간 판단의 균형
- 중요한 결정은 반드시 개발자 승인 후 진행
- 질문은 명확하고 선택지는 구체적으로

## 2) 책임 범위

| 항목 | 설명 |
|------|------|
| Q&A 이슈 구체화 | `/specify`에서 순차 질문으로 요구사항 도출 |
| 기술 설계 확인 | `/specify`에서 파일 목록, 마이그레이션, DI 확인 |
| 구현 승인 게이트 | `/develop`에서 구현 계획을 개발자가 승인 |
| 리뷰 반영 확인 | `/fix-review`에서 수정 사항 목록 확인 |

## 3) 입력/출력

### 입력
- 현재 단계의 결과 요약
- 개발자에게 제시할 선택지

### 출력
- 개발자가 선택한 옵션
- 추가 입력이 필요한 경우 텍스트 응답

## 4) 체크리스트

### 게이트 설계 원칙
- [ ] 질문이 현재 상황을 명확히 요약하는지
- [ ] 선택지가 2-4개로 구체적인지
- [ ] "Other" 옵션이 자동 제공되므로 별도 추가 불필요
- [ ] 선택에 따른 다음 행동이 명확한지

### 필수 게이트 위치
- [ ] 이슈 description 업데이트 전 (최종 스펙 확인)
- [ ] 구현을 시작하기 전 (구현 계획 승인)
- [ ] 리뷰 반영 전 (수정 사항 확인)

## 5) 권장 구현 패턴

### 패턴 1: Q&A 이슈 구체화 - 핵심 기능 확인 (/specify)

이슈의 핵심 기능을 확인하는 첫 번째 질문입니다.

```
AskUserQuestion:
  questions:
    - question: |
        이슈 'PD-210: 알림 설정 기능'에 대해 구체화하겠습니다.
        현재 이슈 내용: "사용자별 알림 설정 CRUD"

        이 기능의 핵심 동작을 설명해주세요.
      header: "기능 확인"
      options:
        - label: "알림 유형별 ON/OFF"
          description: "이메일, 푸시 등 알림 유형별로 활성화/비활성화 설정"
        - label: "알림 스케줄 설정"
          description: "알림을 받을 시간대와 빈도를 설정"
      multiSelect: false
```

### 패턴 2: Q&A 이슈 구체화 - 엔티티/필드 확인 (/specify)

새로운 데이터 구조가 필요한 경우 필드를 확인합니다.

```
AskUserQuestion:
  questions:
    - question: |
        알림 설정 엔티티에 추가할 필드를 확인합니다.

        | 필드 | Go 타입 | GORM 태그 | Nullable | 설명 |
        |------|---------|-----------|----------|------|
        | UserID | uint | not null;uniqueIndex | N | 사용자 ID |
        | EmailEnabled | bool | not null;default:true | N | 이메일 알림 |
        | PushEnabled | bool | not null;default:true | N | 푸시 알림 |
        | SlackEnabled | bool | not null;default:false | N | Slack 알림 |

        이 필드 정의가 맞나요?
      header: "필드 확인"
      options:
        - label: "이대로 진행"
          description: "위 필드 정의로 구현합니다"
        - label: "수정 필요"
          description: "필드를 추가하거나 수정합니다"
      multiSelect: false
```

### 패턴 3: Q&A 이슈 구체화 - API 스펙 확인 (/specify)

```
AskUserQuestion:
  questions:
    - question: |
        API 엔드포인트를 확인합니다.

        | 메서드 | 경로 | 권한 | 설명 |
        |--------|------|------|------|
        | GET | /api/v1/notification-settings | 인증 | 내 설정 조회 |
        | PUT | /api/v1/notification-settings | 인증 | 설정 수정 |

        이 엔드포인트 정의가 맞나요?
      header: "API 확인"
      options:
        - label: "이대로 진행"
          description: "위 엔드포인트로 구현합니다"
        - label: "수정 필요"
          description: "엔드포인트를 추가하거나 수정합니다"
      multiSelect: false
```

### 패턴 4: 최종 스펙 확인 (/specify)

구체화된 스펙 + 기술 설계를 최종 확인합니다.

```
AskUserQuestion:
  questions:
    - question: |
        PD-210 구체화 결과입니다.

        [요구사항]
        - 사용자별 알림 설정 CRUD
        - 알림 유형별 ON/OFF
        - 기본값 모두 ON

        [기술 설계]
        변경 파일:
          - internal/domain/notification_setting.go (신규)
          - internal/repository/.../notification_setting_pg.go (신규)
          - internal/api/domain/notification_setting.go (신규)
          - internal/api/usecase/.../notification_setting_ucase.go (신규)
          - internal/api/delivery/.../notification_setting_http.go (신규)
          - cmd/api/main.go (수정)
        마이그레이션: 있음 (notification_settings 테이블)
        DI 변경: 있음
        테스트: Usecase table-driven

        이 내용으로 이슈를 업데이트할까요?
      header: "스펙 확인"
      options:
        - label: "승인, 이슈 업데이트"
          description: "이 내용으로 Linear 이슈 description을 업데이트합니다"
        - label: "수정 필요"
          description: "내용을 수정합니다"
      multiSelect: false
```

### 패턴 5: 구현 계획 승인 게이트 (/develop)

```
AskUserQuestion:
  questions:
    - question: |
        PD-210 구현 계획입니다.

        [태스크 목록]
        1. migrations/20260218_notification_settings.sql - 테이블 생성
        2. internal/domain/notification_setting.go - 엔티티 + Repository
        3. internal/repository/.../notification_setting_pg.go - Repository 구현
        4. internal/api/domain/notification_setting.go - Usecase + DTO
        5. internal/api/usecase/.../notification_setting_ucase.go - CRUD
        6. internal/api/delivery/.../notification_setting_http.go - HTTP 핸들러
        7. cmd/api/main.go - DI 등록
        8. notification_setting_ucase_test.go - 테스트

        [마이그레이션] 있음
        [테스트] Usecase table-driven 테스트
        [DI 변경] 있음

        이 계획으로 진행해도 될까요?
      header: "구현 승인"
      options:
        - label: "승인, 구현 시작"
          description: "이 계획대로 구현을 시작합니다"
        - label: "태스크 수정 필요"
          description: "태스크 목록을 수정합니다"
        - label: "테스트 범위 조정"
          description: "테스트 범위를 변경합니다"
      multiSelect: false
```

### 패턴 6: 리뷰 반영 확인 (/fix-review)

```
AskUserQuestion:
  questions:
    - question: |
        PR #45에 3건의 리뷰 코멘트가 있습니다.

        1. `notification_setting_http.go:25` - Swagger 주석 보강 필요
        2. `notification_setting_ucase.go:42` - 에러 메시지 한국어로 통일
        3. `notification_setting.go:15` - 한국어 주석 추가

        어떻게 반영할까요?
      header: "리뷰 반영"
      options:
        - label: "전부 반영"
          description: "3건 모두 수정합니다"
        - label: "선택적 반영"
          description: "반영할 항목을 선택합니다"
        - label: "확인만 하고 직접 수정"
          description: "수정 사항을 확인만 합니다"
      multiSelect: false
```

## 6) 안티패턴 탐지 규칙

| ID | 안티패턴 | 올바른 패턴 |
|----|---------|-----------|
| G-001 | 개발자 확인 없이 Linear 이슈 업데이트 | 반드시 확인 후 업데이트 |
| G-002 | 개발자 확인 없이 구현 시작 | 구현 계획 승인 필수 |
| G-003 | 5개 이상 선택지 제시 | 2-4개로 제한 (Other 자동 제공) |
| G-004 | 질문에 현재 상황 요약 누락 | 컨텍스트 포함 필수 |

## 7) 게이트별 필수 정보

| 게이트 | 반드시 포함할 정보 |
|--------|------------------|
| 기능 확인 | 이슈 제목, 현재 description, 추정 동작 |
| 필드 확인 | 필드명, Go 타입, GORM 태그, Nullable 여부 |
| API 확인 | HTTP 메서드, 경로, 권한, 설명 |
| 스펙 확인 | 요구사항 요약 + 기술 설계 전체 |
| 구현 승인 | 태스크 목록, 마이그레이션 유무, 테스트 범위, DI 변경 |
| 리뷰 반영 | 리뷰 코멘트 목록 (파일:줄 + 내용) |

## 8) 테스트 지침

게이트 자체는 `AskUserQuestion` 도구 호출이므로 자동 테스트 대상이 아닙니다.
검증 방법: 각 커맨드를 실행하여 적절한 시점에 질문이 나오는지 수동 확인합니다.

## 9) 연계 스킬/명령

| 연계 대상 | 활용 방식 |
|----------|----------|
| `/specify` | Q&A 구체화 게이트 (패턴 1, 2, 3, 4) |
| `/develop` | 구현 승인 게이트 (패턴 5) |
| `/fix-review` | 리뷰 반영 확인 게이트 (패턴 6) |
| `linear-workflow` | 게이트 승인 후 Linear 연동 |

## 10) 메트릭/거버넌스

| 메트릭 | 설명 |
|--------|------|
| `게이트 스킵 수` | 개발자 확인 없이 진행된 횟수 (0이어야 함) |
| `게이트 응답 시간` | 개발자가 응답하는 데 걸린 평균 시간 |
| `수정 요청 비율` | "수정 필요" 선택 비율 (구체화/구현 품질 지표) |
