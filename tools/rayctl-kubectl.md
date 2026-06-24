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

## 4. rayctl 节点查询

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
