# mccl-platform-yaml.md - 平台纳管 MCCL YAML 模板

本文件只保存新机器平台纳管前 MCCL 验收使用的 YAML / vcjob / PodGroup 模板、检查命令和判断规则。

流程判断以 `skills/mccl-test/SKILL.md` 为准。
入口边界、写操作确认、安全红线以 `MEMORY.md` 为准。

---

## 0. 适用范围

适用：

* 新 MUXI 机器平台纳管前 MCCL 验收。
* 通过平台 YAML / vcjob / PodGroup 起多机 MCCL 任务。
* 新机器验收时的 Pod / PodGroup / Event / 日志检查。
* 后续 Huawei / 910B / 910C 平台纳管 MCCL 验收扩展。

不适用：

* 已纳管 MUXI 机器维修后单节点验收。
* 宿主机上直接运行 `~sensetime/mccl.sh`。
* MCCL 通过后 `rayctl node uncordon`。
* D 集群物理机器 ansible 操作。

---

## 1. MUXI 平台纳管 MCCL

### 1.1 前置确认

创建任务前必须确认：

* vcluster。
* namespace。
* 目标节点范围。
* 机器类型。
* 节点数。
* 每节点卡数。
* `CARD_NUM`。
* worker replicas。
* `minAvailable`。
* image。
* command。
* 是否需要 label / selector。
* 测试完成后是否清理任务 / label。

所有创建 YAML / vcjob、打 label、删 label、删除任务都属于写操作，必须确认。

---

### 1.2 只读检查：节点范围和状态

host cluster 视角查询节点：

```bash
export KUBECONFIG=/root/kubeconfig

NODE_QUERY='<node-ip-fragment-or-selector>'

rayctl node get -A "$NODE_QUERY"
rayctl node check "$NODE_QUERY"
```

确认 vcluster kubeconfig：

```bash
ls -1 /root/D

export KUBECONFIG=/root/D/<实际 kubeconfig 文件名>
```

---

### 1.3 YAML 创建原则

* 先生成 YAML。
* 先 dry-run / 预览。
* 用户确认后再创建。
* 创建后必须检查 Pod / PodGroup / Event / 日志。
* 失败时不得自动扩大节点范围。
* 排除坏节点时必须同步调整 replicas、minAvailable、CARD_NUM 和节点选择范围。

如果本文件没有维护可用 YAML 模板，必须停止并要求用户提供已验证模板，不得临时编造生产 YAML。

---

### 1.4 YAML 模板占位

当前未固化生产可用 YAML 模板。

在补充真实模板前，只允许记录以下占位字段，不得直接执行：

```yaml
# TODO: paste verified MCCL platform onboarding YAML here.
# Required fields:
# - apiVersion:
# - kind:
# - metadata.name:
# - metadata.namespace:
# - worker replicas:
# - minAvailable:
# - CARD_NUM:
# - image:
# - command:
# - resources:
# - nodeSelector / affinity / label selector:
# - volume / PVC if needed:
```

补充真实 YAML 时，必须标明：

* 适用 vcluster / 机器类型。
* 节点数。
* 每节点卡数。
* image。
* command。
* 资源名。
* label / selector 规则。
* 成功标准。
* 清理方式。

---

### 1.5 创建后检查

```bash
export KUBECONFIG=/root/D/<实际 kubeconfig 文件名>

NS='<namespace>'
JOB='<job-name>'

kubectl get pod -n "$NS" -o wide | grep -F -- "$JOB"
kubectl get podgroup -n "$NS" | grep -F -- "$JOB"
kubectl get events -n "$NS" --sort-by=.lastTimestamp | grep -F -- "$JOB"
```

查看日志：

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

查看任务所有 Pod 分布：

```bash
kubectl get pods -n "$NS" -o wide | grep -F -- "$JOB"
```

---

### 1.6 结果判断

通过条件至少包括：

* 所有 MCCL Pod 正常完成。
* `rc=0`。
* `Out of bounds values : 0 OK`。
* 未出现 error / failed / timeout / hang。
* `Avg bus bandwidth` 达到当前验收标准。

验收标准如果未在本文件定义，必须返回：

```text
不确定，需要用户提供当前纳管验收标准。
```

不得自行编造带宽阈值。

---

### 1.7 坏节点排除

如果发现坏节点，排除前必须重新确认：

* 排除哪个节点。
* 剩余节点数量。
* worker replicas。
* `minAvailable`。
* `CARD_NUM`。
* label / selector 范围。
* 是否重新创建任务。

禁止只排除节点但不调整任务参数。

---

### 1.8 清理

删除任务 / 删除 label / 清理 YAML 产生的资源属于写操作，必须确认。

清理前必须展示：

* 目标 namespace。
* job name。
* Pod / PodGroup / vcjob。
* label key / value。
* 预览命令或 dry-run。

---

## 2. Huawei 平台纳管 MCCL

当前未定义。

后续新增时必须补充：

* 适用 vcluster / 机器类型。
* 资源名。
* 镜像。
* YAML 模板。
* 环境变量。
* HCCL / 集合通信测试命令。
* 通过标准。
* 常见失败模式。
* 清理方式。

当前没有 Huawei SOP 时：

* 不复用 MUXI `mccl.sh`。
* 不复用 MUXI YAML。
* 停止并说明需要补充 Huawei 平台纳管 MCCL 流程。
