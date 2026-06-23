# vcjob-debug Skill

## 触发条件

用户询问以下任意问题时使用本 skill：

- vcjob / Volcano Job / PodGroup。
- 任务 Pending、Startup、Running 异常、Aborting、Failed。
- `NotEnoughResources`、`Insufficient cpu/memory/accelerator`。
- Pod 已分配 / 未分配、调度慢、事件异常。
- 任务所在 vcluster、namespace、Pod、PodGroup 定位。
- 某个 VC / 分区任务概览、Pending 分布、前端任务数异常、历史任务堆积、哪个 VC 大量提交任务。

## 相关工具文件

- `tools/rayctl-kubectl.md`
- `tools/job-templates.md`

## 前置原则

- 必须遵守 `MEMORY.md` 的入口路由。
- Kubernetes / vcluster 资源一律走 D 集群开发机。
- 先判断问题粒度：单任务问题优先用单任务查询；VC / 分区级问题优先用分区级任务概览。
- `vcjob` / `PodGroup` / vcluster 内 Pod / Event / 日志必须使用对应 vcluster kubeconfig。
- 禁止用 host cluster kubeconfig 查询或兜底 `vcjob` / `PodGroup`。
- 写操作必须先确认。

## 标准流程

### 0. 判断问题粒度

先判断用户问的是哪类问题：

- 单个任务 / Pod / UID 的状态、Pending 原因、日志、PodGroup：走单任务查询。
- 某个 VC / 分区整体任务情况、Pending 分布、前端任务数异常、历史任务堆积、哪个 VC 大量提交任务：走分区级任务概览。

### 1. 单任务快速定位

在开发机上：

```bash
export KUBECONFIG=/root/kubeconfig
rayctl job get <job-name-or-pod-name-or-uid>
```

输出重点：

- VC。
- namespace。
- job / pod / PodGroup 名称。
- 当前状态。
- 结论 / 下一步。
- 是否提示需要带 vcluster kubeconfig 重查。

### 1.1 分区级任务概览

当问题是 VC / 分区级时，先用概览定位范围。具体命令以 `tools/rayctl-kubectl.md` 的“分区级任务查询”为准。

常用目标：

- 所有 VC / 分区任务概览。
- 所有 VC / 分区 Pending 任务。
- 指定 VC / 分区任务概览。
- 指定 VC / 分区 Pending 任务。

分区级概览只用于定位异常集中点；需要解释某个任务为什么 Pending / Failed 时，再回到单任务查询和 vcluster 内资源查询。

### 2. 定位实际 vcluster kubeconfig

如果 rayctl 输出 VC 名称，先确认实际文件名：

```bash
ls -1 /root/D
ls -1 /root/D | grep -E '<vc关键字，如 c550|jiaofu|h3c|a3|llmit>'
```

如果找不到 kubeconfig：

- 停止。
- 说明“未定位到实际 vcluster kubeconfig 文件”。
- 不要改用 host cluster kubeconfig 查询 vcluster 内资源。

### 3. vcluster 内资源查询

```bash
export KUBECONFIG=/root/D/<实际 kubeconfig 文件名>
kubectl get vcjob -A | grep <job-name>
kubectl get pod -A | grep <job-name>
kubectl get podgroup -A | grep <job-name>
```

查看 Pod：

```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl get pod <pod-name> -n <namespace> -o yaml
```

查看 PodGroup：

```bash
kubectl describe podgroup <podgroup-name> -n <namespace>
kubectl get podgroup <podgroup-name> -n <namespace> -o yaml
```

查看事件：

```bash
kubectl get events -n <namespace> --sort-by=.lastTimestamp | tail -100
kubectl get events -A --sort-by=.lastTimestamp | grep -E '<job-name>|<pod-name>|<podgroup-name>'
```

### 4. NotEnoughResources 精确判断

不要只看总资源量。必须判断是否存在同一台节点同时满足所有 requests 和调度约束。

先看目标 Pod requests 和调度约束：

```bash
kubectl get pod <pod-name> -n <namespace> -o json | jq -r '
  "nodeName=" + (.spec.nodeName // ""),
  "schedulerName=" + (.spec.schedulerName // ""),
  "priorityClass=" + (.spec.priorityClassName // ""),
  "nodeSelector=" + ((.spec.nodeSelector // {})|tojson),
  "tolerations=" + ((.spec.tolerations // [])|tojson),
  "affinity=" + ((.spec.affinity // {})|tojson),
  (.spec.containers[] | "container=" + .name + " requests=" + ((.resources.requests // {})|tojson) + " limits=" + ((.resources.limits // {})|tojson))
'
```

再按节点检查：

```bash
kubectl get nodes -o wide
kubectl describe node <node-name>
kubectl get pods -A --field-selector spec.nodeName=<node-name> -o wide
```

判断口径：

```text
单节点剩余资源 = node.status.allocatable - 已调度且未完成 Pod 的 requests
```

结论分类：

- `fit > 0`：理论上有节点可容纳，继续查 taint/toleration、affinity、队列、配额、调度器事件。
- `fit = 0` 且所有候选节点同一类资源不足：资源确实不足。
- `fit = 0` 但不同资源分散在不同节点：资源碎片化。
- 资源足够但仍 Pending：查 PodGroup、队列、quota、scheduler、node unschedulable、webhook、Kube-OVN 等。

### 5. 输出格式

输出时按这个结构：

```text
结论：
证据：
- rayctl 输出说明 ...
- PodGroup 事件说明 ...
- Pod requests / node allocatable 说明 ...
判断：
- 确定事实 ...
- 推断 ...
下一步：
1. 只读检查 ...
2. 如需写操作，等待确认 ...
```

## 禁止事项

- 不要使用旧命令 `rayctl job check`、`rayctl job get job`、`rayctl job get pg`。
- 不要用 `kubectl get jobs -A` 判断 Volcano 任务。
- 不要把 VC 显示名直接拼成 `/root/D/<VC>`。
- 不要在找不到 vcluster kubeconfig 时回退到 host cluster kubeconfig 查 `vcjob` / `PodGroup`。
- 不要只凭 vcluster 总资源量判断是否够。
