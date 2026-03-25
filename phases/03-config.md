# Phase 3: Configure Hooks

生成 Claude Code hooks.json 配置，并安装到项目的 `.claude/hooks/` 目录。

## Objective

- 生成 `hooks.json` Claude Code hooks 配置
- 安装到项目的 `.claude/hooks/` 目录
- 确保 PreToolUse guard（guard-bash.sh）也被包含

## Input

- Dependency: Phase 2 输出（hook 脚本已生成）

## Execution Steps

### Step 1: Generate hooks.json

```javascript
const hooksJson = {
  hooks: [
    {
      name: "session-start",
      event: "SessionStart",
      hookType: "external",
      command: `${process.env.HOME}/.wf2-autonomous/hooks/session-start.sh`,
      once: true,
      async: false
    },
    {
      name: "post-tool",
      event: "PostToolUse",
      hookType: "external",
      command: `${process.env.HOME}/.wf2-autonomous/hooks/post-tool.sh`,
      matcher: { tool: "Bash" },
      async: true
    },
    {
      name: "post-tool-fail",
      event: "PostToolUseFailure",
      hookType: "external",
      command: `${process.env.HOME}/.wf2-autonomous/hooks/post-tool-fail.sh`,
      matcher: { tool: "Bash" },
      async: true
    },
    {
      name: "stop",
      event: "Stop",
      hookType: "external",
      command: `${process.env.HOME}/.wf2-autonomous/hooks/stop.sh`,
      async: false
    }
  ]
};

Write(`${process.env.HOME}/.claude/hooks/hooks.json`, JSON.stringify(hooksJson, null, 2));
```

### Step 2: Ensure guard-bash.sh is in place

```javascript
// 如果 guard-bash.sh 不存在，从已有路径复制
const guardSrc = `${process.env.HOME}/auto-git/wt/bash-20260325-112900/scripts/guard-bash.sh`;
const guardDest = `${process.env.HOME}/.wf2-autonomous/hooks/guard-bash.sh`;

const guardExists = Bash(`test -f "${guardDest}" && echo "yes" || echo "no"`);

if (guardExists.trim() === "no") {
  // 尝试从已知位置复制
  Bash(`cp "${guardSrc}" "${guardDest}" 2>/dev/null || echo "guard-bash.sh will be created separately"`);
}
```

### Step 3: Create project .claude/hooks symlink (optional)

```javascript
// 在当前项目创建 .claude/hooks 软链接指向全局 hooks.json
const projectHooksDir = `${process.env.HOME}/auto-git/worktree-first-v2/.claude/hooks`;
Bash(`mkdir -p "${projectHooksDir}"`);

// 复制 hooks.json 到项目（本地配置优先）
Bash(`cp "${process.env.HOME}/.claude/hooks/hooks.json" "${projectHooksDir}/hooks.json"`);
```

## Output

- **File**: `~/.claude/hooks/hooks.json`
- **File**: `~/.claude/hooks/guard-bash.sh`（如果存在）
- **File**: `{project}/.claude/hooks/hooks.json`

## Quality Checklist

- [ ] `hooks.json` 包含所有 4 个 hook 配置
- [ ] `hooks.json` 每个 hook 有正确的 `event` 字段
- [ ] `post-tool` 和 `post-tool-fail` 都有 `matcher: { tool: "Bash" }`
- [ ] `session-start` 有 `once: true`
- [ ] `stop` 没有 `async`（需要同步执行）
- [ ] `guard-bash.sh` 在正确位置

## Next Phase

→ [Phase 4: Validation](04-validate.md)
