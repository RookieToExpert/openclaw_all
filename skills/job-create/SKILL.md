# 训练 / 推理任务创建流程

## 触发条件

用于训练 / 推理任务创建、重跑、生成 `rayctl job create` 命令，包括：

* 创建训练任务。
* 创建推理任务。
* 根据 vcluster / 机器类型选择任务模板。
* 生成 rayctl job create 命令。
* 重跑已有任务。
* 创建任务并挂载 PVC。

## 相关工具文件

* `tools/job-templates.md`
* `tools/rayctl-kubectl.md`
* `skills/pvc-afs/SKILL.md`，仅当任务需要 AFS / PVC 时读取。

---

## 前置原则

* 创建任务是写操作，必须遵守 `MEMORY.md` 的写操作确认规则。
* 提交任务必须使用对应 vcluster kubeconfig。
* 禁止使用 host cluster kubeconfig 提交 vcluster 任务。
* 优先使用 `rayctl job create`。
* 不要只凭 vcluster 名称判断模板；必要时用节点 label、allocatable、rayctl 输出或用户提供信息确认。
* 如果任务依赖 PVC，必须先确认 PVC 已 Bound。
* PVC Pending 时必须停止任务创建。

---

## 标准流程

### 1. 收集必要信息

创建任务前必须明确：

* 目标 vcluster。
* namespace。
* job name。
* 任务类型：训练 / 推理 / 测试。
* 机器类型或模板族。
* image。
* command。
* CPU。
* memory。
* accelerator。
* nodes，单机任务可为空。
* 是否挂载 PVC。
* 可能影响范围。

缺少关键字段时，停止并返回缺失项。

---

### 2. 确认 kubeconfig 和模板

1. 定位实际 vcluster kubeconfig。
2. 选择模板族。
3. 如不确定机器类型，先只读验证。
4. 模板和资源范围以 `tools/job-templates.md` 为准。

禁止：

* 不要把 VC 显示名直接拼成 `/root/D/<VC>`。
* 不要使用 host cluster kubeconfig 创建 vcluster 任务。
* 不要使用未确认的模板。

---

### 3. PVC 前置检查

如果任务需要 PVC：

1. 进入 `skills/pvc-afs/SKILL.md`。
2. 按 `tools/rayctl-kubectl.md` → 5 查询 AFS / PVC / PV。
3. 如果 PVC 不存在，需要创建 PVC 时必须重新确认。
4. 如果 PVC Pending，必须停止任务创建。
5. 只有 PVC Bound 后，才允许继续生成任务命令。

---

### 4. 生成预览命令

先生成 `--what-if` 或预览命令，不直接提交。

命令模板以 `tools/job-templates.md` 为准。

---

### 5. 写操作确认

执行前必须说明：

* 将要创建的任务。
* vcluster / namespace。
* template。
* image。
* command。
* CPU / memory / accelerator / nodes。
* PVC 挂载情况。
* 可能影响。
* 停止或回滚方式。

用户明确确认后才允许执行真实创建。

---

### 6. 创建后检查

创建后必须检查：

* `rayctl job get <job-name-or-uid>`。
* Pod / PodGroup 状态。
* PVC 挂载情况，如有。
* Pending / Failed 原因，如出现异常则转入 `skills/vcjob-debug/SKILL.md`。

具体查询命令以 `tools/rayctl-kubectl.md` → 3. 查询任务为准。

---

## 禁止事项

* 不要在 PVC Pending 时继续创建挂载它的任务。
* 不要自动替用户选择未知 image / command。
* 不要自动扩大到其他 vcluster 查询。
* 不要直接创建任务而跳过预览和确认。
* 不要把平台纳管 MCCL YAML 当成普通训练任务模板；该场景走 `skills/mccl-test/SKILL.md`。
