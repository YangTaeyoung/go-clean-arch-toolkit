---
name: develop
description: "구현 - Linear 이슈의 스펙을 기반으로 코드를 구현하고, 셀프 리뷰 후 PR을 생성합니다. 부모 이슈 또는 개별 Sub Task를 지원합니다. --worktree 옵션으로 병렬 구현을 지원합니다"
argument-hint: "<Linear 이슈 ID (부모 또는 Sub Task)> [--worktree]"
---

# 구현 워크플로우

당신은 개발 팀의 리더(dev-lead)입니다. 아래 단계를 **정확히** 따라 진행하세요.

## 인자 처리

Linear 이슈 ID(예: `PD-210`)를 필수로 받습니다.

### 옵션
- `--worktree`: git worktree를 사용하여 별도 디렉토리에서 구현합니다. 이 옵션을 사용하면 현재 작업 디렉토리를 변경하지 않고 별도의 worktree에서 작업하므로, **여러 이슈를 동시에 병렬 구현**할 수 있습니다.

## 1단계: 이슈 분류

1. ToolSearch로 `+linear get_issue` 검색하여 MCP 도구 로드
2. 이슈 조회하여 **부모 이슈인지 Sub Task인지** 판별

### 부모 이슈인 경우

자식 이슈(Sub Task) 목록을 조회합니다:
- `get_issue` 결과에서 children/sub-issues 필드 확인
- 또는 ToolSearch로 `+linear list_issues` 로드하여 부모 이슈 필터로 조회

자식 이슈 목록을 개발자에게 보여주고 진행 방식을 확인합니다:

```
질문: "<부모 이슈 제목>의 Sub Task N건입니다.

      1. <Sub Task 1 ID>: <제목>
      2. <Sub Task 2 ID>: <제목>
      ...

      어떻게 진행할까요?"

선택지:
  - "전부 순차 구현"
  - "특정 Sub Task만 선택"
```

### Sub Task인 경우

해당 Sub Task 하나만 처리합니다. → 2단계로 진행

## 2단계: 스펙 확인

이슈의 **description**에서 `/specify`로 구체화된 스펙을 읽습니다.

확인 사항:
- "## 요구사항" 섹션 존재 여부
- "## 기술 설계" 섹션 존재 여부

스펙이 없거나 부족하면 사용자에게 알리고 `/specify` 커맨드 사용을 권장합니다:
```
"이 이슈에 구체화된 스펙이 없습니다. 먼저 `/specify <이슈 ID>`로 이슈를 구체화해주세요."
```

---

## 이하 각 Sub Task별 반복 실행

### 3단계: 구현 계획 확인

**절대 이 단계를 건너뛰지 마세요.**

`AskUserQuestion`으로 구현 계획을 제시합니다:

```
질문: "<이슈 ID> 구현 계획입니다.

      [태스크 목록]
      1. <파일 경로> - <변경 내용>
      2. <파일 경로> - <변경 내용>
      ...

      [마이그레이션] <있음/없음>
      [테스트] <테스트 파일 및 범위>
      [DI 변경] <있음/없음>

      이 계획으로 진행해도 될까요?"

선택지:
  - "승인, 구현 시작"
  - "태스크 수정 필요"
  - "테스트 범위 조정"
```

### 4단계: 브랜치 생성

#### 일반 모드 (기본)

```bash
git checkout dev
git pull origin dev
git checkout -b feature/<linear-issue-id-소문자>
```

#### Worktree 모드 (`--worktree` 옵션 사용 시)

현재 작업 디렉토리를 변경하지 않고, 프로젝트 내 `./worktrees/` 디렉토리에서 작업합니다.
(`/worktrees`는 `.gitignore`에 등록되어 있어 메인 repo에 영향을 주지 않습니다.)

```bash
# dev 브랜치 최신화
git fetch origin dev

# worktrees 디렉토리 생성 (없는 경우)
mkdir -p worktrees

# worktree 생성
git worktree add ./worktrees/<linear-issue-id-소문자> -b feature/<linear-issue-id-소문자> origin/dev
```

- worktree 경로: `./worktrees/<linear-issue-id-소문자>` (예: `./worktrees/pd-210`)
- 이후 모든 명령어(빌드, 테스트, 커밋 등)는 이 worktree 디렉토리에서 실행합니다.
- **중요**: worktree 모드에서는 모든 Bash 명령에 worktree 경로를 명시적으로 사용하세요.
  ```bash
  # 예시: worktree 디렉토리에서 빌드
  cd ./worktrees/<linear-issue-id-소문자> && make build
  ```

커밋 메시지 컨벤션: `[<Linear ID>] <설명>` (예: `[PD-210] Brand 캐싱 기능 추가`)

### 5단계: TDD 기반 구현

CLAUDE.md의 가이드라인을 **100% 준수**하며 구현합니다.

#### 구현 순서 (CRUD의 경우)

1. **마이그레이션** (필요한 경우)
   ```bash
   make goose <migration_name> sql
   ```
   생성된 SQL 파일에 Up/Down 마이그레이션 작성

2. **Domain 엔티티** (`internal/domain/<entity>.go`)
   - `gorm.Model` 임베딩
   - `GetID() uint` 메서드
   - 복수형 타입 정의
   - Repository 인터페이스 + QueryOption

3. **Repository 구현체** (`internal/repository/<entity>/postgresql/<entity>_pg.go`)
   - `NewBaseRepository[T]` 임베딩
   - List 메서드 + Scope 함수

4. **API Domain** (`internal/api/domain/<entity>.go`)
   - Usecase 인터페이스
   - Request/Response DTO (lowerCamelCase JSON)
   - 한국어 주석

5. **Usecase** (`internal/api/usecase/<entity>/<entity>_ucase.go`)
   - CRUD 메서드 구현
   - 인라인 DTO 변환

6. **HTTP Delivery** (`internal/api/delivery/<entity>/http/<entity>_http.go`)
   - Register 함수 (GET목록 → GET단건 → POST → PUT → DELETE)
   - Swagger 주석
   - AdminOnly 미들웨어 적용

7. **DI 등록** (`cmd/api/main.go`)
   - fx.Provide에 Repository, Usecase 추가
   - fx.Invoke에 HTTP Delivery Register 추가

8. **테스트** (해당 usecase 및 repository)
   - table-driven 테스트 패턴
   - mockery + testify 사용

### 6단계: 빌드 및 테스트 검증

```bash
make build
go test ./internal/api/usecase/<entity>/...
go test ./...
make swagger
```

모든 테스트가 통과하고 빌드가 성공할 때까지 수정합니다.

### 7단계: 셀프 리뷰

구현이 완료되면, **PR 생성 전에** 리뷰어 에이전트로 자체 검증합니다.

**필수 리뷰어:**
- `subagent_type: "go-code-reviewer"` - Go 코드 품질 종합 리뷰

**조건부 리뷰어** (변경 범위에 따라 해당하는 것만 스폰):
- Delivery 파일 변경 시: `subagent_type: "security-auditor"` - 보안 검증
- Repository/Usecase 변경 시: `subagent_type: "perf-auditor"` - 성능 검증
- 마이그레이션 파일 포함 시: `subagent_type: "data-auditor"` - 데이터 무결성 검증

리뷰어 에이전트는 **병렬로** 스폰합니다. 반드시 하나의 메시지에서 모든 Task tool call을 동시에 보내세요.

각 리뷰어의 prompt에는 변경된 파일 경로를 포함하세요.

#### 셀프 리뷰 결과 처리

- **Blocking 이슈 발견**: 즉시 자동 수정 후 재검증 (make build && go test)
- **Major 이슈 발견**: 자동 수정 후 재검증
- **Minor/Nit 이슈**: 자동 수정 (가능한 경우)

수정 후 다시 빌드/테스트를 실행하여 문제가 없는지 확인합니다.

### 8단계: 커밋 + PR 생성

변경된 파일을 확인하고 커밋합니다:
```bash
git add <변경된 파일들>
git commit -m "[<Linear ID>] <기능 설명>"
```

**Draft PR**을 생성합니다 (항상 Draft로 생성):
```bash
git push -u origin feature/<linear-issue-id>
gh pr create --draft --title "[<Linear ID>] <기능 설명>" --body "$(cat <<'EOF'
## Summary
- <변경 요약>

## Linear Issue
<Linear 이슈 링크>

## Changes
- <변경 파일 1>: <변경 내용>
- <변경 파일 2>: <변경 내용>

## Test
- [x] `go test ./...` 통과
- [x] `make build` 성공
- [x] `make swagger` 생성 확인

## Self Review
- [x] go-code-reviewer 통과
- [x] <조건부 리뷰어 이름> 통과
EOF
)"
```

> **참고**: PR은 항상 Draft 상태로 생성됩니다. 개발자가 직접 리뷰 후 Ready for Review로 전환하세요.

### 9단계: Worktree 정리 (worktree 모드인 경우)

worktree 모드로 작업한 경우, PR 생성 후 worktree를 정리합니다:
```bash
# 프로젝트 루트로 돌아가서 worktree 제거
git worktree remove ./worktrees/<linear-issue-id-소문자>
```

> **참고**: worktree를 삭제해도 브랜치와 PR은 그대로 유지됩니다.

### 10단계: Linear 이슈 업데이트

ToolSearch로 `+linear update_issue` 검색하여 MCP 도구를 로드한 후,
이슈 상태를 "In Review"로 업데이트합니다.

---

## 반복 종료 후: 완료 안내

모든 Sub Task 처리가 끝나면:
- 생성된 PR 목록을 요약하여 표시 (Draft PR임을 명시)
- 각 PR의 URL 제공
- 개발자에게 Draft PR을 리뷰 후 Ready for Review로 전환할 것을 안내
- 리뷰 후 `/fix-review` 커맨드로 피드백을 반영할 수 있음을 안내
