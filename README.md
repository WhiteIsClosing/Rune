# Rune

跨项目共享的 Claude Code 工作流规则，Mancer 生态的一部分。

## 文件结构

- `rules.md` — 每次 session 启动注入到 Claude 上下文的规则正文
- `templates/` — 按任务类别积累的验收标准模板，随用随加

## 接入方式

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
