# v3 Autonomous Requirements

基于 `AUTONOMOUS-v3-SPEC.md` 的需求摘要。

## 核心行为

| 行为 | 触发条件 | 用户交互 |
|------|---------|---------|
| 自动 Commit | 未 commit：>5 文件 或 >100 行 | 无 |
| Merge 提示 | 测试全过 + milestone | AskUser 确认 |
| Release 提示 | milestone 到达 | AskUser 确认 |

## 事件驱动模型

- PostToolUse (Bash) → 解析 stdout 检测测试通过
- PostToolUseFailure (Bash) → 检测 exit non-zero
- Stop → 自动 commit + AskUser

## 关键约束

| 约束 | 值 |
|------|-----|
| Stop hook 输出上限 | <64KB |
| async hook stdin | 无 trailing newline |
| async hook 超时 | 15 秒 |

## 阈值配置

| 参数 | 默认值 |
|------|--------|
| uncommitted_files_threshold | 5 |
| uncommitted_lines_threshold | 100 |
| milestone_commits_threshold | 10 |
