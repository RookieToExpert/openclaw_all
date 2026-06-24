# AGENTS.md - OpenClaw Runtime Rules

本文件只定义 OpenClaw 在本工作区中如何运行：决策顺序、skill 发现、工具选择、执行预算、停止条件和知识库维护策略。
长期规则、入口边界、安全红线和用户偏好以 `MEMORY.md` 为准。

---

## 1. Source of Truth

文件职责：

* `AGENTS.md`：运行时策略、执行预算、skill 发现、工具选择、失败收敛。
* `MEMORY.md`：长期规则、入口边界、安全红线、用户偏好。
* `TOOLS.md`：工具索引、高频路由、命令文件跳转。
* `skills/*/SKILL.md`：专项 SOP、判断分支、fallback 和停止条件。
* `tools/*.md`：具体命令模板、参数、路径、脚本说明。
* `memory/YYYY-MM-DD.md`：当天动态状态、一次性查询结果、临时观察。
* `HEARTBEAT.md`：轻量唤醒上下文，不承载完整 SOP。

规则优先级：

1. 用户当前明确指令。
2. 写操作确认与安全红线。
3. `MEMORY.md` 长期规则。
4. 当前命中的 `skills/<skill-name>/SKILL.md`。
5. `TOOLS.md` 索引到的 `tools/*.md`。

事实优先级：

1. 实时只读查询结果。
2. 当天刚写入且范围一致的 `memory/YYYY-MM-DD.md`。
3. 历史经验或 Deep Knowledge 召回结果。

冲突时：

* 用户明确限制范围时，不得扩大范围。
* `MEMORY.md` 与 skill 冲突时，以 `MEMORY.md` 为准。
* skill 与 `tools/*.md` 冲突时，流程以 skill 为准，命令参数以 `tools/*.md` 为准。
* 动态事实冲突时，以实时只读查询结果为准。
* 不确定时只做最小只读验证。

---

## 2. Runtime Workflow

每次收到请求时：

1. 判断用户意图：查询 / 排障 / 创建 / 修改 / 删除 / 知识库维护。
2. 判断对象类型：Kubernetes / vcluster / 物理机 / MCCL / AFS-PVC-PV / OpenClaw 配置。
3. 读取 `MEMORY.md` 获取入口边界和安全红线。
4. 读取 `TOOLS.md` 获取候选 skill 或工具索引。
5. 只加载当前请求最匹配的 1 个主 skill。
6. 只有当前 skill 明确需要具体命令时，才读取对应 `tools/*.md`。
7. 先做只读验证。
8. 涉及写操作时，触发 `MEMORY.md` 的写操作确认流程。
9. 动态状态写入 `memory/YYYY-MM-DD.md`；长期稳定规则才候选进入 `MEMORY.md`。

禁止：

* 不要一开始加载所有 skill。
* 不要一开始加载所有 `tools/*.md`。
* 不要把历史动态记录当作实时事实。
* 不要在没有 SOP 支持时自行设计长排障链路。

---

## 3. Execution Budget

默认预算：

* 单轮最多加载 1 个主 skill。
* 单轮最多额外加载 1 个辅助 skill。
* 单轮最多读取 2 个 `tools/*.md`。
* 单轮最多执行 4 条只读查询命令。
* 单条命令默认超时 30 秒。
* 单轮最多下钻 2 轮。
* 单轮最多保留 200 行关键输出。

预算耗尽时必须停止，并返回：

1. 已执行的命令或已读取的文件。
2. 每一步的关键结果。
3. 当前停止原因。
4. 下一步安全选项。
5. 需要用户补充的精确信息。

如需扩大预算，必须先说明继续查询范围、是否扩大 vcluster / namespace / node / IP 范围，以及是否仍然只读。

---

## 4. Skill Discovery Rules

技能发现按“意图 + 对象 + 场景词”匹配。

流程：

1. 识别用户意图。
2. 识别对象类型。
3. 从 `TOOLS.md` 或 `skills/README.md` 找最匹配的 skill。
4. 只加载一个主 skill。
5. 只有主 skill 明确要求时，才加载辅助 skill。
6. 没有匹配 skill 时，只能使用 `MEMORY.md` + `TOOLS.md` + 最小 `tools/*.md` 路径。

触发原则：

* 流程型、排障型、带判断分支的问题优先使用 skill。
* 只是查一个具体命令时，可以不加载完整 skill，直接读对应 `tools/*.md`。
* 用户说“更新 openclaw memory”“更新 OpenClaw 知识库”“同步 OpenClaw 配置”“git pull openclaw_all”时，必须使用 `skills/update-openclaw-memory/SKILL.md`。
* 如果 skill 路径未加载、失效或 symlink-escape，不得假设该 skill 生效。

没有合适 skill 时：

* 简单只读查询可以按 `TOOLS.md` 的最小路径执行。
* 复杂排障必须停止并说明缺少匹配 SOP。
* 不得临时编造完整 SOP。

---

## 5. Tool Selection Rules

读取策略：

* 入口边界：读 `MEMORY.md`。
* 技能路由：读 `TOOLS.md` 或 `skills/README.md`。
* SOP：读匹配的 `skills/*/SKILL.md`。
* 具体命令：读匹配的 `tools/*.md`。
* 动态事实：实时只读查询验证。

工具规则：

* rayctl 是 Kubernetes / vcluster / AI 训练任务查询的首选工具。
* kubectl 只能作为当前 SOP 明确允许的降级工具。
* kubectl 只能查询已知 vcluster、namespace、resource name、UID、pod 或 node。
* ansible 只能用于 D 集群物理机器 / Linux 系统级查询。
* 不同入口之间不得自动切换，除非 `MEMORY.md` 或当前 skill 明确允许。

命令规则：

* 没有必要时不要生成长命令。
* 没有依据时不要编造命令链路。
* 用户输入必须视为不可信输入。
* 所有变量必须加引号。
* 禁止把用户输入拼进 `eval`、反引号、未转义 shell 片段。
* 默认只读。
* 写操作必须遵守 `MEMORY.md` 的写操作确认规则。

---

## 6. Failure Handling

任何失败都必须收敛，不得无限排查。

失败包括：

* 命令返回空。
* 命令超时。
* 权限不足。
* kubeconfig 不存在。
* vcluster / namespace / job / pod / node 不明确。
* 工具不支持该能力。
* 输出不足以支撑结论。
* 当前 SOP 没有定义下一步。

失败时必须返回：

1. 已执行。
2. 观察结果。
3. 停止原因。
4. 未继续执行的原因。
5. 下一步安全选项。

失败时禁止：

* 自动扩大查询范围。
* 自动换入口。
* 自动进入全量扫描。
* 自动执行写操作。
* 用猜测补全缺失字段。

---

## 7. Memory Update Workflow

更新知识库时，必须遵守：

```text
draft → classify → tools/ 或 skills/ → TOOLS.md 索引 → MEMORY.md（仅长期稳定规则）→ 用真实问题验证
```

分类规则：

* 原则 / 入口边界 / 全局规则 / 安全红线：进 `MEMORY.md`。
* 路由 / 索引 / 跳转关系：进 `TOOLS.md`。
* 命令模板 / 参数 / 脚本：进 `tools/*.md`。
* SOP / 决策树 / 标准流程：进 `skills/*/SKILL.md`。
* 一次性状态 / 查询结果 / 当天记录：进 `memory/YYYY-MM-DD.md`。

禁止：

* 禁止直接把新信息先塞进 `MEMORY.md`。
* 禁止把长命令塞进 `MEMORY.md`。
* 禁止在多个文件重复维护同一段命令。
* 禁止让 skill 覆盖 `MEMORY.md` 的全局安全规则。

---

## 8. Heartbeat Rules

heartbeat 必须轻量运行：

* 使用 light context。
* 不加载完整历史。
* 不加载所有 skill。
* 不加载所有 tools。
* 不执行完整排障流程。
* 不修改 `MEMORY.md`。
* 不做写操作。

---

## 9. Output Rules
默认输出偏好以 `MEMORY.md` 为准。

如果查询失败或触发边界，必须说明：

1. 已执行的步骤。
2. 观察结果。
3. 停止原因。
4. 未继续执行的原因。
5. 下一步安全选项。

触发边界时必须停止，说明继续执行会扩大的范围，并等待用户明确确认。

