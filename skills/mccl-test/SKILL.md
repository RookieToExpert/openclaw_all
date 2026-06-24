# mccl-test Skill

## 触发条件

用于 MCCL 验收、MCCL 排障和 MCCL 结果判断，包括：

* 已纳管 MUXI 机器维修后 MCCL 验收。
* MCCL 通过后节点放回集群 / uncordon。
* 新 MUXI 机器平台纳管前 MCCL 验收。
* 通过 YAML / vcjob / PodGroup 起多机 MCCL 任务。
* `Avg bus bandwidth`、`Out of bounds`、MCCL timeout / failed / hang。
* Huawei / 910B / 910C 平台纳管 MCCL 验收的未来扩展。

---

## 1. 必须先区分场景

MCCL 相关请求必须先分流，禁止混用流程。

| 场景                    | 典型触发语                           | 入口                                                      | 宿主机跑 mccl.sh | 创建 YAML / vcjob |          uncordon |
| --------------------- | ------------------------------- | ------------------------------------------------------- | -----------: | --------------: | ----------------: |
| 场景 A：已纳管 MUXI 机器维修后验收 | 维修后验收、单节点验收、MCCL 通过后放回、uncordon | 堡垒机 → 跳板机 → sensetime → ansible 单 IP                    |            是 |               否 | 默认否；通过后且用户明确要求才允许 |
| 场景 B：新 MUXI 机器平台纳管前验收 | 新机器纳管、平台纳管验收、通过 YAML 起多机 MCCL   | D 集群开发机 → vcluster kubeconfig → YAML / vcjob / PodGroup |            否 |               是 |       不适用或按纳管 SOP |
| 场景 C：Huawei 平台纳管 MCCL | 910B / 910C 新机器纳管               | 未来扩展                                                    |           待定 |              待定 |                待定 |

如果用户没有说清楚是“已纳管维修后验收”还是“新机器平台纳管验收”，必须停止并询问场景，不得自行选择流程。

---

## 2. 场景 A：已纳管 MUXI 机器维修后验收

### 2.1 场景定义

适用于已经纳管到平台的 MUXI 节点，维修后需要在宿主机上验证单节点 MCCL 是否恢复正常。

入口：

```text
本地 → 堡垒机 → 跳板机 → sensetime 用户 → ansible 单 IP
```

强规则：

* 不创建 vcjob。
* 不起 YAML。
* 不打 Kubernetes label。
* 不删 Kubernetes label。
* 不为了测试先 uncordon。
* 不在本地或 D 集群开发机直接执行 ansible。
* 节点原本处于 cordon / 维修状态时，应保持原状态。
* 只有 MCCL 通过后，且用户明确要求“放回 / uncordon”，才切回 D 集群开发机执行 `rayctl node uncordon`。

### 2.2 相关工具

* `tools/dcluster-ansible.md`
* `tools/mccl-commands.md`
* `tools/rayctl-kubectl.md` → 4. rayctl 节点查询，仅用于通过后放回集群

### 2.3 前置确认

只读检查可以直接执行；跑测试和写入脚本前必须确认。

确认内容：

* 目标 IP / hostname。
* 测试卡数，默认 8 卡。
* 测试轮数，默认 5 轮。
* 输出日志路径，默认 `~sensetime/mccl-*.log`。
* 是否保持节点当前 cordon / 维修状态。
* MCCL 通过后是否需要放回集群。

### 2.4 只读检查

按 `tools/dcluster-ansible.md` 和 `tools/mccl-commands.md` 执行：

1. 确认在跳板机 `sensetime` 用户下。
2. 检查目标机 `hostname && date`。
3. 检查 `mx-smi`。
4. 检查 `~sensetime/mccl.sh` 是否存在。

如果 `mccl.sh` 不存在：

* 写入 `~sensetime/mccl.sh` 属于写操作。
* 必须先向用户确认。
* 脚本内容和下发方式以 `tools/mccl-commands.md` 为准。
* 不要手写超长内联 ansible shell。

### 2.5 执行 MCCL 测试

用户确认后，按 `tools/mccl-commands.md` 的“五轮八卡测试”执行。

判断重点：

* 每轮每个 benchmark 的 `Avg bus bandwidth`。
* `Out of bounds values : 0 OK`。
* `rc=0`。
* 是否出现 error / failed / timeout / hang。

输出时必须保留原始关键行，不擅自改写原始结果。

### 2.6 通过后放回集群

只有同时满足以下条件才允许进入 uncordon：

1. MCCL 测试通过。
2. 用户明确要求“放回 / uncordon”。
3. 已确认 node name。
4. 已完成写操作确认。

放回流程：

* 切回 Kubernetes 入口。
* 使用 `tools/rayctl-kubectl.md` → 4. rayctl 节点查询。
* 优先 `rayctl node uncordon <node-name>`。
* 不要优先使用 `kubectl uncordon`，除非 rayctl 不可用或用户明确要求。

---

## 3. 场景 B：新 MUXI 机器平台纳管前 MCCL 验收

### 3.1 场景定义

适用于新到 MUXI 机器，在正式纳管到平台前，通过平台任务方式验证多机 MCCL。

入口：

```text
本地 → D 集群开发机 → vcluster kubeconfig → YAML / vcjob / PodGroup
```

强规则：

* 不走宿主机 ansible 单 IP 的维修验收流程。
* 不直接在物理机上跑 `mccl.sh` 代替平台纳管验收。
* 必须通过平台任务 / YAML / vcjob / PodGroup 验证。
* 创建 YAML / vcjob、打 label、删 label、删除任务都属于写操作，必须确认。
* 如果排除坏节点，必须同步调整 replicas、minAvailable、CARD_NUM、节点选择范围。

### 3.2 相关工具

* `tools/mccl-platform-yaml.md`
* `tools/rayctl-kubectl.md` → 3. 查询任务、4. rayctl 节点查询
* `tools/job-templates.md` 仅在明确使用 rayctl job create 模板时读取

### 3.3 前置确认

必须明确：

* 目标 vcluster。
* namespace。
* 机器类型。
* 目标节点范围。
* 节点数。
* 每节点卡数。
* `CARD_NUM`。
* worker replicas。
* `minAvailable`。
* image。
* command。
* 是否需要 label / selector。
* 测试完成后是否删除任务 / label。

### 3.4 标准流程

1. 确认目标 vcluster、namespace、节点范围、机器类型、卡数、节点数。
2. 只读查询节点是否已被平台识别。
3. 确认是否需要 label / selector / taint / nodeName。
4. 选择或生成 MCCL YAML。
5. 先 dry-run / 预览。
6. 用户确认后创建任务。
7. 观察 Pod / PodGroup / Event / 日志。
8. 根据 `Avg bus bandwidth`、`Out of bounds`、退出码和 timeout 判断是否通过。
9. 输出是否满足纳管条件。
10. 清理任务 / label 前必须再次确认。

### 3.5 坏节点排除

如果平台纳管 MCCL 任务发现坏节点，可以动态排除，但必须同步调整：

* worker replicas。
* `minAvailable`。
* `CARD_NUM`。
* 节点选择范围。
* PodGroup / vcjob 模板中的节点数。

不得只删节点、不改任务参数。

---

## 4. 场景 C：Huawei 平台纳管 MCCL

Huawei / 910B / 910C 新机器平台纳管 MCCL 暂作为未来扩展。

规则：

* 不复用 MUXI 的 `mccl.sh`。
* 不复用 MUXI YAML，除非工具文件明确支持。
* 需要新增 Huawei 专用模板、镜像、资源名、环境变量和判断标准。
* 当前没有明确 SOP 时，停止并说明需要补充 Huawei 纳管 MCCL 流程。

---

## 5. 输出要求

默认输出偏好以 `MEMORY.md` 为准。

MCCL 结果必须保留原始关键行：

* `Avg bus bandwidth`
* `Out of bounds`
* `rc=`
* error / failed / timeout / hang

场景 A 输出建议：

```text
场景：已纳管 MUXI 维修后验收
结论：
证据：
- Avg bus bandwidth 原始行：
- Out of bounds 原始行：
- rc：
- error / failed / timeout：
下一步：
- 是否建议放回：
- 如需 uncordon，等待确认：
```

场景 B 输出建议：

```text
场景：新 MUXI 机器平台纳管前验收
结论：
证据：
- Pod / PodGroup 状态：
- Avg bus bandwidth 原始行：
- Out of bounds 原始行：
- rc：
- error / failed / timeout：
判断：
- 是否满足纳管条件：
- 待验证点：
下一步：
- 是否需要排除节点：
- 是否需要清理任务 / label：
```

---

## 禁止事项

* 不要把已纳管 MUXI 维修后验收做成 vcjob / YAML。
* 不要用宿主机 `mccl.sh` 代替新机器平台纳管验收。
* 不要为了单机维修验收打 label / 删 label。
* 不要为了单机维修验收先 uncordon。
* 不要在本地或开发机直接执行 D 集群机器 ansible。
* 不要在没有用户确认时创建任务、打 label、删 label、删除任务或 uncordon。
* 不要擅自汇总用户要求保留原始输出的 MCCL 结果。
