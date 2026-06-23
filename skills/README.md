# skills/README.md

`skills/` 只放专项 SOP：什么时候触发、按什么顺序查、什么场景禁止做什么。

具体命令不要在 skill 里重复维护太多，优先引用 `tools/*.md`：

| Skill | 触发场景 | 主要依赖工具文件 |
|---|---|---|
| `vcjob-debug` | vcjob / Volcano / PodGroup / Pending / NotEnoughResources | `tools/rayctl-kubectl.md` |
| `pvc-afs` | AFS / PVC / PV 查询与创建 | `tools/rayctl-kubectl.md` |
| `mccl-test` | 单机 MCCL 验收、多机 MCCL 纳管测试 | `tools/dcluster-ansible.md`、`tools/mccl-commands.md`、`tools/job-templates.md` |
| `k8s-cleanup` | 批量删除 Pod / vcjob / 历史资源 | `tools/k8s-cleanup.md` |
| `dcluster-machine-op` | D 集群物理机目录、日志、进程、磁盘、NPU、DNS | `tools/dcluster-ansible.md` |
| `update-openclaw-memory` | 更新 OpenClaw memory / tools / skills / 知识库 | `TOOLS.md`、`git pull --ff-only` |

优先级：

1. 当前用户明确指令。
2. `MEMORY.md` 的安全红线和入口路由。
3. 当前 skill 的流程。
4. `tools/*.md` 的具体命令模板。
5. `memory/YYYY-MM-DD.md` 的动态记录。

如果 skill 与 `MEMORY.md` 冲突，以 `MEMORY.md` 为准。
如果 skill 与 `tools/*.md` 命令参数冲突，以 `tools/*.md` 为准。
