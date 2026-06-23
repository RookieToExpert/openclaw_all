# dcluster-machine-op Skill

## 触发条件

用户询问以下任意问题时使用本 skill：

- 查看某台 D 集群物理机目录、文件、日志。
- 查看进程、磁盘、内存、CPU、NPU、网卡、路由、DNS。
- 在某台物理机上执行本地脚本。
- 检查 `~sensetime`、`mccl.sh`、本地服务状态。

## 相关工具文件

- `tools/dcluster-ansible.md`
- `tools/environment-entry.md`

## 前置原则

- 这是物理机器 / 操作系统级操作，不是 Kubernetes Node 查询。
- 入口必须是：本地 → 堡垒机 → 跳板机 → sensetime 用户 → ansible 单 IP。
- Ansible 单 IP 只能在跳板机 sensetime 用户下执行。
- 不要在本地 OpenClaw exec 环境直接执行 ansible。
- 不要在开发机执行 D 集群机器类 ansible。
- 只读查询可以直接执行。
- 写操作、脚本执行、重启、改配置、删除文件必须先确认。

## 标准流程

### 1. 判断是否物理机器操作

如果用户说：

- “看这个 IP 的目录”。
- “看这台机器的日志”。
- “看 mx-smi / 磁盘 / 内存 / 网卡 / DNS”。
- “跑机器上的脚本”。

全部走本 skill。

如果用户说：

- “这个节点属于哪个 vc”。
- “cordon / uncordon node”。
- “看 Pod 调度到哪个节点”。

这是 Kubernetes / rayctl 场景，不走本 skill。

### 2. 进入跳板机 sensetime

手动流程见 `tools/dcluster-ansible.md 的“手动进入跳板机流程”`。
expect 模板见 `tools/dcluster-ansible.md 的 expect 模板`。

### 3. 只读查询模板

```bash
ansible all -i '<目标IP>,' -m shell -a 'hostname && date'
ansible all -i '<目标IP>,' -m shell -a 'ls -la ~sensetime/'
ansible all -i '<目标IP>,' -m shell -a 'df -h'
ansible all -i '<目标IP>,' -m shell -a 'free -h'
ansible all -i '<目标IP>,' -m shell -a 'mx-smi'
ansible all -i '<目标IP>,' -m shell -a 'ip addr'
ansible all -i '<目标IP>,' -m shell -a 'ip route'
ansible all -i '<目标IP>,' -m shell -a 'cat /etc/resolv.conf'
ansible all -i '<目标IP>,' -m shell -a 'ps -ef | head -50'
ansible all -i '<目标IP>,' -m shell -a 'tail -n 100 <log-path>'
```

### 4. 写操作确认模板

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

确认后再执行：

```bash
ansible all -i '<目标IP>,' -m shell -a '<写操作命令>'
```

## 输出格式

```text
结论：
证据：
- hostname/date ...
- 命令输出 ...
判断：
下一步：
```

## 禁止事项

- 不要把物理机器查询发到开发机。
- 不要从开发机 SSH 到目标物理节点。
- 不要使用 inventory 作为单机查询默认方式。
- 不要在没有确认当前 shell 是跳板机 sensetime 用户时执行 ansible。
- 不要把机器脚本执行当成只读操作。
