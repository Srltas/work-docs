# CUBRIDQA-1432 서브모듈 SHA 자동 PR 생성 (B안) 구현 계획

- 이슈: https://jira.cubrid.org/browse/CUBRIDQA-1432
- PR: <열리면 링크>
- 상태: 계획 (구현 예정)
- 날짜: 2026-07-10

## 배경 / 이슈

- 부모 저장소 `CUBRID/cubrid`는 서브모듈 3개(`cubrid-jdbc`, `cubrid-cci`, `cubridmanager`)를 가진다.
- 각 서브모듈에 변경이 머지될 때마다 부모의 서브모듈 SHA(gitlink)를 **수동으로** 갱신해 왔다 → 누락·지연 발생.
- 방식 검토 결과 **B안(자체 구축)** 선정: 서브모듈 push를 감지해 **실시간으로 부모에 갱신 PR을 자동 생성**. 머지는 사람이 수행(자동 머지 미도입).
- 본 문서는 **각 저장소에 만들 워크플로**와 **기존 CI(tc-branch-sync) 수정 방법**을 구체화한다.

## 원인 분석 (AS-IS)

- 서브모듈 갱신 → 부모 gitlink 수동 커밋. 반영이 사람 손에 의존.
- 자동 PR 생성 시 부딪히는 CUBRID 고유 제약(프로토타입으로 확인):
  - `pr-style`(제목 정규식 `^\[[A-Z]+-\d+\]\s.+`) 필수 체크 → B안은 원본 커밋의 `[이슈번호]`를 제목에 상속해 **자연 통과**.
  - `tc-branch-sync`가 **모든 PR**에 대해 테스트케이스(TC) 저장소에 draft PR을 생성 → bump PR엔 TC가 없어 **`Check TC PRs` 게이트가 막힘**. → 봇 PR 제외 필요.
- 봇 PR이 필수 체크를 정상 트리거하려면 기본 `GITHUB_TOKEN`이 아닌 **GitHub App 토큰**으로 PR을 열어야 함(별도 발급 요청 진행).

## 변경 / 해결 (TO-BE)

### 1) 만들거나 수정할 파일

| 저장소 | 파일 | 작업 | 역할 |
|---|---|---|---|
| CUBRID/cubrid | `.github/workflows/submodule-bump-receiver.yml` | 신규 | **수신**: dispatch → 해당 서브모듈 bump → PR 생성/갱신 |
| CUBRID/cubrid | `.github/CODEOWNERS` | 수정 | 서브모듈 경로 리뷰어 자동 지정 |
| CUBRID/cubrid | `.github/workflows/tc-branch-sync.yml` | 수정 | 봇 PR 제외(필수) |
| CUBRID/cubrid | `.github/workflows/tc-branch-finalize.yml` | 수정 | 봇 PR 제외(선택) |
| CUBRID/cubrid-jdbc | `.github/workflows/notify-parent.yml` | 신규 | **발신**: push → 부모로 dispatch |
| CUBRID/cubrid-cci | `.github/workflows/notify-parent.yml` | 신규 | 발신 |
| CUBRID/cubrid-manager-server | `.github/workflows/notify-parent.yml` | 신규 | 발신 |

### 2) 수신 워크플로 (부모: submodule-bump-receiver.yml)

```yaml
name: Submodule bump (receiver)
on:
  repository_dispatch:
    types: [submodule-bump]
  workflow_dispatch:
    inputs:
      submodule_path: { description: "cubrid-jdbc | cubrid-cci | cubridmanager", required: true }
      commit_message: { description: "원본 커밋 제목", default: "" }
      pusher:         { description: "assignee", default: "" }
permissions:
  contents: write
  pull-requests: write
env:
  BASE_BRANCH: develop
  FALLBACK_TAG: "[CBRD-0000]"   # 제목에 [XXX-000] 없을 때 대체(pr-style 통과 보장)
concurrency:
  group: submodule-bump-${{ github.event.client_payload.submodule_path || inputs.submodule_path }}
  cancel-in-progress: true
jobs:
  bump:
    runs-on: ubuntu-latest
    env:
      SUB_PATH:   ${{ github.event.client_payload.submodule_path || inputs.submodule_path }}
      COMMIT_MSG: ${{ github.event.client_payload.commit_message || inputs.commit_message }}
      PUSHER:     ${{ github.event.client_payload.pusher || inputs.pusher }}
    steps:
      - name: Validate submodule path
        run: |
          case "$SUB_PATH" in
            cubrid-jdbc|cubrid-cci|cubridmanager) ;;
            *) echo "::error::Unknown submodule path: $SUB_PATH"; exit 1 ;;
          esac
      - uses: actions/create-github-app-token@v1
        id: app-token
        with:
          app-id: ${{ secrets.SUBMODULE_BOT_APP_ID }}
          private-key: ${{ secrets.SUBMODULE_BOT_APP_PRIVATE_KEY }}
      - uses: actions/checkout@v4
        with:
          ref: ${{ env.BASE_BRANCH }}
          fetch-depth: 0
          token: ${{ steps.app-token.outputs.token }}
      - name: Update submodule pointer
        id: bump
        run: |
          set -euo pipefail
          git submodule update --init -- "$SUB_PATH"
          git submodule update --remote -- "$SUB_PATH"
          git diff --quiet -- "$SUB_PATH" && { echo "changed=false" >>"$GITHUB_OUTPUT"; exit 0; }
          echo "changed=true" >>"$GITHUB_OUTPUT"
      - name: Build PR title (이슈번호 상속 + fallback)
        if: steps.bump.outputs.changed == 'true'
        id: title
        run: |
          set -euo pipefail
          TAG=$(printf '%s' "$COMMIT_MSG" | head -n1 | grep -oE '^\[[A-Z]+-[0-9]+\]' || true)
          [ -z "$TAG" ] && TAG="$FALLBACK_TAG"
          echo "value=$TAG Update $SUB_PATH submodule" >>"$GITHUB_OUTPUT"
      - name: Create or update PR
        if: steps.bump.outputs.changed == 'true'
        uses: peter-evans/create-pull-request@v7   # 실제 반영 시 커밋 SHA로 pin
        with:
          token: ${{ steps.app-token.outputs.token }}
          branch: bump/${{ env.SUB_PATH }}         # 고정 브랜치 → 같은 PR 갱신(중복 방지)
          base: ${{ env.BASE_BRANCH }}
          title: ${{ steps.title.outputs.value }}
          commit-message: ${{ steps.title.outputs.value }}
          labels: Submodule Auto Update
          assignees: ${{ env.PUSHER }}
          delete-branch: true
```

- 리뷰어는 워크플로에 하드코딩하지 않고 **CODEOWNERS**로 자동 지정(아래 4).

### 3) 발신 워크플로 (각 서브모듈: notify-parent.yml)

3개 저장소 모두 동일 구조. **`SUB_PATH` 값만 저장소별로 다름**(부모 `.gitmodules`의 path 기준).

| 발신 저장소 | `SUB_PATH` 값 | 추적 브랜치 |
|---|---|---|
| CUBRID/cubrid-jdbc | `cubrid-jdbc` | develop |
| CUBRID/cubrid-cci | `cubrid-cci` | develop |
| CUBRID/cubrid-manager-server | `cubridmanager` | develop |

> 주의: 매니저는 **저장소명(cubrid-manager-server)과 부모 내 경로명(cubridmanager)이 다르다.** dispatch에는 **경로명 `cubridmanager`**를 보내야 한다.

```yaml
name: Notify parent (submodule bump)
on:
  push:
    branches: [develop]
permissions: {}
jobs:
  notify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/create-github-app-token@v1
        id: app-token
        with:
          app-id: ${{ secrets.SUBMODULE_BOT_APP_ID }}
          private-key: ${{ secrets.SUBMODULE_BOT_APP_PRIVATE_KEY }}
          owner: CUBRID
          repositories: cubrid          # 부모 저장소 범위 토큰
      - name: Dispatch to parent
        env:
          GH_TOKEN:   ${{ steps.app-token.outputs.token }}
          SUB_PATH:   cubrid-jdbc       # ← 저장소별로 변경 (위 표)
          SHA:        ${{ github.sha }}
          PUSHER:     ${{ github.actor }}
          COMMIT_MSG: ${{ github.event.head_commit.message }}
        run: |
          set -euo pipefail
          TITLE=$(printf '%s' "$COMMIT_MSG" | head -n1)
          PAYLOAD=$(jq -n --arg p "$SUB_PATH" --arg s "$SHA" --arg m "$TITLE" --arg u "$PUSHER" \
            '{event_type:"submodule-bump",client_payload:{submodule_path:$p,sha:$s,commit_message:$m,pusher:$u}}')
          curl -sS -f -X POST \
            -H "Authorization: Bearer $GH_TOKEN" -H "Accept: application/vnd.github+json" \
            https://api.github.com/repos/CUBRID/cubrid/dispatches -d "$PAYLOAD"
```

- 커밋 제목은 `env`로만 전달(스크립트 문자열에 직접 보간하지 않음 → injection 방지).

### 4) CODEOWNERS 추가 (부모)

기존 `.github/CODEOWNERS` 끝에 추가(담당자/팀은 실제 값으로):

```
/cubrid-jdbc     @CUBRID/<maintainers>
/cubrid-cci      @CUBRID/<maintainers>
/cubridmanager   @CUBRID/<maintainers>
/.gitmodules     @CUBRID/<maintainers>
```

- 트레일링 슬래시 없이(gitlink 경로 매칭). 프로토타입에서 이 방식으로 리뷰어 자동 요청 확인.
- 경로 추가만으로 "승인 필수"가 되지는 않음(현행 "아무 승인 1건" 유지). 코드오너 승인 강제는 branch protection의 `Require review from Code Owners`를 켤 때만.

### 5) tc-branch-sync 봇 제외 (구체 수정)

문제: App가 연 bump PR도 정상 PR로 취급되어 `tc-branch-sync`가 TC 저장소에 draft PR을 생성 → `Check TC PRs` 게이트가 막음.
해결: **첫 job `detect-pr-type`에 봇 제외 가드 1줄** 추가. 이 job이 skip되면 이를 `needs`로 받는 `sync-tc-branch`·`comment-on-engine-pr`도 연쇄 skip → TC draft PR이 만들어지지 않음 → `Check TC PRs`가 "열린 TC PR 없음 → 통과".

AS-IS (`.github/workflows/tc-branch-sync.yml`):
```yaml
  detect-pr-type:
    name: Detect PR Type
    runs-on: ubuntu-24.04
```

TO-BE:
```yaml
  detect-pr-type:
    name: Detect PR Type
    if: github.event.pull_request.user.login != 'cubrid-submodule-bot[bot]'   # ← 추가
    runs-on: ubuntu-24.04
```

- 판단 기준은 `github.event.pull_request.user.login`(PR 작성자 = App 봇 `cubrid-submodule-bot[bot]`). **봇 PR일 때만 skip이라 사람 PR 동작은 100% 그대로.**
- `tc-branch-sync`의 잡들은 develop의 **필수 체크가 아니므로** skip돼도 머지에 영향 없음.
- (선택) `tc-branch-finalize.yml`의 `finalize-tc-branch` job에도 동일 가드 추가 → 머지/클로즈 시 불필요한 TC 정리 시도·노이즈 방지.

### 6) 사전 준비 (조직 차원, 별도 진행)

- **GitHub App**(`cubrid-submodule-bot`) 생성·설치(CUBRID/cubrid) + 개인키 발급 → 조직 admin 요청(별도 요청 문서).
- **조직 레벨 Actions Secret**: `SUBMODULE_BOT_APP_ID`, `SUBMODULE_BOT_APP_PRIVATE_KEY` → 대상 저장소(cubrid + 서브모듈 3) 공유.

## 검증

프로토타입(개인 저장소 2개, 부모/서브모듈)으로 동작 확인 완료:

- 서브모듈 push → 부모에 bump PR **자동 생성** 확인.
- PR 제목 `[DEMO-123] Update <sub> submodule` → `pr-style` **자연 통과** 확인.
- CODEOWNERS로 **리뷰어 자동 요청** 확인.
- 연속 push 시 **기존 PR 갱신**(중복 PR 없음) 확인.

실 적용 후 검증 계획:
- `workflow_dispatch`로 수신 워크플로 수동 실행 → 서브모듈별 bump PR 생성 확인.
- 봇 제외 반영 후 bump PR에서 `Check TC PRs` 통과 확인.
- 3개 서브모듈 각각 push → end-to-end(발신→수신→PR) 확인.

## 결과 / 영향

- 서브모듈 SHA 수동 갱신 토일 제거, 반영 지연 해소(실시간).
- 기존 CI 변경은 **봇 전용 가드 1줄(+선택 1줄)** 뿐 → 사람 PR 동작 불변.
- 롤백 용이: 워크플로 파일 삭제 + App 설치 해제로 즉시 원복.
- 남은 과제: App 발급 완료 후 워크플로 반영 PR, 액션 커밋 SHA pin, CODEOWNERS 담당자 확정.

## 참고

- JIRA: https://jira.cubrid.org/browse/CUBRIDQA-1432
- PR: <열리면 링크>
- Commit: <반영 시 링크>
- 액션: peter-evans/create-pull-request, actions/create-github-app-token
- 방식 검토 및 App 발급 요청 문서(사내)
