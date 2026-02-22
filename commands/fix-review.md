---
name: fix-review
description: "리뷰 반영 - GitHub PR의 리뷰 코멘트를 읽고 코드를 수정하여 push합니다"
argument-hint: "[PR 번호]"
---

# 리뷰 반영 워크플로우

당신은 리뷰 반영 담당자입니다. GitHub PR에 달린 개발자의 리뷰 코멘트를 읽고, 코드를 수정하여 push합니다.

## 인자 처리

- PR 번호 (예: `45`): 해당 PR의 리뷰 코멘트 조회
- Resolved 된 코멘트는 무시하고 Unresolved 코멘트만 처리
- 인자 없음: 현재 브랜치의 PR을 찾아서 처리
  ```bash
  gh pr list --head $(git branch --show-current) --json number,title,state
  ```

## Worktree 감지

리뷰 반영 전에 해당 브랜치의 worktree가 존재하는지 확인합니다:
```bash
git worktree list
```

해당 PR의 브랜치에 대응하는 worktree가 `./worktrees/` 에 존재하면, 해당 worktree 디렉토리에서 작업합니다.
worktree가 없으면 일반 모드(현재 브랜치에서 직접)로 작업합니다.

worktree가 없고 현재 브랜치도 해당 PR 브랜치가 아닌 경우, worktree를 새로 생성합니다:
```bash
mkdir -p worktrees
git worktree add ./worktrees/<branch-name> <branch-name>
```

## 1단계: PR 리뷰 코멘트 조회

1. PR의 리뷰 코멘트를 조회합니다:
   ```bash
   # PR의 리뷰 목록 조회
   gh api repos/{owner}/{repo}/pulls/{pr_number}/reviews

   # 리뷰별 코멘트 조회
   gh api repos/{owner}/{repo}/pulls/{pr_number}/comments
   ```

2. 일반 PR 코멘트도 조회합니다:
   ```bash
   gh pr view {pr_number} --comments
   ```

3. 리뷰 코멘트를 확인했을 때 반영하면 안되거나, 반영되었을 때 우려가 있는 코멘트의 경우 해당 코멘트에 Reply로 해당 이유를 달고, 다시 해당 액션이 수행되었을 때 그럼에도 사용자가 수정하라고 지시하는 경우에는 그 지시를 따라 반영합니다.
    - 단순 질문 코멘트의 경우 답변을 달고, 사용자가 그 답변을 보고 수정하라고 지시하는 경우에는 그 지시를 따라 반영합니다.

4. 리뷰 코멘트를 파싱하여 수정 요청 사항을 정리합니다:
   - 파일 경로 + 줄 번호
   - 리뷰어의 요청 내용
   - 코드 변경 제안 (있는 경우)

## 2단계: 수정 사항 요약 및 확인

`AskUserQuestion`으로 수정 사항 목록을 제시합니다:

```
질문: "PR #<번호>에 N건의 리뷰 코멘트가 있습니다.

      1. `<파일>:<줄>` - <리뷰 내용 요약>
      2. `<파일>:<줄>` - <리뷰 내용 요약>
      ...

      어떻게 반영할까요?"

선택지:
  - "전부 반영"
  - "선택적 반영" (반영할 항목을 선택)
  - "확인만 하고 직접 수정"
```

## 3단계: 코드 수정

승인된 리뷰 코멘트에 대해 코드를 수정합니다.

수정 시 유의사항:
- CLAUDE.md의 코딩 규약을 준수
- 리뷰어의 의도를 정확히 반영
- 관련 테스트가 있으면 테스트도 함께 수정
- **worktree 모드**: worktree 디렉토리(`./worktrees/<branch-name>`)에서 파일을 수정합니다

## 4단계: 빌드/테스트 검증

```bash
# 일반 모드
make build
go test ./...

# worktree 모드
cd ./worktrees/<branch-name> && make build && go test ./...
```

빌드와 테스트가 모두 통과할 때까지 수정합니다.

## 5단계: 커밋 + push

변경된 파일을 확인하고 커밋합니다:
```bash
# 일반 모드
git add <변경된 파일들>
git commit -m "[<Linear ID>] 코드리뷰 반영: <수정 요약>"
git push

# worktree 모드 (worktree 디렉토리에서 실행)
cd ./worktrees/<branch-name>
git add <변경된 파일들>
git commit -m "[<Linear ID>] 코드리뷰 반영: <수정 요약>"
git push
```

커밋 메시지에는 어떤 리뷰를 반영했는지 간결하게 기술합니다.

## 6단계: 완료 안내

수정된 항목을 요약하여 보고합니다:
- 반영된 리뷰 코멘트 수
- 변경된 파일 목록
- 빌드/테스트 결과
