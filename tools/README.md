# tools/README.md

`tools/` 只维护具体环境入口、命令模板、YAML 模板和通用参考表，不维护长流程。

长流程放到：

```text
skills/<skill-name>/SKILL.md
```

长期原则、入口边界、安全红线、写操作确认放到根目录：

```text
MEMORY.md
```

根目录：

```text
TOOLS.md
```

只作为快速路由索引，负责把用户请求路由到对应 skill / tools 文件。

---

## 当前工具文件

| 文件                      | 内容                                                            | 典型使用场景                                              |
| ----------------------- | ------------------------------------------------------------- | --------------------------------------------------- |
| `environment-entry.md`  | 开发机、堡垒机、跳板机入口速查；复杂 SSH heredoc 通用规则                           | 入口不确定、需要登录开发机、需要复杂远程脚本                              |
| `rayctl-kubectl.md`     | kubeconfig、rayctl、kubectl、节点、任务、PVC / PV / AFS、ECS / AIS 命令模板 | vcjob 查询、节点查询、PVC 查询、rayctl 操作                      |
| `machine-types.md`      | 机器类型、芯片、vcluster 族、资源名、IP 段初筛映射                               | 判断 IP / vcluster / machine-type 是 MUXI、910B 还是 910C |
| `job-templates.md`      | `rayctl job create` 任务创建模板和资源范围                               | 创建训练 / 推理任务、生成 rayctl job create 命令                 |
| `dcluster-ansible.md`   | D 集群物理机 ansible 单 IP、JumpServer / 跳板机入口、转义规则                  | 查物理机目录、日志、mx-smi、磁盘、DNS、本地脚本                        |
| `mccl-commands.md`      | 已纳管 MUXI 维修后宿主机 MCCL 单机测试命令和日志检查                              | 维修后单节点 MCCL 验收                                      |
| `mccl-platform-yaml.md` | 新机器平台纳管前 MCCL YAML / vcjob / PodGroup 模板和检查命令                 | 新 MUXI 机器纳管前通过平台任务跑 MCCL                            |
| `k8s-cleanup.md`        | Kubernetes 清理类命令模板                                            | 批量删除 Pod、vcjob TTL patch、Aborted 清理                 |
| `hc-system-kubectl.md`  | host cluster 系统组件查询、VC 控制面定位、VC 控制面 reset、通过 prod-lepton / lepton_service 数据库调整 VC flavor 的命令模板 | 重启 VC 控制面、扩容 VC flavor、查询 HC 系统组件 |
| `fault-records.md`      | 运维故障 / 维修记录表查询                                                | Failed 任务疑似坏节点、维修记录对照                               |

---

## 文件职责边界

### `environment-entry.md`

只保存入口和远程执行通用规则。

适合：

* 怎么登录 D 集群开发机。
* 怎么区分开发机入口和物理机入口。
* 复杂远程命令如何用 heredoc。
* 入口失败时怎么停止。

不放：

* 具体 rayctl / kubectl 业务命令。
* 具体 ansible 排障命令。
* 长流程 SOP。

---

### `rayctl-kubectl.md`

保存 Kubernetes / vcluster / rayctl / kubectl 具体命令模板。

适合：

* `rayctl job get`
* `rayctl job get cluster`
* `rayctl node get / check / describe`
* vcluster 内 `vcjob` / Pod / PodGroup / Event / log 查询
* PVC / AFS / PV 查询与创建
* ECS / AIS 查询

不放：

* vcjob 排障流程。
* PVC 生命周期流程。
* 批量清理确认流程。
* 物理机 ansible 命令。

---

### `machine-types.md`

保存机器类型判断的通用参考信息。

适合：

* vcluster 族和芯片类型映射。
* machine-type 和芯片类型映射。
* MUXI / 910B / 910C 初筛规则。
* IP 段初筛映射。
* 节点资源名、label、allocatable 判断口径。

注意：

* 本文件只用于初筛。
* 最终结论必须用 rayctl 或 Kubernetes 实时查询验证。
* 不要把本文件当作实时库存。

---

### `job-templates.md`

保存 `rayctl job create` 的任务创建模板。

适合：

* 普通训练 / 推理任务创建。
* 单机 / 多机 rayctl job create 模板。
* vcluster 到模板族映射。
* 资源范围说明。

不适合：

* 新机器平台纳管 MCCL YAML。
* 已纳管 MUXI 维修后宿主机 MCCL。
* PVC 创建流程。

---

### `mccl-commands.md`

保存已纳管 MUXI 维修后，在宿主机上执行 MCCL 的底层命令。

适合：

* 检查 `~sensetime/mccl.sh`。
* 下发 `mccl.sh`。
* 五轮八卡 MCCL 测试。
* 查看 MCCL 日志。
* grep `Avg bus bandwidth` / `Out of bounds` / `rc=`。

不适合：

* 新机器平台纳管 YAML。
* 通过 vcjob / PodGroup 起多机平台任务。
* rayctl uncordon。

---

### `mccl-platform-yaml.md`

保存新机器平台纳管前 MCCL 验收相关 YAML / vcjob / PodGroup 模板。

适合：

* 新 MUXI 机器纳管前 MCCL 验收。
* 通过平台 YAML / vcjob / PodGroup 跑多机 MCCL。
* 平台任务创建后 Pod / Event / log 检查。
* 后续 Huawei / 910B / 910C 纳管 MCCL 扩展。

不适合：

* 已纳管 MUXI 维修后宿主机 `mccl.sh`。
* 物理机 ansible 单 IP 验收。
* `rayctl node uncordon`。

---

### `dcluster-ansible.md`

保存 D 集群物理机 ansible 单 IP 命令模板。

适合：

* 进入堡垒机 / 跳板机。
* 确认 `sensetime@host-10-140-220-33`。
* 单 IP ansible 查询。
* ansible 命令转义规则。
* 批量 ansible 的基础模板。

不适合：

* Kubernetes Node 查询。
* vcjob / Pod / PVC 查询。
* rayctl 操作。

---

### `k8s-cleanup.md`

保存 Kubernetes 清理命令模板。

适合：

* Failed Pod 预览和删除。
* 历史 vcjob TTL patch。
* TTL 后集合比对核验。
* Aborted 未回收检查。
* 直接 delete 兜底命令。

不适合：

* 清理流程确认机制。
* 批量操作策略决策。
* 是否允许删除的判断。

这些流程判断放在：

```text
skills/k8s-cleanup/SKILL.md
```

---

### `hc-system-kubectl.md`

保存 host cluster 系统组件操作命令模板。

适合：

* 确认 HC / VC kubeconfig 类型。
* 查询 HC namespace 下 VC 控制面 Pod、owner、replicas、selector。
* 精确 delete 控制面 Pod 触发重建。
* 通过 `prod-lepton` / `lepton_service` 数据库调整 VC flavor。
* 操作后复核。

不适合：

* HC 系统组件操作的流程判断。
* 什么时候允许 delete / SQL update。
* 缺少标识时是否继续扩大范围。

这些流程判断放在：

```text
skills/hc-system-op/SKILL.md
```

---

### `fault-records.md`

保存维修记录 / 故障记录查询方法。

适合：

* 根据节点 IP 查询维修记录。
* 对照任务失败时间和维修时间。
* 判断是否疑似坏节点、坏卡、坏端口。

不适合：

* vcjob Failed 排障完整流程。
* Pod 日志分析完整流程。

---

## 新增 tools 文件时的规则

新增 `tools/*.md` 前，先判断它是不是“命令 / 模板 / 参考表”。

适合放 tools：

* 命令模板。
* YAML 模板。
* jq / awk / ansible / kubectl / rayctl 片段。
* 静态参考表。
* 查询字段说明。
* 环境入口模板。

不适合放 tools：

* 多步骤 SOP。
* 判断分支。
* 停止条件。
* 写操作确认流程。
* 场景边界。
* 用户输出格式。

这些应该放到：

```text
skills/<skill-name>/SKILL.md
```

---

## tools 文件写法要求

* 文件开头说明用途和边界。
* 说明对应的 skill。
* 写操作命令必须标明“确认后执行”。
* 命令中用户输入必须放变量并加引号。
* grep 用户输入时使用：

```bash
grep -F -- "$VAR"
```

* 复杂命令不要写成超长一行。
* 复杂 SSH 远程命令优先用 `environment-entry.md` 的 heredoc 模板。
* 不要保存真实密码、token、AK/SK、cookie、Bearer token。
