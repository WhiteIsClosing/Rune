# 接入 Rune

新项目接入 Rune 的一页纸 playbook。首个参考实现：[Prospex 接入 commit `dc1e03e`](https://github.com/WhiteIsClosing/Prospex/commit/dc1e03e)（已过时：用的是下文"反模式"里点名的 stderr warning 模式）。硬化后的参考实现：[SIGIL PR #101](https://github.com/WhiteIsClosing/SIGIL/pull/101)。

## 为什么是"复制 4 份模板"而不是"引用 Rune"

Rune 通过 submodule 挂进消费项目。Submodule 未初始化时（clone 时漏了 `--recurse-submodules`，或网络故障），`.rune/` 整个消失——包括 `rules.md`、`integration.md`、本文档本身。

**自保护逻辑不能依赖被保护的东西可达**。如果你的 hook、bootstrap、诊断指南都住在 Rune 里靠引用，那么 Rune 一缺席，它们也一起消失，消费项目就没有任何机制能发现并修复。

所以接入 Rune = 把 4 份模板**原样复制**进消费项目自己的 git 历史。复制完后这 4 份文件就是项目自有资产，Rune 再缺席也能靠它们自救。

## 前置假设

- 项目是 git 仓库
- 使用 Claude Code，约定配置在 `.claude/`
- 采用 SessionStart hook 机制（现有 hook 合并；没有则新建）

---

## 接入步骤

### 1. 挂载 Rune 为 submodule

```bash
git submodule add <rune-url> .rune
```

`<rune-url>` 选择：

- **同 org 项目** → 相对路径 `../Rune`，跨 https / ssh 自动解析，最可移植
- **跨 org / fork** → 用绝对 URL（`https://github.com/whiteisclosing/Rune.git`）

### 2. 复制 4 份自保护模板到项目自有路径

#### 2.1 `.claude/hooks/session-start.sh` — 失败探测 hook

```bash
#!/bin/bash
#
# SessionStart hook — Rune L1 failure detector
# --------------------------------------------
# Rune L1 rules are loaded via CLAUDE.md's `@.rune/rules.md` import.
# This hook handles failure modes the @-import can't signal:
# submodule missing on disk.  Missing-file behavior of `@` import is
# not documented by Claude Code, so the stdout banner here is the
# only reliable signal of Layer 1 failure.

set -euo pipefail

cd "$CLAUDE_PROJECT_DIR"

_check_rune_rules() {
    # Happy path: rules file present → @-import in CLAUDE.md loads it.
    if [ -f ".rune/rules.md" ]; then
        return 0
    fi

    # Missing — attempt a bounded one-shot auto-init.
    if [ -f ".gitmodules" ] && grep -q 'path = \.rune' .gitmodules; then
        if timeout 30 git submodule update --init --recursive .rune \
                >/dev/null 2>&1; then
            if [ -f ".rune/rules.md" ]; then
                echo "(note: .rune submodule auto-initialized at session start)"
                return 0
            fi
        fi
    fi

    # Auto-init failed — emit a LOUD banner on stdout.  Claude's input
    # stream is stdout; stderr is invisible to the agent.
    cat <<'BANNER'

========================================================================
⚠ RUNE L1 RULES NOT LOADED — SESSION DEGRADED
========================================================================
.rune/rules.md 不可读。跨项目的 Rune L1 工作流规则**整体**未生效——
不只是某一两条具体规则，而是 Rune 当前所有 L1 规则与未来新增的
规则全部缺失。

修复（继续任何实质性工作之前请先执行）：
    git submodule update --init --recursive

如果 auto-init 失败是网络问题，告诉用户这一情况；如果继续进行
任务性工作，必须显式向用户声明"L1 规则缺失、按无 L1 模式降级运行"
以免 L1 静默失效被再次忽视。
========================================================================

BANNER
    return 1
}

_check_rune_rules || true  # do not block session start on Rune absence

# --- 项目自己的 hook 逻辑放在下面 ---
```

把 hook 注册到 `.claude/settings.json`：

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

#### 2.2 `scripts/bootstrap.sh` — 一次性 dev 环境 setup

```bash
#!/usr/bin/env bash
#
# bootstrap.sh — one-shot dev-environment setup
# Idempotent; safe to re-run.

set -euo pipefail

cd "$(git rev-parse --show-toplevel)"

echo "=== bootstrap ==="

echo "  [1/N] git submodule update --init --recursive"
git submodule update --init --recursive

if [ ! -f ".rune/rules.md" ]; then
    echo "  ERROR: .rune/rules.md still missing after submodule init" >&2
    echo "  Check network access to the Rune remote in .gitmodules" >&2
    exit 1
fi

# --- 项目专属 install / pre-commit / 其他 bootstrap 步骤 ---

echo "=== bootstrap complete ==="
```

项目可以按需扩充其他步骤（`pip install -e .[dev]`、`pre-commit install`、等等）。关键是**第一步永远是 submodule init**——Rune 缺席时后续一切都不工作。

#### 2.3 `CLAUDE.md` — 与 Rune 的协同 + L2 执行框架

在项目 `CLAUDE.md`（或等价的 Claude Code memory 文件）加以下章节。`@.rune/rules.md` 是 **Claude Code 原生 `@` 导入**——会把 Rune L1 规则作为 CLAUDE.md 的结构一部分自动加载到 session 上下文。

~~~markdown
## 与 Rune 的协同

本项目通过 `.rune/` submodule 接入 [Rune](https://github.com/WhiteIsClosing/Rune)。Rune L1 规则通过 CLAUDE.md 的 `@` 导入直接载入 session 上下文：

@.rune/rules.md

`.claude/hooks/session-start.sh` 不承担规则注入，只保留**失败探测**职责（submodule 缺失时 auto-init；仍失败则 stdout banner）。

### 分层

- **L1 通用层** — `.rune/rules.md`：跨项目的 agentic 协作规则
- **L2 项目层** — 本文件：本项目专属的硬约束 / 工程正确性规则

### 优先级

冲突时以本项目的 L2 为准。Rune 不写项目专属规则，项目也不把 L2 约束外溢到 Rune。

### Rune L1 规则在本项目的可观测执行

**原则**：L1 的每一条规则必须在实际响应里**可观测地合规**——审查者能一眼看出"这条规则有没有被遵守"。

**合规可观测性来源**（按优先级）：

1. **Rune 本身指定的输出标记**（若 L1 规则文档里写明了硬形式约束）
2. **L2 在下方"规则级约束表"里追加的项目级标记**
3. **响应末尾"规则合规清单"兜底**（不推荐）

**规则级约束表**（示例——每个项目按自己的团队文化替换）：

| Rune L1 规则 | 可观测性来源 | L2 形式约束（示例） |
|---|---|---|
| 规则 1（先定验收标准） | L2 | task-type 响应以 `## 验收标准` 开头 + 四子段（SIGIL 的选择） |
| 规则 3（沉淀模板） | L2 | 任务完成响应含"沉淀?"三选一询问（SIGIL 的选择） |
| 元规则（持续打磨 Rune） | 声明即可 | 发现改进点时主动提出 |
| 未来新增的 Rune 规则 | 待定 | 按下方"新规则加入流程" |

**新规则加入流程**：当 `.rune` submodule bump 带入新规则时：

1. bump commit message **必须点名**每条新增/修改的规则
2. **同一 PR**（不是后续 PR）必须更新本表
3. Rune 已自带硬形式约束 → 引用 Rune 规则号
4. Rune 只给语义约束 → L2 必须给出形式约束，否则该 bump PR 被 review 拒
5. 缺步骤 2 的 bump = "规则生效但合规不可观测"，回到硬化前的静默失效状态

### 维护

- 升级 Rune pin：`git submodule update --remote .rune`，单独 commit & PR
- 新 clone：`bash scripts/bootstrap.sh`
- `.rune/rules.md` 缺失时：`@` 导入可能静默失败，但 hook 会尝试 auto-init；仍失败则 hook 向 stdout 打印 banner
~~~

**必须由每个消费者项目自己决定的部分**：

- "规则级约束表"里具体的 L2 形式约束——上面列的 `## 验收标准` 和 "沉淀?" 是 **SIGIL 的选择**，其他项目应按自己团队文化重选
- 本项目与 Rune 的"优先级"条款里具体哪些 L2 约束压倒 L1 风格建议

框架本身（三层可观测性来源、规则级约束表结构、新规则加入流程）是所有消费者项目通用。

#### 2.4 `docs/rune-integration-troubleshooting.md` — 失败排查文档

~~~markdown
# Rune 接入常见失败模式

Rune L1 对本项目的生效依赖**三层串联契约**。

## 契约三层

```
Layer 1 — .rune submodule 已初始化（.rune/rules.md 可读）
        ↓
Layer 2 — 规则以**两个独立通道**进入 session 上下文（冗余是故意的）
           (a) CLAUDE.md 里的 `@.rune/rules.md` 导入（主通道）
           (b) .claude/hooks/session-start.sh 的 auto-init + banner
                （失败探测通道；Layer 1 断裂时 banner 让 Claude 可见）
        ↓
Layer 3 — Claude 看到规则后合规（按 L2 规则级约束表的形式标记）
```

两个通道在 Layer 2 各司其职：

- `@` 导入：Claude Code 原生机制；**缺失文件行为未文档化**，可能静默
- hook banner：显式失败信号；在 `@` 导入可能静默失败的场景下是唯一可靠的"规则没到"信号

## 失败模式 1：submodule 未初始化 → L1 静默失效

### 症状
- `git submodule status` 输出前缀 `-`（未初始化）
- Claude session 里**没有**任何 L1 规则相关内容
- Claude 按无 L1 模式工作

### 修法
Hook 已自动 auto-init；仍失败时 stdout banner 可见。手动修：
```bash
git submodule update --init --recursive
```

### 预防
新 clone 一律用 `bash scripts/bootstrap.sh` 或 `git clone --recurse-submodules <url>`。

## 失败模式 2：submodule pin 过期

### 症状
规则载入成功，但 Rune 上游最近的规则 N 未生效（submodule 指针停在旧 commit）。

### 修法
```bash
git submodule update --remote .rune
git add .rune && git commit -m "chore: bump Rune pin"
```
独立 commit、独立 PR，**同一 PR 同步更新 L2 规则级约束表**。

## 失败模式 3：规则已读但合规不可观测

### 症状
- L1 规则注入成功（前两层正常）
- 但 Claude 响应里看不到明确合规证据
- 审查者只能靠"感觉"判断

### 修法
L2 `CLAUDE.md` 的规则级约束表为每条 L1 规则绑定可观测形式标记。`.rune` bump 带入新规则时同一 PR 必须更新该表。

## 自检清单

**Layer 1 磁盘状态**：
- [ ] `git submodule status .rune` 无 `-` 前缀
- [ ] `cat .rune/rules.md | head -5` 能读到

**Layer 2 规则是否真的进入 Claude 上下文**：
- [ ] Session 启动日志**没有** `⚠ RUNE L1 RULES NOT LOADED` 横幅（hook 观测点）
- [ ] 让 Claude 复述 Rune 规则 1 的四个字段——能准确复述说明 `@.rune/rules.md` 导入成功（行为观测点）

**Layer 3 合规形式**：
- [ ] 第一次 task-type 响应符合 L2 规则级约束表的形式标记

全对 → L1 生效；任何一条不对 → 按上面的失败模式查。**必须两个 Layer 2 观测点都看**——只看 hook banner（没出现）但 `@` 导入静默失败的场景，会让规则"看起来载入了实际没载入"。
~~~

### 3. 初次 bootstrap + 自检

```bash
bash scripts/bootstrap.sh
```

然后按 `docs/rune-integration-troubleshooting.md` 的自检清单逐项核对——**让 Claude 复述 Rune 规则 1 的四字段**是 `@` 导入真实生效的唯一可信测试。

---

## 维护

- **升级 Rune**：`git submodule update --remote .rune`，独立 commit & PR，**同一 PR 同步更新本项目 L2 规则级约束表**
- **新 clone**：`bash scripts/bootstrap.sh`
- **发现 Rune 需要改进**（规则不清晰、模板过时、误判等）→ Rune 元规则要求**主动**提 Rune 的 PR/issue

## 反模式

- ❌ 把 `rules.md` 拷贝进项目仓库（漂移）
- ❌ 把项目专属规则搬进 Rune（L2 留在项目本地）
- ❌ hook 用 stderr warning 报告失败（stderr 对 Claude 不可见 → 静默失效）
- ❌ hook 因 `.rune/` 缺失而 exit 非 0（block session）
- ❌ 注入逻辑放在环境判断之后（本地 session 拿不到规则）
- ❌ 依赖 `@` 导入的失败行为做失败检测（未文档化，可能静默——必须靠 hook banner）
- ❌ `.rune` pin bump 不在同一 PR 更新 L2 规则级约束表（新规则生效但合规不可观测）
- ❌ 把自保护模板（hook / bootstrap / CLAUDE.md / troubleshooting）用符号链接或 `@` 引用的方式指回 Rune（Rune 缺席时这些也一起消失）
