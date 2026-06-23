# environment-entry.md - 环境入口与远程命令通用规则

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

- 端口是 `32222`，不是 `22`。
- 出现 banner exchange timeout 最多重试 3 次。
- 重试 3 次仍失败，停止并说明原因。
- 不要从本地直接执行 D 集群 kubeconfig / rayctl 操作。
- 不要在开发机执行 D 集群物理机器 ansible。
- 不要从开发机 SSH 到 D 集群目标物理节点。

---

## 2. D 集群机器操作入口

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

Ansible 单 IP 只能在以下环境执行：

```text
sensetime@host-10-140-220-33
```

标准形式：

```bash
ansible all -i '<目标IP>,' -m shell -a '<命令>'
```

---

## 3. 复杂远程命令用 heredoc

通过 SSH 到开发机执行复杂命令时，禁止生成超长单行命令。
包含 `awk`、`jq`、多层引号、管道、正则、变量 `$NF` 等时，使用 heredoc：

```bash
cat <<'REMOTE' | ssh -p 32222 root@10.140.158.149 'bash -s'
set -euo pipefail
<远程脚本内容>
REMOTE
```

---

## 4. 只读 / 写操作区分

只读命令可以直接执行，例如：

```bash
kubectl get ...
kubectl describe ...
rayctl job get ...
rayctl node get ...
ansible all -i '<IP>,' -m shell -a 'hostname'
```

写操作必须先确认，例如：

```bash
kubectl delete ...
kubectl patch ...
kubectl apply ...
kubectl label ...
rayctl job create ...
rayctl pvc create ...
rayctl node cordon ...
rayctl node uncordon ...
ansible all -i '<IP>,' -m shell -a 'sudo bash ...'
```
