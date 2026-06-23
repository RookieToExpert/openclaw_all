# TOOLS.md - OpenClaw Tool Index

本文件只保留工具入口索引、文件分工和高频跳转。
长期原则、安全红线、入口路由放到 `MEMORY.md`。
具体命令模板放到 `tools/*.md`。
专项流程放到 `skills/<skill-name>/SKILL.md`。

---

## 0. 文件分工

| 文件 / 目录 | 作用 |
|---|---|
| `MEMORY.md` | 长期稳定原则、入口路由、安全红线、跨任务优先级 |
| `TOOLS.md` | 工具索引，只负责告诉 OpenClaw 去哪里找命令 |
| `tools/environment-entry.md` | 开发机、堡垒机、跳板机、远程命令通用规则 |
| `tools/rayctl-kubectl.md` | kubeconfig、rayctl、kubectl、单任务与分区级任务查询、节点查询、AFS/PVC/PV、ECS/AIS |
| `tools/job-templates.md` | rayctl job create 模板、vcluster 到模板映射、资源范围 |
| `tools/dcluster-ansible.md` | D 集群物理机操作、JumpServer、expect、ansible 单 IP、批量 Ansible |
| `tools/mccl-commands.md` | MCCL 单机测试底层命令、`mccl.sh`、日志检查命令 |
| `tools/k8s-cleanup.md` | 批量清理 Pod / vcjob、历史 vcjob TTL 回收、断线续查和事后核验模板 |
| `tools/fault-records.md` | 运维故障记录表入口和查询关键字 |
| `skills/` | 按任务场景组织的 SOP，负责流程，不重复维护所有命令 |
| `memory/YYYY-MM-DD.md` | 动态查询结果、当天状态、临时结论；下次使用前必须重新验证 |

---

## 1. 快速路由

| 用户请求 | 先读 | 再读 |
|---|---|---|
| 查单任务 Pending, error, failed / vcjob / PodGroup / NotEnoughResources | `skills/vcjob-debug/SKILL.md` | `tools/rayctl-kubectl.md` |
| 查某个 VC / 分区任务概览、Pending 分布、前端任务数异常 | `skills/vcjob-debug/SKILL.md` | `tools/rayctl-kubectl.md` 的分区级任务查询 |
| 创建 / 查询 PVC、AFS、PV | `skills/pvc-afs/SKILL.md` | `tools/rayctl-kubectl.md` |
| 单机 MUXI MCCL 维修验收 / MUXI 节点放回 / 机器uncordon / MCCL 通过后放回 | `skills/mccl-test/SKILL.md` | `tools/dcluster-ansible.md` + `tools/mccl-commands.md` |
| 多机 / 平台纳管 MCCL | `skills/mccl-test/SKILL.md` | `tools/rayctl-kubectl.md` + `tools/job-templates.md` |
| 查看 D 集群物理机目录 / 日志 / NPU / 磁盘 / DNS | `skills/dcluster-machine-op/SKILL.md` | `tools/dcluster-ansible.md` |
| 批量删除 Pod / vcjob / Failed 资源、历史 vcjob TTL 回收 | `skills/k8s-cleanup/SKILL.md` | `tools/k8s-cleanup.md` |
| 更新 OpenClaw memory / tools / skills / 知识库 | `skills/update-openclaw-memory/SKILL.md` | `git pull --ff-only` |
| 查询节点属于哪个 vcluster | `MEMORY.md` 入口路由 | `tools/rayctl-kubectl.md` 的节点查询 |
| 创建训练 / 推理任务 | `MEMORY.md` 写操作确认 | `tools/job-templates.md` |
| 查询故障 / 维修记录 | `MEMORY.md` 节点维修原则 | `tools/fault-records.md` |

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

- 需要流程判断时，优先看 `skills/<skill>/SKILL.md`。
- 需要具体命令时，跳到 `tools/*.md`。
- 如果 skill 与 `MEMORY.md` 冲突，以 `MEMORY.md` 为准。
- 如果 skill 与 `tools/*.md` 命令参数冲突，以 `tools/*.md` 最新模板为准。
- 写操作必须遵守 `MEMORY.md` 的确认机制。
- 动态查询结果不要写回 `MEMORY.md` 或 `TOOLS.md`，写入 `memory/YYYY-MM-DD.md` 并标注复核要求。
