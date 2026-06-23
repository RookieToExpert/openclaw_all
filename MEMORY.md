# MEMORY.md - OpenClaw Long-term Operating Memory

本文件只保存长期稳定的背景、判断原则、安全红线、入口路由和跨任务优先级。
具体 IP、登录方式、命令模板、rayctl 参数、脚本内容放到 `TOOLS.md` 或对应 `skills/<skill>/SKILL.md`。
一次性查询结果、当天状态、节点列表、故障现象、动态资源统计写入 `memory/YYYY-MM-DD.md`，并标注查询时间和复核方式。

## OpenClaw 知识库更新准则

新增或更新 SOP 时，必须按以下顺序处理：

1. 先写到草稿。
2. 判断内容类型：原则 / 命令 / 流程 / 临时记录。
3. 如果是命令、路径、参数、脚本模板，先进 `tools/`。
4. 如果是完整处理流程，再进 `skills/<skill-name>/SKILL.md`。
5. 如果新增了 tools 或 skills，需要在 `TOOLS.md` 补索引。
6. 只有全局长期规则、安全红线、入口路由、跨多个 SOP 都适用的原则，才进入 `MEMORY.md`。
7. 更新后必须检查是否存在重复、冲突、过期命令。
8. 最后用一个真实问题测试 OpenClaw 是否能走对流程。

判断准则：

- `MEMORY.md`：长期稳定原则、安全红线、入口路由、优先级。
- `TOOLS.md`：全局索引，不放大段命令。
- `tools/*.md`：具体命令模板、路径、参数、工具用法。
- `skills/*/SKILL.md`：专项 SOP、处理流程、判断逻辑。
- `memory/YYYY-MM-DD.md`：动态查询结果、临时状态、一次性记录。

禁止事项：

- 不要把一次性查询结果写进 `MEMORY.md`。
- 不要把长命令和大段脚本塞进 `MEMORY.md`。
- 不要在多个文件里重复维护同一段长命令。
- 不要让 skill 和 `MEMORY.md` 的全局原则冲突。
---

## 0. 全局优先级

处理任何请求时按以下顺序决策：

1. 用户当前明确指令。
2. 写操作确认、安全红线、不可逆操作保护。
3. 本文件中的长期规则和入口路由。
4. 当前命中的 skill 流程。
5. `TOOLS.md` 中的最新命令模板和环境路径。
6. `memory/YYYY-MM-DD.md` 中的动态记录。

冲突处理：

- skill 与 `MEMORY.md` 冲突时，以 `MEMORY.md` 为准。
- skill 与 `TOOLS.md` 冲突时，流程以 skill 为准，命令参数以 `TOOLS.md` 最新模板为准。
- 动态记录与实时查询冲突时，以实时查询为准。
- 不确定时必须说明不确定点，并优先做只读验证。

---

## 1. 先判断对象类型，再选择入口

执行任何动作前，先判断目标属于哪一类。

| 请求类型 | 正确入口 | 说明 |
|---|---|---|
| Kubernetes / vcluster / host cluster / rayctl / kubectl / vcjob / Pod / PVC / PV / AFS / Service / Endpoint / Webhook / PodGroup | D 集群开发机 | Kubernetes 逻辑资源操作 |
| 查询某个物理节点 IP / hostname 属于哪个 vcluster | D 集群开发机 + rayctl | host cluster 视角节点归属查询 |
| 查看某台 D 集群机器上的目录、文件、日志、进程、磁盘、内存、NPU、网卡、路由、DNS、本地脚本 | 堡垒机 → 跳板机 → sensetime 用户 → ansible 单 IP | 物理机器 / 操作系统级查询 |
| 在某台 D 集群机器上执行脚本、改配置、删除文件、重启服务 | 堡垒机 → 跳板机 → sensetime 用户 → ansible 单 IP，并且必须先确认 | 机器写操作或高风险操作 |

关键判断：

- “看某个 IP / hostname 上的目录、文件、日志、进程、磁盘、内存、NPU、网卡、路由、DNS”属于 D 集群机器类操作。
- “看 Pod、vcjob、PVC、PV、AFS、vcluster、PodGroup、Service、Endpoint、Webhook”属于 Kubernetes / 集群类操作。
- “看某个节点属于哪个 vc / vcluster”属于节点归属查询，走开发机上的 rayctl。
- 不要因为用户说“D 集群节点”就默认走开发机。
- 不要把“物理机器文件系统查询”误判为“Kubernetes Node 查询”。

---

## 2. 平台长期背景

当前平台是基于 Kubernetes 的 AI 训练 / 推理平台。
底层包含宿主 Kubernetes 集群，上层部署多套 vcluster。
vcluster 与 host cluster 之间的 Pod、Node 等资源通过 syncer 同步。

常见对象关系：

- Host Cluster：真实物理节点、底层资源、宿主集群 kubeconfig。
- vCluster：租户视角 Kubernetes 集群。
- rayctl：平台优先使用的任务 / AFS / PVC / Node 查询与操作工具。
- kubectl：rayctl 信息不足或能力不满足时的降级工具。
- Volcano / vcjob：训练任务常见调度与任务 CRD。
- D 集群机器：真实物理机 / 节点，通过跳板机上的 sensetime 用户使用 Ansible 操作。

### 2.1 机器类型判断原则

- 每个 vcluster 通常只对应一种主要机器类型，不要默认一个 vcluster 内混合多种机器类型。
- 判断任务模板、资源规格、机器类型时，可以先根据 vcluster 名称推断，再用节点 label / allocatable / rayctl 输出验证。
- 不要仅凭 `ring-controller.atlas: ascend-910b` 判断机器类型，该标签可能是默认标签。
- 机器类型最终应结合 `resource.compute.sensecore.cn/machine-type`、节点 allocatable、rayctl 模板映射共同判断。
- IP 段只能作为初筛线索，不作为最终结论；实际排障前必须用 rayctl 或 Kubernetes 实时信息验证。

常用 vcluster 族与芯片初筛：

| vcluster 族 | 芯片 / 类型 |
|---|---|
| `a2-*` | 华为 / NPU 910B |
| `a3-*` | 华为 / NPU 910C |
| `c550-jiaofu` | 沐曦 / MUXI 风冷 |
| `c550-ai4s` | 沐曦 / MUXI 风冷 |
| `c550-h3c` | 沐曦 / MUXI 液冷 |
| `c550-mohe` | 沐曦 / MUXI 超节点 |

---

## 3. Kubernetes / vcluster 操作原则

任何 Kubernetes 逻辑资源相关操作统一走：

```text
本地 → D 集群开发机 → rayctl 优先 → kubectl 兜底
```

原则：

1. 先到 D 集群开发机。
2. 优先使用 rayctl。
3. 生成 Kubernetes / vcluster / Node / Job / PVC / AFS / ECS 命令前，先参考 `TOOLS.md` 的 rayctl 小节。
4. rayctl 信息不足、能力不满足或排障需要时，再用 kubectl。
5. 读操作可以直接执行。
6. 写操作必须先输出计划并等待用户明确确认。

特别注意：

- 查询某个节点属于哪个 vcluster：走开发机，用 host cluster 视角 rayctl。
- cordon / uncordon Kubernetes Node：默认走 rayctl 节点调度控制，不要优先生成 `kubectl cordon` / `kubectl uncordon`，除非 rayctl 不可用或用户明确要求 kubectl。
- 查看物理机器目录、日志、进程、磁盘、NPU、网卡、DNS：不是 Kubernetes 操作，不能走开发机。
- 开发机不是 D 集群物理机器操作入口。

---

## 4. D 集群机器类操作原则

任何 D 集群物理机器 / 节点 / 单机操作系统环境查询或操作统一走：

```text
本地 → 堡垒机 → 跳板机 → sensetime 用户 → ansible 单 IP
```

适用对象：

- 某台 D 集群物理 IP / hostname。
- 机器上的文件、目录、日志。
- `~sensetime` 家目录。
- 本地进程、磁盘、内存、CPU。
- NPU / `mx-smi`。
- 网卡、路由、DNS。
- 本地脚本，例如 `mccl.sh`。

原则：

1. 不要从本地直接 SSH 到 D 集群目标节点。
2. 不要从开发机跳到 D 集群目标节点。
3. 不要在本地 OpenClaw exec 环境直接执行 ansible。
4. 不要在开发机执行 D 集群机器类 ansible。
5. 单机操作必须在跳板机 sensetime 用户下执行 Ansible 单 IP。
6. 默认不要依赖 inventory 文件做单机操作。
7. 对机器执行写操作、重启、改配置、跑影响业务的脚本前，必须先向用户确认。

---

## 5. vcjob / Volcano / 任务生命周期原则

- VC 集群中的训练任务优先用 rayctl 查询。
- 任务排障前先判断粒度：单个任务 / Pod / UID 问题走单任务诊断入口；某个 VC / 分区整体任务情况、Pending 分布、前端任务数异常、历史任务堆积、哪个 VC 大量提交任务等问题走分区级任务概览入口。
- 分区级概览只用于先定位范围和异常集中点；需要解释单个任务原因时，再下钻到单任务诊断。
- 具体 rayctl 命令、参数和入口变化以 `TOOLS.md` 索引到的 `tools/rayctl-kubectl.md` 与 `skills/vcjob-debug/SKILL.md` 为准。
- 需要 kubectl 兜底查 `vcjob` / `PodGroup` / Pod 时，必须先切到对应 vcluster kubeconfig。
- `vcjob` / `PodGroup` 是 vcluster 内资源，禁止用 host cluster kubeconfig 查询或兜底。
- rayctl 输出的 `VC` 显示名不一定等于 `/root/D/` 下的 kubeconfig 文件名，必须先确认实际 kubeconfig 文件存在。
- 不再使用已废弃的旧任务查询入口。
- 不要用 Kubernetes 原生 `jobs` 判断 Volcano 任务。
- 如果 vcluster kubeconfig 不存在或查不到资源，不能回退到 host cluster kubeconfig 查询 `vcjob` / `PodGroup`。
- 对 `NotEnoughResources` 的判断不能只看总资源量，应按单节点 requests / allocatable / 已调度 requests 精确判断。
- 目标 Pod 可调度的条件是同一台节点同时满足 CPU / memory / accelerator / machine-type / nodeSelector / affinity / taint / unschedulable 等约束。

任务创建原则：

- 提交任务必须使用 vcluster kubeconfig。
- 禁止使用 host cluster kubeconfig 提交 vcluster 任务。
- 创建任务优先使用 rayctl。
- 创建任务属于写操作，必须先确认目标 vcluster、namespace、job name、template、image、command、资源、PVC、影响范围。

---

## 6. AFS / PVC / PV 生命周期原则

- 查询 AFS / PVC / PV 优先使用 rayctl。
- AFS 名称由用户提供，不要擅自拼接或修改前缀。
- 如果 rayctl 返回 Host PV 名称，再直接查询该 PV。
- 已有完整资源名时，不要再用 grep 模糊匹配。
- 在 vcluster 中创建 PVC 禁止直接用 `kubectl create pvc`，必须使用 rayctl 创建。
- PVC 命名必须严格为 `pvc-<afs-name>`。
- 禁止给 PVC 名私自追加后缀、时间戳、随机字符串或 UUID。
- PVC 创建属于写操作，必须先确认。
- 创建 PVC 后必须检查。
- 如果 PVC 是 Pending，必须立即停止后续任务创建，把 Pending 原因返回给用户。

---

## 7. MCCL 测试原则

MCCL 测试必须先区分场景：

1. 单机 MUXI MCCL 维修验收测试。
2. 多机 / 平台纳管 MCCL 测试。

单机测试原则：

- 走 D 集群机器类入口：堡垒机 → 跳板机 → sensetime → ansible 单 IP。
- 不创建 vcjob。
- 不打 Kubernetes node label。
- 不删 Kubernetes node label。
- 不为了测试先 uncordon。
- 节点如果原本处于维修 / cordon 状态，应保持原状态。
- 只有用户明确要求“通过后 uncordon”时，测试通过后才走开发机用 rayctl uncordon。

多机测试原则：

- 走 Kubernetes / vcluster 入口：开发机 → rayctl / kubectl。
- 该场景才可能涉及 Kubernetes node label、vcjob、PodGroup、动态排除节点。
- 打标、删标、创建任务、删除任务都属于写操作，必须先确认。
- 出现坏节点时，可以动态排除，但必须同步调整 `minAvailable`、worker replicas、`CARD_NUM`。
- 测试结果输出时，保留原始日志，不自行改写。
- 如需摘要，应基于日志中的原始 `Avg bus bandwidth` 行。

具体流程见 `skills/mccl-test/SKILL.md`。

---

## 8. 写操作强制确认

任何写操作执行前，必须先输出：

1. 将要执行的操作。
2. 影响范围。
3. 可能风险。
4. 回滚方式或停止方式。
5. 需要用户确认的问题。
6. 涉及时间窗口筛选时，必须显式核对北京时间与集群 UTC 时间的 8 小时时区转换，严禁直接混用。

用户明确回复以下类似内容前，禁止执行真实修改：

```text
确认
同意
可以执行
执行
yes
```

如果用户只是问“怎么做”，只能给命令和解释，不得代为执行。

写操作包括但不限于：

- `kubectl delete` / `patch` / `create` / `apply` / `label` / `annotate` / `scale` / `rollout restart`。
- `rayctl job create` / `rayctl pvc create` / `rayctl node cordon` / `rayctl node uncordon`。
- 删除或修改文件。
- 修改 OpenClaw 配置、gateway、agent、model 配置。
- 修改 `MEMORY.md`、`TOOLS.md`、skill 文件。
- 创建临时脚本。
- 重启服务、关机、重启机器。
- 删除 PVC / PV / Pod / Job / Webhook。
- 执行会影响业务任务的脚本。

批量删除或通过 TTL 回收 Pod、vcjob、PVC、PV、Webhook、Node label 等高风险操作必须分阶段：

1. 只读筛选目标。
2. 展示数量、namespace、时间窗口、状态条件、样例。
3. 明确 kubeconfig / vcluster。
4. 给出 dry-run 或预览命令。
5. 等待确认后再执行。
6. 必须按原始对象清单复查资源和关联对象是否仍存在。
7. TTL 无法回收的对象若要改为直接 delete，必须重新展示剩余范围并再次确认。

---

## 9. Skills 使用原则

- D 集群日常操作以本文件和 `TOOLS.md` 的入口规则为准。
- skill 用于专项任务流程，不用于覆盖全局安全规则。
- 旧的 `d-cluster` skill 不作为日常操作依据，除非确认内容最新且无冲突。
- `mccl-test` skill 仅在 MCCL 测试场景使用。
- `vcjob-debug` skill 用于 Volcano / vcjob / PodGroup / Pending / NotEnoughResources 排障。
- `pvc-afs` skill 用于 AFS / PVC / PV 查询、创建、Pending 排障。
- `k8s-cleanup` skill 用于批量删除 Pod / vcjob、历史 vcjob TTL 回收、断线续查和清理后核验。
- `dcluster-machine-op` skill 用于 D 集群物理机器单机查询和操作。

如果 OpenClaw 日志出现：

```text
Skipping escaped skill path outside its configured root
reason="symlink-escape"
```

说明该 skill 没有被安全加载，不能假设该 skill 生效。

重新启用或新增 skill 前必须确认：

1. skill 内容是最新的。
2. skill 路径没有 symlink-escape。
3. skill 规则不与 `MEMORY.md` / `TOOLS.md` 冲突。

---

## 10. 输出风格与排障原则

- 优先给可执行、可验证的步骤。
- 对不确定的事实明确说“不确定”，不要猜。
- 复杂操作先给只读检查命令。
- 排障结论要区分“确定事实”“推断”“下一步验证”。
- 用户给出日志时，先解释日志说明了什么，再给下一步命令。
- 用户问“是不是某原因”时，不要直接肯定；要说明支持证据、反证和还需要看的点。
