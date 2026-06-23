# k8s-cleanup Skill

## 触发条件

用户询问以下任意问题时使用本 skill：

- 批量删除 Pod 或历史 vcjob。
- 清理 Failed / Succeeded / Evicted 资源。
- 给历史 vcjob 设置 `ttlSecondsAfterFinished`。
- 按时间窗口清理 Kubernetes 资源。
- TTL 后核验 vcjob / Pod，或处理 TTL 未回收的 Aborted 任务。

## 相关工具文件

- `tools/k8s-cleanup.md`
- `tools/rayctl-kubectl.md`
- `tools/environment-entry.md`

## 前置原则

- 必须先确认目标 vcluster、真实 kubeconfig、namespace、资源类型和时间窗口。
- vcluster 内 Pod / vcjob 禁止用 host cluster kubeconfig 查询或清理。
- 批量 delete 和批量 TTL patch 都是高风险写操作，必须分阶段执行。
- 第一步永远是只读筛选、计数、状态分布和样例展示。
- 默认排除 `Running` / `Pending`；处理它们必须另行明确确认。
- 用户按北京时间指定日期时，必须先换算成 UTC 窗口再匹配 `creationTimestamp`。
- TTL patch 可能使历史任务立即被删除，风险级别按批量删除处理。
- 用户确认前只能生成清单、快照和预览命令，不能 patch / delete。

## 标准流程

### 1. 明确范围

必须明确：

- vcluster 与 `/root/D/<实际 kubeconfig 文件名>`。
- namespace 或全部 namespace。
- 资源类型和状态条件。
- 北京时间窗口及换算后的 UTC 窗口。
- 是否排除 `Running` / `Pending`。
- 清理方式：TTL patch 或直接 delete。
- 是否核验关联 Pod。

### 2. 只读生成清单

命令以 `tools/k8s-cleanup.md` 为准。vcjob 清单至少包含：

```text
namespace / name / phase / creationTimestamp / 当前 TTL
```

必须展示：总数、phase 分布、TTL 分布、同一窗口内 Running/Pending 数量和样例。

如果清单包含 `Running` / `Pending`，立即停止。

### 3. 保存执行前证据

保存待处理 TSV、筛选时的 vcjob JSON 快照和候选对象 JSON 备份。
备份主要用于审计与定位，不保证能直接恢复删除后的对象。

### 4. 展示写操作并等待确认

必须说明：

- 精确数量、kubeconfig、namespace、UTC 时间窗口和 phase。
- TTL 对历史任务可能立即生效。
- API 请求量、apiserver 压力和预计执行时间。
- `Ctrl-C` 等停止方式。
- 删除后不能直接回滚。

### 5. 确认后实时复核

执行前重新拉取快照并重新生成清单，确认：

- 数量仍与用户确认的一致。
- `Running/Pending` 为 0。
- phase 仍属于确认过的终态。
- TTL 分布未发生意外变化。

数量或状态变化时停止并重新确认。

### 6. 执行 TTL patch

- 默认串行执行。
- 写入 patch 日志并定期输出成功/失败计数。
- 数量较大时使用 tmux 或可续查后台脚本，记录清单和日志路径。
- SSH 断开后不要从头重跑；先查日志和实时剩余对象，只补未完成项。

### 7. TTL 后核验

等待 controller 处理后，必须拿原始 namespace/name 清单做集合比对：

- 原目标 vcjob 还剩多少。
- 剩余对象的 phase、TTL、finalizer、deletionTimestamp。
- 与原目标 job 关联的 Pod 还剩多少。
- patch 失败对象是否仍存在。

不能只重新按时间和状态筛选，因为对象被删除后筛选结果会自然减少。

### 8. Aborted 特殊处理

部分 `Aborted` vcjob 只有状态转换时间，没有 TTL controller 可识别的 `completionTime` / `finishedAt`。
即使 TTL 已设置，这类对象也可能长期不产生 deletionTimestamp。

处理步骤：

1. 确认剩余对象全部是 `Aborted`。
2. 确认 TTL 已设置并等待后仍未删除。
3. 检查 completion / finished 字段和 controller 日志。
4. 保存剩余 Aborted 清单。
5. 不要反复 patch TTL。
6. 如需直接 delete，必须重新展示数量、样例和命令，并获得新的明确确认。

### 9. 直接 delete 兜底

直接 delete 是新的写操作，不能沿用“只确认 TTL patch”的授权。
确认后默认串行并使用 `--wait=false --ignore-not-found`，完成后再次按原始清单核验 vcjob 和 Pod。

## 输出格式

```text
目标范围：vcluster / kubeconfig / namespace / 北京时间与 UTC 窗口 / phase
执行前：总数 / phase 分布 / Running-Pending / TTL 分布
执行结果：成功 / 失败 / 剩余 vcjob / 剩余关联 Pod
特殊剩余：Aborted 是否缺少完成时间
下一步：是否需要直接 delete 并重新确认
```

## 禁止事项

- 不要省略预览和实时复核。
- 不要处理 `Running` / `Pending`，除非用户明确要求且再次确认。
- 不要用 host cluster kubeconfig 清理 vcluster 内资源。
- 不要把 `.status.state` 当 phase；正确字段是 `.status.state.phase`。
- 不要默认高并发。
- 不要把 TTL patch 描述成无风险“打标”。
- 不要自动把 TTL 无法处理的 Aborted 对象改成直接 delete。
- 不要在 SSH 中断后直接从头重跑。
