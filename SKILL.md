---
name: worktree-first-autonomous
description: Autonomous Git workflow with event-driven hooks. Auto-commits on large uncommitted changes, prompts for merge when tests pass + milestone, prompts for release on tags. Triggers on "setup autonomous git", "autonomous git workflow", "setup git hooks".
allowed-tools: Agent, Read, Write, Glob, Grep, Bash
---

# Worktree-First Autonomous

全自动驾驶 Git 工作流 — 纯事件驱动，Claude Code Hooks 实现。

## Architecture Overview

```
Phase 0: Prerequisites (Read existing v2/v3 design docs)
         ↓
Phase 1: Setup          → 创建 ~/.wf2-autonomous/ 目录结构
         ↓
Phase 2: Hook Scripts    → 生成 4 个 hook 脚本
         ↓
Phase 3: Configuration   → 生成 hooks.json 配置
         ↓
Phase 4: Validation      → 验证所有组件正确安装
```

## Key Design Principles

1. **纯事件驱动** — 所有操作由 Claude Code Hooks 事件触发，无需定时/轮询
2. **状态共享** — `~/.wf2-autonomous/state.json` 跨 hook 共享状态
3. **v2 兼容** — v2 wrapper 处理 main 分支自动分支创建，v3 hooks 处理 feature 分支自动驾驶
4. **最小交互** — 只有 merge/release 两个 AskUser 确认点

---

## Mandatory Prerequisites

> **Do NOT skip**: Before performing any operations, you **must** completely read the following documents.

### Design Documents (Required Reading)

| Document | Purpose | When |
|----------|---------|------|
| [@/Users/c/auto-git/worktree-first-v2/docs/AUTONOMOUS-v3-SPEC.md](file:///Users/c/auto-git/worktree-first-v2/docs/AUTONOMOUS-v3-SPEC.md) | 需求规格说明书 — 行为定义、验收标准 | **Must read before execution** |
| [@/Users/c/auto-git/worktree-first-v2/docs/AUTONOMOUS-v3-IMPLEMENTATION.md](file:///Users/c/auto-git/worktree-first-v2/docs/AUTONOMOUS-v3-IMPLEMENTATION.md) | 完整实现方案 — hook 脚本代码、配置、测试 | **Must read before execution** |

---

## Execution Flow

### Phase 1: Setup Directory Structure
→ **Refer to**: `phases/01-setup.md`

创建 `~/.wf2-autonomous/` 目录结构、state.json 和 config.json。

### Phase 2: Generate Hook Scripts
→ **Refer to**: `phases/02-hooks.md`

生成 4 个 hook 脚本：session-start.sh, post-tool.sh, post-tool-fail.sh, stop.sh。

### Phase 3: Configure Hooks
→ **Refer to**: `phases/03-config.md`

生成 Claude Code hooks.json 配置，安装到项目 `.claude/hooks/`。

### Phase 4: Validation
→ **Refer to**: `phases/04-validate.md`

运行测试验证所有组件正确安装。

## Directory Setup

```javascript
const skillDir = `${HOME}/.claude/skills/worktree-first-autonomous`;
Bash(`mkdir -p "${skillDir}/phases" "${skillDir}/specs" "${skillDir}/scripts"`);
```

## Output Structure

```
~/.claude/skills/worktree-first-autonomous/
├── SKILL.md                     # 本文件
├── phases/
│   ├── 01-setup.md             # Phase 1
│   ├── 02-hooks.md             # Phase 2
│   ├── 03-config.md             # Phase 3
│   └── 04-validate.md           # Phase 4
└── specs/
    └── v3-requirements.md       # 需求规格摘要

~/.wf2-autonomous/
├── state.json                  # 运行时状态
├── config.json                 # 用户配置
└── hooks/
    ├── session-start.sh        # 初始化
    ├── post-tool.sh           # 测试成功检测
    ├── post-tool-fail.sh      # 测试失败检测
    └── stop.sh                # 自动 commit + AskUser
```

## Reference Documents by Phase

### Phase 1: Setup Directory Structure
Documents to reference when executing Phase 1

| Document | Purpose | When to Use |
|---------|---------|-------------|
| [phases/01-setup.md](phases/01-setup.md) | Phase 1 执行指南 | 创建目录和配置文件 |

### Phase 2: Hook Scripts
Documents to reference when executing Phase 2

| Document | Purpose | When to Use |
|---------|---------|-------------|
| [phases/02-hooks.md](phases/02-hooks.md) | Phase 2 执行指南 | 生成 4 个 hook 脚本 |
| [@/Users/c/auto-git/worktree-first-v2/docs/AUTONOMOUS-v3-IMPLEMENTATION.md](file:///Users/c/auto-git/worktree-first-v2/docs/AUTONOMOUS-v3-IMPLEMENTATION.md) | 完整实现代码 | 复制粘贴 hook 脚本 |

### Phase 3: Configuration
Documents to reference when executing Phase 3

| Document | Purpose | When to Use |
|---------|---------|-------------|
| [phases/03-config.md](phases/03-config.md) | Phase 3 执行指南 | 生成 hooks.json |

### Phase 4: Validation
Documents to reference when executing Phase 4

| Document | Purpose | When to Use |
|---------|---------|-------------|
| [phases/04-validate.md](phases/04-validate.md) | Phase 4 执行指南 | 运行验证测试 |

### Debugging & Troubleshooting

| Issue | Solution Document |
|-------|------------------|
| Hook 脚本不执行 | 检查 hooks.json 路径是否正确配置 |
| state.json 不更新 | 检查 ~/.wf2-autonomous/ 目录权限 |
| AskUser 不出现 | 验证 stop.sh exit 2 逻辑 |
| 合并到 main 失败 | 检查 git remote 和分支状态 |

### Reference & Background

| Document | Purpose | Notes |
|----------|---------|-------|
| [@/Users/c/auto-git/worktree-first-v2/docs/AUTONOMOUS-v3-DESIGN.md](file:///Users/c/auto-git/worktree-first-v2/docs/AUTONOMOUS-v3-DESIGN.md) | 完整设计文档 | 20 个 hooks 研究、状态机图 |
| [@/Users/c/auto-git/worktree-first-v2/docs/RESEARCH_SUMMARY.md](file:///Users/c/auto-git/worktree-first-v2/docs/RESEARCH_SUMMARY.md) | GitHub issues 研究 | Stop hook pipe deadlock 等已知限制 |
