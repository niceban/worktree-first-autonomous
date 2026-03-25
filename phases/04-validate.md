# Phase 4: Validation

运行测试验证所有组件正确安装。

## Objective

- 验证 `~/.wf2-autonomous/` 目录结构
- 验证 4 个 hook 脚本存在且可执行
- 验证 `hooks.json` 配置正确
- 验证 `state.json` 可写

## Input

- Dependency: Phase 1, 2, 3 输出

## Execution Steps

### Step 1: Verify Directory Structure

```javascript
const checks = {
  autonomousDir: Bash(`test -d "${process.env.HOME}/.wf2-autonomous" && echo "OK" || echo "MISSING"`).trim(),
  hooksDir: Bash(`test -d "${process.env.HOME}/.wf2-autonomous/hooks" && echo "OK" || echo "MISSING"`).trim(),
  stateJson: Bash(`test -f "${process.env.HOME}/.wf2-autonomous/state.json" && echo "OK" || echo "MISSING"`).trim(),
  configJson: Bash(`test -f "${process.env.HOME}/.wf2-autonomous/config.json" && echo "OK" || echo "MISSING"`).trim(),
};

console.log("Directory checks:", checks);
```

### Step 2: Verify Hook Scripts

```javascript
const hookScripts = ["session-start.sh", "post-tool.sh", "post-tool-fail.sh", "stop.sh"];
for (const script of hookScripts) {
  const path = `${process.env.HOME}/.wf2-autonomous/hooks/${script}`;
  const exists = Bash(`test -f "${path}" && echo "OK" || echo "MISSING"`).trim();
  const executable = Bash(`test -x "${path}" && echo "OK" || echo "NOT_EXECUTABLE"`).trim();
  console.log(`${script}: exists=${exists}, executable=${executable}`);
}
```

### Step 3: Verify hooks.json

```javascript
const hooksJsonPath = `${process.env.HOME}/.claude/hooks/hooks.json`;
const hooksJsonExists = Bash(`test -f "${hooksJsonPath}" && echo "OK" || echo "MISSING"`).trim();

if (hooksJsonExists === "OK") {
  const hooksJson = JSON.parse(Read(hooksJsonPath));
  const hookNames = hooksJson.hooks.map(h => h.name);
  console.log("Configured hooks:", hookNames.join(", "));

  const requiredHooks = ["session-start", "post-tool", "post-tool-fail", "stop"];
  const missing = requiredHooks.filter(n => !hookNames.includes(n));
  if (missing.length > 0) {
    console.log("MISSING HOOKS:", missing.join(", "));
  } else {
    console.log("All required hooks configured.");
  }
}
```

### Step 4: Run Unit Test

```javascript
// 运行 test-hooks.sh（从 IMPLEMENTATION.md 的测试方案）
const testScript = `#!/usr/bin/env bash
set -euo pipefail

AUTONOMOUS_DIR="\${HOME}/.wf2-autonomous"
SCRIPT_DIR="\${AUTONOMOUS_DIR}/hooks"

echo "=== Testing Hook Scripts ==="

# Test 1: session-start.sh
echo "Test 1: session-start.sh"
bash "\${SCRIPT_DIR}/session-start.sh" && echo "  PASS"

# Test 2: state.json initialized
jq -e '.version == "3.0"' "\${AUTONOMOUS_DIR}/state.json" && echo "  PASS: state.json valid"

# Test 3: hook scripts are executable
for script in session-start post-tool post-tool-fail stop; do
  test -x "\${SCRIPT_DIR}/\${script}.sh" && echo "  PASS: \${script}.sh executable" || echo "  FAIL: \${script}.sh not executable"
done

echo ""
echo "=== Validation Complete ==="
`;

// Write to temp file and run
const testFile = `/tmp/test-wf2-autonomous-hooks.sh`;
Write(testFile, testScript);
const testResult = Bash(`bash "${testFile}" 2>&1`);
console.log(testResult);
```

## Output

- **Console**: 验证结果输出
- **Status**: PASS/FAIL per component

## Quality Checklist

- [ ] `~/.wf2-autonomous/` 目录存在
- [ ] `state.json` 有效（version=3.0）
- [ ] `config.json` 有效（包含阈值字段）
- [ ] 所有 4 个 hook 脚本存在且可执行
- [ ] `hooks.json` 包含所有必需 hook
- [ ] `guard-bash.sh` 存在（可选）

## Completion

验证全部通过后，安装完成。Skill 执行完毕。

### 下一步

用户可以开始使用自动驾驶 Git 工作流：

1. 在 feature 分支上开发
2. 测试全过后，session 结束自动触发 merge AskUser
3. 用户确认后自动 merge + release 提示
