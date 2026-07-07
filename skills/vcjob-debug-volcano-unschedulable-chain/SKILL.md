---
name: "vcjob-debug-volcano-unschedulable-chain"
description: "Volcano unschedulable chain for vcjob debug"
---

# vcjob-debug Volcano 无法调度查询链路提案

## 适用目标

把飞书 Wiki《volcano 问题排查指南》中的平台任务无法调度、Volcano 查询部分，整理进现有 OpenClaw 工作区任务排障链路：

- `skills/vcjob-debug/SKILL.md`：放流程、分支、停止条件和输出结构。
- `tools/rayctl-kubectl.md`：放具体只读命令模板。
- `TOOLS.md`：现有 `vcjob / Volcano Job / PodGroup / Pending / Failed 单任务排障` 路由已命中，不需要新增主路由。
- `MEMORY.md`：只保留全局长期边界，本次不建议写入。

## 建议修改 `skills/vcjob-debug/SKILL.md`

### 1. 触发条件补充

在触发条件中增加：

- Volcano scheduler 未生效，Pod 或 workload 模板的 `schedulerName` 不是 `volcano`。
- PodGroup Pending / Inqueue 但 Pod 未创建。
- PodGroup `Status.Conditions` 中出现 gang unschedulable、资源不足、NodeAffinity / nodeSelector 不匹配、capacity overused、log by search keywords 等调度失败线索。
- Volcano controller / admission / scheduler configmap 或日志可能导致任务无法调度。
- 昇腾 NPU 任务中 910C / 910B 的 annotation、nodeSelector、MinResources、volcano-npu plugin 架构不匹配等调度问题。

### 2. 单任务诊断增加分支

在 `### 2. 单任务诊断` 后增加：

```markdown
#### 2.x Volcano 无法调度专项分支

进入条件：

* `rayctl job get` 或用户描述显示任务无法调度、Pending、Pod 未创建、PodGroup 异常。
* 已确认实际 vcluster kubeconfig。
* 已知 namespace，且 job / pod / PodGroup 至少有一个关键标识。

查询顺序：

1. 确认任务是否实际走 Volcano：Pod 已存在时查 `spec.schedulerName`；Pod 未创建时查 vcjob / workload 模板；AMS / Deployment / LWS 场景检查触发 Volcano admission 的 annotation。
2. 查 PodGroup：关注 phase、minAvailable / minMember、queue、Status.Conditions。
3. 按 PodGroup / Event 文案分类：
   * `pod group is not ready`：确认 Pod 是否全部创建，继续查 controller 是否 `Failed to create pod`。
   * `Insufficient cpu/memory/accelerator`：切到 NotEnoughResources 资源判断模板。
   * `NodeAffinity predicates failed` / selector mismatch：核对 Pod nodeSelector / affinity 与候选节点 label。
   * `capacity overused` 且出现 `attachable-volumes-csi-csi-provisioner`：检查 PVC storageClass provisioner 是否需要被 Volcano ignored provisioners 忽略。
   * `node check failed` / `log by search keywords`：查 scheduler 关键日志；NPU 场景查 device info configmap。
4. PodGroup Pending 但配置看似正常时，只读检查 `volcano-scheduler-configmap` 的 actions / plugins；修改 configmap 或重启 scheduler 必须停止并等待用户确认。
5. PodGroup Inqueue 但 Pod 未创建时，查 `volcano-controllers` 日志、MPI plugin 与单机任务是否冲突、`priorityClassName` 是否存在。
6. 昇腾 NPU 任务额外检查：910C 的 `sp-block` annotation、910C / 910B 的 `accelerator-type` nodeSelector、PodGroup MinResources、volcano-npu plugin 架构后缀。

停止条件：

* 需要查 scheduler / controller / admission，但未确认组件 namespace 或组件名。
* 需要修改 configmap、重启 scheduler / controller / admission、删除异常 Pod / vcjob。
* 需要扩大到全 vcluster / 全 namespace / 全 VC 扫描。
* 缺少 job / pod / PodGroup / namespace / vcluster kubeconfig 关键标识。

输出结构：

结论：当前最可能卡点是 schedulerName / PodGroup / 资源不足 / selector-affinity / Volcano 配置 / controller 创建 Pod / NPU 专项 / 不确定。
证据：rayctl 初筛、Pod 或 vcjob 模板、PodGroup phase 与 conditions、Event / controller / scheduler 关键日志。
判断：区分确定事实、推断、待验证点。
下一步：列只读可继续项；如需写操作，说明影响范围并等待确认。
```

## 建议修改 `tools/rayctl-kubectl.md`

在 `3.3 vcluster 内资源下钻模板` 后新增：

```markdown
### 3.3.x Volcano 无法调度查询模板

前提：

* 已确认实际 vcluster kubeconfig。
* 已知 namespace。
* 已知 job / pod / PodGroup 至少一个关键标识。
* 本节只读；任何 patch / delete / rollout restart 都必须回到写操作确认流程。

变量：

```bash
export KUBECONFIG=/root/D/<实际 kubeconfig 文件名>

NS='<namespace>'
JOB='<job-name>'
POD='<pod-name>'
PG='<podgroup-name>'
```

确认 Pod 是否走 Volcano：

```bash
kubectl get pod "$POD" -n "$NS" -o json | jq -r '
  "schedulerName=" + (.spec.schedulerName // ""),
  "nodeName=" + (.spec.nodeName // ""),
  "priorityClassName=" + (.spec.priorityClassName // ""),
  "annotations=" + ((.metadata.annotations // {}) | tojson),
  "nodeSelector=" + ((.spec.nodeSelector // {}) | tojson),
  "affinity=" + ((.spec.affinity // {}) | tojson),
  "tolerations=" + ((.spec.tolerations // []) | tojson),
  (.spec.containers[] | "container=" + .name + " requests=" + ((.resources.requests // {}) | tojson) + " limits=" + ((.resources.limits // {}) | tojson))
'
```

Pod 未创建时查 vcjob：

```bash
kubectl get vcjob "$JOB" -n "$NS" -o yaml
kubectl describe vcjob "$JOB" -n "$NS"
```

查 PodGroup：

```bash
kubectl get podgroup "$PG" -n "$NS" -o wide
kubectl describe podgroup "$PG" -n "$NS"
kubectl get podgroup "$PG" -n "$NS" -o json | jq -r '
  "phase=" + (.status.phase // ""),
  "minAvailable=" + ((.spec.minMember // .spec.minAvailable // "") | tostring),
  "queue=" + (.spec.queue // ""),
  ((.status.conditions // [])[]? | "condition type=" + (.type // "") + " status=" + (.status // "") + " reason=" + (.reason // "") + " message=" + (.message // ""))
'
```

查事件：

```bash
kubectl get events -n "$NS" --sort-by=.lastTimestamp | tail -100
kubectl get events -n "$NS" --sort-by=.lastTimestamp | grep -F -- "$JOB"
kubectl get events -n "$NS" --sort-by=.lastTimestamp | grep -F -- "$POD"
kubectl get events -n "$NS" --sort-by=.lastTimestamp | grep -F -- "$PG"
```

只读查看 Volcano 组件。组件 namespace 需先确认，不确定时不要全量扫描；常见为 vcluster 内 `kube-system`：

```bash
VC_SYSTEM_NS='kube-system'

kubectl get cm -n "$VC_SYSTEM_NS" | grep -F -- 'volcano'
kubectl get pods -n "$VC_SYSTEM_NS" | grep -E 'volcano-scheduler|volcano-controllers|volcano-admission'
kubectl get cm volcano-scheduler-configmap -n "$VC_SYSTEM_NS" -o yaml
kubectl get cm volcano-admission-configmap -n "$VC_SYSTEM_NS" -o yaml
```

查日志关键字：

```bash
kubectl logs -n "$VC_SYSTEM_NS" deploy/volcano-controllers --tail=200 | grep -F -- 'Failed to create pod'
kubectl logs -n "$VC_SYSTEM_NS" deploy/volcano-scheduler --tail=300 | grep -E "$JOB|$POD|$PG|log by search keywords|overused|nil NPU|NodeAffinity|Insufficient"
```

NPU 专项：

```bash
kubectl get cm -n kube-system | grep -F -- 'mindx-dl-deviceinfo-'
```

分类规则：

* `pod group is not ready`：Pod 未全部创建，继续查 vcjob / controller 创建 Pod 失败原因。
* `Insufficient cpu` / `Insufficient memory` / accelerator 不足：切到 NotEnoughResources 资源判断模板。
* `NodeAffinity predicates failed` / selector mismatch：核对 Pod nodeSelector / affinity 与候选节点 label。
* `capacity overused` 且出现 `attachable-volumes-csi-csi-provisioner`：检查 PVC storageClass provisioner 与 Volcano ignored provisioners。
* `node check failed` / `log by search keywords`：继续查 scheduler 日志关键字；NPU 场景检查 device info configmap。
```

## 建议不做的事

- 不把 Wiki 中“修正 configmap / 重启 scheduler / 删除异常 pod 或 vcjob”等动作做成自动步骤；这些都是写操作或影响业务动作，必须在只读证据充分后等待用户确认。
- 不把该文档全文搬进 `MEMORY.md`。
- 不新增全局扫描入口。
- 不绕过 `rayctl` 初筛和 vcluster kubeconfig 校验。

## 验证方式

用一个真实 Pending / PodGroup 异常任务验证：

1. `rayctl job get` 能否定位 VC、namespace、job / PodGroup。
2. 按新增 Volcano 分支能否产出 PodGroup conditions、schedulerName、events。
3. 如果命中资源不足，是否能正确切到 NotEnoughResources 模板。
4. 如果命中 configmap / scheduler / controller 问题，是否只输出建议并停在写操作确认前。
