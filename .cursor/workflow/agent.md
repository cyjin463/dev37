# Agent workflow <!-- 에이전트 작업 워크플로우 -->

## Branch ↔ Jira ticket <!-- 브랜치명으로 Jira 티켓 연동 -->

Before starting work or committing, always check the **current branch name** first.

- If the branch name contains a Jira ticket pattern (`XXX-123`, e.g. `PROJ-42`, `ABC-1001`), treat it as Jira-linked.
- Extract the ticket key from the branch name. (e.g. `feature/PROJ-42-login` → `PROJ-42`)

### Commit message format <!-- 커밋 메시지 형식 (subject + body) -->

Every commit message **must** have:

1. **Subject** (first line): short summary of the work  
2. **Blank line**  
3. **Body**: detailed bullet list of what changed

#### Language <!-- 커밋 메시지 언어 -->

- Write the **entire** commit message (subject and body) in **Korean**.
- Ticket prefix stays as-is: `[XXX-123]`.
- For `commit-auto`, the agent always drafts subject/body in Korean.
- For `commit-<work summary>`, if the user text after `commit-` is not Korean, translate it into a natural Korean subject; body bullets are always Korean from the diff.

#### Subject <!-- 제목 줄 -->

| Branch contains `XXX-123` | Subject |
|---------------------------|---------|
| Yes | `[XXX-123] <한국어 작업 요약>` |
| No | `<한국어 작업 요약>` |

#### Body <!-- 본문 불릿 -->

- Always include a body with concrete details (not just the subject).
- Use `- ` bullets, one change per line, in **Korean**.
- Cover notable items such as: deps installed/configured, files/features added, bug fixes, rules/workflow updates, removals.
- Derive body bullets from the actual diff (`git status` / `git diff`). Do not invent unrelated items.

#### Full example <!-- 전체 예시 -->

Branch `feature/XXX-1-setup`, changes include library setup, new code, and a bug fix:

```text
[XXX-1] 기능 셋업 작업 추가

- 예제 라이브러리 설치 및 설정
- 예제 기능 코드 추가
- 예제 버그 수정
```

Commit via HEREDOC (required for multi-line messages):

```bash
git commit -m "$(cat <<'EOF'
[XXX-1] 기능 셋업 작업 추가

- 예제 라이브러리 설치 및 설정
- 예제 기능 코드 추가
- 예제 버그 수정

EOF
)"
```


---

## Pre-commit build gate <!-- 커밋 전 빌드 검증 (필수) -->

Before **any** commit (`commit-<work summary>` or `commit-auto`), run the project build test:

```bash
npm run build
```

### Rules <!-- 빌드 게이트 규칙 -->

- The build **must** succeed (exit code `0`) with **no real build errors or warnings** from the app toolchain (Next.js, TypeScript, bundler, ESLint-during-build, etc.).
- If the build exits non-zero, or the **build output** includes real **errors** or **warnings**, **abort the commit immediately**.
- Do **not** run `git add` / `git commit` after a failed or warning build.
- Report the failure to the user: command used, exit code, and the relevant error/warning lines.
- Do not “fix forward” by committing anyway.

### Ignore (agent environment noise) <!-- 무시할 경고 -->

- **Ignore** Cursor/agent sandbox npm noise that is not part of the project build, especially:
  - `npm warn Unknown env config "devdir"`
  - Other `npm warn Unknown env config "..."` lines caused by the agent runner injecting npm config
- These do **not** count as build failures. If `npm run build` exits `0` and Next/TypeScript/etc. report success with no app-level warnings, proceed with the commit.

---

## `commit-<work summary>` command <!-- 사용자 지정 메시지로 커밋 -->

When the user requests `commit-<work summary>`, run the steps below **in order**.

1. Inspect changes with `git status` / `git diff` (needed for the body bullets)
2. Try to extract the Jira ticket key (`XXX-123`) from the current branch name
3. Build the commit message
   - **Subject**: `[XXX-123] <한국어 작업 요약>` if ticket found, otherwise `<한국어 작업 요약>` (from the user’s text after `commit-`, translated to Korean if needed)
   - **Body**: Korean bullet list of concrete changes from the diff (required)
4. Run the **Pre-commit build gate** (`npm run build`). If it fails or has real build warnings (see Ignore list), **stop and report** — do not continue.
5. Stage all changes
   ```bash
   git add .
   ```
6. Commit with subject + body (HEREDOC)
   ```bash
   git commit -m "$(cat <<'EOF'
   <subject>

   - <상세 1>
   - <상세 2>

   EOF
   )"
   ```

### Examples <!-- 예시 -->

User input:

```text
commit-로그인 폼 추가
```

If the branch is `feature/AUTH-12-login` and the build passes cleanly:

```bash
npm run build
git add .
git commit -m "$(cat <<'EOF'
[AUTH-12] 로그인 폼 추가

- 로그인 폼 UI 추가
- 이메일/비밀번호 제출 핸들러 연결

EOF
)"
```

### Notes <!-- 주의사항 -->

- `commit-` performs a **commit only**. Push only when the user asks separately (`push`).
- Never commit with subject-only; body bullets are mandatory.
- Subject and body must be in **Korean**.
- If there is nothing to commit, do not commit; report the status instead.
- If files that look like secrets (e.g. `.env`) are included, warn before committing.

---

## `commit-auto` command <!-- 변경 내용 기반 자동 커밋 메시지 -->

When the user requests `commit-auto`, the workflow is the **same as `commit-<work summary>`**.  
The difference is that the agent writes **both** the subject summary and the body bullets from the changes — the user does not provide them.

### Steps <!-- 실행 순서 -->

1. Inspect changes with `git status` / `git diff` (staged + unstaged) and, if needed, recent commit style via `git log`
2. Write a concise **subject summary in Korean** (one line, focus on why)
3. Write a **body** of Korean `- ` bullets detailing the concrete changes (required)
4. Try to extract the Jira ticket key (`XXX-123`) from the current branch name — **do not skip this**
5. Build the full commit message
   - Subject: `[XXX-123] <한국어 요약>` or `<한국어 요약>`
   - Body: the agent-written Korean bullets
6. Run the **Pre-commit build gate** (`npm run build`). If it fails or has real build warnings (see Ignore list), **stop and report** — do not continue.
7. Stage all changes
   ```bash
   git add .
   ```
8. Commit with subject + body (HEREDOC)
   ```bash
   git commit -m "$(cat <<'EOF'
   <subject>

   - <상세 1>
   - <상세 2>

   EOF
   )"
   ```

### Examples <!-- 예시 -->

User input:

```text
commit-auto
```

If the change adds a login form, the branch is `feature/AUTH-12-login`, and the build is clean:

```bash
npm run build
git add .
git commit -m "$(cat <<'EOF'
[AUTH-12] 로그인 폼 추가

- 로그인 폼 UI 추가
- 이메일/비밀번호 제출 핸들러 연결

EOF
)"
```

### Notes <!-- 주의사항 -->

- `commit-auto` also performs a **commit only**. Push only via `push`.
- Jira linking and subject+body format are **identical** to `commit-<work summary>`.
- Always inspect the diff before writing subject/body; do not invent a message from guesswork alone.
- Never commit with subject-only; body bullets are mandatory.
- Subject and body must be in **Korean**.
- If there is nothing to commit, do not commit; report the status instead.
- If files that look like secrets (e.g. `.env`) are included, warn before committing.

---

## `push` command <!-- 현재 브랜치 push -->

When the user requests `push`, push the **current branch** to the remote.

### Steps <!-- 실행 순서 -->

1. Confirm the current branch name
2. Push the current branch (set upstream if it does not track a remote yet)
   ```bash
   git push -u origin HEAD
   ```
   If upstream is already set:
   ```bash
   git push
   ```
3. Report the result (branch name, remote, success or error output)

### Notes <!-- 주의사항 -->

- Do **not** force-push unless the user explicitly asks for it.
- Do not push to a different branch than the current one.

---

## `merge-<other-branch>` command <!-- 다른 브랜치를 현재 브랜치에 merge -->

When the user requests `merge-<other-branch>` (e.g. `merge-main`, `merge-dev`), merge **`<other-branch>` into the current branch**.

### Steps <!-- 실행 순서 -->

1. Confirm the current branch name
2. Verify that `<other-branch>` exists (local or fetch as needed)
3. Merge
   ```bash
   git merge <other-branch>
   ```
4. Check the result

### On conflicts — mandatory <!-- 충돌 시 필수 규칙 -->

- If **any** merge conflict occurs, **stop immediately** and **report** to the user.
- Report at least: current branch, source branch, conflicted file paths, and relevant `git status` output.
- **Never** auto-resolve conflicts.
- **Never** invent or apply fixes on your own.
- **Never** run `git add` / complete the merge while conflicts remain, unless the user later gives an explicit `cmd:` to resolve them.
- Leave the repo in the conflicted merge state for the user to decide, or abort only if the user explicitly asks.

### Notes <!-- 주의사항 -->

- Successful merge with no conflicts: report that the merge completed cleanly.
- Do not push after merge unless the user also requests `push`.

---

## `switch-<branch>` command <!-- 기존 브랜치로 이동 -->

When the user requests `switch-<branch>` (e.g. `switch-dev`, `switch-feature/login`), switch to that **existing** branch.

### Steps <!-- 실행 순서 -->

1. Confirm the current branch name
2. Switch to `<branch>`
   ```bash
   git switch <branch>
   ```
3. Report the result (previous branch → new branch, or error)

### Notes <!-- 주의사항 -->

- Do **not** create a new branch. If `<branch>` does not exist, stop and report — suggest `checkout-<branch>` if the user meant to create it.
- If uncommitted changes block the switch, stop and report; do not discard changes unless the user explicitly asks.

---

## `checkout-<branch>` command <!-- 브랜치 생성 후 이동 -->

When the user requests `checkout-<branch>` (e.g. `checkout-feature/login`), **create** `<branch>` and switch to it.

### Steps <!-- 실행 순서 -->

1. Confirm the current branch name (new branch starts from here)
2. Create and switch
   ```bash
   git switch -c <branch>
   ```
3. Report the result (base branch → new branch, or error)

### Notes <!-- 주의사항 -->

- If `<branch>` already exists, do **not** overwrite or reset it. Stop and report; suggest `switch-<branch>` instead.
- If uncommitted changes block the operation, stop and report; do not discard changes unless the user explicitly asks.
