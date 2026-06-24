# environment-entry.md - 环境入口与远程命令通用规则

本文件只保存进入远程环境和执行复杂远程命令的通用模板。

具体 Kubernetes / vcluster / rayctl / kubectl 命令见：

```text
tools/rayctl-kubectl.md
```

具体 D 集群物理机器 ansible 命令见：

```text
tools/dcluster-ansible.md
```

入口边界、写操作确认和安全红线以：

```text
MEMORY.md
```

为准。

---

## 1. Kubernetes / vcluster 操作入口

统一入口：

```text
本地 → D 集群开发机 → rayctl 优先 → kubectl 兜底
```

开发机：

```text
10.140.158.149:32222
```

登录：

```bash
ssh -p 32222 root@10.140.158.149
```

注意：

* 端口是 `32222`，不是 `22`。
* 出现 banner exchange timeout 最多重试 3 次。
* 重试 3 次仍失败，停止并说明原因。
* 不要从本地直接执行 D 集群 kubeconfig / rayctl / kubectl 操作。
* 不要在开发机执行 D 集群物理机器 ansible。
* 不要从开发机 SSH 到 D 集群目标物理节点。

适用场景：

* vcjob / Pod / PodGroup 查询。
* PVC / PV / AFS 查询或创建。
* vcluster 内资源查询。
* Kubernetes Node 查询。
* rayctl job / node / pvc / afs / ecs 查询。
* rayctl job create。
* rayctl node cordon / uncordon。

---

## 2. D 集群物理机器操作入口

统一入口：

```text
本地 → 堡垒机 → 跳板机 → sensetime 用户 → ansible 单 IP
```

机器关系：

| 角色 | 地址 | 用途 |
|---|---|---|
| 堡垒机 | `10.140.3.216:5906` | JumpServer 入口，用户 `test6`，密码 `6RV&QnwPrq` |
| 跳板机 | `10.140.220.33` | D 集群内部中转机，也可在 JumpServer 中输入 `220.33` |
| 开发机 | `10.140.158.149:32222` | 只用于 Kubernetes / rayctl / kubectl 操作 |

物理机器入口和 ansible 命令模板见：

```text
tools/dcluster-ansible.md
```

Ansible 单 IP 只能在以下环境执行：

```text
sensetime@host-10-140-220-33
```

标准形式：

```bash
ansible all -i '<目标IP>,' -m shell -a '<命令>'
```

适用场景：

* 查看物理机目录、文件、日志。
* 查看进程、磁盘、内存、CPU、NPU、网卡、路由、DNS。
* 查看 `mx-smi`。
* 检查或执行物理机本地脚本。
* 已纳管 MUXI 机器维修后宿主机 MCCL 验收。

禁止：

* 不要在本地 OpenClaw exec 环境直接执行 ansible。
* 不要在 D 集群开发机执行物理机器类 ansible。
* 不要从 D 集群开发机 SSH 到目标物理节点。
* 不要把 Kubernetes Node 查询误当成物理机 ansible 查询。

---

## 3. 入口选择规则

先判断对象类型，再选入口。

### 3.1 走 Kubernetes / vcluster 入口

用户问题涉及以下对象时，走 D 集群开发机：

```text
vcjob
Volcano Job
PodGroup
Pod
PVC
PV
AFS
Service
Endpoint
Webhook
Kubernetes Node
vcluster
rayctl job
rayctl node
rayctl pvc
rayctl afs
rayctl ecs
```

入口：

```text
本地 → D 集群开发机 → rayctl 优先 → kubectl 兜底
```

### 3.2 走物理机器入口

用户问题涉及以下对象时，走堡垒机 / 跳板机 / ansible 单 IP：

```text
物理 IP
hostname
机器目录
机器文件
机器日志
进程
磁盘
内存
CPU
NPU
mx-smi
网卡
路由
DNS
本地脚本
mccl.sh
```

入口：

```text
本地 → 堡垒机 → 跳板机 → sensetime 用户 → ansible 单 IP
```

### 3.3 不要混用入口

* 查 Pod / vcjob / PVC：不要走 ansible。
* 查物理机目录 / mx-smi / 本地日志：不要走开发机 rayctl。
* `rayctl node check` 是 Kubernetes / 平台视角。
* `ansible all -i '<IP>,' -m shell -a 'mx-smi'` 是物理机 OS 视角。
* 需要从一个入口切换到另一个入口时，必须说明原因。

---

## 4. 复杂远程命令用 heredoc

通过 SSH 到开发机执行复杂命令时，禁止生成超长单行命令。

包含以下内容时，优先使用 heredoc：

* `awk`
* `jq`
* 多层引号
* 管道
* 正则
* shell 变量
* `$NF`
* 多行脚本
* 循环
* 条件判断

模板：

```bash
cat <<'REMOTE' | ssh -p 32222 root@10.140.158.149 'bash -s'
set -euo pipefail

<远程脚本内容>
REMOTE
```

示例：

```bash
cat <<'REMOTE' | ssh -p 32222 root@10.140.158.149 'bash -s'
set -euo pipefail

export KUBECONFIG=/root/kubeconfig
rayctl cluster set d
rayctl node get -A | head
REMOTE
```

注意：

* heredoc 标记使用单引号：`<<'REMOTE'`。
* 这样可以避免本地 shell 提前展开 `$`、反引号和变量。
* 远程脚本里再设置 `set -euo pipefail`。
* 不要把复杂 `jq` / `awk` 命令压成一行远程 SSH 命令。

---

## 5. 只读和写操作

只读操作可以直接执行，但必须走正确入口。

只读示例：

```bash
kubectl get ...
kubectl describe ...
rayctl job get ...
rayctl node get ...
rayctl pvc check ...
rayctl afs check ...
ansible all -i '<IP>,' -m shell -a 'hostname'
ansible all -i '<IP>,' -m shell -a 'mx-smi'
```

写操作必须遵守 `MEMORY.md` 的写操作确认规则。

写操作示例：

```bash
kubectl delete ...
kubectl patch ...
kubectl apply ...
kubectl label ...
kubectl annotate ...
rayctl job create ...
rayctl pvc create ...
rayctl node cordon ...
rayctl node uncordon ...
ansible all -i '<IP>,' -m shell -a 'sudo bash ...'
```

创建临时脚本、修改文件、删除文件、重启服务、执行影响业务的脚本，也属于写操作。


---

## 6. 失败处理

### 6.1 开发机登录失败

如果登录开发机出现 banner exchange timeout：

1. 最多重试 3 次。
2. 仍失败则停止。
3. 返回失败原因。
4. 不要改用本地执行 kubeconfig / rayctl / kubectl。
5. 不要改走物理机入口。

### 6.2 跳板机入口失败

如果堡垒机 / JumpServer / 跳板机登录失败：

1. 停止。
2. 返回卡在哪一步。
3. 不要改从开发机 SSH 到目标物理机。
4. 不要在本地直接 ansible 目标 IP。

### 6.3 kubeconfig 不确定

如果不确定当前 kubeconfig 是 host cluster 还是 vcluster：

1. 停止。
2. 先确认 kubeconfig 路径。
3. 不要继续执行 kubectl。
4. 不要用 host cluster kubeconfig 查询 vcluster 内 vcjob / PodGroup / Pod。

---

## 7. 本文件不负责的内容

本文件不保存以下内容：

* rayctl / kubectl 具体业务命令。
* PVC / AFS / PV 具体命令。
* vcjob / PodGroup / Pending 排障流程。
* MCCL 脚本内容。
* Kubernetes 清理脚本。
* 物理机 ansible 详细命令。
* 长期安全红线。
* 每日动态记录。

对应文件：

```text
tools/rayctl-kubectl.md
tools/dcluster-ansible.md
tools/mccl-commands.md
tools/k8s-cleanup.md
skills/vcjob-debug/SKILL.md
skills/pvc-afs/SKILL.md
skills/mccl-test/SKILL.md
skills/k8s-cleanup/SKILL.md
MEMORY.md
memory/YYYY-MM-DD.md
```



