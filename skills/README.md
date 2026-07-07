# skills/README.md

`skills/` 只放专项 SOP：什么时候触发、按什么顺序查、如何分支、什么场景禁止做什么、什么时候停止。

具体命令不要在 skill 里重复维护，优先引用：

```text
tools/*.md
```

长期原则、安全红线、入口边界、写操作确认以根目录：

```text
MEMORY.md
```

为准。

根目录：

```text
TOOLS.md
```

负责快速路由，把用户请求映射到 skill 和 tools 文件。

---

## 当前 Skill

| Skill                    | 触发场景                                                                                      | 主要依赖工具文件                                                                                                         |
| ------------------------ | ----------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| `vcjob-debug`            | vcjob / Volcano Job / PodGroup / Pending / NotEnoughResources / Failed 单任务排障；VC / 分区级任务概览 | `tools/rayctl-kubectl.md`；疑似坏节点时读 `tools/fault-records.md`                                                       |
| `pvc-afs`                | AFS / PVC / PV 查询与创建；PVC Pending；任务挂载 PVC 失败                                              | `tools/rayctl-kubectl.md`                                                                                        |
| `job-create`             | 创建训练 / 推理任务；生成 `rayctl job create` 命令；重跑任务                                                | `tools/job-templates.md`、`tools/rayctl-kubectl.md`；需要 PVC 时读 `skills/pvc-afs/SKILL.md`                           |
| `mccl-test`              | 已纳管 MUXI 维修后 MCCL 验收；MCCL 通过后放回；新机器平台纳管前 MCCL 验收                                          | `tools/dcluster-ansible.md`、`tools/mccl-commands.md`、`tools/mccl-platform-yaml.md`、必要时 `tools/rayctl-kubectl.md` |
| `k8s-cleanup`            | 批量删除 Pod / vcjob / Failed 资源；历史 vcjob TTL 回收；Aborted 未回收处理                                | `tools/k8s-cleanup.md`、必要时 `tools/rayctl-kubectl.md`                                                             |
| `dcluster-machine-op`    | D 集群物理机目录、日志、进程、磁盘、NPU、DNS、宿主机训练网互通、本地脚本                                            | `tools/dcluster-ansible.md`；必要时读 `tools/rayctl-kubectl.md` 找候选宿主机                                             |
| `image-build-push`      | 堡垒机内镜像制作 / 打标 / 打驱动 / push；MUXI 镜像加 driver；非 MUXI 镜像推送到指定 ccr | `tools/image-build-push.md`、`tools/environment-entry.md` |
| `hc-system-op`           | HC 上平台 / VC 控制面系统组件操作；重启 / 重置 VC 控制面；通过平台数据库调整 VC flavor；更新 `disallow-privileged-containers` policy；查询 HC 系统组件状态；给 VC / Subnet / AFS / CCR / AIS 授权或移除授权 | `tools/hc-system-kubectl.md`、`tools/rayctl-kubectl.md` |

---

## Skill 职责边界

### `vcjob-debug`

用于 Kubernetes / vcluster 任务排障。

典型问题：

* 单个 vcjob Pending。
* PodGroup 异常。
* NotEnoughResources。
* Failed 任务日志分析。
* Failed 任务疑似坏节点。
* 某个 VC / 分区 Pending 分布。

不处理：

* 创建任务。
* 创建 PVC。
* 批量清理历史资源。
* 物理机目录 / 日志 / mx-smi 查询。
* MCCL 验收流程。

---

### `pvc-afs`

用于 AFS / PVC / PV 生命周期。

典型问题：

* 查询 AFS UID / secretName / Host PV。
* 查询 PVC 是否 Bound。
* 创建 PVC。
* PVC Pending。
* 任务挂载 PVC 失败。

不处理：

* 创建训练 / 推理任务本身。
* vcjob Pending 根因完整排障。
* 删除 PVC / PV 的高风险清理。

---

### `job-create`

用于普通训练 / 推理任务创建。

典型问题：

* 生成 `rayctl job create` 命令。
* 根据 vcluster / 机器类型选择模板。
* 创建单机 / 多机训练任务。
* 创建任务前检查 PVC。
* 重跑任务。

不处理：

* 新机器平台纳管 MCCL YAML。
* 已纳管 MUXI 维修后宿主机 MCCL。
* vcjob Pending / Failed 深度排障。

这些分别走：

```text
skills/mccl-test/SKILL.md
skills/vcjob-debug/SKILL.md
```

---

### `mccl-test`

用于 MCCL 验收和 MCCL 结果判断。

必须先区分两个互斥场景：

| 场景               | 入口                                                      | 说明                                 |
| ---------------- | ------------------------------------------------------- | ---------------------------------- |
| 已纳管 MUXI 机器维修后验收 | 堡垒机 → 跳板机 → sensetime → ansible 单 IP                    | 宿主机跑 `mccl.sh`，通过后才可按确认执行 uncordon |
| 新机器平台纳管前验收       | D 集群开发机 → vcluster kubeconfig → YAML / vcjob / PodGroup | 通过平台任务跑 MCCL，不走宿主机单 IP 验收          |

禁止混用：

* 不要把已纳管维修后验收做成 vcjob / YAML。
* 不要用宿主机 `mccl.sh` 代替新机器平台纳管验收。
* 不要为了单机维修验收先 uncordon。

---

### `k8s-cleanup`

用于 Kubernetes 批量清理和历史资源回收。

典型问题：

* 删除 Failed Pod。
* 清理历史 vcjob。
* 给历史 vcjob patch `ttlSecondsAfterFinished`。
* TTL 后核验是否回收。
* Aborted 未被 TTL 回收后的处理。

必须遵守：

* 先只读预览。
* 展示数量、状态分布、样例。
* 写操作前确认。
* 执行前实时复核。
* 删除后按原始清单核验。

---

### `dcluster-machine-op`

用于 D 集群物理机器 / Linux 系统级操作。

典型问题：

* 查看某台物理机目录。
* 查看日志。
* 查看 `mx-smi`。
* 查看磁盘、内存、进程、网卡、DNS。
* 检查两台宿主机训练网 / 数据网 IP 是否互通。
* 执行本地脚本。

不处理：

* Kubernetes Node 查询。
* 查询节点属于哪个 vcluster。
* 查询节点上有哪些 Pod / 任务。
* cordon / uncordon。

这些走：

```text
tools/rayctl-kubectl.md
```

或相关 Kubernetes skill。

---

### `image-build-push`

用于堡垒机内镜像制作和推送。

典型问题：

* MUXI 镜像 `docker pull` 后重新 tag、打驱动、push。
* `wget` 镜像包后 `docker load`、打驱动、push。
* 非 MUXI 镜像 tag 到用户指定 ccr 并 push。

不处理：

* Kubernetes / vcluster / rayctl / kubectl 查询与写操作。
* 物理机 ansible 单 IP 查询。
* HC 控制面重置、VC flavor、HC policy 更新。

必须遵守：

* 走堡垒机内 `3.216` 入口。
* MUXI 和非 MUXI 路径必须先区分。
* 非 MUXI 目标 ccr 必须由用户明确指定。
* `wget` 固定在 `/data/xie` 下执行。

---

### `hc-system-op`

用于 host cluster 下的平台 / VC 控制面系统组件操作。

典型问题：

* 重启 / 重置某个 VC 控制面 Pod。
* 查询某个 VC 控制面组件状态。
* 调整某个 VC 的 flavor。
* 更新 `disallow-privileged-containers` policy，把某个 VC 加入例外。

不处理：

* VCJob / PodGroup / Pending / Failed 排障。
* 批量清理历史资源。
* PVC / AFS / PV 生命周期。
* 物理机目录 / 日志 / NPU。

必须遵守：

* VC 内业务现象用 VC kubeconfig 验证。
* HC 系统组件定位、delete、数据库相关操作用 HC kubeconfig。
* HC policy 查询优先走 `rayctl policy get disallow-privileged-containers`，更新优先走 `rayctl policy update disallow-privileged-containers`，HC `kubectl` 只做只读兜底和人工兜底。
* 写操作前必须只读验证、展示影响范围和风险，并等待确认。

---

## Skill 与 tools 的关系

Skill 负责：

* 什么时候触发。
* 按什么顺序查。
* 如何分支。
* 何时停止。
* 何时需要确认。
* 输出哪些证据。
* 哪些行为禁止。

Tools 负责：

* 具体命令。
* YAML 模板。
* jq / awk / ansible / kubectl / rayctl 片段。
* 静态参考表。
* 环境入口模板。

不要在 skill 中维护大段命令。
不要在 tools 中维护长流程判断。

---

## 优先级

1. 当前用户明确指令。
2. `MEMORY.md` 的安全红线、入口路由、写操作确认。
3. 当前 skill 的流程和停止条件。
4. `tools/*.md` 的具体命令模板。
5. `memory/YYYY-MM-DD.md` 的动态记录。

如果 skill 与 `MEMORY.md` 冲突，以 `MEMORY.md` 为准。

如果 skill 与 `tools/*.md` 命令参数冲突，以 `tools/*.md` 为准。

如果动态记录与实时查询冲突，以实时查询为准。

---

## 新增 Skill 的规则

新增 skill 前先判断：

* 是否是一个明确场景。
* 是否有多步流程。
* 是否有分支判断。
* 是否有停止条件。
* 是否涉及写操作确认。
* 是否容易和其他场景混淆。

如果只是命令模板或参考表，不要新增 skill，放到：

```text
tools/*.md
```

推荐 skill 结构：

```markdown
# <skill-name> Skill

## 触发条件

## 相关工具文件

## 前置原则

## 标准流程

### 1. 判断问题类型

### 2. 只读定位

### 3. 分支判断

### 4. 写操作确认

### 5. 执行后检查

### 6. 停止条件

## 输出要求

## 禁止事项
```

通用输出偏好不要在每个 skill 中重复，交给：

```text
MEMORY.md
```

只保留本场景特有的输出结构。
