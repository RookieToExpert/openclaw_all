# skills/dcluster-machine-op/SKILL.md - D 集群物理机器操作

## 触发条件

用于 D 集群物理机器 / Linux 系统级查询和操作，包括：

- 查看某台 D 集群物理机目录、文件、日志。
- 查看进程、磁盘、内存、CPU、NPU、网卡、路由、DNS。
- 检查两台物理机宿主机层面的训练网 / 数据网 IP 是否互通。
- 检查 `~sensetime`、`mccl.sh`、本地服务状态。
- 在某台物理机上执行本地脚本。

## 相关工具文件

- `tools/dcluster-ansible.md`
- `tools/environment-entry.md`

---

## 前置原则

- 这是物理机器 / 操作系统级操作，不是 Kubernetes Node 查询。
- 入口必须是：本地 → 堡垒机 → 跳板机 → sensetime 用户 → ansible 单 IP。
- Ansible 单 IP 只能在跳板机 sensetime 用户下执行。
- 不要在本地 OpenClaw exec 环境直接执行 ansible。
- 不要在 D 集群开发机执行物理机器类 ansible。
- 只读查询可以执行。
- 写操作、脚本执行、重启、改配置、删除文件必须先确认。

---

## 标准流程

### 1. 判断是否物理机器操作

以下请求走本 skill：

- 看某个 IP / hostname 的目录、文件、日志。
- 看 mx-smi / 磁盘 / 内存 / 网卡 / DNS。
- 看宿主机训练网 / 数据网网卡 IP，或让两台物理机互 ping 某类网卡 IP。
- 看进程、本地服务、本地脚本。
- 跑机器上的脚本。

以下请求不走本 skill：

- 查询节点属于哪个 vcluster。
- 查询节点上有哪些 Pod / 任务。
- cordon / uncordon Kubernetes node。
- 查看 Pod 调度到哪个节点。

这些属于 Kubernetes / rayctl 场景，应走 `tools/rayctl-kubectl.md` 或对应 skill。

例外：如果用户要检查“某个分区 / superpod 的宿主机训练网是否互通”，可以先用 `tools/rayctl-kubectl.md` 找到候选宿主管理 IP；拿到物理机 IP 后，网卡查询和互通测试仍回到本 skill。

---

### 2. 进入跳板机 sensetime 用户

手动进入方式见：

- `tools/dcluster-ansible.md` → 1. 手动进入跳板机流程

自动进入模板见：

- `tools/dcluster-ansible.md` → 2. expect 自动进入跳板机模板

执行 ansible 前必须确认当前 shell 是跳板机 sensetime 用户。

---

### 3. 只读查询

常见只读查询命令见：

- `tools/dcluster-ansible.md` → 3. 只读单机查询模板

只读查询包括：

- hostname / date。
- 目录。
- 磁盘。
- 内存。
- mx-smi。
- 网卡。
- 路由。
- DNS。
- 进程。
- 日志 tail。

---

### 3.1 宿主机训练网 / 数据网互通检查

适用于快速判断两台物理机在宿主机网络层是否能互通，不等价于 MCCL / RDMA 验收。

流程：

1. 如果用户只给分区、superpod 或 vcluster，先通过 `tools/rayctl-kubectl.md` 找到 2 台候选宿主管理 IP。
2. 进入跳板机 sensetime 用户，只对这 2 台物理机做 ansible 单 IP 只读查询。
3. 分别查询两台机器的全局 IPv4 网卡，优先识别训练网 / 数据网候选 IP。
4. 对同网段候选 IP 做 source-bound ping，至少覆盖 A → B；需要排除单向 ACL / 路由异常时再做 B → A。
5. 输出每一组源 IP、目标 IP、网卡、丢包率和结论。

命令模板见：

- `tools/dcluster-ansible.md` → 3.1 宿主机训练网 / 数据网只读互通检查模板

停止条件：

- 找不到两台候选物理机时停止。
- 查询不到训练网 / 数据网候选 IP 时停止。
- 同网段候选 IP 全部 ping 失败时，只能说明宿主机 ICMP / L3 检查失败，不直接下 MCCL / RDMA 结论。
- 不自动扩大到更多节点，除非用户确认扩大范围。

---

### 4. 写操作

写操作包括但不限于：

- 修改文件。
- 删除文件。
- 创建临时脚本。
- 执行影响业务的脚本。
- 重启服务。
- 重启机器。
- 批量 ansible 写操作。

执行前必须向用户展示：

```text
将要执行：
目标机器：
入口位置：跳板机 sensetime 用户 ansible 单 IP
影响范围：
风险：
回滚 / 停止方式：
请确认是否执行。
``` 

确认后再按 `tools/dcluster-ansible.md` 执行。

复杂命令、长命令、多行命令优先写临时脚本；创建临时脚本属于写操作。

---

## 输出要求

默认输出偏好以 `MEMORY.md` 为准。

物理机查询必须说明：

* 目标 IP / hostname。
* 执行入口是否为跳板机 sensetime 用户。
* 已执行的只读命令。
* 关键结果。
* 是否需要进一步写操作确认。

---

## 禁止事项

* 不要把物理机器查询发到 D 集群开发机。
* 不要从开发机 SSH 到目标物理节点。
* 不要使用 inventory 作为单机查询默认方式。
* 不要在没有确认当前 shell 是跳板机 sensetime 用户时执行 ansible。
* 不要把机器脚本执行当成只读操作。
