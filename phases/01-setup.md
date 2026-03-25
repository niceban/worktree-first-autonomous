# Phase 1: Setup Directory Structure

创建 `~/.wf2-autonomous/` 目录结构、state.json 和 config.json。

## Objective

- 创建 `~/.wf2-autonomous/` 目录
- 生成 `state.json`（初始状态）
- 生成 `config.json`（用户可配置阈值）

## Input

- 无外部依赖

## Execution Steps

### Step 1: Create Directory Structure

```javascript
const autonomousDir = `${HOME}/.wf2-autonomous`;
const hooksDir = `${autonomousDir}/hooks`;

Bash(`mkdir -p "${hooksDir}"`);
```

### Step 2: Generate state.json

```javascript
const state = {
  version: "3.0",
  session_id: "",
  branch: "",
  branch_type: "feature",
  test_passed: false,
  test_passed_at: null,
  test_failed_at: null,
  uncommitted_files: 0,
  uncommitted_lines: 0,
  last_commit_at: null,
  last_commit_message: null,
  milestone: false,
  milestone_reason: null,
  awaiting_squash_confirmation: false,
  squash_requested: false,
  awaiting_merge_confirmation: false,
  awaiting_release_confirmation: false,
  commits_since_last_tag: 0,
  created_at: new Date().toISOString()
};

Write(`${process.env.HOME}/.wf2-autonomous/state.json`, JSON.stringify(state, null, 2));
```

### Step 3: Generate config.json

```javascript
const config = {
  uncommitted_files_threshold: 5,
  uncommitted_lines_threshold: 100,
  milestone_commits_threshold: 10,
  auto_commit_message_prefix: "checkpoint: auto-save",
  merge_delete_branch: true,
  release_tag_prefix: "v"
};

Write(`${process.env.HOME}/.wf2-autonomous/config.json`, JSON.stringify(config, null, 2));
```

## Output

- **File**: `~/.wf2-autonomous/state.json`
- **File**: `~/.wf2-autonomous/config.json`
- **Format**: JSON

## Quality Checklist

- [ ] `~/.wf2-autonomous/` 目录存在
- [ ] `state.json` 包含所有必需字段
- [ ] `config.json` 阈值符合设计规格
- [ ] `hooks/` 子目录存在

## Next Phase

→ [Phase 2: Generate Hook Scripts](02-hooks.md)
