# go-clean-arch-toolkit

Go/Echo/GORM 클린 아키텍처 개발을 위한 Claude Code 플러그인입니다.

이슈 구체화부터 구현, 셀프 리뷰, PR 생성, 리뷰 반영까지 전체 개발 워크플로우를 자동화하고,
레이어별 코딩 규약 검증과 코드 감사를 제공합니다.

## 설치

```bash
# 마켓플레이스에서 설치하는 경우
/plugin marketplace add YangTaeyoung/go-clean-arch-toolkit
/plugin install go-clean-arch-toolkit

# 또는 로컬에서 직접 테스트
claude --plugin-dir ./go-clean-arch-toolkit
```

## 포함된 커맨드

| 커맨드 | 설명 |
|--------|------|
| `/specify <이슈 ID>` | Linear 이슈를 Q&A로 구체화하고 기술 설계를 포함하여 description 업데이트 |
| `/develop <이슈 ID> [--worktree]` | 스펙 기반 코드 구현 → 셀프 리뷰 → Draft PR 생성 |
| `/fix-review [PR 번호]` | PR 리뷰 코멘트를 읽고 코드 수정 후 push |

## 포함된 에이전트

| 에이전트 | 설명 |
|----------|------|
| `perf-auditor` | N+1 쿼리, 인덱스 누락, 비효율 쿼리 패턴 분석 |
| `data-auditor` | 마이그레이션-엔티티 정합성, FK 규약, 소프트 삭제 점검 |
| `pattern-auditor` | CLAUDE.md 코딩 규약 준수 여부 검증 |
| `security-auditor` | AdminOnly 누락, SQL 인젝션, 민감 정보 노출 점검 |
| `arch-auditor` | 레이어 의존성 방향, DI 등록 누락 분석 |
| `go-code-reviewer` | PR 변경 사항 종합 코드 리뷰 |

## 포함된 스킬

| 스킬 | 설명 |
|------|------|
| `developer-gate` | AskUserQuestion 기반 개발자 의사결정 게이트 패턴 |
| `domain-layer` | 엔티티, Repository, QueryOption, Enum, 에러 정의 검증 |
| `package-layer` | pkg/ 공유 패키지 인터페이스 설계, 생성자 패턴 검증 |
| `usecase-layer` | CRUD 구현, DTO 변환, 에러 처리, API Domain 검증 |
| `interface-layer` | HTTP/SQS Delivery, Repository 구현체 검증 |
| `infrastructure-layer` | DI 등록, Config, Migration, Queue 검증 |
| `pr-workflow` | GitHub CLI 기반 PR 생성/리뷰 반영 패턴 |
| `linear-workflow` | Linear 이슈 조회, description 업데이트, 상태 전이 패턴 |

## 대상 프로젝트 아키텍처

이 플러그인은 다음 스택을 사용하는 프로젝트에 적합합니다:

- **언어**: Go
- **웹 프레임워크**: Echo
- **ORM**: GORM + PostgreSQL
- **DI**: Uber fx
- **마이그레이션**: Goose
- **메시지 큐**: AWS SQS
- **이슈 관리**: Linear
- **아키텍처**: 4계층 클린 아키텍처 (Domain → Repository → Usecase → Delivery)

## 라이선스

MIT
