# vcjob-debug Skill

## 触发条件

用户询问以下任意问题时使用本 skill：

- vcjob / Volcano Job / PodGroup。
- 任务 Pending、Startup、Running 异常、Aborting、Failed。
- `NotEnoughResources`、`Insufficient cpu/memory/accelerator`。
- Pod 已分配 / 未分配、调度慢、事件异常。
- 任务所在 vcluster、namespace、Pod、PodGroup 定位。
- 某个 VC / 分区任务概览、Pending 分布、前端任务数异常、历史任务堆积、哪个 VC 大量提交任务。
- 排查任务是否是坏节点导致的失败。

## 相关工具文件

- `tools/rayctl-kubectl.md`
- `tools/job-templates.md`
- `tools/fault-records.md`（维修记录查询）

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

### 5. Failed 任务坏节点排查（新增）

当 Failed 任务的 Pod 日志中**没有明确的代码级报错**（如 Python traceback、loss NAN、数据加载失败），或者日志中出现**以下硬件/通信相关错误**时，应优先排查是否由坏节点导致，而不是先进入代码调试。

#### 5.0 触发线索

Pod 日志中出现以下关键词之一，提示可能为硬件/节点问题：

| 关键词 | 可能指向 |
|---|---|
| `hccl` / `fftsplus` / `task timeout` / `timeout: [0-9]+s` | HCCL 集合通信超时（NPU 通信链路问题、光模块/光缆故障） |
| `HCCL` / `hccl comm` / `hcom` / `hccl_*` | NCCL 类通信故障（类似 GPU 场景的 NCCL timeout） |
| `SIGABRT` / `signal` / `ExitCode 134` / `killed` | 被系统 OOM 或健康检查杀死 |
| `OOM` / `out of memory` / `OutOfMemory` | 节点内存耗尽 |
| `link down` / `Link_DOWN` / `Link Down` | 网络链路故障 |
| `NPU` / `掉卡` / `缺卡` / `GPU`（伴随异常） | NPU/GPU 硬件故障 |
| `health` / `健康` / `hwlog` / `dmsg` | 硬件健康告警 |
| `ACL` / `AclrtSynchronizeStreamWithTimeout` | 昇腾 ACL 超时，通常是通信或 NPU 异常 |
| `error code: 107020` | 昇腾 ACL 超时的常见错误码 |

即使 Pod 日志中有 Python traceback，但如果 traceback 是因为通信超时或进程被 SIGTERM 终止，也应优先排查节点。

#### 5.1 定位任务涉及的节点

先通过 vcluster kubeconfig 查出所有 Pod 及其所在节点：

```bash
export KUBECONFIG=/root/D/<实际 kubeconfig>
kubectl get pods -n <namespace> -l 'volcano.sh/job-name=<job-name>' -o wide
```

重点关注：

- 是否所有 Pod 都死了（全部 Error），还是只有个别异常。
- 所有 Pod 同时间段失败，说明可能共用某个坏节点。
- 异常 Pod 所在的 node name。

#### 5.2 查看 Pod 日志确定失败时间

看 master/worker 0 等有代表性的 Pod 日志尾部，确定失败时间：

```bash
kubectl logs <pod-name> -n <namespace> --tail=100
```

重点找：

- 最后一个 `2026-xx-xx xx:xx:xx` 的时间戳。
- timeout 持续时长（如 `timeout: 1836s` 表示任务卡了 31 分钟才崩）。
- 进程是如何结束的（SIGTERM、SIGKILL、ExitCode）。

#### 5.3 将节点 IP 映射到维修表中的设备编号

从 `kubectl get pods -o wide` 获取的 node name 通常是 `host-<IP>` 格式，从中提取业务 IP。

维修表记录规则（以 A924 910B 机型为例）：

- node name `host-10-140-213-80` → 业务 IP `10.140.213.80`。
- 同一 IP 段（如 `10.140.213.*` / `10.140.214.*`）的设备在维修表中通常对应同一型号（如 `A924`）。
- 具体设备编号如 `A924-058` 需通过维修表查询确认。

#### 5.4 查询维修表

维修表工具链接见 `tools/fault-records.md`。

查询步骤：

1. 先获取整表数据，定位相关 IP 或设备号。
2. 按任务失败时间段筛选维修记录。
3. 对异常节点查询其最近的工单记录。
4. 获取工单的故障时间、故障现象、处理结论和时间线。

使用飞书表格 CLI 查询：

```bash
lark-cli sheets +csv-get --as user --url "<sheets-url>" --sheet-name "故障记录表（南洋维护）" --range "A1:Z<max-row>" | grep -E "<节点IP>|<设备编号>" | grep -E "<失败日期范围>"
```

实际 URL 以 `tools/fault-records.md` 为准。

#### 5.5 对照失败时间与维修时间

判断逻辑：

- 如果节点维修时间**早于或等于**任务失败时间，且故障现象涉及 NPU/网络/HCCS/光模块：→ 该节点很可能是坏节点，直接导致任务失败。
- 如果节点维修时间**晚于**任务失败时间但间隔很近（几小时内）：→ 节点在任务运行期间已经出现闪断/异常，但维修动作稍后才开始，仍然是根因。
- 如果同节点有**多条连续故障记录**（同一端口/同一 NPU 反复修）：→ 该节点存在持续性硬件缺陷，不是偶然问题。

#### 5.6 输出格式

按以下结构输出：

```text
结论：
- 任务是否由坏节点导致（是/否/不确定）
- 如果是，哪个节点、哪个端口/NPU

证据：
- Pod 日志关键错误行 ...
- 所有 Pod 状态和所在节点 ...
- 维修表中该节点的故障记录 ...
- 失败时间 vs 维修时间对照 ...

判断：
- 确定事实 ...
- 推断 ...

下一步（按实际情况选填）：
- 告知用户坏节点工单号，供运营跟进
- 任务是否需要重跑
- 是否需要排除坏节点后重试
```

### 6. 输出格式

Failed 任务如果已经执行了坏节点排查，按 5.6 的格式输出。
其他情况按以下通用格式：

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
