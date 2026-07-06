# hc-system-op Skill

## 触发条件

用于以下 host cluster 系统组件相关场景：

- 重启 / 重置 VC 控制面。
- 扩容 VC / 调整 VC flavor / 修改 VC 规格。
- 更新 `disallow-privileged-containers` policy，把某个 VC 加入例外。
- 查询 HC 上某个 VC 控制面组件状态。

不适用场景：

- VCJob / PodGroup / Pending / Failed 排障，走 `skills/vcjob-debug/SKILL.md`
- 批量清理历史资源，走 `skills/k8s-cleanup/SKILL.md`
- PVC / AFS / PV，走 `skills/pvc-afs/SKILL.md`
- 物理机目录 / 日志 / NPU，走 `skills/dcluster-machine-op/SKILL.md`

## 相关工具文件

- `tools/hc-system-kubectl.md`
- `tools/rayctl-kubectl.md`

## 前置原则

- VC 内业务资源检查使用 VC kubeconfig。
- HC 系统组件查询和操作使用 HC kubeconfig。
- kubeconfig 类型不明确必须停止。
- namespace、VC UUID、目标组件、目标 Pod 不明确必须停止。
- 默认只读。
- delete / SQL update 都是写操作，必须确认。
- `rayctl policy update disallow-privileged-containers ...` 和对应的 HC policy 修改属于写操作，必须确认。
- 不允许为了猜目标而全量扫描所有 namespace。
- 禁止用 HC kubeconfig 查询、创建、删除 VC 内业务资源。
- 需要验证 VC 内 get/list / Pod 现象时，必须切回 VC kubeconfig。

## 场景选择

### 场景 A：重启 / 重置 VC 控制面

### 1. 收集必要信息

必须明确：

- VC 名称或 VC UID。
- VC kubeconfig 路径。
- HC kubeconfig / context。
- HC namespace。
- 目标控制面组件，例如 apiserver / syncer / controller。
- 可选：VC 内用于验证的 namespace 和 pod name。

缺少任一关键标识时停止。

如果用户只提供 VC 名称或别名、但后续定位需要 VC UID，必须先按 `tools/rayctl-kubectl.md` → 4.2 平台 VC 信息查询执行：

```bash
export KUBECONFIG=/root/kubeconfig

VC='<vc-name-or-uid>'

rayctl vc get "$VC"
```

要求：

- 必须唯一确认目标 VC UID。
- 查不到 VC 时停止。
- 查到多个候选且无法唯一判断时停止。
- 不要为了猜 VC UID 自动遍历所有 vcluster 或全 namespace。

### 2. VC kubeconfig 只读验证

先验证 VC 内是否确实存在 get/list 异常或业务现象。
命令以 `tools/hc-system-kubectl.md` → 3. VC kubeconfig 下验证 get/list 异常为准。

如果没有复现异常：

- 不直接建议 delete HC 控制面 Pod。
- 只返回当前观察、证据和下一步安全选项。

### 3. HC kubeconfig 只读定位

命令以 `tools/hc-system-kubectl.md` → 2. 确认 kubeconfig / context、4. HC namespace 下定位 VC 控制面组件、5. 查询 owner / replicas / selector 为准。

必须确认：

- 当前 context 是 host cluster。
- 已通过 `rayctl vc get` 或用户提供信息唯一确认目标 VC UID。
- 只在已知 HC namespace 下查询。
- 目标 Pod、ownerReference、Deployment / StatefulSet、Ready、restart、age、events 一致。

以下情况必须停止：

- VC UID 不明确。
- owner 不明确。
- 目标不唯一。
- HC namespace 不明确。
- 只读结果与用户描述不一致且无法解释。

### 4. 写操作确认

确认前必须输出：

- 准备删除的 HC namespace 和 Pod name。
- Pod UID。
- owner 类型和 owner 名称。
- 影响范围。
- 风险。
- 删除前最后复核方式。
- 删除后复核方式。
- 预览命令。

等待用户明确回复：

```text
确认 / 同意 / 可以执行 / 执行 / yes
```

### 5. 确认后执行

确认后仍必须先做最后复核：

- 重新 get 目标 Pod 和 UID。
- UID 不一致则停止。
- owner 不再一致则停止。

然后再按 `tools/hc-system-kubectl.md` → 6. 确认后删除控制面 Pod 执行：

- delete 精确 Pod。
- 等待新 Pod Ready。
- 回到 VC kubeconfig 下复查 get/list 或目标业务现象是否恢复。

---

### 场景 B：扩容 VC / 调整 VC flavor

适用场景：

- 扩容 VC。
- 调整 VC 规格。
- 把某个 VC 改成 minimum / small / medium / large / unlimit。
- 修改 VC flavor。
- 查询某个 VC 当前 flavor。

不适用说明：

- 这不是 Kubernetes Deployment / StatefulSet scale。
- 不修改 VC 控制面 Pod 副本数。
- 不通过 kubectl scale 完成。
- 不处理普通训练任务资源扩容。
- 不处理 vcjob Pending / NotEnoughResources 排障。

### 1. 收集必要信息

必须明确：

- VC UID。
- HC kubeconfig / context。
- namespace：`prod-lepton`。
- 执行 SQL 的 Pod：默认 `ubuntu`。
- 目标 flavor。
- 数据库连接信息来源：`lepton-ecp-service` ConfigMap 或用户明确提供的连接串。

缺少 VC UID 或目标 flavor 时停止。

允许列表：

- `minimum`
- `small`
- `medium`
- `large`
- `unlimit`

目标 flavor 不在允许列表中时停止。
目标 flavor 是 `unlimit` 时，必须提示不建议使用，并要求用户二次确认。

### 2. 只读定位

命令以 `tools/hc-system-kubectl.md` → 2. 确认 kubeconfig / context、7. Flavor 调整相关模板为准。

必须确认：

- 当前 context 是 host cluster。
- `prod-lepton` namespace 下存在 `ubuntu` Pod。
- `lepton-ecp-service` ConfigMap 中可获取数据库连接所需信息。
- 可以进入或通过 `kubectl exec` 在 `ubuntu` Pod 内连接数据库。
- 只读 SQL 可唯一查到目标 VC 当前 flavor。

以下情况停止：

- 查不到记录。
- 查到多条记录。
- 当前 flavor 已经等于目标 flavor。
- 无法确认当前 flavor。
- 无法连接数据库。

### 3. 写操作确认

确认前必须输出：

- 准备修改的 VC UID。
- 当前 flavor。
- 目标 flavor。
- SQL 影响范围：`virtual_cluster` 表中 `not deleted and uid = '<VC_UID>'` 的单条记录。
- 风险：平台规格配置变化，可能影响 VC 可用资源 / 调度 / 控制面行为，错误 UID 会改错 VC。
- 回滚方式：把 flavor 改回原 flavor。
- 执行前最后复核方式：重新 `select` 当前 flavor。
- 执行后复核方式：再次 `select flavor`。
- 预览 SQL。

等待用户明确确认后才能继续。
目标 flavor 为 `unlimit` 时，必须额外做一次二次确认。

### 4. 确认后执行

执行前重新只读复核：

- 重新查询当前 flavor。
- 如果当前 flavor 与确认前不一致，停止。

然后按 `tools/hc-system-kubectl.md` → 7. Flavor 调整相关模板执行 update 和复核。

执行后必须确认：

- update 影响范围是 1 条记录，或通过 update 后 select 明确验证唯一记录已变更。
- 输出变更前后对比。

---

### 场景 C：更新 `disallow-privileged-containers` policy

适用场景：

- `rayctl policy update disallow-privileged-containers <vc-name-or-uid>`
- 给某个 VC 添加 `disallow-privileged-containers` policy 例外
- 核对某个 VC 是否已在 `disallow-privileged-containers` 例外中

不适用说明：

- 当前只覆盖 `disallow-privileged-containers`
- 不泛化到其他 policy 名称
- 不处理普通 VC 控制面重启
- 不处理 VC flavor 调整

### 1. 收集必要信息

必须明确：

- policy 名称：`disallow-privileged-containers`
- VC 名称或 VC UID
- HC kubeconfig / context
- 可选：VC kubeconfig，用于只读验证该 VC 基本信息

缺少 VC 标识或 kubeconfig 类型不明确时停止。

### 2. 只读定位

命令以 `tools/rayctl-kubectl.md` 中的 policy 小节和 `tools/hc-system-kubectl.md` → 8. `disallow-privileged-containers` policy 更新模板为准。

先只读确认：

- 当前 context 是 host cluster
- `rayctl vc get <vc-name-or-uid>` 能唯一定位目标 VC
- `rayctl policy get disallow-privileged-containers <vc-name-or-uid>` 能核对目标 VC 是否已经在白名单中
- 如 rayctl 输出不明确，再用 HC `kubectl` 只读查看 `clusterpolicies/disallow-privileged-containers`

以下情况停止：

- policy 不存在
- 查不到目标 VC UID
- 查到多个 VC 候选，无法唯一判断
- 当前 policy 已经包含该 VC
- rayctl 与 kubectl 只读核对结果冲突且无法解释
- 当前 policy 结构与预期差异过大，无法安全判断应修改的位置

### 3. 写操作确认

确认前必须输出：

- 准备更新的 policy 名称：`disallow-privileged-containers`
- 目标 VC 名称和 VC UID
- 准备新增的 selector 关键内容
- 影响范围
- 风险：错误 VC UID 会把错误 VC 加入例外
- 回滚方式：删除刚加入的该 VC 对应例外项
- 执行前最后复核方式：重新 `rayctl vc get` 和 `rayctl policy get`
- 执行后复核方式：优先 `rayctl policy get`，必要时 HC `kubectl` 只读核对 YAML
- 预览命令

等待用户明确确认后才能继续。

### 4. 确认后执行

执行前重新只读复核：

- 再次用 `rayctl vc get` 确认目标 VC UID
- 再次用 `rayctl policy get disallow-privileged-containers <vc-name-or-uid>` 确认该 VC 尚未在白名单中
- 如 rayctl 输出不明确，再确认 policy 仍存在，且该 VC UID 尚未出现在 policy YAML 中

然后优先按 `tools/rayctl-kubectl.md` 中的 `rayctl policy update disallow-privileged-containers` 执行。

执行后必须复核：

- `rayctl policy get disallow-privileged-containers <vc-name-or-uid>` 显示目标 VC 已在白名单中
- 如 `rayctl` 输出不明确，再用 HC `kubectl` 只读检查 policy YAML

如果 `rayctl` 不可用或结果不明确，只能把 `kubectl edit` 作为人工兜底方案展示给用户，不要在未确认结构前自动 edit。

---

## 输出要求

输出要先给结论，再给证据，区分：

- 已确认事实。
- 推断。
- 待验证点。
- 需要用户确认的写操作。

## 禁止事项

- 禁止用 HC kubeconfig 查询或删除 VC 内业务 Pod / vcjob / PodGroup。
- 禁止不知道 namespace 时 `kubectl get pod -A` 猜目标。
- 禁止没有确认就 delete / SQL update / policy update。
- 禁止 owner 不明确时删除控制面 Pod。
- 禁止把 `k8s-cleanup` 当作 VC 控制面重置流程。
- 禁止在 kubeconfig 类型不明确时继续。
- 禁止缺少关键标识时自动扩大范围。
- 当前 SOP 未定义下一步时必须停止。
