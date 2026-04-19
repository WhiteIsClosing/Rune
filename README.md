# Rune（试点托管于 Almanac）

跨项目共享的 Claude Code 工作流规则，Mancer 生态的一部分。

## 当前状态

**临时托管在 Almanac 的 `.claude/rune/` 下**，待 `WhiteIsClosing/Rune` 仓的 cloud 授权完成后迁出为独立仓库。迁移后本目录整体删除，`.claude/settings.json` 的 SessionStart hook 从本地 `cat` 改为 `curl` 中心仓的 `rules.md`。

## 文件结构

- `rules.md` — 每次 session 启动注入到 Claude 上下文的规则正文
- `templates/` — 按任务类别积累的验收标准模板，随用随加

## 当前接入方式（Almanac 本地）

`.claude/settings.json` 的 SessionStart hook 直接 `cat .claude/rune/rules.md` 注入规则。

## 未来接入方式（Rune 独立仓后）

每个接入项目的 `.claude/settings.json`：

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "curl -sfL https://raw.githubusercontent.com/WhiteIsClosing/Rune/main/rules.md"
          }
        ]
      }
    ]
  }
}
```

## 修改规则

直接改 `rules.md` 并 commit。当前 session 需要重启才会重新注入（或者手动重读 hook 输出）。
