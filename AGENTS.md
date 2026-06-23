# AGENTS.md - OpenClaw Runtime Rules

本文件只定义 OpenClaw 在本工作区中如何工作：运行时规则、决策顺序、技能发现、工具选择、记忆更新和执行策略。
它不是业务知识库；长期业务知识、入口路由和安全规则以 `MEMORY.md` 为准。

## 1. Source Of Truth

```text
AGENTS.md
↓
MEMORY.md
↓
TOOLS.md
↓
skills/*/SKILL.md
↓
tools/*.md
↓
实时执行 / 实时查询
```

文件职责：
- `AGENTS.md`：workflow / runtime policy / discovery / execution
- `MEMORY.md`：长期稳定知识、入口路由、全局优先级、安全规则
- `TOOLS.md`：路由、索引、参考引用
- `skills/*/SKILL.md`：SOP、工作流、决策树
- `tools/*.md`：命令模板、脚本、参数
- `memory/YYYY-MM-DD.md`：动态记录、一次性查询结果、当天状态

优先级与冲突处理：

1. 用户明确指令
2. 写操作确认与安全红线
3. `MEMORY.md`
4. 匹配的 skill
5. `TOOLS.md`
6. 匹配的 `tools/*.md`
7. 实时查询结果

- 动态记录与实时查询冲突时，以实时查询为准
- 不确定时先只读验证

## 2. Runtime Workflow

每次收到请求时：

1. 判断用户意图：查询 / 排障 / 创建 / 修改 / 删除 / 知识库维护
2. 判断对象类型：Kubernetes / vcluster / 物理机 / MCCL / AFS-PVC-PV / 知识库
3. 读取 `MEMORY.md` 获取入口路由和全局约束
4. 读取 `TOOLS.md` 获取候选 skill 或工具索引
5. 只加载当前请求匹配的 skill
6. 仅在需要具体命令时读取对应 `tools/*.md`
7. 先做只读验证，再决定是否进入写操作确认
8. 执行后决定写入 `memory/YYYY-MM-DD.md` 还是更新长期知识

不要一开始加载所有 skill、tools 或历史记录。

## 3. Skill Discovery Rules

技能发现按“语义匹配 + 对象匹配”进行：

1. 识别请求意图
2. 识别对象类型
3. 去 `TOOLS.md` 或 `skills/README.md` 找最匹配的 skill
4. 只加载一个主 skill；必要时再加载辅助 skill
5. 没有匹配 skill 时，再退回 `MEMORY.md` + `TOOLS.md` + `tools/*.md`

触发原则：
- 请求包含明确场景词时，优先命中对应 skill
- 流程型、排障型、带判断分支的问题优先使用 skill
- 只是查一个具体命令时，可不加载完整 skill，直接按索引读取对应 `tools/*.md`
- 当用户说“更新 openclaw memory”“更新 OpenClaw 知识库”“同步 OpenClaw 配置”“git pull openclaw_all”时，必须使用 `skills/update-openclaw-memory/SKILL.md`

## 4. Tool Selection Rules

工具选择最小化：

- 先选 skill，再选 tool
- 先选索引，再读命令模板
- 没有必要时不要读取无关 `tools/*.md`
- 没有必要时不要生成长命令
- 没有依据时不要编造命令链路

读取策略：
- 入口路由：读 `MEMORY.md`
- 技能路由：读 `TOOLS.md` 或 `skills/README.md`
- SOP：读匹配的 `skills/*/SKILL.md`
- 具体命令：读匹配的 `tools/*.md`

## 5. Memory Update Workflow

更新知识库时，必须遵守：

```text
draft
↓
classify
↓
tools/ 或 skills/
↓
TOOLS.md 索引
↓
MEMORY.md（仅长期稳定知识）
↓
用真实问题验证
```

分类规则：
- 原则 / 入口路由 / 全局规则 / 安全红线：进 `MEMORY.md`
- 路由 / 索引 / 跳转关系：进 `TOOLS.md`
- 命令模板 / 参数 / 脚本：进 `tools/*.md`
- SOP / 决策树 / 标准流程：进 `skills/*/SKILL.md`
- 一次性状态 / 查询结果 / 当天记录：进 `memory/YYYY-MM-DD.md`

禁止直接把新信息先塞进 `MEMORY.md`。

## 6. Write Confirmation Policy

任何写操作执行前，必须先输出：

1. 将要执行的操作
2. 影响范围
3. 可能风险
4. 回滚方式或停止方式
5. 需要用户确认的问题
6. 涉及时间窗口时，显式核对北京时间与集群 UTC 时间换算

用户明确回复以下内容前，禁止执行真实修改：

```text
确认
同意
可以执行
执行
yes
```

如果用户只是问“怎么做”，只能给命令和解释，不得代为执行。

## 7. Execution Policy

执行时默认遵循：

- 先分类，再路由
- 先只读，再写入
- 先验证，再下结论
- 先读索引，再读细节
- 只加载当前请求需要的最小上下文

执行中禁止：
- 把物理机问题误判成 Kubernetes 问题
- 把 Kubernetes 资源问题误判成物理机问题
- 在没有依据时混用入口
- 在多个文件重复维护同一段长命令
- 让 skill 与 `MEMORY.md` 的全局原则冲突

## 8. Efficiency Rules

- `AGENTS.md` 只描述 HOW，不承载业务知识
- 路由表放在 `TOOLS.md`
- 长期业务知识放在 `MEMORY.md`
- 具体命令只放在 `tools/*.md`
- 流程和判断逻辑只放在 `skills/*/SKILL.md`
- 优先局部读取，避免全量加载
