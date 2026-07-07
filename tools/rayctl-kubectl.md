# rayctl-kubectl.md - rayctl / kubectl 命令模板

本文件只保存 rayctl / kubectl / jq 的具体命令模板。
流程判断以 `skills/*/SKILL.md` 为准。
入口边界、写操作确认、安全红线以 `MEMORY.md` 为准。

---

## 0. 使用边界

* Kubernetes / vcluster / AI 训练平台资源优先使用 rayctl。
* kubectl 默认只做只读兜底。
* kubectl 兜底必须满足：

  * 当前 SOP 明确允许。
  * 已知 vcluster、namespace、resource name、UID、pod 或 node 等关键标识。
  * 已确认 kubeconfig 类型正确。
* vcluster 内 `vcjob` / `PodGroup` / Pod / Event / 日志必须使用对应 vcluster kubeconfig。
* 禁止用 host cluster kubeconfig 查询 vcluster 内 `vcjob` / `PodGroup` / Pod。
* 写操作必须遵守 `MEMORY.md` 的写操作确认规则。
* 用户输入必须放入变量并加引号；grep 使用 `grep -F -- "$VAR"`。
* bearer token 属于敏感信息，不得写入知识库、命令模板、日志、聊天记录或 shell history；只允许通过临时环境变量在当前会话中使用，用完立即 unset。

---

## 1. kubeconfig

D host cluster：

```bash
export KUBECONFIG=/root/kubeconfig
```

Dcluster host cluster：

```bash
export KUBECONFIG=/root/kubeconfigc
```

vcluster kubeconfig 目录：

```bash
/root/D/
```

定位实际 vcluster kubeconfig 文件名：

```bash
ls -1 /root/D
ls -1 /root/D | grep -E 'c550|jiaofu|h3c|a3|a2|llmit|ai4chem'
```

常见示例：

```bash
export KUBECONFIG=/root/D/a3-llmit
export KUBECONFIG=/root/D/c550-h3c
export KUBECONFIG=/root/D/c550-jiaofu
export KUBECONFIG=/root/D/ai4chem
```

红线：

* rayctl 输出的 `VC` 显示名不保证等于 `/root/D/` 文件名。
* 如果找不到对应 vcluster kubeconfig，停止并反馈。
* 不要改用 host cluster kubeconfig 查询 vcluster 内资源。

---

## 2. rayctl 全局用法

当前能力总览：

```text
rayctl cluster set <d|dcloud>
rayctl cluster get <vc-name-or-uid>
rayctl ecs check <ais-name-or-ecs-name-or-uid> [...]
rayctl ecs login <ais-name-or-ecs-name-or-uid>
rayctl node get [profile-or-selector]
rayctl node check <node-name-or-ip> [...]
rayctl node describe <node-name-or-ip> [...]
rayctl node cordon <node-name>
rayctl node uncordon <node-name>
rayctl job get <job-name-or-pod-name-or-uid> [...]
rayctl job get cluster [-A|<vc-name>] [pending]
rayctl job create <template>
rayctl vc get [vc-name-or-uid]
rayctl policy get disallow-privileged-containers [vc-name-or-uid]
rayctl policy update disallow-privileged-containers <vc-name-or-uid>
rayctl user get <username-or-userid> [--jobs]
rayctl auth check user <username-or-userid>
rayctl auth check vc <vc-name-or-uid>
rayctl auth check subnet <subnet-name>
rayctl auth check afs <afs-name>
rayctl auth check ccr <namespace-name>
rayctl auth check ais <ais-name>
rayctl auth check groups <group-name-or-id>
rayctl auth roles <afs|ais|ccr|subnet|vc>
rayctl auth grant vc <vc-name> [--user <username-or-userid>|--group <group-name-or-id>] --role <user|admin|role_name> [--dry-run]
rayctl auth grant subnet <subnet-name> [--user <username-or-userid>|--group <group-name-or-id>] --role <reader|editor|role_name> [--dry-run]
rayctl auth grant afs <afs-name> [--user <username-or-userid>|--group <group-name-or-id>] --role <editor|reader|owner|role_name> [--dry-run]
rayctl auth grant ccr <namespace-name> [--user <username-or-userid>|--group <group-name-or-id>] --role <user|imageUser|owner|role_name> [--dry-run]
rayctl auth grant ais <ais-name> [--user <username-or-userid>|--group <group-name-or-id>] --role <owner|role_name> [--dry-run]
rayctl auth remove vc <vc-name> [--user <username-or-userid>|--group <group-name-or-id>] --role <user|admin|role_name> [--dry-run]
rayctl auth remove subnet <subnet-name> [--user <username-or-userid>|--group <group-name-or-id>] --role <reader|editor|role_name> [--dry-run]
rayctl auth remove afs <afs-name> [--user <username-or-userid>|--group <group-name-or-id>] --role <editor|reader|owner|role_name> [--dry-run]
rayctl auth remove ccr <namespace-name> [--user <username-or-userid>|--group <group-name-or-id>] --role <user|imageUser|owner|role_name> [--dry-run]
rayctl auth remove ais <ais-name> [--user <username-or-userid>|--group <group-name-or-id>] --role <owner|role_name> [--dry-run]
rayctl rbac get <vc-name-or-uid> [-l <label-selector>]
rayctl afs check <afs-name-or-uid> [...]
rayctl pvc check <pvc-name> [...]
rayctl pvc create
```

全局参数：

```bash
rayctl --kubeconfig /path/to/kubeconfig <command>
rayctl -k /path/to/kubeconfig <command>
```

推荐初始化：

```bash
export KUBECONFIG=/root/kubeconfig
rayctl cluster set d
```

host cluster 视角查询：

```bash
export KUBECONFIG=/root/kubeconfig
rayctl node get -A 10.140.215
```

vcluster 视角查询：

```bash
export KUBECONFIG=/root/D/<实际 kubeconfig 文件名>
rayctl -k /root/D/<实际 kubeconfig 文件名> job get <job-name-or-pod-name-or-uid>
```

policy 只读查询与更新：

```bash
export KUBECONFIG=/root/kubeconfig

VC_QUERY='<vc-name-or-uid>'

rayctl policy get disallow-privileged-containers "$VC_QUERY"
rayctl policy update disallow-privileged-containers "$VC_QUERY"
```

注意：

* `rayctl policy get` 是只读查询。
* `rayctl policy update` 是写操作，执行前必须确认。

---

## 3. 查询任务

先按问题粒度选择入口：

* 单个任务、Pod、UID 的状态或 Pending 原因：使用单任务查询。
* 某个 VC / 分区整体任务情况、Pending 分布、前端任务数异常、历史任务堆积、哪个 VC 大量提交任务：使用分区级任务查询。
* Failed 且疑似 HCCL / ACL / timeout / OOM / 信号终止 / 坏节点：使用 Failed 任务节点定位模板。

---

### 3.1 单任务查询

标准流程：

1. 先通过 host cluster kubeconfig 用 rayctl 定位任务所属 VC：

```bash
export KUBECONFIG=/root/kubeconfig

JOB='<job-name-or-pod-name-or-uid>'
rayctl job get "$JOB"
```

输出关注：

* VC。
* namespace。
* job / pod / PodGroup 名称。
* 当前状态。
* Pending / Failed / Aborting 原因。
* 是否提示需要 vcluster kubeconfig 重查。

如果 rayctl 输出缺少 VC / namespace / pod / PodGroup 等关键字段，停止并返回缺失信息，不要自动扩大到全 vcluster 扫描。

2. 找到对应 vcluster kubeconfig：

```bash
VC_HINT='<vc关键字>'

ls -1 /root/D
ls -1 /root/D | grep -E "$VC_HINT"
```

注意：rayctl 输出的 `VC` 显示名不保证等于 `/root/D/` 文件名，需要核实。

3. 切到 vcluster kubeconfig 查询详情：

```bash
export KUBECONFIG=/root/D/<实际 kubeconfig 文件名>

JOB='<job-name>'
NS='<namespace>'

kubectl get vcjob "$JOB" -n "$NS" -o wide
kubectl get vcjob "$JOB" -n "$NS" -o yaml
kubectl describe vcjob "$JOB" -n "$NS"
```

如果 rayctl 在 host cluster 阶段要求 vcluster kubeconfig，也可以直接走 vcluster 查：

```bash
rayctl -k /root/D/<实际 kubeconfig 文件名> job get "$JOB"
```

不要再使用旧命令：

```bash
rayctl job check <job-name>
rayctl job get job <job-name>
rayctl job get pg <podgroup-name-or-uid>
```

---

### 3.2 分区级任务查询

用于查看所有 VC / 分区的任务概览：

```bash
export KUBECONFIG=/root/kubeconfig
rayctl job get cluster -A
```

只看所有 VC / 分区的 Pending 任务：

```bash
export KUBECONFIG=/root/kubeconfig
rayctl job get cluster -A pending
```

查看指定 VC / 分区的任务概览：

```bash
export KUBECONFIG=/root/kubeconfig

VC='<vc-name>'
rayctl job get cluster "$VC"
```

只看指定 VC / 分区的 Pending 任务：

```bash
export KUBECONFIG=/root/kubeconfig

VC='<vc-name>'
rayctl job get cluster "$VC" pending
```

注意：

* 命令关键字是 `cluster`，不要写成 `culster`。
* 分区级概览只用于定位范围和异常集中点。
* 需要解释单个任务 Pending / Failed 原因时，再回到单任务查询。
* 如果用户没有授权全局概览，不要主动执行所有 VC / 分区扫描。

---

### 3.3 vcluster 内资源下钻模板

前提：

* 已确认实际 vcluster kubeconfig。
* 已知 namespace。
* 已知 job / pod / PodGroup 至少一个关键标识。

```bash
export KUBECONFIG=/root/D/<实际 kubeconfig 文件名>

JOB='<job-name>'
NS='<namespace>'
POD='<pod-name>'
PG='<podgroup-name>'
```

查 vcjob：

```bash
kubectl get vcjob "$JOB" -n "$NS" -o wide
kubectl get vcjob "$JOB" -n "$NS" -o yaml
kubectl describe vcjob "$JOB" -n "$NS"
```

查 Pod：

```bash
kubectl get pod -n "$NS" | grep -F -- "$JOB"
kubectl get pod "$POD" -n "$NS" -o wide
kubectl describe pod "$POD" -n "$NS"
kubectl get pod "$POD" -n "$NS" -o yaml
```

查 PodGroup：

```bash
kubectl get podgroup "$PG" -n "$NS" -o wide
kubectl get podgroup "$PG" -n "$NS" -o yaml
kubectl describe podgroup "$PG" -n "$NS"
```

查事件：

```bash
kubectl get events -n "$NS" --sort-by=.lastTimestamp | tail -100
kubectl get events -n "$NS" --sort-by=.lastTimestamp | grep -F -- "$JOB"
kubectl get events -n "$NS" --sort-by=.lastTimestamp | grep -F -- "$POD"
kubectl get events -n "$NS" --sort-by=.lastTimestamp | grep -F -- "$PG"
```

查日志：

```bash
kubectl logs "$POD" -n "$NS" --tail=100
kubectl logs "$POD" -n "$NS" --previous --tail=100
```

查 Pod 创建时间：

```bash
kubectl get pod "$POD" -n "$NS" -o jsonpath='{.metadata.creationTimestamp}{"\n"}'
```

查 Pod 退出状态：

```bash
kubectl get pod "$POD" -n "$NS" -o json | jq -r '
  .status.containerStatuses[]
  | {
      name: .name,
      restartCount: .restartCount,
      state: .state,
      lastState: .lastState
    }
'
```

如果 namespace 未知，但已确认 vcluster kubeconfig 且已知 job / pod / PodGroup 关键标识，可以在该 vcluster 内使用 `-A` 兜底定位 namespace：

```bash
export KUBECONFIG=/root/D/<实际 kubeconfig 文件名>

JOB='<job-name>'

kubectl get vcjob -A | grep -F -- "$JOB"
kubectl get pod -A | grep -F -- "$JOB"
kubectl get podgroup -A | grep -F -- "$JOB"
```

边界：

* 只允许在已确认的 vcluster kubeconfig 内使用 `-A` 兜底定位 namespace。
* 禁止在 host cluster kubeconfig 或未知 vcluster 上使用 `-A` 扫描 vcluster 内资源。
* 找不到目标资源且 SOP 没有下一步 fallback 时，停止并返回缺失信息。

---

### 3.3.1 Volcano 无法调度查询模板

前提：

* 已确认实际 vcluster kubeconfig。
* 已知 namespace。
* 已知 job / pod / PodGroup 至少一个关键标识。
* 本节只读；任何 patch / delete / rollout restart 都必须回到写操作确认流程。

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

只读查看 Volcano 组件。

组件 namespace 需先确认，不确定时不要全量扫描；常见为 vcluster 内 `kube-system`：

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

如果组件不是 Deployment 或名称不匹配，先用已确认 namespace 内的 `kubectl get pods` 结果定位具体 Pod 名称，再查日志；不要扩大到未知 namespace。

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

输出模板：

```text
结论：
- 当前最可能卡点：

证据：
- schedulerName / annotations：
- PodGroup phase / conditions：
- Events：
- Volcano component config/log：

判断：
- 确定事实：
- 推断：
- 待验证点：

下一步：
- 只读继续项：
- 写操作建议（如有，需确认）：
```

---

### 3.4 kubectl 兜底查询

rayctl 不够明确时，必须先进入已确认的 vcluster kubeconfig：

```bash
export KUBECONFIG=/root/D/<实际 kubeconfig 文件名>

JOB='<job-name>'
NS='<namespace>'

kubectl get vcjob "$JOB" -n "$NS" -o wide
kubectl get pod -n "$NS" | grep -F -- "$JOB"
kubectl get podgroup -n "$NS" | grep -F -- "$JOB"
```

如果只知道 Pod：

```bash
export KUBECONFIG=/root/D/<实际 kubeconfig 文件名>

POD='<pod-name>'
NS='<namespace>'

kubectl get pod "$POD" -n "$NS" -o wide
kubectl describe pod "$POD" -n "$NS"
kubectl get pod "$POD" -n "$NS" -o yaml
```

如果只知道 PodGroup：

```bash
export KUBECONFIG=/root/D/<实际 kubeconfig 文件名>

PG='<podgroup-name>'
NS='<namespace>'

kubectl get podgroup "$PG" -n "$NS" -o wide
kubectl describe podgroup "$PG" -n "$NS"
kubectl get podgroup "$PG" -n "$NS" -o yaml
```

---

### 3.5 NotEnoughResources 资源判断模板

前提：

* 已确认 vcluster kubeconfig。
* 已知 namespace 和目标 Pod。
* 不得只看 vcluster 总资源量。
* 必须判断是否存在同一台节点同时满足 requests 和调度约束。

```bash
export KUBECONFIG=/root/D/<实际 kubeconfig 文件名>

NS='<namespace>'
POD='<pod-name>'
NODE='<node-name>'
```

查看目标 Pod requests 和调度约束：

```bash
kubectl get pod "$POD" -n "$NS" -o json | jq -r '
  "nodeName=" + (.spec.nodeName // ""),
  "schedulerName=" + (.spec.schedulerName // ""),
  "priorityClass=" + (.spec.priorityClassName // ""),
  "nodeSelector=" + ((.spec.nodeSelector // {}) | tojson),
  "tolerations=" + ((.spec.tolerations // []) | tojson),
  "affinity=" + ((.spec.affinity // {}) | tojson),
  (.spec.containers[] | "container=" + .name + " requests=" + ((.resources.requests // {}) | tojson) + " limits=" + ((.resources.limits // {}) | tojson))
'
```

查看节点列表：

```bash
kubectl get nodes -o wide
```

查看单个节点详情：

```bash
kubectl describe node "$NODE"
```

查看单个节点 labels / taints / allocatable / unschedulable：

```bash
kubectl get node "$NODE" -o json | jq -r '
  "unschedulable=" + ((.spec.unschedulable // false) | tostring),
  "labels=" + (.metadata.labels | tojson),
  "taints=" + ((.spec.taints // []) | tojson),
  "allocatable=" + (.status.allocatable | tojson)
'
```

查看节点上已调度 Pod：

```bash
kubectl get pods -A --field-selector spec.nodeName="$NODE" -o wide
```

查看节点上未完成 Pod 的 requests：

```bash
kubectl get pods -A --field-selector spec.nodeName="$NODE" -o json | jq -r '
  .items[]
  | select(.status.phase != "Succeeded" and .status.phase != "Failed")
  | [
      .metadata.namespace,
      .metadata.name,
      .status.phase,
      ((.spec.containers[].resources.requests // {}) | tojson)
    ]
  | @tsv
'
```

查看 PodGroup：

```bash
PG='<podgroup-name>'

kubectl get podgroup "$PG" -n "$NS" -o yaml
kubectl describe podgroup "$PG" -n "$NS"
```

查看调度事件：

```bash
kubectl get events -n "$NS" --sort-by=.lastTimestamp | tail -100
kubectl get events -n "$NS" --sort-by=.lastTimestamp | grep -F -- "$POD"
kubectl get events -n "$NS" --sort-by=.lastTimestamp | grep -F -- "$PG"
```

判断口径：

```text
单节点剩余资源 = node.status.allocatable - 已调度且未完成 Pod 的 requests
```

结论分类：

* `fit > 0`：理论上有节点可容纳，继续查 taint/toleration、affinity、queue、quota、scheduler event。
* `fit = 0` 且同一类资源不足：资源确实不足。
* `fit = 0` 但资源分散在不同节点：资源碎片化。
* 资源足够但仍 Pending：查 PodGroup、quota、scheduler、node unschedulable、webhook、Kube-OVN。

---

### 3.6 Failed 任务节点定位模板

前提：

* 已确认 vcluster kubeconfig。
* 已知 namespace 和 job name。

```bash
export KUBECONFIG=/root/D/<实际 kubeconfig 文件名>

NS='<namespace>'
JOB='<job-name>'
```

查看任务所有 Pod 和所在节点：

```bash
kubectl get pods -n "$NS" -l "volcano.sh/job-name=$JOB" -o wide
```

如果 label 不匹配，用 job name 兜底：

```bash
kubectl get pods -n "$NS" -o wide | grep -F -- "$JOB"
```

查看代表性 Pod 日志：

```bash
POD='<pod-name>'

kubectl logs "$POD" -n "$NS" --tail=200
kubectl logs "$POD" -n "$NS" --previous --tail=200
```

查看 Pod 退出状态：

```bash
kubectl get pod "$POD" -n "$NS" -o json | jq -r '
  .status.containerStatuses[]
  | {
      name: .name,
      restartCount: .restartCount,
      state: .state,
      lastState: .lastState
    }
'
```

查看 Pod 所在节点：

```bash
kubectl get pod "$POD" -n "$NS" -o wide
kubectl get pod "$POD" -n "$NS" -o jsonpath='{.spec.nodeName}{"\n"}'
```

从 node name 提取业务 IP：

```bash
NODE='host-10-140-213-80'
IP="${NODE#host-}"
IP="${IP//-/.}"
echo "$IP"
```

批量提取任务 Pod 的节点和 IP：

```bash
kubectl get pods -n "$NS" -l "volcano.sh/job-name=$JOB" -o json | jq -r '
  .items[]
  | [
      .metadata.name,
      .status.phase,
      (.spec.nodeName // "")
    ]
  | @tsv
' | while IFS=$'\t' read -r pod phase node; do
  ip="${node#host-}"
  ip="${ip//-/.}"
  printf "%s\t%s\t%s\t%s\n" "$pod" "$phase" "$node" "$ip"
done
```

随后按 `tools/fault-records.md` 查询维修记录，并对照任务失败时间与维修时间。

---

## 4. rayctl 基础查询

### 4.1 节点查询

节点列表：

```bash
export KUBECONFIG=/root/kubeconfig

rayctl node get
rayctl node get ecp
rayctl node get ecs
rayctl node get -A
rayctl node get -A -l
```

按 IP / hostname / selector 查询：

```bash
export KUBECONFIG=/root/kubeconfig

NODE_QUERY='<node-ip-or-hostname-or-selector>'

rayctl node get -A "$NODE_QUERY"
rayctl node get -A -l "$NODE_QUERY"
```

示例：

```bash
rayctl node get -A 10.12.138.26
rayctl node get -A 10.12.138
rayctl node get -A '10.12.138|10.140.214'
rayctl node get 'node-role.compute.sensecore.cn/prod=ecs'
```

grep 兜底：

```bash
NODE='host-10-12-138-26'

rayctl node get -A | grep -F -- "$NODE"
```

节点详情：

```bash
NODE='<node-name-or-ip>'

rayctl node check "$NODE"
rayctl node describe "$NODE"
```

查某个节点上是否有任务 / Pod 在运行：

```bash
export KUBECONFIG=/root/kubeconfig

NODE='<node-ip-or-hostname>'

rayctl node check "$NODE"
```

输出关注：

* 所属 VC。
* GPU / CPU / MEM 占用。
* PODS 数量。
* 具体 Pod 列表。
* 是否空闲。
* 是否处于 cordon / unschedulable / 维修状态。

节点调度控制是写操作，执行前必须确认：

```bash
NODE='<node-name>'

rayctl node cordon "$NODE"
rayctl node uncordon "$NODE"
```

kubectl cordon / uncordon 只在 rayctl 不可用或用户明确要求时使用，且属于写操作：

```bash
NODE='<node-name>'

kubectl cordon "$NODE"
kubectl uncordon "$NODE"
```

---

### 4.2 平台 VC 信息查询

用于列出 VC 或查询单个 VC 详情。

查询单个 VC：

```bash
export KUBECONFIG=/root/kubeconfig

VC='<vc-name-or-uid>'

rayctl vc get "$VC"
```

列出 VC：

```bash
export KUBECONFIG=/root/kubeconfig

rayctl vc get
```

输出关注：

* VC 名称或显示名。
* VC UID。
* 与 HC / 控制面 / namespace 映射相关的信息。
* 是否能唯一定位目标 VC。

适用场景：

* 查询 VC 基础信息。
* 查询或核对 VC UID。
* policy 更新前确认目标 VC 是否唯一。

边界：

* 这是 host cluster / 平台侧只读查询。
* 无参 `rayctl vc get` 可能列出全部 VC，只有用户明确要求列表、概览或全量时才执行。
* 它不替代 vcluster kubeconfig 下的业务资源查询。
* 需要查 vcjob / Pod / PodGroup / PVC / Event 时，仍回到对应 vcluster kubeconfig 和原有模板。

---

### 4.3 vcluster 与 host namespace 映射查询

用于查看某个 vcluster 在 host cluster 中的控制面 namespace，以及逻辑 namespace 到 host resource namespace 的映射关系。

```bash
export KUBECONFIG=/root/kubeconfig

VC='<vc-name-or-uid>'

rayctl cluster get "$VC"
```

示例：

```bash
rayctl cluster get vc-a3-llmit
rayctl cluster get vc-019d28e0-9610-74ef-a722-9242dede9e37
```

输出关注：

* 分区名。
* VC UID。
* 控制面 namespace。
* 资源 namespace 数量。
* 每个 `RESOURCE NAMESPACE` 对应的 `VIRTUAL NAMESPACE`。

适用场景：

* 查询某个 vcluster 的控制面 namespace。
* 查询 vcluster 内逻辑 namespace 对应的 host resource namespace。
* 做 host cluster 侧 namespace 映射核对。

边界：

* 这是 host cluster 视角只读查询。
* 它只解决 namespace 映射关系，不替代 vcluster kubeconfig 下的资源查询。
* 需要查 vcjob / Pod / PodGroup / PVC / Event 时，仍回到对应 vcluster kubeconfig 和原有模板。

---

### 4.4 平台用户查询

用于根据 username 或 userid 查询平台用户信息。

```bash
export KUBECONFIG=/root/kubeconfig

USER_QUERY='<username-or-userid>'

rayctl user get "$USER_QUERY"
```

同时查询该用户在当前租户下提交的活跃任务：

```bash
export KUBECONFIG=/root/kubeconfig

USER_QUERY='<username-or-userid>'

rayctl user get "$USER_QUERY" --jobs
```

示例：

```bash
rayctl user get zhangjinouwen
rayctl user get 0198ef7e-988a-7196-8ce1-12d1bfce08e1
rayctl user get zhangjinouwen --jobs
```

输出关注：

* ID。
* USERNAME。
* NAME。
* TENANT CODE。
* STATUS。
* SOURCE。
* 如使用 `--jobs`，补充查看该用户在当前租户下的活跃任务。

边界：

* `--jobs` 只查询当前租户下的活跃任务，不等价于全平台历史任务检索。
* 需要继续排障具体任务时，再回到 3. 查询任务。
* 用户名、用户 ID 必须明确；缺少关键标识时停止，不要扩大范围扫描。

---

### 4.5 平台授权与 RBAC 查询

用于查询平台资源授权关系：

* `rayctl auth check vc`：查看 VC 归属的用户 / 用户组授权。
* `rayctl auth check subnet`：查看 Subnet 归属的用户 / 用户组授权。
* `rayctl auth check afs`：查看 AFS 归属的用户 / 用户组授权。
* `rayctl auth check ccr`：查看 CCR namespace 归属的用户 / 用户组授权。
* `rayctl auth check ais`：查看 AIS / AI Space 归属的用户 / 用户组授权。
* `rayctl auth check user`：查看某个用户所属组和权限。
* `rayctl auth check groups`：查看某个用户组信息和权限。
* `rayctl auth roles`：查看指定资源类型可授权角色。
* `rayctl rbac get`：查看某个 VC 集群维度的 ClusterRoleBinding。

查询用户权限：

```bash
export KUBECONFIG=/root/kubeconfig

USER_QUERY='<username-or-userid>'

rayctl auth check user "$USER_QUERY"
```

查询 VC 归属授权：

```bash
export KUBECONFIG=/root/kubeconfig

VC='<vc-name-or-uid>'

rayctl auth check vc "$VC"
```

查询 Subnet 归属授权：

```bash
export KUBECONFIG=/root/kubeconfig

SUBNET='<subnet-name>'

rayctl auth check subnet "$SUBNET"
```

查询 AFS 归属授权：

```bash
export KUBECONFIG=/root/kubeconfig

AFS='<afs-name>'

rayctl auth check afs "$AFS"
```

查询 CCR namespace 归属授权：

```bash
export KUBECONFIG=/root/kubeconfig

CCR='<namespace-name>'

rayctl auth check ccr "$CCR"
```

查询 AIS / AI Space 归属授权：

```bash
export KUBECONFIG=/root/kubeconfig

AIS='<ais-name>'

rayctl auth check ais "$AIS"
```

查询用户组权限：

```bash
export KUBECONFIG=/root/kubeconfig

GROUP_QUERY='<group-name-or-id>'

rayctl auth check groups "$GROUP_QUERY"
```

`groups` 也支持别名 `group`：

```bash
rayctl auth check group "$GROUP_QUERY"
```

查询资源可授权角色：

```bash
export KUBECONFIG=/root/kubeconfig

rayctl auth roles vc
rayctl auth roles subnet
rayctl auth roles afs
rayctl auth roles ccr
rayctl auth roles ais
```

查询 VC 集群维度 RBAC：

```bash
export KUBECONFIG=/root/kubeconfig

VC='<vc-name-or-uid>'

# 在交互 shell 中输入 token，不要把 token 写进命令历史或文档。
printf 'Bearer token: '
stty -echo
IFS= read -r RAYCTL_BEARER_TOKEN
stty echo
printf '\n'
export RAYCTL_BEARER_TOKEN

rayctl rbac get "$VC"

unset RAYCTL_BEARER_TOKEN BEARER_TOKEN
```

指定 ClusterRoleBinding labelSelector：

```bash
export KUBECONFIG=/root/kubeconfig

VC='<vc-name-or-uid>'
LABEL_SELECTOR='<label-selector>'

printf 'Bearer token: '
stty -echo
IFS= read -r RAYCTL_BEARER_TOKEN
stty echo
printf '\n'
export RAYCTL_BEARER_TOKEN

rayctl rbac get "$VC" -l "$LABEL_SELECTOR"

unset RAYCTL_BEARER_TOKEN BEARER_TOKEN
```

`rayctl rbac get` 默认 selector：

```text
resource.compute.sensecore.cn/control
```

token 要求：

* `rayctl rbac get` 需要 console bearer token。
* 命令从 `RAYCTL_BEARER_TOKEN` 或 `BEARER_TOKEN` 读取 token。
* 不要把 token 写成命令行参数、不要写入 SOP、不要回显 token、不要保存到持久环境文件。
* 如果没有 token，停止并说明需要用户在交互 shell 中临时设置 `RAYCTL_BEARER_TOKEN` 或 `BEARER_TOKEN`。

边界：

* 以上命令都是只读查询。
* 查询对象必须明确：VC 名 / UID、Subnet 名、AFS 名、CCR namespace 名、AIS 名、用户名 / 用户 ID、用户组名 / ID。
* `rayctl auth check vc / subnet / afs / ccr / ais` 查询资源授权归属，不替代对应资源自身的详情、状态或映射查询。
* `rayctl auth roles <resource-type>` 只查询该资源类型可授权角色，不查询某个具体资源的已有授权。
* `rayctl rbac get` 查询的是 VC 集群维度 ClusterRoleBinding，不替代 vcluster kubeconfig 下的业务资源查询。
* 缺少关键标识或 token 时停止，不要扩大到全量扫描。

---

### 4.6 平台资源授权增删写操作模板

用于给平台资源新增或移除用户 / 用户组授权：

* `rayctl auth grant vc`：给 VC 授权。
* `rayctl auth grant subnet`：给 Subnet 授权。
* `rayctl auth grant afs`：给 AFS 授权。
* `rayctl auth grant ccr`：给 CCR namespace 授权。
* `rayctl auth grant ais`：给 AIS / AI Space 授权。
* `rayctl auth remove vc`：移除 VC 授权。
* `rayctl auth remove subnet`：移除 Subnet 授权。
* `rayctl auth remove afs`：移除 AFS 授权。
* `rayctl auth remove ccr`：移除 CCR namespace 授权。
* `rayctl auth remove ais`：移除 AIS / AI Space 授权。

前提：

* 使用 HC kubeconfig。
* 已明确资源类型、资源名称、授权对象和角色。
* 角色名不确定时，先用 `rayctl auth roles <afs|ais|ccr|subnet|vc>` 查询该资源类型可授权角色，再选择 `grant` / `remove` 的 `--role`。
* 用户和用户组二选一，不要同时传 `--user` 和 `--group`。
* Bearer token 只能通过临时环境变量读取，不要写入命令行参数、文档、聊天记录或持久环境文件。
* 新增授权和移除授权都是写操作，执行前必须完成 `MEMORY.md` 写操作确认流程。

角色速查：

```text
vc: user / admin / 完整 role_name
subnet: reader / editor / 完整 role_name
afs: editor / reader / owner / 完整 role_name
ccr: user / imageUser / owner / 完整 role_name
ais: owner / 完整 role_name
```

通用变量：

```bash
export KUBECONFIG=/root/kubeconfig

RESOURCE_NAME='<resource-name>'
ROLE='<role>'
USER_QUERY='<username-or-userid>'
GROUP_QUERY='<group-name-or-id>'
SCOPE='<optional-scope>'
```

登录或临时设置 token：

```bash
rayctl auth login
rayctl auth token
```

租户登录凭据：

```text
~/.openclaw/workspace/platform.json
```

使用规则：

* 优先执行 `rayctl auth token`，登录态有效时不要读取凭据文件。
* 只有登录态失效且需要 `rayctl auth login` 时，才读取对应集群和租户的账号密码。
* 不要打印、复制、转存或记录 `platform.json` 中的密码。
* 不要把密码放到命令行参数；如需交互输入，用不回显方式输入。

如需手动输入 token：

```bash
printf 'Bearer token: '
stty -echo
IFS= read -r RAYCTL_BEARER_TOKEN
stty echo
printf '\n'
export RAYCTL_BEARER_TOKEN
```

新增授权 dry-run 预检示例：

```bash
# 授权给用户。
rayctl auth grant vc "$RESOURCE_NAME" --user "$USER_QUERY" --role "$ROLE" --dry-run
rayctl auth grant subnet "$RESOURCE_NAME" --user "$USER_QUERY" --role "$ROLE" --dry-run
rayctl auth grant afs "$RESOURCE_NAME" --user "$USER_QUERY" --role "$ROLE" --dry-run
rayctl auth grant ccr "$RESOURCE_NAME" --user "$USER_QUERY" --role "$ROLE" --dry-run
rayctl auth grant ais "$RESOURCE_NAME" --user "$USER_QUERY" --role "$ROLE" --dry-run

# 授权给用户组。
rayctl auth grant vc "$RESOURCE_NAME" --group "$GROUP_QUERY" --role "$ROLE" --dry-run
rayctl auth grant subnet "$RESOURCE_NAME" --group "$GROUP_QUERY" --role "$ROLE" --dry-run
rayctl auth grant afs "$RESOURCE_NAME" --group "$GROUP_QUERY" --role "$ROLE" --dry-run
rayctl auth grant ccr "$RESOURCE_NAME" --group "$GROUP_QUERY" --role "$ROLE" --dry-run
rayctl auth grant ais "$RESOURCE_NAME" --group "$GROUP_QUERY" --role "$ROLE" --dry-run
```

如资源需要手动指定 scope，在 dry-run 和真实执行中保持一致：

```bash
rayctl auth grant vc "$RESOURCE_NAME" --user "$USER_QUERY" --role "$ROLE" --scope "$SCOPE" --dry-run
```

移除授权 dry-run 预检示例：

```bash
# 移除用户授权。
rayctl auth remove vc "$RESOURCE_NAME" --user "$USER_QUERY" --role "$ROLE" --dry-run
rayctl auth remove subnet "$RESOURCE_NAME" --user "$USER_QUERY" --role "$ROLE" --dry-run
rayctl auth remove afs "$RESOURCE_NAME" --user "$USER_QUERY" --role "$ROLE" --dry-run
rayctl auth remove ccr "$RESOURCE_NAME" --user "$USER_QUERY" --role "$ROLE" --dry-run
rayctl auth remove ais "$RESOURCE_NAME" --user "$USER_QUERY" --role "$ROLE" --dry-run

# 移除用户组授权。
rayctl auth remove vc "$RESOURCE_NAME" --group "$GROUP_QUERY" --role "$ROLE" --dry-run
rayctl auth remove subnet "$RESOURCE_NAME" --group "$GROUP_QUERY" --role "$ROLE" --dry-run
rayctl auth remove afs "$RESOURCE_NAME" --group "$GROUP_QUERY" --role "$ROLE" --dry-run
rayctl auth remove ccr "$RESOURCE_NAME" --group "$GROUP_QUERY" --role "$ROLE" --dry-run
rayctl auth remove ais "$RESOURCE_NAME" --group "$GROUP_QUERY" --role "$ROLE" --dry-run
```

确认后真实新增授权示例：

```bash
rayctl auth grant vc "$RESOURCE_NAME" --user "$USER_QUERY" --role "$ROLE" --yes
rayctl auth grant subnet "$RESOURCE_NAME" --user "$USER_QUERY" --role "$ROLE" --yes
rayctl auth grant afs "$RESOURCE_NAME" --user "$USER_QUERY" --role "$ROLE" --yes
rayctl auth grant ccr "$RESOURCE_NAME" --user "$USER_QUERY" --role "$ROLE" --yes
rayctl auth grant ais "$RESOURCE_NAME" --user "$USER_QUERY" --role "$ROLE" --yes
```

确认后真实移除授权示例：

```bash
rayctl auth remove vc "$RESOURCE_NAME" --user "$USER_QUERY" --role "$ROLE" --yes
rayctl auth remove subnet "$RESOURCE_NAME" --user "$USER_QUERY" --role "$ROLE" --yes
rayctl auth remove afs "$RESOURCE_NAME" --user "$USER_QUERY" --role "$ROLE" --yes
rayctl auth remove ccr "$RESOURCE_NAME" --user "$USER_QUERY" --role "$ROLE" --yes
rayctl auth remove ais "$RESOURCE_NAME" --user "$USER_QUERY" --role "$ROLE" --yes
```

执行后清理 token：

```bash
unset RAYCTL_BEARER_TOKEN BEARER_TOKEN
```

边界：

* 未确认前只允许执行 `rayctl auth grant ... --dry-run` 或 `rayctl auth remove ... --dry-run`。
* 确认后执行时，必须只执行用户确认的那一个操作类型、资源类型、资源名、授权对象和角色。
* 不要为了补齐资源名、用户、用户组或角色而全量扫描。
* `--yes` 只能用于用户已经按 `MEMORY.md` 明确确认后的真实新增或移除授权。
* `--bearer-token` 参数会把 token 暴露在命令行中，默认不要使用。
* `--debug-auth` 只打印脱敏来源信息；一般无需使用。

---

## 5. AFS / PVC / PV 查询

### 5.1 查询 AFS

```bash
export KUBECONFIG=/root/kubeconfig

AFS='<afs-name-or-uid>'

rayctl afs check "$AFS"
rayctl afs check -l "$AFS"
```

### 5.2 查询 PVC

```bash
export KUBECONFIG=/root/D/<实际 kubeconfig 文件名>

PVC='<pvc-name>'
NS='<namespace>'

rayctl pvc check "$PVC"
rayctl pvc check -l "$PVC"

kubectl get pvc "$PVC" -n "$NS" -o yaml
kubectl describe pvc "$PVC" -n "$NS"
```

如果 namespace 未知，但已确认 vcluster kubeconfig 且 PVC 名称明确，可以在该 vcluster 内定位：

```bash
export KUBECONFIG=/root/D/<实际 kubeconfig 文件名>

PVC='<pvc-name>'

kubectl get pvc -A | grep -F -- "$PVC"
```

### 5.3 查询 Host PV

```bash
export KUBECONFIG=/root/kubeconfig

PV='<host-pv-name>'

kubectl get pv "$PV"
kubectl get pv "$PV" -o yaml
kubectl describe pv "$PV"
```

---

## 6. 创建 PVC

创建 PVC 是写操作，执行前必须确认。

前置确认：

* vcluster kubeconfig。
* namespace。
* AFS 名称。
* AFS UID。
* secret name。
* PVC 名称。
* size。
* 影响范围。
* PVC Pending 后是否停止任务创建。

PVC 命名规则：

```text
pvc-<afs-name>
```

创建命令：

```bash
export KUBECONFIG=/root/D/<实际 kubeconfig 文件名>

PVC='pvc-<afs-name>'
AFS_UID='<afs-uuid>'
SECRET='<secret-name>'
NS='<namespace>'
SIZE='1000Mi'

rayctl pvc create \
  --name "$PVC" \
  --uid "$AFS_UID" \
  --secret "$SECRET" \
  --ns "$NS" \
  --size "$SIZE"
```

创建后检查：

```bash
rayctl pvc check "$PVC"
kubectl get pvc "$PVC" -n "$NS" -o yaml
kubectl describe pvc "$PVC" -n "$NS"
```

PVC Pending 时，停止后续任务创建，并返回 Pending 原因。

---

## 7. ECS / AIS

查询 ECS / AIS：

```bash
export KUBECONFIG=/root/kubeconfig

AIS='<ais-name-or-ecs-name-or-uid>'

rayctl ecs check "$AIS"
```

批量查询：

```bash
export KUBECONFIG=/root/kubeconfig

rayctl ecs check <id-1> <id-2>
```

登录 ECS / AIS 对应 VM：

```bash
export KUBECONFIG=/root/kubeconfig

AIS='<ais-name-or-ecs-name-or-uid>'

rayctl ecs login "$AIS"
```

注意：

* `rayctl ecs login` 是登录动作，不等同于物理机 ansible 操作。
* 如果用户要看物理机目录、日志、进程、磁盘、NPU、网卡、DNS、本地脚本，应按 `MEMORY.md` 的 D 集群物理机器入口处理。
