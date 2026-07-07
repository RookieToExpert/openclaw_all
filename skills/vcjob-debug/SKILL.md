# vcjob-debug Skill

## 触发条件

用于 vcjob / Volcano Job / PodGroup / Pod 调度与任务状态排障，包括：

* Pending / NotEnoughResources / Startup / Running 异常 / Failed。
* 单任务、Pod、UID、PodGroup 定位。
* Volcano scheduler 未生效、PodGroup Pending / Inqueue 但 Pod 未创建、Volcano 组件配置或日志导致无法调度。
* 昇腾 NPU 任务中 910B / 910C 的 annotation、nodeSelector、MinResources、volcano-npu plugin 架构不匹配等调度问题。
* VC / 分区级任务概览、Pending 分布、历史任务堆积。
* Failed 任务疑似坏节点、HCCL / ACL / timeout / OOM / 信号终止排查。

## 相关工具文件

* `tools/rayctl-kubectl.md`
* `tools/fault-records.md`
* `tools/job-templates.md` （仅用户要求重跑 / 创建任务 / 生成任务模板时读取）

## 前置原则

* 必须遵守 `MEMORY.md` 的入口路由、写操作确认和禁止扩大范围规则。
* Kubernetes / vcluster 资源走 D 集群开发机。
* 优先 rayctl；kubectl 只按当前 SOP 允许的范围兜底。
* `vcjob` / `PodGroup` / vcluster 内 Pod / Event / 日志必须使用对应 vcluster kubeconfig。
* 禁止用 host cluster kubeconfig 查询或兜底 vcluster 内 `vcjob` / `PodGroup`。
* 缺少 vcluster、namespace、job、pod、node、UID 等关键标识时，必须停止并返回缺失信息。

---

## 标准流程

### 1. 判断问题粒度

先判断用户问题属于哪类：

* 单个任务 / Pod / UID / PodGroup：走单任务诊断。
* 某个 VC / 分区整体任务情况、Pending 分布、历史任务堆积：走分区级概览。
* Failed 且疑似通信、硬件、节点问题：走 Failed 坏节点排查。

---

### 2. 单任务诊断

目标：

1. 用 rayctl 定位任务。
2. 获取 VC、namespace、job、pod、PodGroup、状态和初步原因。
3. 判断是否需要切换到 vcluster kubeconfig 下钻。
4. 如果 rayctl 未返回足够关键标识，停止并返回缺失项。

具体命令以 `tools/rayctl-kubectl.md` 的单任务查询模板为准。

下钻条件：

* 需要看 vcluster 内 Pod / PodGroup / Event。
* 需要确认 Pending / Failed 原因。
* 需要查看 Pod requests、nodeSelector、affinity、toleration。
* 需要确认 schedulerName、PodGroup conditions、Volcano controller / scheduler / admission 配置或日志。
* 需要查看 Pod 日志。

停止条件：

* 未定位到实际 vcluster kubeconfig。
* namespace / pod / job / PodGroup 不明确。
* 目标资源不存在，且当前 SOP 没有允许扩大范围。
* 用户没有提供足够关键标识，rayctl 也无法补齐。

---

### 2.1 Volcano 无法调度专项分支

进入条件：

* `rayctl job get` 或用户描述显示任务无法调度、Pending、Pod 未创建、PodGroup 异常。
* 已确认实际 vcluster kubeconfig。
* 已知 namespace，且 job / pod / PodGroup 至少有一个关键标识。

查询顺序：

1. 确认任务是否实际走 Volcano：
   * Pod 已存在时看 `spec.schedulerName`。
   * Pod 未创建时看 vcjob / workload 模板里的 schedulerName、annotations、batch scheduler 配置。
   * AMS / Deployment / LWS 等原生 workload，检查是否存在触发 Volcano admission 的 annotation，例如 `ams.studio.sensecore.cn/scheduler: volcano`。
2. 查 PodGroup：
   * `phase` 是 Pending / Inqueue / Running。
   * `minAvailable` / `minMember` 与实际期望副本数是否一致。
   * `Status.Conditions` 的 reason / message 是本分支的主要依据。
3. 按 PodGroup / Event 文案分类：
   * `pod group is not ready`：优先确认 Pod 是否全部创建、controller 是否报 `Failed to create pod`。
   * `Insufficient cpu` / `Insufficient memory` / accelerator 不足：切到 NotEnoughResources 资源判断模板，不只看总量。
   * `NodeAffinity predicates failed` 或 node selector mismatch：核对目标 Pod 的 nodeSelector / affinity 与候选节点 label。
   * `capacity overused` 且出现 `attachable-volumes-csi-csi-provisioner`：检查 PVC storageClass provisioner 是否应在 Volcano `--ignored-provisioners` 中忽略。
   * `node check failed` / `log by search keywords`：继续查 scheduler 日志关键字；如涉及 NPU，检查 NPU device info configmap。
4. PodGroup Pending 但配置看似正常时：
   * 检查 `volcano-scheduler-configmap` 的 actions 是否仍包含不合适的 `enqueue`。
   * 检查 scheduler plugins 中 `proportion` / `capacity` 的使用是否符合当前平台预期。
   * 这里只读判断并输出建议；修改 configmap、重启 volcano-scheduler 属于写操作，必须停止并等待用户确认。
5. PodGroup Inqueue 但 Pod 未创建时：
   * 查 `volcano-controllers` 日志是否有 `Failed to create pod`。
   * 检查 vcjob 是否启用了 MPI plugin 但实际是单机任务。
   * 检查 `priorityClassName` 是否存在。
6. 昇腾 NPU 任务额外检查：
   * 910C：检查 vcjob / PodGroup 是否有 `sp-block` annotation。
   * 910C / 910B：检查 nodeSelector 是否匹配 `accelerator-type: module-910c-8` 或 `module-910b-8`。
   * 检查 PodGroup 是否设置 MinResources，关注 `nil NPU` 相关日志。
   * 检查 volcano-npu plugin 文件名是否与 scheduler 所在节点架构匹配：x86 使用 `x86_64` 后缀，arm 使用 `aarch64` 后缀。

停止条件：

* 需要查 scheduler / controller / admission 的日志或 configmap，但未确认其所在 namespace 或组件名。
* 需要修改 configmap、重启 scheduler / controller / admission、删除异常 Pod / vcjob。
* 需要扩大到全 vcluster / 全 namespace / 全 VC 扫描。
* 缺少 job / pod / PodGroup / namespace / vcluster kubeconfig 关键标识。

输出结构：

```text
结论：
- 当前最可能卡点：schedulerName / PodGroup / 资源不足 / selector-affinity / Volcano 配置 / controller 创建 Pod / NPU 专项 / 不确定

证据：
- rayctl 初筛：
- Pod / vcjob 模板：
- PodGroup phase 与 conditions：
- Event / controller / scheduler 关键日志：

判断：
- 确定事实：
- 推断：
- 待验证点：

下一步：
- 只读可继续查什么：
- 如需写操作，需用户确认的操作与影响范围：
```

具体命令以 `tools/rayctl-kubectl.md` 的 Volcano 无法调度查询模板为准。

---

### 3. 分区级任务概览

适用：

* 用户询问某个 VC / 分区整体任务情况。
* 用户询问 Pending 分布、前端任务数异常、历史任务堆积、哪个 VC 大量提交任务。
* 用户明确要求全局或分区级概览。

目标：

1. 用分区级查询定位异常集中点。
2. 输出 VC / namespace / 状态 / 数量 / 样例。
3. 如需解释单个任务原因，再回到单任务诊断。

具体命令以 `tools/rayctl-kubectl.md` 的分区级任务查询模板为准。

边界：

* 分区级概览只用于定位范围。
* 不要用分区级结果直接判断单任务 Pending / Failed 根因。
* 如果用户没有授权全局概览，不得遍历所有 vcluster。

---

### 4. vcluster 内资源下钻

下钻前必须确认：

* 实际 vcluster kubeconfig 文件存在。
* namespace 已知。
* job / pod / PodGroup 至少有一个关键标识已知。

查询目标：

* `vcjob`
* Pod
* PodGroup
* Event
* Pod YAML
* Pod 日志

具体命令以 `tools/rayctl-kubectl.md` 的 vcluster 资源查询模板为准。

禁止：

* 不要把 VC 显示名直接拼成 `/root/D/<VC>`。
* 找不到 vcluster kubeconfig 时，不得回退到 host cluster kubeconfig 查询 `vcjob` / `PodGroup`。
* 不要用 Kubernetes 原生 `jobs` 判断 Volcano 任务。

---

### 5. NotEnoughResources 判断

判断原则：

* 不要只看 vcluster 总资源量。
* 必须判断是否存在同一台节点同时满足所有 requests 和调度约束。
* 目标 Pod 可调度条件包括 CPU、memory、accelerator、machine-type、nodeSelector、affinity、taint/toleration、unschedulable、quota、PodGroup 等。

诊断步骤：

1. 获取目标 Pod requests、limits、nodeSelector、affinity、tolerations、schedulerName。
2. 获取候选节点 allocatable、taints、labels、unschedulable 状态。
3. 获取候选节点已调度 Pod 的 requests。
4. 判断单节点是否可容纳目标 Pod。
5. 如果资源看似足够，继续检查 PodGroup、队列、quota、scheduler event、webhook、Kube-OVN 等。

结论分类：

* `fit > 0`：理论上有节点可容纳，继续查调度约束或调度器事件。
* `fit = 0` 且同一类资源不足：资源确实不足。
* `fit = 0` 但资源分散在不同节点：资源碎片化。
* 资源足够但仍 Pending：查 PodGroup、quota、scheduler、node 状态、webhook、Kube-OVN。

具体命令以 `tools/rayctl-kubectl.md` 的资源判断模板为准。

---

### 6. Failed 任务坏节点排查

当 Failed 任务日志没有明确代码级报错，或者出现通信、硬件、系统终止线索时，优先排查坏节点。

触发线索：

* HCCL / hcom / hccl comm / fftsplus。
* task timeout / timeout。
* ACL / AclrtSynchronizeStreamWithTimeout / error code 107020。
* SIGABRT / SIGTERM / SIGKILL / ExitCode 134。
* OOM / OutOfMemory / killed。
* link down / Link_DOWN。
* NPU / 掉卡 / 缺卡 / health / hwlog / dmesg。

流程：

1. 定位任务所有 Pod 及所在 node。
2. 查看代表性 Pod 日志，确认失败时间、错误关键词和退出方式。
3. 提取异常 Pod 所在节点 IP。
4. 按 `tools/fault-records.md` 查询维修记录。
5. 对照任务失败时间与维修时间。
6. 判断是否存在坏节点、坏端口、坏卡或持续性硬件缺陷。
7. 输出任务是否需要重跑、是否建议排除节点后重试。

判断逻辑：

* 维修时间早于或等于任务失败时间，且故障现象涉及 NPU / 网络 / HCCS / 光模块：坏节点可能性高。
* 维修时间晚于任务失败时间但间隔很近：节点可能在任务运行期间已经异常。
* 同节点同端口 / 同 NPU 多次维修：倾向持续性硬件缺陷。
* 维修记录无关，且日志有明确代码级报错：不优先判断为坏节点。

输出结构：

```text
结论：
- 是否由坏节点导致：是 / 否 / 不确定
- 涉及节点 / 端口 / NPU：

证据：
- Pod 状态与所在节点：
- Pod 日志关键错误：
- 维修记录：
- 失败时间与维修时间对照：

判断：
- 确定事实：
- 推断：
- 待验证点：

下一步：
- 是否建议重跑：
- 是否建议排除节点：
- 是否需要继续查维修记录 / 节点健康：
```

---

## 禁止事项

* 不要用 `kubectl get jobs -A` 判断 Volcano 任务。
* 不要把 VC 显示名直接拼成 `/root/D/<VC>`。
* 不要在找不到 vcluster kubeconfig 时回退到 host cluster kubeconfig 查 `vcjob` / `PodGroup`。
* 不要只凭 vcluster 总资源量判断是否够。
* 不要在缺少关键标识时扩大到全 namespace、全 vcluster 或全量扫描。
