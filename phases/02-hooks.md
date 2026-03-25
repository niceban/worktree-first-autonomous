# Phase 2: Generate Hook Scripts

生成 5 个 hook 脚本：session-start.sh, pre-push.sh, post-tool.sh, post-tool-fail.sh, stop.sh。

## Objective

- 生成 `~/.wf2-autonomous/hooks/session-start.sh`
- 生成 `~/.wf2-autonomous/hooks/pre-push.sh`（squash-on-PUSH）
- 生成 `~/.wf2-autonomous/hooks/post-tool.sh`
- 生成 `~/.wf2-autonomous/hooks/post-tool-fail.sh`
- 生成 `~/.wf2-autonomous/hooks/stop.sh`
- 设置可执行权限

## Input

- Dependency: Phase 1 输出（目录已创建）

## Execution Steps

### Step 1: Read Implementation Reference

```javascript
// 读取 AUTONOMOUS-v3-IMPLEMENTATION.md 获取完整脚本代码
const implPath = `${process.env.HOME}/auto-git/worktree-first-v2/docs/AUTONOMOUS-v3-IMPLEMENTATION.md`;
const impl = Read(implPath);
// 提取 Section 4 中的脚本代码
```

### Step 2: Write session-start.sh

```javascript
const sessionStart = `#!/usr/bin/env bash
# session-start.sh — 初始化 state.json（once: true）
# Claude Code hooks 配置: event=SessionStart, once=true

set -euo pipefail

AUTONOMOUS_DIR="\${HOME}/.wf2-autonomous"
STATE_FILE="\${AUTONOMOUS_DIR}/state.json"

mkdir -p "$AUTONOMOUS_DIR"

# 检测当前分支
branch=$(git symbolic-ref --short HEAD 2>/dev/null || echo "unknown")
is_main_branch=false
[[ "$branch" == "main" || "$branch" == "master" ]] && is_main_branch=true

# 检测 branch_type
branch_type="feature"
if $is_main_branch; then
  branch_type="main"
elif [[ "$branch" == feature/* ]]; then
  branch_type="feature"
fi

# 获取 commits since last tag
commits_since_tag=0
if command -v git &>/dev/null && git rev-parse --git-dir &>/dev/null 2>&1; then
  commits_since_tag=$(git rev-list --count HEAD ^$(git describe --tags --abbrev=0 2>/dev/null) 2>/dev/null || echo 0)
fi

# 初始化或更新 state.json
if [[ -f "$STATE_FILE" ]]; then
  now=$(date -u +%Y-%m-%dT%H:%M:%SZ)
  tmp_file=$(mktemp)
  jq \\
    --arg session_id "$$" \\
    --arg branch "$branch" \\
    --arg branch_type "$branch_type" \\
    --arg commits_since_tag "$commits_since_tag" \\
    --arg now "$now" \\
    '.session_id = $session_id | .branch = $branch | .branch_type = $branch_type | .commits_since_last_tag = ($commits_since_tag | tonumber) | .created_at = $now' \\
    "$STATE_FILE" > "$tmp_file" && mv "$tmp_file" "$STATE_FILE"
else
  cat > "$STATE_FILE" <<EOF
{
  "version": "3.0",
  "session_id": "$$",
  "branch": "$branch",
  "branch_type": "$branch_type",
  "test_passed": false,
  "test_passed_at": null,
  "test_failed_at": null,
  "uncommitted_files": 0,
  "uncommitted_lines": 0,
  "last_commit_at": null,
  "last_commit_message": null,
  "milestone": false,
  "milestone_reason": null,
  "awaiting_merge_confirmation": false,
  "awaiting_release_confirmation": false,
  "commits_since_last_tag": $commits_since_tag,
  "created_at": "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
}
EOF
fi

echo "[session-start] initialized: branch=$branch, type=$branch_type"
`;

Write(`${process.env.HOME}/.wf2-autonomous/hooks/session-start.sh`, sessionStart);
```

### Step 3: Write post-tool.sh

```javascript
const postTool = `#!/usr/bin/env bash
# post-tool.sh — 检测测试成功（PostToolUse, async=true）
# Claude Code hooks 配置: event=PostToolUse, matcher=Bash, async=true

set -euo pipefail

# 读取 hook 输入（stdin）
input=""
while IFS= read -r line || [[ -n "$line" ]]; do
  input+="$line"$'\\n'
done

TOOL_RESPONSE=$(echo "$input" | jq -r '.tool_response // empty' 2>/dev/null || echo "")
[[ -z "$TOOL_RESPONSE" ]] && exit 0

AUTONOMOUS_DIR="\${HOME}/.wf2-autonomous"
STATE_FILE="\${AUTONOMOUS_DIR}/state.json"
[[ ! -f "$STATE_FILE" ]] && exit 0

# 检测测试成功 pattern
TEST_PATTERNS="(PASS|OK|0 failures|All tests passed|All \\\\d+ tests passed|\\\\u2713.*passed)"
if echo "$TOOL_RESPONSE" | grep -qE "$TEST_PATTERNS"; then
  now=$(date -u +%Y-%m-%dT%H:%M:%SZ)
  was_passed=$(jq -r '.test_passed' "$STATE_FILE" 2>/dev/null || echo "false")

  jq \\
    --arg now "$now" \\
    '.test_passed = true | .test_passed_at = $now | .test_failed_at = null' \\
    "$STATE_FILE" > "\${STATE_FILE}.tmp" && mv "\${STATE_FILE}.tmp" "$STATE_FILE"

  [[ "$was_passed" != "true" ]] && echo "[post-tool] tests passed, updated state"
fi

# 同时更新 uncommitted 统计
if command -v git &>/dev/null && git rev-parse --git-dir &>/dev/null 2>&1; then
  uncommitted_files=$(git status --short 2>/dev/null | wc -l | tr -d ' ')
  uncommitted_lines=$(git diff --stat 2>/dev/null | tail -1 | awk '{print $1}' || echo 0)
  jq \\
    --arg files "$uncommitted_files" \\
    --arg lines "$uncommitted_lines" \\
    '.uncommitted_files = ($files | tonumber) | .uncommitted_lines = ($lines | tonumber)' \\
    "$STATE_FILE" > "\${STATE_FILE}.tmp" && mv "\${STATE_FILE}.tmp" "$STATE_FILE"
fi

exit 0
`;

Write(`${process.env.HOME}/.wf2-autonomous/hooks/post-tool.sh`, postTool);
```

### Step 4: Write post-tool-fail.sh

```javascript
const postToolFail = `#!/usr/bin/env bash
# post-tool-fail.sh — 检测测试失败（PostToolUseFailure, async=true）
# Claude Code hooks 配置: event=PostToolUseFailure, matcher=Bash, async=true

set -euo pipefail

input=""
while IFS= read -r line || [[ -n "$line" ]]; do
  input+="$line"$'\\n'
done

ERROR=$(echo "$input" | jq -r '.error // empty' 2>/dev/null || echo "")
if ! echo "$ERROR" | grep -q "non-zero status"; then
  exit 0
fi

AUTONOMOUS_DIR="\${HOME}/.wf2-autonomous"
STATE_FILE="\${AUTONOMOUS_DIR}/state.json"
[[ ! -f "$STATE_FILE" ]] && exit 0

now=$(date -u +%Y-%m-%dT%H:%M:%SZ)
jq \\
  --arg now "$now" \\
  '.test_passed = false | .test_failed_at = $now' \\
  "$STATE_FILE" > "\${STATE_FILE}.tmp" && mv "\${STATE_FILE}.tmp" "$STATE_FILE"

echo "[post-tool-fail] non-zero exit detected, test_passed=false"
exit 0
`;

Write(`${process.env.HOME}/.wf2-autonomous/hooks/post-tool-fail.sh`, postToolFail);
```

### Step 5: Write pre-push.sh

```javascript
const prePush = `#!/usr/bin/env bash
# pre-push.sh — PreToolUse hook: intercept git push, squash commits, force-push
# Claude Code hooks 配置: event=PreToolUse, matcher={tool:Bash}, async=false
# exit 0 = allow command, exit 1 = deny command

set -euo pipefail

AUTONOMOUS_DIR="\${HOME}/.wf2-autonomous"
STATE_FILE="\${AUTONOMOUS_DIR}/state.json"
CONFIG_FILE="\${AUTONOMOUS_DIR}/config.json"

# 读取 hook 输入（stdin）
input=""
while IFS= read -r line || [[ -n "$line" ]]; do
  input+="$line"$'\\n'
done

# 解析命令
COMMAND=$(echo "$input" | jq -r '.command // empty' 2>/dev/null || echo "")
[[ -z "$COMMAND" ]] && exit 0

# 检查是否是 git push 命令
if ! echo "$COMMAND" | grep -qE '^[[:space:]]*git[[:space:]]+push'; then
  exit 0
fi

# 只在 feature 分支工作
if [[ ! -f "$STATE_FILE" ]]; then
  exit 0
fi

branch_type=$(jq -r '.branch_type // "feature"' "$STATE_FILE")
milestone=$(jq -r '.milestone // false' "$STATE_FILE")
awaiting_squash_push=$(jq -r '.awaiting_squash_push // false' "$STATE_FILE")

[[ "$branch_type" != "feature" ]] && exit 0
[[ "$milestone" != "true" && "$awaiting_squash_push" != "true" ]] && exit 0

# 检查是否有 commits 需要 squash（至少 2 个 commits 相对于 main）
commits_to_push=$(git rev-list --count origin/main..HEAD 2>/dev/null || echo 0)
if [[ "$commits_to_push" -lt 2 ]]; then
  exit 0
fi

echo "[pre-push] squash-on-push triggered: $commits_to_push commits → 1 commit"

# 执行 non-interactive squash: git reset --soft
current_branch=$(git symbolic-ref --short HEAD 2>/dev/null || echo "feature")
first_commit=$(git rev-list --max-parents=0 HEAD 2>/dev/null | tail -1)
base_commit=$(git merge-base HEAD origin/main 2>/dev/null || echo "$first_commit")

# 获取所有 commit messages 用于生成 squash message
squash_msg=""
if [[ -n "$first_commit" && "$first_commit" != "$base_commit" ]]; then
  squash_msg=$(git log --format="%s" "$base_commit"..HEAD 2>/dev/null | head -20 | paste -sd ' | ' || echo "squash: $current_branch commits")
else
  squash_msg=$(git log --format="%s" -10 HEAD 2>/dev/null | paste -sd ' | ' || echo "squash: $current_branch")
fi

echo "[pre-push] squashing: $squash_msg"

# 执行 soft reset 到 origin/main，保留所有更改在 staging
if ! git reset --soft origin/main 2>/dev/null; then
  echo "[pre-push] reset failed, allowing normal push"
  exit 0
fi

# 创建 squash commit
final_msg="squash(${current_branch}): ${squash_msg}"
if ! git commit -m "$final_msg" 2>/dev/null; then
  echo "[pre-push] commit failed, allowing normal push"
  git reset --hard HEAD 2>/dev/null || true
  exit 0
fi

# Force push with lease（安全 force push）
if git push --force-with-lease origin "$current_branch" 2>/dev/null; then
  echo "[pre-push] squashed commit pushed successfully"
  now=$(date -u +%Y-%m-%dT%H:%M:%SZ)
  jq \\
    --arg now "$now" \\
    --arg msg "$final_msg" \\
    --arg reason "squash-on-push completed" \\
    '.milestone = false | .milestone_reason = null | .awaiting_squash_push = false | .last_commit_at = $now | .last_commit_message = $msg' \\
    "$STATE_FILE" > "\${STATE_FILE}.tmp" && mv "\${STATE_FILE}.tmp" "$STATE_FILE"
  echo "[pre-push] state updated: milestone cleared"
else
  echo "[pre-push] force-push failed"
  git reset --hard HEAD 2>/dev/null || true
  exit 0
fi

exit 0
`;

Write(`${process.env.HOME}/.wf2-autonomous/hooks/pre-push.sh`, prePush);
```

### Step 6: Write stop.sh

```javascript
// stop.sh — 从 IMPLEMENTATION.md 复制完整代码
// 注意：需要简化版本避免超过 64KB 限制
const stopSh = `#!/usr/bin/env bash
# stop.sh — 自动 commit + AskUser merge/release（Stop hook）
# Claude Code hooks 配置: event=Stop
# exit 0 = 正常结束，exit 2 = 阻止 session 结束

set -euo pipefail

AUTONOMOUS_DIR="\${HOME}/.wf2-autonomous"
STATE_FILE="\${AUTONOMOUS_DIR}/state.json"
CONFIG_FILE="\${AUTONOMOUS_DIR}/config.json"

[[ ! -f "$STATE_FILE" ]] && exit 0

# 加载配置默认值
UNCOMMITTED_FILES_THRESHOLD=5
UNCOMMITTED_LINES_THRESHOLD=100
MILESTONE_COMMITS_THRESHOLD=10
AUTO_COMMIT_PREFIX="checkpoint: auto-save"
MERGE_DELETE_BRANCH=true
TAG_PREFIX="v"

if [[ -f "$CONFIG_FILE" ]]; then
  UNCOMMITTED_FILES_THRESHOLD=$(jq -r '.uncommitted_files_threshold // 5' "$CONFIG_FILE")
  UNCOMMITTED_LINES_THRESHOLD=$(jq -r '.uncommitted_lines_threshold // 100' "$CONFIG_FILE")
  MILESTONE_COMMITS_THRESHOLD=$(jq -r '.milestone_commits_threshold // 10' "$CONFIG_FILE")
  AUTO_COMMIT_PREFIX=$(jq -r '.auto_commit_message_prefix // "checkpoint: auto-save"' "$CONFIG_FILE")
  MERGE_DELETE_BRANCH=$(jq -r '.merge_delete_branch // true' "$CONFIG_FILE")
  TAG_PREFIX=$(jq -r '.release_tag_prefix // "v"' "$CONFIG_FILE")
fi

branch_type=$(jq -r '.branch_type // "feature"' "$STATE_FILE")
test_passed=$(jq -r '.test_passed // false' "$STATE_FILE")
awaiting_merge=$(jq -r '.awaiting_merge_confirmation // false' "$STATE_FILE")
awaiting_release=$(jq -r '.awaiting_release_confirmation // false' "$STATE_FILE")

# 只在 feature 分支工作
[[ "$branch_type" != "feature" ]] && exit 0

uncommitted_files=$(git status --short 2>/dev/null | wc -l | tr -d ' ')
uncommitted_lines=$(git diff --stat 2>/dev/null | tail -1 | awk '{print $1}' || echo 0)

# =====================================================================
# 处理 merge 确认响应
# =====================================================================
if [[ "$awaiting_merge" == "true" ]]; then
  user_response=""
  if [[ -p /dev/stdin ]] || [[ -s /dev/stdin ]]; then
    user_response=$(cat /dev/stdin 2>/dev/null | tr '[:upper:]' '[:lower:]')
  fi

  if echo "$user_response" | grep -qE "(是|yes|y|合并)"; then
    echo "[stop] executing merge..."
    git fetch origin main 2>/dev/null || true
    if git merge --no-ff origin/main -m "merge: feature branch into main"; then
      echo "[stop] merge successful"
      current_branch=$(git symbolic-ref --short HEAD 2>/dev/null || echo "")
      if [[ "$MERGE_DELETE_BRANCH" == "true" ]] && [[ -n "$current_branch" ]] && [[ "$current_branch" != "main" ]]; then
        git branch -d "$current_branch" 2>/dev/null && echo "[stop] branch deleted"
      fi
      jq '.awaiting_merge_confirmation = false | .awaiting_release_confirmation = true' "$STATE_FILE" > "\${STATE_FILE}.tmp" && mv "\${STATE_FILE}.tmp" "$STATE_FILE"
    else
      echo "[stop] merge failed, aborting"
      git merge --abort 2>/dev/null || true
    fi
  else
    jq '.awaiting_merge_confirmation = false' "$STATE_FILE" > "\${STATE_FILE}.tmp" && mv "\${STATE_FILE}.tmp" "$STATE_FILE"
    echo "[stop] merge declined by user"
  fi
fi

# =====================================================================
# 处理 release 确认响应
# =====================================================================
if [[ "$awaiting_release" == "true" ]]; then
  user_response=""
  if [[ -p /dev/stdin ]] || [[ -s /dev/stdin ]]; then
    user_response=$(cat /dev/stdin 2>/dev/null | tr '[:upper:]' '[:lower:]')
  fi

  if echo "$user_response" | grep -qE "(是|yes|y|release|打tag)"; then
    last_tag=$(git describe --tags --abbrev=0 2>/dev/null || echo "")
    new_tag="\${TAG_PREFIX}1.0.0"
    if [[ -n "$last_tag" ]]; then
      patch=$(echo "$last_tag" | grep -oE '[0-9]+$' || echo "0")
      new_patch=$((patch + 1))
      major_minor=$(echo "$last_tag" | sed 's/[0-9]*$//')
      new_tag="\${major_minor}\${new_patch}"
    fi
    git tag -a "$new_tag" -m "release: $new_tag $(date +%Y-%m-%d)"
    git push --tags origin main
    echo "[stop] release tag $new_tag pushed"
    jq '.awaiting_release_confirmation = false | .milestone = false' "$STATE_FILE" > "\${STATE_FILE}.tmp" && mv "\${STATE_FILE}.tmp" "$STATE_FILE"
  else
    jq '.awaiting_release_confirmation = false' "$STATE_FILE" > "\${STATE_FILE}.tmp" && mv "\${STATE_FILE}.tmp" "$STATE_FILE"
    echo "[stop] release declined by user"
  fi
fi

# =====================================================================
# 自动 Commit（超过阈值）
# =====================================================================
if [[ "$uncommitted_files" -gt "$UNCOMMITTED_FILES_THRESHOLD" ]] || [[ "$uncommitted_lines" -gt "$UNCOMMITTED_LINES_THRESHOLD" ]]; then
  echo "[stop] auto-committing: $uncommitted_files files, $uncommitted_lines lines"
  git add -A
  commit_msg="\${AUTO_COMMIT_PREFIX} $(date +%Y%m%d-%H%M%S)"
  if git commit -m "$commit_msg"; then
    now=$(date -u +%Y-%m-%dT%H:%M:%SZ)
    jq \\
      --arg msg "$commit_msg" \\
      --arg now "$now" \\
      '.last_commit_at = $now | .last_commit_message = $msg | .uncommitted_files = 0 | .uncommitted_lines = 0' \\
      "$STATE_FILE" > "\${STATE_FILE}.tmp" && mv "\${STATE_FILE}.tmp" "$STATE_FILE"
    echo "[stop] committed: $commit_msg"
  fi
fi

# =====================================================================
# Milestone 检测 + squash-on-PUSH 触发
# =====================================================================
commits_since_tag=$(git rev-list --count HEAD ^$(git describe --tags --abbrev=0 2>/dev/null) 2>/dev/null || echo 0)
last_commit_msg=$(git log -1 --format="%s" 2>/dev/null || echo "")

milestone_detected=false
milestone_reason=""

if [[ "$commits_since_tag" -ge "$MILESTONE_COMMITS_THRESHOLD" ]]; then
  milestone_detected=true
  milestone_reason="$commits_since_tag commits since last tag"
fi

if echo "$last_commit_msg" | grep -qE "^(feat|fix|perf|ci):"; then
  milestone_detected=true
  milestone_reason="conventional commit: $last_commit_msg"
fi

if [[ "$milestone_detected" == "true" ]] && [[ "$test_passed" == "true" ]]; then
  jq \\
    --arg reason "$milestone_reason" \\
    '.milestone = true | .milestone_reason = $reason' \\
    "$STATE_FILE" > "\${STATE_FILE}.tmp" && mv "\${STATE_FILE}.tmp" "$STATE_FILE"

  commits_to_push=$(git rev-list --count origin/main..HEAD 2>/dev/null || echo 0)
  if [[ "$commits_to_push" -ge 1 ]]; then
    echo ""
    echo "=== Milestone Reached ==="
    echo "Reason: $milestone_reason"
    echo "Tests: PASSED"
    echo "Commits to push: $commits_to_push"
    git log --oneline origin/main..HEAD 2>/dev/null | head -10
    echo ""
    echo "下次 'git push' 时将自动 squash 为 1 个 commit 并 force-push"
    echo ""

    jq '.awaiting_squash_push = true' "$STATE_FILE" > "\${STATE_FILE}.tmp" && mv "\${STATE_FILE}.tmp" "$STATE_FILE"
    echo "[stop] squash-on-push armed: next push will squash commits"
  fi
fi

echo "[stop] session ending normally"
exit 0
`;

Write(`${process.env.HOME}/.wf2-autonomous/hooks/stop.sh`, stopSh);
```

### Step 7: Set Executable Permissions

```javascript
Bash(`chmod +x "${process.env.HOME}/.wf2-autonomous/hooks/"*.sh`);
```

## Output

- **File**: `~/.wf2-autonomous/hooks/session-start.sh`
- **File**: `~/.wf2-autonomous/hooks/pre-push.sh`（squash-on-PUSH 核心）
- **File**: `~/.wf2-autonomous/hooks/post-tool.sh`
- **File**: `~/.wf2-autonomous/hooks/post-tool-fail.sh`
- **File**: `~/.wf2-autonomous/hooks/stop.sh`
- **Format**: Shell script (executable)

## Quality Checklist

- [ ] 所有 5 个 hook 脚本存在
- [ ] 所有脚本有可执行权限
- [ ] session-start.sh 包含 `set -euo pipefail`
- [ ] post-tool.sh 正确处理无 newline 的 stdin（async hook bug workaround）
- [ ] stop.sh 输出小于 64KB（避免 pipe buffer deadlock）
- [ ] pre-push.sh 正确检测 git push 命令
- [ ] pre-push.sh 使用 `git reset --soft` 实现 non-interactive squash
- [ ] pre-push.sh 使用 `--force-with-lease` 安全 force push
- [ ] stop.sh milestone 检测后设置 `awaiting_squash_push = true`

## Next Phase

→ [Phase 3: Configure Hooks](03-config.md)
