# rayctl-kubectl.md - rayctl / kubectl 命令模板

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

红线：rayctl 输出的 `VC` 显示名不保证等于 `/root/D/` 文件名。如果找不到对应 vcluster kubeconfig，停止并反馈，不要改用 host cluster kubeconfig 查询 vcluster 内资源。

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

推荐：

```bash
export KUBECONFIG=/root/kubeconfig
rayctl cluster set d
rayctl node get -A 10.140.215

export KUBECONFIG=/root/D/<实际 kubeconfig 文件名>
rayctl -k /root/D/<实际 kubeconfig 文件名> job get <job-name-or-pod-name-or-uid>
```

---

## 3. 查询任务

先按问题粒度选择入口：

- 单个任务、Pod、UID 的状态或 Pending 原因：使用单任务查询。
- 某个 VC / 分区整体任务情况、Pending 分布、前端任务数异常、历史任务堆积、哪个 VC 大量提交任务：使用分区级任务查询。

### 3.1 单任务查询

标准流程：

1. 先通过 host cluster kubeconfig 用 rayctl 定位任务所属 VC：

```bash
export KUBECONFIG=/root/kubeconfig
rayctl job get <job-name-or-pod-name-or-uid>
```

输出包含 VC 列，确定任务在哪个 vcluster。

2. 找到对应 vcluster 的 kubeconfig：

```bash
ls -1 /root/D | grep -E '<vc关键字>'
```

注意：rayctl 输出的 `VC` 显示名不保证等于 `/root/D/` 文件名，需要核实。

3. 切到 vcluster kubeconfig 查询 vcjob / PodGroup / Pod 详细信息和创建时间：

```bash
export KUBECONFIG=/root/D/<实际 kubeconfig 文件名>
kubectl get vcjob <job-name> -n <namespace> -o wide
kubectl get podgroup <podgroup-name> -n <namespace>
kubectl get pod -n <namespace> | grep <job-name>
kubectl describe pod <pod-name> -n <namespace>
kubectl get pod <pod-name> -n <namespace> -o yaml | grep creationTimestamp
```

如果 rayctl 在 HC 阶段要求 vcluster kubeconfig，也可以直接走 vcluster 查：

```bash
rayctl -k /root/D/<实际 kubeconfig 文件名> job get <job-name-or-pod-name-or-uid>
```

不要再使用旧命令：

```bash
rayctl job check <job-name>
rayctl job get job <job-name>
rayctl job get pg <podgroup-name-or-uid>
```

### 3.2 分区级任务查询

用于查看所有 VC / 分区的任务概览：

```bash
export KUBECONFIG=/root/kubeconfig
rayctl job get cluster -A
```

只看所有 VC / 分区的 Pending 任务：

```bash
rayctl job get cluster -A pending
```

查看指定 VC / 分区的任务概览：

```bash
rayctl job get cluster <vc-name>
```

只看指定 VC / 分区的 Pending 任务：

```bash
rayctl job get cluster <vc-name> pending
```

注意：命令关键字是 `cluster`，不要写成 `culster`。

分区级概览用于定位范围和异常集中点；需要解释单个任务 Pending / 失败原因时，再回到单任务查询。

### 3.3 kubectl 兜底查询

rayctl 不够明确时进入 vcluster：

```bash
export KUBECONFIG=/root/D/<实际 kubeconfig 文件名>
kubectl get vcjob -A | grep <job-name>
kubectl get pod -A | grep <job-name>
kubectl describe pod <pod-name> -n <namespace>
```

查 PodGroup：

```bash
export KUBECONFIG=/root/D/<实际 kubeconfig 文件名>
kubectl get podgroup -A | grep <vcjob-name>
kubectl describe podgroup <podgroup-name> -n <namespace>
```

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
rayctl node get -A 10.12.138.26
rayctl node get -A 10.12.138
rayctl node get -A '10.12.138|10.140.214'
rayctl node get -A | grep host-10-12-138-26
rayctl node get 'node-role.compute.sensecore.cn/prod=ecs'
```

节点详情：

```bash
rayctl node check <node-name-or-ip>
rayctl node describe <node-name-or-ip>
```

查某个节点上是否有任务 / Pod 在运行：

```bash
export KUBECONFIG=/root/kubeconfig
rayctl node check <node-ip-or-hostname>
```

输出包含：所属 VC、GPU/CPU/MEM 占用、PODS 数及具体 Pod 列表。
适用于「看某个节点上跑的是什么任务」「节点是否空闲」。

节点调度控制，写操作，执行前必须确认：

```bash
rayctl node cordon <node-name>
rayctl node uncordon <node-name>
```

兜底 kubectl，仅在 rayctl 不可用或用户明确要求时使用：

```bash
kubectl cordon <node-name>
kubectl uncordon <node-name>
```

---

## 5. AFS / PVC / PV 查询

查询 AFS：

```bash
export KUBECONFIG=/root/kubeconfig
rayctl afs check <afs-name>
rayctl afs check -l <afs-name-or-uid>
```

查询 PVC：

```bash
export KUBECONFIG=/root/D/<实际 kubeconfig 文件名>
rayctl pvc check <pvc-name>
rayctl pvc check -l <pvc-name>
kubectl get pvc <pvc-name> -n <namespace> -o yaml
kubectl describe pvc <pvc-name> -n <namespace>
```

查询 Host PV：

```bash
export KUBECONFIG=/root/kubeconfig
kubectl get pv <host-pv-name>
kubectl get pv <host-pv-name> -o yaml
kubectl describe pv <host-pv-name>
```

---

## 6. 创建 PVC

写操作，执行前必须确认。

```bash
export KUBECONFIG=/root/D/<实际 kubeconfig 文件名>

rayctl pvc create   --name pvc-<afs-name>   --uid <afs-uuid>   --secret <secret-name>   --ns <namespace>   --size 1000Mi
```

创建后检查：

```bash
rayctl pvc check pvc-<afs-name>
kubectl get pvc pvc-<afs-name> -n <namespace> -o yaml
kubectl describe pvc pvc-<afs-name> -n <namespace>
```

PVC Pending 时停止后续任务创建。

---

## 7. ECS / AIS

```bash
export KUBECONFIG=/root/kubeconfig
rayctl ecs check <ais-name-or-ecs-name-or-uid>
rayctl ecs check <id-1> <id-2>
```

登录 ECS / AIS 对应 VM：

```bash
rayctl ecs login <ais-name-or-ecs-name-or-uid>
```
