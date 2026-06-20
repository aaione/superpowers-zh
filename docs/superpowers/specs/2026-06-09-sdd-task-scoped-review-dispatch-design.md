# SDD 任务级审查调度

通过将每个任务的审查 prompt 限定在该任务范围内并停止冗余工作，使 subagent-driven-development 中针对单个任务的审查更便宜、更快捷而不削弱其力度——同时最终分支审查保持宽泛。

## 问题

SDD 中针对单个任务的代码质量审查者，经常在单任务 diff 上做分支审查级别的工作。证据来自两次真实的本地 SDD 会话：`a1a6719a-6109-453a-9933-34ae396f5bae`（sen-core-v2）和 `0cc1a12d-9984-4c35-8615-9d42dadb2c47`（serf），均位于 `~/.claude/projects/`：

- 在 sen-core-v2 会话中，8 个质量审查者中有 7 个运行了仓库级 grep；开销最大的那个在约 200 秒内运行了 50+ 次 Bash 命令。跨两次会话，质量审查者在相同任务上的开销是 spec 审查者的 4-8 倍。
- spec 审查者，其 prompt 中含有"Only read files in this diff. Do not crawl the broader codebase"，保持紧凑：6-16 次工具调用，14-65 秒。
- 没有审查者自主运行过重型测试。观察到的每一次包级或重复测试运行，都是由 controller 写入的 prompt 明确要求的（"check all uses"、"run tests if useful, especially race-focused ones"、"does anything else read `Meta()`?"）。

根本原因，按影响排序：

1. **单任务质量 prompt 继承了一份合并就绪审查。** `code-quality-reviewer-prompt.md` 委托给 `requesting-code-review/code-reviewer.md`，后者询问架构、可扩展性、安全性、生产就绪度，并以"Ready to merge?"收尾。这一框架在单任务 diff 上授权了分支级别的广度。spec prompt 的 diff 范围护栏从未被传递过来。
2. **controller 在编写审查者 prompt 方面得不到任何指导**，于是它编造开放式指令（"check all uses"），审查者则照字面理解。
3. **流水线中存在重复工作。** 质量模板的"Plan alignment"维度重新检查了 spec 审查者刚刚验证过的内容。审查者重新运行了实现者已经运行过（并带着 TDD 证据报告过）的测试套件——而且是在相同的代码上。
4. **单任务审查与最终审查共用一个模板**，所以任何地方都没有"单任务窄、最终宽"的表示。

一份现场报告（`~/2026-06-09-code-quality-reviewer-scope-budget-issue.md`）首次标记了此问题。其引用的会话和标题数字无法验证，但其定性诊断在两次真实本地会话中得到确认。对其有一处修正：横向切面的审计（锁顺序、被改动的契约）有时是*正确的*审查方法——修复方案必须把广度置于一个被明确陈述的具体风险之后，而不是禁止它。

## 目标

- 单任务审查限定在任务范围内：以 diff 优先阅读、有依据地扩大范围、无冗余测试运行。
- 最终全分支审查保持当前的广度。
- 审查能发现的内容不减少。

## 非目标 / 明确保留

- **完整复审保留。** 当审查者在修复后复审时，它仍以完整的阅读广度审查整个任务。（它不会重新运行实现者刚刚在修改后的代码上运行过的测试。）此处有意拒绝了现场报告提出的"复审预算"补救方案：其最严重被引用案例（一次复审运行了 `-race` 和 `-count=100` 循环）的开销由下方的测试预算加以遏制，而不是靠收窄复审者的阅读范围。
- ~~**两个审查阶段保持分离。** Spec 合规性和代码质量仍是独立的 subagent，串行把关。不合并。~~ **被下方的成本迭代所取代**：线上 eval 经济性显示 per-dispatch 开销主导了成本，维护者把一切都摆上了台面。单任务阶段现在合并为一个任务审查者并产出两份裁决；独立的宽泛最终审查保留。
- **coordinator 保留模型判断权。** 不在审查中强制任何方向的模型层级。
- **`requesting-code-review/` 保持不动。** 它仍是最终分支审查和临时审查的宽泛模板。
- 裁决顺序（spec 合规性先于质量报告）、修复并复审循环、以及修复 Critical/Important 发现的要求，均不变。

## 成本迭代（上线后 eval 经济性）

线上 before/after 运行，在质量硬化文案（证据规则、约束传递、洁净输出）落地后，暴露出一个成本回归：go-fractals 从 42.8 分钟 / 14.5M token（首个任务级范围版本）变为 69.9 分钟 / 32.2M（硬化版本），同时达到与基线持平的质量（盲评 8.5 vs 8.5）。对每个 subagent 轮次的性能剖析将成本归因于，按顺序：廉价模型在多步工作上花费 2-3 倍轮次（1197 次 subagent 轮次中 678 次是 haiku）、per-dispatch 开销（每个任务 3 次 subagent 启动，每次重新推导 diff；controller 协调占了一半费用）、以及证据规则式的叙述。

- **迭代 1：**轮次数胜过 token 单价的模型指导（多步工作中层下限）、可选的内联 diff、cite-don't-narrate 证据、Important = 不能信任直到修复、仅对 Critical/Important 派发修复。结果：68.2 分钟 / 22.9M——token 降 29%，墙上时间持平；当措辞设为可选时，controller 在 22 次审查调度中仅有 2 次粘贴了 diff。
- **迭代 2：**单任务 spec 与质量审查合并进一个 `task-reviewer-prompt.md`（一个审查者、一次 diff 阅读、两份裁决；一次修复调度同时处理两类发现）；实现者迭代时运行聚焦测试，提交前运行一次完整套件。结果（go-fractals）：47.5 分钟 / 15.7M / $13.55——在每一维度上都超越基线，盲评 9/10 vs 基线 7/10。
- **迭代 3：**校准把阻断合并的可维护性损害（逐字重复、吞掉错误、无断言测试）命名为 Important，Minor 发现必须被粘贴进最终审查以供分诊；审查者的怀疑延伸到实现者的设计理由（"left it per YAGNI" 是一个主张，不是裁决）；diff 以文件形式交给审查者（`git diff > /tmp/sdd-task-N.diff`，重定向使它永不进入 controller 的上下文；审查者一次 Read 调用）——此举发生在把 diff 粘贴进 prompt 的指导无人采纳（11-17 次调度中 0-6 次）之后，出于本地合理的上下文经济性原因。
- **最终冻结配置（e355795），全部五个场景通过：**go-fractals 44.4 分钟 / 13.4M / $11.67（vs 基线时间 -32%、token -37%、美元 -27%）；svelte-todo 62.8 / 19.7M / $15.76（-21% / -28% / -25%）；rejects-extra-features $1.31（vs $1.88）；spec-reviewer-flaws 持平；planted-defect 场景（v3：对判断类调用设开放标记透明门槛，对名称承诺了验证却不执行的测试设必须修复门槛）通过，缺陷被捕获并修复。

### 迭代 4-5（2026-06-10）：方差诚实、结构性修复、正向配方

一次同配置重跑暴露了运行间方差（相同 prompt 下 44.4→57.1 分钟；审查者逃生舱胃口在 1.0→6.3 次工具调用/审查间摆动），因此所有后续论断都使用区间。在 go-fractals 上的五个并行实验变体，加上对真实本地会话的 transcript 挖掘（含负面结果的完整日志：`evals/docs/experiments/2026-06-10-sdd-cost-experiments.md`），产生了最终配置：

- **采纳：**final-review 包（最终审查者在 controller 模型价格下从 33→6 轮）；两个模板中 REQUIRED 的 `model:` 行（prose 指导曾在一次会话中途退化，继承了 opus 达 17 次调度，+$5）；task-brief + report 文件（`scripts/task-brief`；保真锚点、温和的上下文节省）；位于 `<git-dir>/sdd/progress.md` 的进度账本（真实会话曾在压缩后重新派发整段已完成的任务序列——约 22 个任务对应 269 次调度）；omnibus 最终修复器（某真实会话中按发现逐项修复的一波开销超过其全部任务）；范围限定的修复测试；唯一 SHA 范围的附属物命名（worktree/submodule 安全）；dispatch-composition 配方和审查者命名风险预算（经微测：正向配方 3.0 个逐字转录值 vs 禁令 4.4 vs 对照 3.6——禁令可能适得其反；见 `2026-06-10-positive-instruction-redesign-design.md`）。
- **测试后放弃：**controller 轮次批处理和并行调用流水线（controller 每条消息恰好发出一次工具调用——每次运行中 0 条多工具消息；其轮次中 46% 是思考/叙述，是 prompt 免疫的下限）；后台调度流水线（该机制在 7/28 时被采纳，但在这些场景上收益低于 ±6 分钟的噪声下限）。
- **最终验证配置（b81f35b 家族），全部门槛通过：**go-fractals 54.1-54.7 分钟 / 14.4-16.6M / $12.81-14.31（基线 64.9 / 21.2M / $16.07）；svelte-todo 55.0 分钟 / 19.3M / $14.99（基线 79.7 / 27.3M / $20.98）；planted-defect 通过 / $2.77。跨全部 8 次同设计 fractals 运行：44.4-57.1 分钟 / 13.4-20.0M / $11.67-14.84——最差的一抽在每一维度上都优于基线；典型中段节省约 20-25%。

## 设计

### 共享原则：不要在未改动的代码上重新运行测试

实现者的报告包含针对所审查代码的测试结果和 TDD RED/GREEN 证据。审查者通过阅读来验证。只有当阅读引发某个现有运行无法回答的具体疑虑时，审查者才运行测试——而且是聚焦测试，不是套件。在审查者 subagent 为只读的 harness 上（例如 Antigravity 将审查者模板映射到没有命令访问权限的 `research` 类型），审查者则在报告中指明它会运行哪个测试。

修复之后，实现者重新运行覆盖被修改代码的测试；复审者不重复该运行。当前没有任何机制强制此前提：`implementer-prompt.md` 仅描述初始的 implement-test-commit 流程，没有修复迭代指令。因此本 spec 还向 `implementer-prompt.md` 追加：修复一项审查发现后，重新运行覆盖被修改代码的测试，并将结果纳入修复报告。

该原则出现在两个审查者 prompt、实现者 prompt 和 controller 指导中。

### 1. 新文件：`skills/subagent-driven-development/code-quality-reviewer-prompt.md` 变为自包含

停止委托给 `requesting-code-review/code-reviewer.md`。单任务质量审查者获得自己的范围化 prompt 模板：

- **定调：**"You are reviewing one task's implementation for code quality." 一个任务级范围的门槛，而非合并审查。
- **Spec 合规性已定案：**spec 审查已通过；不要重新就需求或 plan 对齐进行争辩。
- **保留的审查维度：**代码质量（清晰度、重复、错误处理）、测试质量（真实行为，非 mock）、可维护性，以及既有的 SDD 专项检查（单一职责、独立可测性、来自 plan 的文件结构、本次改动贡献的文件增长）。删除：plan 对齐、安全/可扩展性/生产就绪度维度、合并裁决。
- **范围预算：**从 `git diff BASE..HEAD` 开始；先读改动文件；仅当为评估某个你能点名的具体风险时才检查相邻代码。横向切面改动——锁顺序、被改动的函数/API 契约、共享可变状态——都是合理的命名风险，足以支持检查调用点。默认不要爬取整个代码库。
- **测试预算：**上述共享原则，外加：不运行包级套件、竞争检测器，或重复/高计数运行，除非你已首先点名某个具体怀疑的不稳定测试或竞争。否则，在报告中建议进行重型验证，而非亲自运行。实现者报告的测试输出中的告警或噪声属于发现——输出应当洁净（实现者的自检也会检查这一点）。
- **证据规则：**审查者用 file:line 证据回答每一项 What-to-Check 条目，而非光秃秃的 yes/no。（在多次线上 eval 显示审查者放过了 prompt 直接点向它们的缺陷后加入——一个无障碍名称检查和一个临时目录清理检查都得到了无依据的"yes"，而缺陷正躺在被审查的 diff 中。）
- **只读规则**以精简形式保留：不得改动工作树、index、HEAD 或分支状态。当前模板中关于 `git worktree add` 的 how-to 句子不会被搬入本文件——一次范围限定的 diff 审查从不需要检出另一个修订版（与下方 spec-prompt 清理同理）。
- **裁决：**Strengths / Issues（Critical/Important/Minor）/ "Task quality: Approved | Needs fixes."

### 2. `skills/subagent-driven-development/spec-reviewer-prompt.md` 清理

- 移除 `git worktree add` 的 how-to 句子。只读规则保留；一次范围限定的 spec 审查从不需要检出另一个修订版。
- 化解 diff-only 护栏与"独立验证一切"之间的张力：通过将 diff 与需求对照阅读来判定 spec 合规性。实现者的 TDD 证据覆盖"它能跑"——适用共享测试原则。
- 新增第三条裁决通道：无法从 diff 验证的需求（存在于未改动代码中、跨任务）被作为显式的"⚠️ Cannot verify from diff — controller should check X"条目报告，而不是去爬取或默默放行。流程图中的二元通过/失败菱形无法路由此项，因此 controller 指导（§3）定义了处理方式：⚠️ 条目不阻塞调度质量审查者，但 controller 必须在标记任务完成前自行解决每一项（它持有 plan 和跨任务上下文）；controller 确认为真实缺口的条目，被当作失败的 spec 审查退回给实现者。
- 用有依据的怀疑取代虚构的前提"The implementer finished suspiciously quickly"：把实现者的报告当作关于代码的未验证主张。同样的不信任，没有捏造的事实。

### 3. `skills/subagent-driven-development/SKILL.md` controller 改动

- **模型选择：**用判断性指导取代"Architecture, design, and review tasks: use the most capable available model"——像挑选实现者模型那样挑选审查者模型，按 diff 的规模、复杂度和风险进行缩放。"Task complexity signals"清单被重新限定范围，以明确其要点描述的是实现任务；审查者模型选择遵循同一判断，因此窄 diff 审查不会自动映射到"宽代码库理解 → 最强模型"。
- **审查者 prompt 构造**（Red Flags 附近的新指导）：派发审查者时，在没有具体任务相关理由的情况下，不要写开放式指令（"check all uses"、"run race tests if useful"）；不要要求审查者重新运行实现者已在相同代码上运行过的测试；不要替审查者预判发现（绝不指示审查者忽略或不标记某个具体问题——疑似误报在审查循环中裁定）；单任务审查是任务级门槛——宽泛审查只发生一次，即最终的全分支审查。（预判规则在一次线上 eval 抓到 controller 捏造"the plan forbids a shared helper"主张并指示质量审查者不要标记一个植入的 DRY 违规后加入。）controller 还必须把绑定该任务的 spec/design 全局约束——版本下限、命名和文案规则、平台要求——纳入其粘贴的需求中：一次线上运行交付了 `go 1.26.1` 模块下限，违背了"Go 1.21+"的设计，因为没有任何审查者见过该约束。controller 还必须在每次调度时显式指定模型——被省略的模型会继承会话（通常是最贵的）模型，这会悄然抵消模型选择。
- **处理 spec-reviewer ⚠️ 条目**（新指导，与 Handling Implementer Status 并列）：controller 在标记任务完成前自行解决每个"cannot verify from diff"条目；确认的缺口作为失败的 spec 审查退回给实现者。
- **最终审查保持宽泛，且明确化：**最终的全分支审查者派发节点获得一个指向 `../requesting-code-review/code-reviewer.md` 的显式指针。（当前该模板仅能通过单任务质量 prompt 的委托到达；一旦移除该委托，未被引用的最终审查模板就会变成孤儿。）Integration 章节中关于 `superpowers:requesting-code-review` 提供"the code review template for reviewer subagents"的注释被更正为仅适用于最终审查。
- **示例工作流：**示例中的 quality-reviewer 行被更新为新裁决词汇（"Task quality: Approved"）；最终审查者的"ready to merge"行保留。
- 流程图拓扑不变；⚠️ 通道由 controller 指导处理，而非新增一条图分支。

## 本方案未修复的内容（已知、已推迟）

spec 审查者对照 controller 粘贴的任务文本来判定；它无法捕捉在 controller 从 plan 提取过程中被丢掉的需求。这是"controller 提供完整文本"这一架构属性，不是 prompt 问题，超出本次范围。

## 验证

- Plugin 基础设施测试（`tests/`）仍通过。
- 在改动前后运行 SDD skill-behavior evals（`git submodule update --init evals`，然后按 `evals/README.md`）。具体包括：`sdd-go-fractals`、`sdd-svelte-todo`、`sdd-rejects-extra-features`（端到端 SDD，含 spec 审查者的 YAGNI 门槛），以及 `spec-reviewer-catches-planted-flaws`。
- 本次改动暴露的已知 eval 缺口：当前没有场景在单个 SDD 任务内部植入代码质量缺陷并断言单任务质量审查者能捕获它，也没有场景测量每个审查者的探索开销（工具调用/grep 计数）。新增一个覆盖第一个缺口的场景（植入单任务质量缺陷 → 单任务审查者必须在最终审查前标记它）。对于探索开销，跨改动前后 eval transcript 手工比对审查者 subagent 的工具调用计数。
