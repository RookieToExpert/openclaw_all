# mccl-test Skill

## 触发条件

用于 MCCL / HCCL 验收、通信测试、通信排障和结果判断。

适用范围包括：

* 已纳管 MUXI 机器维修后 MCCL 验收。
* MCCL 通过后节点放回集群 / uncordon。
* 新 MUXI 机器平台纳管前 MCCL 验收。
* 通过 YAML / vcjob / PodGroup 起多机 MCCL 任务。
* Huawei 910C 新机器平台纳管前 HCCL 压测。
* Huawei 910C SuperPod HCCL allreduce / allgather / alltoall 压测。
* Huawei 910C 指定节点 HCCL allreduce / allgather / alltoall 压测。
* Huawei 910C HCCL 二分法排坏节点。
* `Avg bus bandwidth`、`alg_bandwidth`、`Out of bounds`、`check_result`、MCCL / HCCL timeout / failed / hang。

Huawei 910B 暂不复用 910C SOP。除非已有明确 910B 工具文件和模板，否则遇到 Huawei 910B HCCL 请求时必须停止，说明当前 SOP 只覆盖 Huawei 910C。

---

## 1. 必须先区分场景

MCCL / HCCL 相关请求必须先分流，禁止混用流程。

| 场景                           | 适用对象                                                | 入口                                                       | 宿主机跑脚本 | 创建 YAML / vcjob |          uncordon |
| ---------------------------- | --------------------------------------------------- | -------------------------------------------------------- | -----: | --------------: | ----------------: |
| 场景 A：已纳管 MUXI 机器维修后验收        | 已纳管 MUXI 单节点维修后验收                                   | 堡垒机 → 跳板机 → sensetime → ansible 单 IP                     |      是 |               否 | 默认否；通过后且用户明确要求才允许 |
| 场景 B：新 MUXI 机器平台纳管前验收        | 新 MUXI 机器纳管前多机 MCCL                                 | D 集群开发机 → vcluster kubeconfig → YAML / vcjob / PodGroup  |      否 |               是 |       不适用或按纳管 SOP |
| 场景 C：Huawei 910C HCCL 平台任务压测 | Huawei 910C 新机器纳管前 HCCL / SuperPod / 指定节点 / 二分法排坏节点 | D 集群开发机 → `~/D/a3-huawei-test` → YAML / vcjob / PodGroup |      否 |               是 |               不适用 |

如果用户没有说清楚是“已纳管维修后验收”还是“新机器平台纳管验收”，必须停止并询问场景，不得自行选择流程。

如果用户说的是 Huawei 910C HCCL，则直接进入场景 C，不要进入 MUXI 场景。

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

必须先读：

```text
tools/dcluster-ansible.md
tools/mccl-commands.md
```

通过后如需放回集群，再读：

```text
tools/rayctl-kubectl.md → 4. rayctl 节点查询
```

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
测试后必须残留核验，放回前必须 MCCL_RESIDUAL_CLEAN。

### 2.6 通过后放回集群

只有同时满足以下条件才允许进入 uncordon：

1. MCCL 测试通过。
2. 用户明确要求“放回 / uncordon”。
3. 已确认 node name。
4. 已完成写操作确认。

放回流程：

只有同时满足以下条件才允许进入 uncordon：
1. MCCL 测试通过。
2. 测试结束后已执行残留清理，或确认无残留。
3. 放回前核验输出 `MCCL_RESIDUAL_CLEAN`。
4. 用户明确要求“放回 / uncordon”。
5. 已确认 Kubernetes node name。
6. 已完成写操作确认。

如果残留核验失败，必须停止，不得 uncordon。
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

必须先读：

```text
tools/mccl-platform-yaml.md
```

必要时再读：

```text
tools/rayctl-kubectl.md → 3. 查询任务、4. rayctl 节点查询
```

仅在用户明确要求使用 `rayctl job create` 模板时，才读：

```text
tools/job-templates.md
```

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

4. 场景 C：Huawei 910C HCCL 平台任务压测
4.1 场景定义

适用于 Huawei Ascend 910C 通过平台任务方式验证多机 HCCL 集合通信。

入口：

本地 → D 集群开发机 → vcluster kubeconfig → YAML / vcjob / PodGroup

当前 Huawei 910C 测试入口：

ssh -p 32222 root@10.140.158.149
export KUBECONFIG=~/D/a3-huawei-test

默认 namespace：

NS=default

强规则：

Huawei 910C 走 HCCL 验收，不复用 MUXI 的 mccl.sh。
不复用 MUXI YAML。
不直接在物理机上手工跑脚本代替平台任务验收。
默认通过 YAML / vcjob / PodGroup 创建平台任务。
创建 YAML / vcjob、删除任务、打 label、删 label 都属于写操作，必须确认。
当前 910C 每节点按 16 卡计算。
HCCL mpirun 必须使用容器内 sshd -p 2222。
不得把 HCCL mpirun 的 Hydra 端口改成 22。
Huawei 910B 暂不复用本 SOP，除非已有独立 910B 工具文件和模板。
4.2 相关工具

必须先读：

tools/huawei-hccl-platform-yaml.md

必要时再读：

tools/rayctl-kubectl.md → 节点查询、任务查询、Pod / PodGroup / Event / 日志查询

不要读取 tools/job-templates.md，除非用户明确要求改成 rayctl job create 方式。

4.3 前置确认

创建任务前必须明确：

vcluster / kubeconfig。
namespace。
节点数。
测试方法：allreduce、allgather 或 alltoall。
调度方式：SuperPod 或指定节点。
SuperPod ID，或指定节点 IP 列表。
image。
CANN / HCCL test 路径。
是否使用零拷贝，仅 allgather 需要确认开 / 关。
是否指定 Device 执行，仅单卡 / 单 die 场景需要确认 Device ID。
generateName。
是否创建任务。
是否清理旧任务。
是否需要二分法排坏节点。

不明确时，只允许做只读确认，不得创建任务。

4.4 测试范围

Huawei 910C HCCL 测试范围由用户当次指定，不在 SOP 中固定。

用户可以指定：

固定节点列表
SuperPod X
某个节点集合
单组 allreduce
单组 allgather
单组 alltoall
allreduce + allgather + alltoall
allgather 零拷贝开 / 关对比
alltoall 每机 1 卡 / 1 die 场景
完整测试矩阵
二分法排坏节点

规则：

不要在 skill 中固定 IP 列表。
不要在 skill 中固定 SuperPod ID。
不要默认扩展成完整测试矩阵。
用户只要求一组测试时，只执行该组。
用户要求完整矩阵时，再根据用户提供的节点范围 / SuperPod 范围生成多组任务。
每组任务创建前都必须单独预览关键字段。
每组任务创建都属于写操作，必须确认。
4.5 标准流程
进入 D 集群开发机。
切换 Huawei 910C 测试 kubeconfig。
确认 namespace、节点数、测试方法、调度方式、image、CANN / HCCL test 路径。
读取 tools/huawei-hccl-platform-yaml.md。
根据用户指定范围生成 YAML。
按 tools 文件的字段同步规则检查 YAML。
创建前展示关键字段。
用户确认后创建任务。
等待 Pod 调度、镜像拉取、sshd 启动和 DNS 就绪。
查看 vcjob / Pod / PodGroup / Event / master logs。
根据日志判断基础通信是否成功。
如果日志无结果但 Pod 都 Running，可以进入 master 手动补跑。
如果用户要求二分法排坏节点，进入 4.8。
清理任务前必须再次确认。

4.5.1 已验证的 910C HCCL 测试矩阵约定

以下矩阵是已在 910C SuperPod4 / SuperPod8 上验证过的补跑方式。只有用户明确要求相同矩阵时才复用，不得默认扩大测试范围。

allgather：

* 使用 `all_gather_test`。
* 数据量可按用户要求使用 `512M, 1G, 2G, 4G`，对应 `-b 512M -e 4G -f 2`。
* 需要分别测试零拷贝关闭和开启：
  * 关闭：`-z 0`。
  * 开启：`-z 1`。
* 已验证场景命名和命令模板只维护在 `tools/huawei-hccl-platform-yaml.md`。

alltoall：

* 使用 `alltoall_test`。
* alltoall 不做零拷贝开 / 关对比，不加 `-z`。
* full16 场景：1 机 / 2 机可按 SuperPod 内 16 台全覆盖测试，并输出平均值。
* 每机 1 卡 / 1 die 场景：4 机 / 8 机 / 16 机使用 hostfile 每行 `:1`、`mpirun -n <节点数>`、`-p 1`。
* 指定单卡时使用 `HCCL_TEST_USE_DEVS="<device-id>"`，例如 `HCCL_TEST_USE_DEVS="4"`。
* 已验证场景命名和命令模板只维护在 `tools/huawei-hccl-platform-yaml.md`。

审计留存：

* 用户要求审计时，脚本、hostfile、summary、原始日志和结果表必须保存到 master pod 的 `/tmp/hccl_<test>_<matrix>_<timestamp>/`。
* 建议目录内至少包含：
  * `scripts/`：启动脚本。
  * `hostfiles/`：每个场景使用的 hostfile。
  * `logs/`：每个场景原始日志。
  * `summary.tsv`：场景、节点数、卡数、Device、rc、日志路径。
  * `*_results.md` / `*_results.tsv`：保留两位小数的结果汇总。
4.6 创建前预览

创建前必须展示：

YAML 路径
namespace
generateName
image
节点数
测试方法
调度方式
SuperPod ID 或指定节点列表
sp-block
minAvailable
master replicas
worker replicas
mpirun 总卡数
TEST_BIN
HYDRA_LAUNCHER_EXTRA_ARGS
零拷贝参数，仅 allgather 展示 `-z 0` 或 `-z 1`
指定 Device，仅单卡 / 单 die 场景展示 `HCCL_TEST_USE_DEVS`
审计目录和脚本路径，如果是手动补跑

发现以下任意问题必须停止：

master 和 worker 调度规则不一致。
SuperPod 和指定节点调度同时存在。
节点数相关字段不一致。
Hydra 端口不是 2222。
alltoall 命令带了零拷贝 `-z` 参数。
allgather 零拷贝场景没有明确 `-z 0` 或 `-z 1`。
每机 1 卡 / 1 die 场景中 hostfile 不是每节点 `:1`，或 `-p` 不是 `1`。
YAML 关键字段和用户要求不一致。
用户未确认写操作。
4.7 结果判断

通过条件至少包括：

Pod / PodGroup 状态符合预期。
master log 有 HCCL test 结果。
check_result 为 success。
未出现 error / failed / timeout / hang。
未出现 HcclGetRootInfo failed。
未出现 received invalid data from root process 0。
未出现 Could not resolve hostname。
alg_bandwidth / Avg bus bandwidth 达到用户提供的验收标准。

如果用户未提供带宽阈值，只能判断基础通信是否成功，不能判断是否满足最终纳管带宽标准。

输出必须区分：

基础通信结果：通过 / 不通过 / 不确定
纳管带宽结论：通过 / 不通过 / 不确定，需要用户提供阈值
4.8 二分法排坏节点

二分法目标是通过多轮缩小失败节点范围，定位坏节点或最小失败集合。

标准策略：

1. 一个大规模任务失败，例如 48 机失败。
2. 停掉失败的大任务。
3. 基于同一批节点，起两个较小任务，例如两个 24 机任务。
4. 等待两个任务结果。
5. 成功的一组可以保留，不继续拆。
6. 失败的一组需要停掉，并继续拆成更小任务，例如两个 12 机任务。
7. 重复以上流程，直到定位到单节点或最小失败集合。

规则：

二分法不是只改 hostfile。
每一轮都应重新生成 YAML。
每一轮都必须按 tools/huawei-hccl-platform-yaml.md 的字段同步规则检查 YAML。
每一轮创建任务前必须预览关键字段。
每一轮创建任务都属于写操作，必须确认。
停掉失败任务属于写操作，必须确认。
成功任务是否保留，由用户决定；不得自动删除。
不要自动扩大到用户未指定的节点范围。
不要固定每次必须二等分；用户可以指定按 24/12/6/3/1 或其他分组方式继续排查。

二分法输出建议：

当前轮次：
原始失败任务：
当前失败节点集合：
本轮新建任务：
- 任务 A：节点数 / 节点列表 / 测试结果
- 任务 B：节点数 / 节点列表 / 测试结果
成功集合：
失败集合：
下一轮建议：
需要用户确认的写操作：
4.9 清理

以下操作都属于写操作，必须确认：

删除 vcjob / YAML 任务。
删除 Pod / PodGroup。
删除 label。
删除临时 YAML 生成的资源。
停掉二分法中的失败任务。

清理前必须展示：

vcluster / kubeconfig
namespace
job name
YAML 路径
将删除的资源类型
影响范围
dry-run 或预览命令

用户确认后才允许执行真实清理。
---

## 5. 输出要求

默认输出偏好以 `MEMORY.md` 为准。

MCCL / HCCL 结果必须保留原始关键行：

* `Avg bus bandwidth`
* `alg_bandwidth`
* `check_result`
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

场景 C 输出建议：

```text
场景：Huawei 910C HCCL 平台任务压测
结论：
证据：
- Pod / PodGroup 状态：
- HCCL 结果原始行：
- check_result：
- alg_bandwidth / Avg bus bandwidth：
- error / failed / timeout / hang：
判断：
- 基础通信结果：
- 纳管带宽结论：
- 待验证点：
下一步：
- 是否需要补跑：
- 是否需要二分排坏节点：
- 是否需要清理任务：
```

---

## 禁止事项

* 不要把已纳管 MUXI 维修后验收做成 vcjob / YAML。
* 不要用宿主机 `mccl.sh` 代替新机器平台纳管验收。
* 不要为了单机维修验收打 label / 删 label。
* 不要为了单机维修验收先 uncordon。
* 不要在本地或开发机直接执行 D 集群机器 ansible。
* 不要在没有用户确认时创建任务、打 label、删 label、删除任务或 uncordon。
* 不要擅自汇总用户要求保留原始输出的 MCCL / HCCL 结果。
* 不要把 Huawei 910C HCCL 任务走 MUXI `mccl.sh`。
* 不要复用 MUXI YAML 跑 Huawei 910C HCCL。
* 不要把 HCCL mpirun 的 Hydra 端口改成 22。
* 不要二分法时只改 hostfile，不改 YAML 节点数相关字段。
