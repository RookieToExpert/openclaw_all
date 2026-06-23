# tools/README.md

`tools/` 只维护具体环境入口和命令模板，不维护长流程。

长流程放到 `skills/<skill-name>/SKILL.md`。
长期原则和安全红线放到根目录 `MEMORY.md`。
根目录 `TOOLS.md` 只是工具索引。

## 当前工具文件

| 文件 | 内容 |
|---|---|
| `environment-entry.md` | 开发机、堡垒机、跳板机、远程命令通用规则 |
| `rayctl-kubectl.md` | kubeconfig、rayctl、kubectl、节点、PVC/PV/AFS、ECS/AIS |
| `job-templates.md` | rayctl 任务创建模板 |
| `dcluster-ansible.md` | D 集群物理机 ansible 单 IP 和 JumpServer 入口 |
| `mccl-commands.md` | MCCL 单机测试命令和日志检查 |
| `k8s-cleanup.md` | Kubernetes 清理类命令模板 |
| `fault-records.md` | 运维故障记录表 |
