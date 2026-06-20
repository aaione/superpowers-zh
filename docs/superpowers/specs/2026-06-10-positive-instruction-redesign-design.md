# 技能指导的正向指令重设计 — 设计 Spec

**状态：** Proposed（2026-06-09 SDD 审查调度工作的后续；按"每个 PR 一个问题"规则独立成 PR）
**驱动：** 实测证据（2026-06-10）表明 skill prose 中某些否定式指令适得其反，而另一些则有效——并且这种差异是可预测的。

## 本 spec 所推广的实测发现

2026-06-10 的微测（opus，每种措辞 5 次重复，程序化打分；harness 见下文）测量了指导措辞如何改变 controller 所编写的内容：

| 用例 | 措辞 | 结果 |
|---|---|---|
| 调度构成（"don't restate the brief"） | 禁令 | **4.4** 个 spec 值被重新键入——*比无指导（3.6）更差* |
| 调度构成 | 正向配方（"your dispatch should contain: (1)…(5)"） | **3.0，零方差**——已采纳 |
| 调度构成 | 配方 + 细化子句（"quote only the fragment…"） | 3.8，噪声大——细化稀释了配方 |
| 测试重跑指令（"do not ask reviewer to re-run tests"） | 禁令 | **0/5 违规**——效果良好（对照：3/5） |
| 测试重跑指令 | 正向配方 | 0/5——效果相当，但更长 |

**该学说**（用它来分类任何否定式指令）：

1. **绊线有效。** 针对具体 token 的短语级自检（"if the prompt you are writing contains 'do not flag' … stop"）能可靠触发。
2. **识别表有效。** Red-Flags/rationalization 表在决策时被读取，而非在编写时。
3. **离散指令禁令有效。** 当模型没有去做 Y 的竞争性动机时，"Do not ask X to do Y" 成立。
4. **编写禁令适得其反**，当模型对输出有自己的议程时（例如，重述 spec 感觉像是乐于助人的策展）。只有正向编写配方能改变这些——而且给一个获胜配方追加细化子句会让它更糟，而非更好。
5. **平手时取更短的措辞。** Codex 在一次长会话中会重读 SKILL.md 约 500 次（2026-06-10 实测）；prose 长度是真实成本。

## 审计结果（2026-06-10，全部约 30 个 skill + prompt 模板）

计数：3 个绊线（保留）、14 个识别表（保留）、约 20 个策略门槛（保留——"never push without permission"是策略，不是编写塑形）、5 个编写禁令：

| # | 位置 | 处置 |
|---|---|---|
| 1 | `subagent-driven-development/task-reviewer-prompt.md` — "Cite, don't narrate" | **排入 PR #1717 批次**：以正向部分开头（"Your report should point at evidence: file:line for every finding…"），删除禁令部分（死重——正向部分已存在且承担主要作用） |
| 2 | `subagent-driven-development/SKILL.md` — "Do not add open-ended directives" | **保持原样**：微测无法在 15 个样本中诱发出失败；任一方向都无证据；更短者胜 |
| 3 | `subagent-driven-development/SKILL.md` — "Do not ask a reviewer to re-run tests" | **保持原样**：实测 0/5 违规；该禁令还能有益地把自身传播进调度中 |
| 4 | `subagent-driven-development/SKILL.md` — "do not re-review on top of it" | **排入 PR #1717 批次**：用三要素清单替换（"Before re-dispatching the reviewer, confirm the fix report contains: the covering tests, the command run, and the output"） |
| 5 | `writing-plans/SKILL.md` — "No Placeholders" 禁用模式列表 | **本 spec 的主要对象**——见下文 |

边界情况，与 #5 一起推迟：`task-reviewer-prompt.md` "Don't flag pre-existing file sizes — focus on what this change contributed"（正向部分存在且承载作用；影响低；若方便可与 #5 一起测试）。

## writing-plans 改动（推迟项 #5）

### 当前状态

`skills/writing-plans/SKILL.md`，"No Placeholders"：一句正向句子（"Every step must contain the actual content an engineer needs"）后跟一个六要点的禁用模式列表（"never write them: 'TBD'、'TODO'、'Add appropriate error handling'、'Write tests for the above'、'Similar to Task N'、…"）。

### 为什么它重要以及为何它确实不确定

- plan 是工作流中**最大的生成产物**，而且模型确实有发出占位符的竞争性动机（它们是长度压力下阻力最小的路径）——正是禁令可测量地适得其反的那种激励结构。
- 但被禁项是**离散的、可识别的 token**——正是禁令可测量地成立的那种形态。
- **该列表在别处承载作用：**该 skill 的 Self-Review 章节引用了它（"Placeholder scan: search your plan for red flags — any of the patterns from the 'No Placeholders' section above"）。这些 token 兼作审查时的扫描清单，而审查时的识别正是有效的那个类别。一次天真的、换成正向清单的替换会破坏该引用，并丢弃好的绊线 token。

### 待测变体

- **V0（当前）：**编写时有正向句子 + 禁用列表；Self-Review 引用该列表。
- **V1（审计者清单）：**编写时仅用正向配方——"Before finalizing a step, confirm it has: the literal code to write, a runnable command with expected output, types and method names defined within this plan, error handling shown explicitly. A step is complete when an engineer could implement it without asking any follow-up questions." Self-Review 保留泛化的占位符扫描。
- **V2（按机制重构——预测赢家）：**编写时仅获得 V1 的正向配方；具名模式整体迁入 Self-Review 的占位符扫描步骤，被重新框架为识别（"when you scan, look for: 'TBD'、'TODO'、'Similar to Task N'、…"）。相同的 token，从引发类目迁移到检测类目。
- **V3（对照）：**仅正向句子，任何地方都无列表。

### 微测设计

- **任务：**opus 从一份故意欠规整的 spec 写出一份 2-3 任务的实现 plan（欠规整正是诱惑占位符的因素）。使用一份 fixture spec，含：一个规整良好的任务、一个 spec 对其错误处理含糊带过的任务、一个与第一个相似的任务（诱惑"Similar to Task 1"）。
- **采样：**每个变体 5+ 次重复，默认温度，模型 `claude-opus-4-8`（实践中编写 plan 的模型）。
- **程序化打分**（除注明外越低越好）：
  - 禁用 token 计数：`TBD|TODO|implement later|fill in details|appropriate error handling|handle edge cases|Similar to Task|Write tests for the above`
  - 在改动代码的步骤中缺少围栏代码块的步骤数
  - 引用了在 plan 输出中任何地方都未定义的类型/函数
  - （越高越好）每个任务中带预期输出的可运行命令数
- **V2 的两阶段打分：**还测试 Self-Review 那一半——把每个生成的 plan 连同该变体的 Self-Review 章节重新喂入，并测量扫描是否真能捕获植入的占位符（向某 fixture plan 插入 2 个已知占位符；检测率是度量）。
- **接受标准：**仅当某变体在禁用 token 计数上击败 V0、且不损失代码块覆盖率或自检检测率时才采纳。预计成本：总计约 $6-10。

### PR 范围

独立 PR（writing-plans 是不同的 skill；其"No Placeholders"列表是被调优内容，贡献者指南要求 eval 证据）。该 PR 必须包含：微测 harness + 结果表、改动前后文本，以及 V2 迁移的依据。

## 微测 harness（方法，以免遗失）

`/tmp/sdd-exp/micro/run-micro.py` 和 `/tmp/sdd-exp/micro2/run-micro2.py`（2026-06-10；将提交到 superpowers-evals 作为 `docs/superpowers/skills/micro-testing-prompt-guidance.md` + 脚本）：

- 每个样本一次 API 调用：system prompt = 置于真实周边上下文中的 skill 指导变体；user = 一个真实的工作流中场景；output = 编写出的产物（调度 prompt、plan、报告）。
- 程序化打分，用 grep 匹配无歧义标记；**在信任任何裁决前手工检查每一处命中**——今晚的一个"违规"是 controller 正确地引用了禁令，而自动化否定检测把另一个误标了。
- 约 $0.15-0.30/样本，每次迭代数秒 vs $12/50 分钟的完整 eval 运行。在此迭代措辞；仅当改动是结构性时才在完整运行中确认赢家。
- 始终包含一个无指导对照——今晚它同时揭示了一个适得其反（重述：禁令比无更糟）和一个有效的禁令（测试重跑：3/5 对照失败 vs 任一措辞 0/5）。

## 结果：writing-plans 微测（2026-06-10 运行，在本 spec 编写之后）

**已定案——无需改动。**阶段 1（3 任务 spec，无压力）：全部 20 份 plan 在四个变体（包括无指导对照）中均 0 占位符。阶段 1b（10 任务 spec，五个近乎相同的命令诱惑"Similar to Task N"，显式约 2,500 词经济性目标）：40/40 干净——唯一的正则命中是 V2 自检*证明*"no TBD/TODO ✓"。当前代际的 opus 即便在刻意施压下，无论有无禁用模式列表，都不产生 plan 占位符。处置：将 No Placeholders 章节保持原样（它成本低，而反事实不可测量）；不要开启后续 PR。V2 迁移设计保留在本文件中，以备未来某代际模型出现回归。

## 同样明确不予放弃（经测试后放弃，附数据）

记录在此，以便任何人在没有新证据的情况下不再重提——完整数字见 2026-06-09 SDD 设计 spec 的 Cost-iterations 章节：

- **controller 轮次批处理 / 单条消息中的并行工具调用：**controller 每条消息恰好发出一次工具调用（每次测量运行中 0 条多工具消息，有无指导皆是）。controller 轮次中 46% 是思考/叙述而无工具调用——一个 prompt 免疫的下限。
- **通过并行调用流水线化审查：**因同样原因已弃。
- **通过 `run_in_background` 流水线化审查：**被提供时机制会被采纳（7/28 调度），但在 45 分钟场景上收益低于运行间噪声下限（每次审查仅约 30-60 秒）；还增加了双结果流协调。仅当审查各自都很长的 plan 时才值得重访。
- **追加到获胜配方之后的细化子句：**可测量地降低它们（C2：3.8 噪声大 vs C：3.0 一致）。通过重新推导配方来迭代，而不是通过追加告诫。
