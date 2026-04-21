# 接入 Rune

新项目接入 Rune 的一页纸 playbook。首个参考实现：[Prospex 接入 commit `dc1e03e`](https://github.com/WhiteIsClosing/Prospex/commit/dc1e03e)。

## 前置假设

- 项目是 git 仓库
- 使用 Claude Code，约定配置在 `.claude/`
- 采用 SessionStart hook 机制（现有 hook 合并；没有则新建）

## 三步接入

### 1. 以 submodule 方式挂载 Rune

```bash
git submodule add <rune-url> .rune
```

`<rune-url>` 选择：

- **同 org 项目** → 相对路径 `../Rune`，跨 https / ssh 自动解析，最可移植
- **跨 org / fork** → 用绝对 URL（`https://github.com/whiteisclosing/Rune.git`）

不要把 `rules.md` 直接拷进项目仓库——绕过 submodule 会导致规则漂移。

### 2. 在 SessionStart hook 中注入 rules

**核心要求**：注入逻辑必须在任何环境判断（`CLAUDE_CODE_REMOTE`、IDE 检测等）之**前**——Rune 规则在所有环境都要生效。

最小骨架（已有 hook 则合并到现有脚本顶部）：

```bash
#!/bin/bash
set -euo pipefail
cd "$CLAUDE_PROJECT_DIR"

# --- Rune rules injection (unconditional) ---
if [ -f ".rune/rules.md" ]; then
  cat ".rune/rules.md"
else
  echo "warning: .rune/rules.md not found; run 'git submodule update --init --recursive'" >&2
fi

# --- 项目自己的 hook 逻辑放在下面 ---
```

并在 `.claude/settings.json` 注册该 hook：

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {"type": "command", "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/session-start.sh"}
        ]
      }
    ]
  }
}
```

hook 必须**不因 `.rune/` 缺失而 exit 非 0**——否则 session 启动被 block。用 stderr warning + 继续的降级策略。

### 3. 在项目 `CLAUDE.md`（或等价文件）声明协同关系

加一节「与 Rune 的协同」，至少包含：

- **分层**：Rune L1（`.rune/rules.md`）= 跨项目通用；项目本地 = L2 专属规则
- **优先级**：本项目硬约束 / 工程正确性规则 ≥ Rune 风格建议；冲突时以项目为准
- **维护**：`git submodule update --remote .rune` 升级 Rune pin，放独立 PR

## 维护

- **升级 Rune**：`git submodule update --remote .rune`，独立 commit & PR
- **新 clone**：`git clone --recurse-submodules`，或 clone 后补跑 `git submodule update --init --recursive`

## 反模式

- ❌ 把 `rules.md` 拷贝进项目仓库（漂移）
- ❌ 把项目专属规则 / A–F 硬约束搬进 Rune（L2 留在项目本地）
- ❌ hook 因 `.rune/` 缺失而 exit 非 0（block session）
- ❌ 注入逻辑放在环境判断之后（本地 session 拿不到规则）
