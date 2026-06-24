# TOOLS.md - OpenClaw Tool Index

本文件只保留工具入口索引和高频跳转。

命中 skills/<skill-name>/SKILL.md 的场景，必须先读取对应 skill，再读取右侧 tools 小节。
不得只读 tools 后直接执行命令。
TOOLS.md 只负责路由，不负责替代 SOP。
长期规则、入口边界、安全红线、用户偏好以 `MEMORY.md` 为准。
运行时优先级、skill 发现、工具选择、执行预算和停止条件以 `AGENTS.md` 为准。
具体命令模板放到 `tools/*.md`。
专项流程放到 `skills/<skill-name>/SKILL.md`。

---

## 1. 快速路由

| 用户请求                                                         | 先读                            | 再读                                                                          |
| ------------------------------------------------------------ | ----------------------------- | --------------------------------------------------------------------------- |
| vcjob / Volcano Job / PodGroup / Pending / Failed 单任务排障      | `skills/vcjob-debug/SKILL.md` | `tools/rayctl-kubectl.md` → 3.1 单任务查询、3.3 vcluster 内资源下钻模板、3.4 kubectl 兜底查询 |
| NotEnoughResources / Insufficient cpu / memory / accelerator | `skills/vcjob-debug/SKILL.md` | `tools/rayctl-kubectl.md` → 3.1 单任务查询、3.5 NotEnoughResources 资源判断模板         |
| Failed 任务疑似坏节点 / HCCL / ACL / timeout / OOM / 信号终止           | `skills/vcjob-debug/SKILL.md` | `tools/rayctl-kubectl.md` 3.1 单任务查询 → 3.6 Failed 任务节点定位模板；再读 `tools/fault-records.md` |
| VC / 分区级任务概览、Pending 分布、前端任务数异常、历史任务堆积、哪个 VC 大量提交任务          | `skills/vcjob-debug/SKILL.md` | `tools/rayctl-kubectl.md` → 3.2 分区级任务查询                                     |
| 已纳管 MUXI 机器维修后 MCCL 验收 / 单节点维修验收 / MCCL 通过后放回集群 / 节点 uncordon | `skills/mccl-test/SKILL.md` → 场景 A | `tools/dcluster-ansible.md` → 1/3/4；`tools/mccl-commands.md` → 1/3/4/5/6；通过后如需放回，再读 `tools/rayctl-kubectl.md` → 4. rayctl 节点查询 |
| 新 MUXI 机器平台纳管前 MCCL 验收 / 通过 YAML 起多机 MCCL / 新机器验收纳管           | `skills/mccl-test/SKILL.md` → 场景 B | `tools/mccl-platform-yaml.md`；必要时读 `tools/rayctl-kubectl.md` → 3. 查询任务、4. rayctl 节点查询                                          |
| Huawei / 910B / 910C 新机器平台纳管 MCCL 验收                          | `skills/mccl-test/SKILL.md` → 场景 C | 当前为未来扩展；没有明确 Huawei SOP 时停止，不得复用 MUXI YAML 或 MUXI mccl.sh                                                                      |
| 创建训练 / 推理任务                                              | `tools/job-templates.md`                 | `tools/rayctl-kubectl.md`                              |
| 创建 / 查询 PVC、AFS、PV                                           | `skills/pvc-afs/SKILL.md`     | `tools/rayctl-kubectl.md` → 5. AFS / PVC / PV 查询；创建 PVC 时再读 6. 创建 PVC       |
| 批量删除 Pod / vcjob / Failed 资源、历史 vcjob TTL 回收             | `skills/k8s-cleanup/SKILL.md`            | `tools/k8s-cleanup.md`                                 |
| 查看 D 集群物理机目录 / 日志 / NPU / 磁盘 / DNS                       | `skills/dcluster-machine-op/SKILL.md`    | `tools/dcluster-ansible.md`                            |
| Kubernetes / vcluster 基础只读查询：节点属于哪个 vcluster、节点上有哪些 Pod / 任务、节点状态 / 资源 | 无需 skill，直接读 `tools/rayctl-kubectl.md` → 4. rayctl 节点查询 | 如需解释具体 vcjob / Pending / Failed，再读 `skills/vcjob-debug/SKILL.md` |
| 机器类型 / 芯片类型 / vcluster 族 / IP 段 / 资源名 / 节点类型映射查询 | 无需 skill，直接读 `tools/machine-types.md` | 如需实时验证某个节点或 IP，再读 `tools/rayctl-kubectl.md` → 4. rayctl 节点查询 |                                       
| 查询故障 / 维修记录                                              | `tools/fault-records.md`                 | -                                                      |

---

## 2. 最高频入口速查

Kubernetes / vcluster 操作入口：

```bash
ssh -p 32222 root@10.140.158.149
```

D 集群物理机操作入口：

```text
本地 → 堡垒机 10.140.3.216:5906 → JumpServer 输入 220.33 → 跳板机 → sudo su sensetime → ansible 单 IP
```

开发机只用于 Kubernetes / rayctl / kubectl。
跳板机 sensetime 用户只用于 D 集群物理机 ansible。

---

## 3. 使用约定

* 需要流程判断时，读 `skills/<skill-name>/SKILL.md`。
* 需要具体命令时，读对应 `tools/*.md`。
* 需要安全边界、入口判断、写操作确认时，以 `MEMORY.md` 为准。
* 需要运行时优先级、执行预算、停止条件时，以 `AGENTS.md` 为准。
* 动态查询结果不要写回 `MEMORY.md` 或 `TOOLS.md`，写入 `memory/YYYY-MM-DD.md` 并标注复核要求。
