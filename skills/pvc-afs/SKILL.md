# skills/pvc-afs/SKILL.md - AFS / PVC / PV 生命周期

## 触发条件

用于 AFS / PVC / PV 生命周期相关问题，包括：

* AFS / PVC / PV 查询。
* Host PV 映射查询。
* PVC 创建。
* PVC Pending / Bound 异常。
* 任务挂载 PVC 失败。
* AFS name、AFS UID、secretName、Host PV 定位。

## 相关工具文件

* `tools/rayctl-kubectl.md` → 5. AFS / PVC / PV 查询
* `tools/rayctl-kubectl.md` → 6. 创建 PVC

---

## 前置原则

* 必须遵守 `MEMORY.md` 的入口路由、写操作确认和禁止扩大范围规则。
* AFS / Host PV 通常从 host cluster 视角查询。
* vcluster PVC 必须使用对应 vcluster kubeconfig。
* PVC 创建必须使用 rayctl，不要直接 `kubectl create pvc`。
* PVC 命名必须为 `pvc-<afs-name>`。
* 不要擅自给 AFS name 添加或删除前缀。
* 不要给 PVC 名追加时间戳、UUID 或随机后缀。
* PVC Pending 时必须停止后续任务创建。

---

## 标准流程

### 1. 判断用户意图

先判断属于哪类：

* 只查 AFS / PVC / PV：走只读查询。
* 创建 PVC：走创建流程。
* PVC Pending / 挂载失败：先查询 PVC / PV / Event，再判断是否能继续。
* 创建任务依赖 PVC：先确认 PVC Bound，再允许进入任务创建。

---

### 2. 只读查询流程

目标：

1. 查询 AFS，获取 AFS name、UID、secretName、Host PV。
2. 定位实际 vcluster kubeconfig。
3. 查询 vcluster PVC 状态。
4. 如 rayctl 返回 Host PV，再查询 Host PV。
5. 输出 AFS / PVC / PV 的当前状态和风险。

具体命令以 `tools/rayctl-kubectl.md` → 5. AFS / PVC / PV 查询为准。

停止条件：

* 用户未提供 AFS / PVC / namespace / vcluster 等关键标识，且 rayctl 无法补齐。
* 找不到实际 vcluster kubeconfig。
* 需要扩大范围时，先返回给用户确认。

---

### 3. PVC 创建流程

创建前必须确认：

* 目标 vcluster。
* namespace。
* AFS name。
* AFS UID。
* secretName。
* PVC name，必须是 `pvc-<afs-name>`。
* size。
* 是否会被任务挂载。

确认后按 `tools/rayctl-kubectl.md` → 6. 创建 PVC 执行。

创建后必须检查：

* `rayctl pvc check`。
* `kubectl get pvc`。
* `kubectl describe pvc`。

如果 PVC Pending：

1. 停止后续任务创建。
2. 输出 Pending 原因。
3. 等用户确认后再继续。

---

### 4. 任务挂载 PVC 失败

处理顺序：

1. 查询 PVC 是否存在。
2. 查询 PVC 是否 Bound。
3. 查询 Host PV 是否存在。
4. 查询 Pod Event / describe 输出中的挂载错误。
5. 如果 PVC Pending 或 Host PV 异常，停止任务创建或重试。
6. 需要改 PVC、重建 PVC 或重跑任务时，必须重新确认。

---

## 输出要求

默认输出偏好以 `MEMORY.md` 为准。

PVC 场景必须说明：

* AFS 状态。
* PVC 状态。
* Host PV 状态。
* 是否可以继续创建 / 重跑任务。
* 如果不能继续，说明阻塞原因。

---

## 禁止事项

* 不要擅自改 AFS 名称。
* 不要给 PVC 名追加时间戳 / UUID / 随机后缀。
* 不要在 PVC Pending 时继续创建挂载它的任务。
* 不要用 host cluster kubeconfig 创建 vcluster PVC。
* 不要直接用 `kubectl create pvc` 创建 vcluster PVC。



