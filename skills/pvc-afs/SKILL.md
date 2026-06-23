# pvc-afs Skill

## 触发条件

用户询问以下任意问题时使用本 skill：

- AFS / PVC / PV 查询。
- PVC 创建。
- PVC Pending / Bound 异常。
- 任务挂载 PVC 失败。
- AFS 名称、UID、secret、Host PV 映射。

## 相关工具文件

- `tools/rayctl-kubectl.md`

## 前置原则

- 必须遵守 `MEMORY.md` 的入口路由。
- AFS / Host PV 通常从 host cluster 视角查询。
- vcluster PVC 必须使用对应 vcluster kubeconfig。
- PVC 创建必须使用 rayctl，不要直接 `kubectl create pvc`。
- PVC 命名必须为 `pvc-<afs-name>`。
- 不要擅自给 AFS 名称加 `pvc-` 前缀。
- 写操作必须先确认。

## 标准查询流程

### 1. 查询 AFS

在开发机：

```bash
export KUBECONFIG=/root/kubeconfig
rayctl afs check <afs-name>
rayctl afs check -l <afs-name-or-uid>
```

重点记录：

- AFS name。
- AFS UID。
- secretName。
- Host PV 名称。
- 所属租户 / vcluster。

### 2. 查询 vcluster PVC

先定位实际 vcluster kubeconfig：

```bash
ls -1 /root/D
ls -1 /root/D | grep -E '<vc关键字>'
```

再查询：

```bash
export KUBECONFIG=/root/D/<实际 kubeconfig 文件名>
rayctl pvc check <pvc-name>
rayctl pvc check -l <pvc-name>
kubectl get pvc <pvc-name> -n <namespace> -o yaml
kubectl describe pvc <pvc-name> -n <namespace>
```

### 3. 查询 Host PV

```bash
export KUBECONFIG=/root/kubeconfig
kubectl get pv <host-pv-name>
kubectl get pv <host-pv-name> -o yaml
kubectl describe pv <host-pv-name>
```

## PVC 创建流程

创建前必须向用户确认：

- 目标 vcluster。
- namespace。
- AFS name。
- AFS UID。
- secretName。
- PVC name。
- size。
- 是否会被任务挂载。

确认后执行：

```bash
export KUBECONFIG=/root/D/<实际 kubeconfig 文件名>
rayctl pvc create \
  --name pvc-<afs-name> \
  --uid <afs-uuid> \
  --secret <secret-name> \
  --ns <namespace> \
  --size 1000Mi
```

创建后必须检查：

```bash
rayctl pvc check pvc-<afs-name>
kubectl get pvc pvc-<afs-name> -n <namespace> -o yaml
kubectl describe pvc pvc-<afs-name> -n <namespace>
```

如果 PVC Pending：

1. 停止后续任务创建。
2. 输出 Pending 原因。
3. 等用户确认后再继续。

## 输出格式

```text
结论：
- AFS / PVC / PV 当前状态 ...
证据：
- rayctl afs check ...
- rayctl pvc check ...
- kubectl describe pvc/pv ...
风险：
- 如果继续创建任务，PVC Pending 会导致挂载失败 ...
下一步：
- 只读检查 ...
- 如需创建 / 修改，等待确认 ...
```

## 禁止事项

- 不要擅自改 AFS 名称。
- 不要给 PVC 名追加时间戳 / UUID / 随机后缀。
- 不要在 PVC Pending 时继续创建挂载它的任务。
- 不要用 host cluster kubeconfig 创建 vcluster PVC。
