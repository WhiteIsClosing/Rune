---
name: promote-to-rune
description: Promote a reusable principle that emerged in ONE sister project UP into the Rune master spec (abstract the domain-agnostic core into master, then hand-adapt a domain version into each other sister). Use when a sister's CLAUDE.md grows a rule/principle worth generalizing for all future projects. The inverse of sync-rune-master (which pushes master changes DOWN to sisters). Judgment-heavy, not a string-patch script.
---

# Promote Sister Innovation to Rune Master

`sync-rune-master` 走"母版 → 姊妹"（**向下**、机械、逐字拷贝）。这个 skill 走反方向："姊妹 → 母版 → 其他姊妹"（**向上**、品味型）——某个姊妹项目长出了一条可复用的原则（如 Mancer 的 `P1-d 建造者视角`），值得让所有未来项目都继承。

核心难点不是搬字符串，而是**判断**：哪部分是域无关的通用核心（进母版）、哪部分是域专属（留给项目自补），以及怎么让同一原则在各姊妹里**域改写**而非逐字复制。所以本 skill 是一套判断 SOP + 检查单，不是 `sync.py` 那样的自动脚本。

## When to use

- 一个姊妹项目的 `CLAUDE.md` 冒出新规则 / 原则 / 方法论，且**多个项目都该有**（规则 6 沉淀判断落在"Rune 母版"那一档）
- 该原则**含可剥离的域无关内核**——剥掉领域例子后，铁律对任何项目都成立

不适用：

- 原则纯域专属、剥不出通用内核（像 P1-c 六视角只对金融成立）→ 留在项目，别污染母版（§八 反模式：把项目专属规则提交到母版）
- 只是想把一条**逐字**规则同步到各仓 → 那是向下传播，用 `sync-rune-master`

## How（以 Mancer `P1-d` → Rune §三 + Cadence 域版 为样例）

1. **取源**：fetch 原则所在的姊妹分支，diff 出新增段落，读全（含它引用的硬约束 / design 锚点）。

2. **切通用 vs 域专属（本 skill 的命门）**：逐条问"剥掉领域词还成立吗？"
   - 成立 → 通用核心，进母版。如 P1-d 的 ①可证伪的赢 ②复杂度匹配已验证现实 ④结果有效 > 过程完美——铁律本身不含任何金融 / 域词。
   - 不成立 → 域专属，**母版只占位、项目自补**。如 P1-d 的 ③物理天花板（宏观的小样本 / 反身性、A 股的因子拥挤 / 数据挖掘）各域不同。

3. **写进母版**：放 Rune 对应章节——方法论类进 §三，硬约束类进 §五并更新规则级约束表。把域专属条降级为"由项目在 P1 自补"的**占位槽**。

4. **保编号跨仓对齐（最易翻车处）**：母版与各姊妹的同一原则**用同一套编号**。P1-d 做法：①②④ 通用、③ 设为"域专属天花板占位槽"——这样 `P1-d①` / `建造者自检①` 在三仓都指实，反模式的 `→ X①` 引用不会指空。**别把通用条重排成 ①②③ 让域槽消失**。

5. **逐个姊妹域化**：对每个目标姊妹，照其领域**手写**域版（不是拷贝母版）。P1-d 在 Cadence 把三天花板换成 数据挖掘小样本 / 因子拥挤 / 真因子 vs 巧合，并对接该仓已有的 A-DM / G。

6. **每个文件的连带更新（清单，漏一处就不自洽）**：
   - [ ] 顶部"域支点 / N 条第一性原理"计数（如 三条 → 四条）
   - [ ] 阅读导航 / header 描述
   - [ ] 反模式段落加对应条目、引用新编号
   - [ ] 相邻原则的措辞对照（如 P1-c 加"对外看市场"对照 P1-d"对内看系统"）

7. **验证 + 落库**：grep 核对新编号自洽、所有 `→ X①` 交叉引用指实；`git diff` 逐行回读；各仓在指定分支 commit + `git push -u`。**源姊妹不动**（它是出处）。

8. **§七 收政策**：确认 Rune `rules.md` §七 元规则有"向上提升"那段（与"向下 CRITICAL 回溯"对仗）。

## Why 不做成脚本

各姊妹拿的是**域改写版**而非逐字拷贝（P1-d 在 Mancer / Cadence 内容不同），`sync.py` 的 `(old, new)` 串对机制不适用。可机械的只有第 7 步的 push；判断（第 2 / 4 / 5 步）是真功夫，留给 agent。

## Files

- `SKILL.md`（本文件）——纯判断 SOP，无脚本

## 参考

- 反方向：`../sync-rune-master/`（母版 → 姊妹，机械传播）
- 母版演化政策：`rules.md` §七 元规则
- 样例：Mancer `P1-d 建造者视角` 上提为 Rune §三「建造者自检」+ Cadence 域版 `P1-d`
