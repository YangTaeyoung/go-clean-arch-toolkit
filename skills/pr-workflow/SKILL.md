---
name: pr-workflow
description: |
  PR 생성/리뷰 반영 공통 패턴 스킬.
  GitHub CLI를 사용한 PR 관리의 표준 워크플로우를 제공합니다.
  /develop, /fix-review 커맨드에서 공통으로 활용됩니다.
---

# PR Workflow 스킬

## 1) 목적 요약

GitHub CLI (`gh`)를 사용한 **PR 생성과 리뷰 피드백 반영**의 표준 패턴을 제공합니다.

## 2) 책임 범위

| 항목 | 설명 |
|------|------|
| PR 생성 | `/develop`에서 구현 완료 후 PR 생성 |
| 리뷰 코멘트 조회 | `/fix-review`에서 PR 리뷰 코멘트 읽기 |
| 리뷰 반영 + push | `/fix-review`에서 수정 후 push |

## 3) 입력/출력

### 입력
- PR 번호 또는 브랜치명
- Linear 이슈 ID

### 출력
- PR URL (생성 시)
- 수정된 파일 목록 (리뷰 반영 시)

## 4) 체크리스트

### PR 생성 전
- [ ] `make build` 성공
- [ ] `go test ./...` 통과
- [ ] `make swagger` 실행 (API 변경 시)
- [ ] 셀프 리뷰 통과 (go-code-reviewer 등)
- [ ] 변경 파일 목록 확인
- [ ] 커밋 메시지 컨벤션: `[<Linear ID>] <설명>`

### PR 생성
- [ ] 브랜치 push: `git push -u origin <branch>`
- [ ] `gh pr create --draft` 사용 (항상 Draft PR로 생성)
- [ ] PR 제목: `[<Linear ID>] <기능 설명>`
- [ ] PR 본문: Summary, Linear Issue, Changes, Test, Self Review 섹션

### 리뷰 반영
- [ ] `gh api`로 리뷰 코멘트 조회
- [ ] 수정 사항을 개발자에게 확인
- [ ] 수정 후 빌드/테스트 검증
- [ ] 커밋 메시지에 "코드리뷰 반영" 포함

## 5) 권장 구현 패턴

### 브랜치 생성 + push (일반 모드)
```bash
# 브랜치 생성
git checkout dev
git pull origin dev
git checkout -b feature/pd-210

# 작업 후 커밋
git add <변경된 파일들>
git commit -m "[PD-210] Review 도메인 CRUD 구현"

# push
git push -u origin feature/pd-210
```

### 브랜치 생성 + push (worktree 모드)
```bash
# worktree로 브랜치 생성 (병렬 작업용)
git fetch origin dev
mkdir -p worktrees
git worktree add ./worktrees/pd-210 -b feature/pd-210 origin/dev

# worktree 디렉토리에서 작업 후 커밋
cd ./worktrees/pd-210
git add <변경된 파일들>
git commit -m "[PD-210] Review 도메인 CRUD 구현"

# push
git push -u origin feature/pd-210

# PR 생성 후 worktree 정리 (프로젝트 루트에서)
cd <프로젝트 루트>
git worktree remove ./worktrees/pd-210
```

### PR 생성 (항상 Draft)
```bash
gh pr create --draft --title "[PD-210] Review 도메인 CRUD 구현" --body "$(cat <<'EOF'
## Summary
- Review 도메인의 CRUD 기능을 구현했습니다
- 마이그레이션, 엔티티, Repository, Usecase, Delivery 전체 스택 포함

## Linear Issue
https://linear.app/chainshift/issue/PD-210

## Changes
- `migrations/20260215_review_ddl.sql`: reviews 테이블 생성
- `internal/domain/review.go`: 엔티티 + Repository 인터페이스
- `internal/repository/review/postgresql/review_pg.go`: Repository 구현
- `internal/api/domain/review.go`: Usecase 인터페이스 + DTO
- `internal/api/usecase/review/review_ucase.go`: CRUD 구현
- `internal/api/delivery/review/http/review_http.go`: HTTP 핸들러
- `cmd/api/main.go`: DI 등록

## Test
- [x] `go test ./...` 통과
- [x] `make build` 성공
- [x] `make swagger` 생성 확인

## Self Review
- [x] go-code-reviewer 통과
- [x] security-auditor 통과 (Delivery 변경)
- [x] data-auditor 통과 (마이그레이션 포함)
EOF
)"
```

### PR 리뷰 코멘트 조회 (/fix-review에서 사용)
```bash
# PR의 리뷰 목록 조회
gh api repos/{owner}/{repo}/pulls/{pr_number}/reviews

# 리뷰별 인라인 코멘트 조회
gh api repos/{owner}/{repo}/pulls/{pr_number}/comments

# PR의 일반 코멘트 조회
gh pr view {pr_number} --comments

# 특정 리뷰의 코멘트만 조회
gh api repos/{owner}/{repo}/pulls/{pr_number}/reviews/{review_id}/comments
```

### 리뷰 반영 후 커밋 + push (/fix-review에서 사용)
```bash
# 수정된 파일 커밋
git add <수정된 파일들>
git commit -m "[PD-210] 코드리뷰 반영: Swagger 주석 보강, 에러 메시지 통일"

# push
git push
```

### 현재 브랜치의 PR 찾기
```bash
# 현재 브랜치명으로 PR 검색
gh pr list --head $(git branch --show-current) --json number,title,state

# 또는 직접 view
gh pr view
```

## 6) 안티패턴 탐지 규칙

| ID | 안티패턴 | 올바른 패턴 |
|----|---------|-----------|
| P-001 | 테스트 실패 상태에서 PR 생성 | `go test ./...` 통과 후 생성 |
| P-002 | 빌드 실패 상태에서 PR 생성 | `make build` 성공 후 생성 |
| P-003 | PR 본문에 변경 파일 목록 누락 | Changes 섹션 필수 |
| P-004 | 커밋 메시지에 Linear ID 누락 | `[PD-XXX]` 접두사 필수 |
| P-005 | 셀프 리뷰 없이 PR 생성 | go-code-reviewer 등 셀프 리뷰 필수 |
| P-006 | 리뷰 반영 후 빌드/테스트 미검증 | 수정 후 반드시 검증 |

## 7) PR 본문 템플릿

```markdown
## Summary
- <변경 요약 1-3줄>

## Linear Issue
https://linear.app/chainshift/issue/<이슈 ID>

## Changes
- `<파일 경로>`: <변경 내용>

## Test
- [x] `go test ./...` 통과
- [x] `make build` 성공
- [x] `make swagger` 생성 확인

## Self Review
- [x] go-code-reviewer 통과
- [x] <조건부 리뷰어> 통과
```

## 8) 테스트 지침

PR 워크플로우 검증:

```bash
# 1. 빌드 확인
make build

# 2. 전체 테스트
go test ./...

# 3. Swagger 생성 (API 변경 시)
make swagger
```

## 9) 연계 스킬/명령

| 연계 대상 | 활용 방식 |
|----------|----------|
| `/develop` | PR 생성 패턴 사용 |
| `/fix-review` | PR 리뷰 코멘트 조회 + 반영 패턴 |
| `developer-gate` | 리뷰 반영 전 수정 사항 확인 |
| `linear-workflow` | PR 생성 후 Linear 상태 업데이트 |

## 10) 메트릭/거버넌스

| 메트릭 | 설명 |
|--------|------|
| `PR 빌드 실패율` | PR 생성 시 빌드/테스트 실패 비율 |
| `셀프 리뷰 통과율` | 셀프 리뷰에서 Blocking 이슈 없이 통과한 비율 |
| `리뷰 반영 사이클` | 리뷰 반영 횟수 평균 |
| `PR 본문 완전성` | Summary, Changes, Test, Self Review 섹션 모두 포함된 비율 |
