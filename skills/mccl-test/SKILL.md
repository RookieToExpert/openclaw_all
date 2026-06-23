# mccl-test Skill

## 触发条件

用户询问以下任意问题时使用本 skill：

- MCCL 测试。
- MUXI 单机维修验收。
- 多机 MCCL / 平台纳管 MCCL。
- 坏节点排除。
- `Avg bus bandwidth`、`Out of bounds`、MCCL timeout / failed / hang。

## 必须先区分场景

MCCL 测试分两类，不能混用流程。

| 场景 | 入口 | 是否创建 vcjob | 是否打 label | 是否 uncordon |
|---|---|---:|---:|---:|
| 单机 MUXI MCCL 维修验收 | 堡垒机 → 跳板机 → sensetime → ansible 单 IP | 否 | 否 | 否，除非用户明确要求通过后 uncordon |
| 多机 / 平台纳管 MCCL | 开发机 → rayctl / kubectl | 可能 | 可能 | 按测试方案，写操作必须确认 |

---

## 单机 MUXI MCCL 维修验收流程

### 1. 前置确认

只读检查可以直接执行；跑测试前必须确认。

确认内容：

- 目标 IP / hostname。
- 测试卡数，默认 8 卡。
- 测试轮数，默认 5 轮。
- 输出日志路径，默认 `~sensetime/mccl-*.log`。
- 不改变节点 cordon / 维修状态。

### 2. 进入正确入口

必须在跳板机 sensetime 用户下执行：

```bash
ansible all -i '<目标IP>,' -m shell -a '<命令>'
```

### 3. 只读检查

```bash
ansible all -i '<目标IP>,' -m shell -a 'hostname && date'
ansible all -i '<目标IP>,' -m shell -a 'mx-smi'
ansible all -i '<目标IP>,' -m shell -a 'test -f ~sensetime/mccl.sh && ls -l ~sensetime/mccl.sh || echo MISSING'
```

### 4. 如果脚本不存在

写入 `~sensetime/mccl.sh` 属于写操作，必须先确认。
脚本内容以 `tools/mccl-commands.md` 的“标准单机 mccl.sh” 为准。

### 5. 执行五轮八卡测试

确认后执行：

```bash
ansible all -i '<目标IP>,' -m shell -a 'cd ~sensetime && LOG=mccl-$(date +%Y%m%d-%H%M%S).log; for i in 1 2 3 4 5; do echo "===== MCCL RUN ${i} =====" | tee -a "$LOG"; sudo bash ./mccl.sh 8 2>&1 | tee -a "$LOG"; echo "rc=${PIPESTATUS[0]}" | tee -a "$LOG"; done; echo LOG=$LOG'
```

### 6. 查看结果

```bash
ansible all -i '<目标IP>,' -m shell -a 'cd ~sensetime && ls -lt mccl-*.log | head'
ansible all -i '<目标IP>,' -m shell -a "cd ~sensetime && grep -E 'Avg bus bandwidth|Out of bounds|error|failed|timeout|rc=' <mccl-log-file> | tail -200"
```

### 7. 判断标准

重点看：

- 每轮每个 benchmark 的 `# Avg bus bandwidth`。
- `# Out of bounds values : 0 OK`。
- `rc=0`。
- 是否出现 error / failed / timeout / hang。

输出结论时保留原始关键行，不擅自改写原始结果。

---

## 多机 / 平台纳管 MCCL 流程

### 1. 前置确认

必须明确：

- 目标 vcluster。
- namespace。
- 目标节点范围。
- 是否需要打 label。
- 是否需要创建 vcjob。
- worker replicas / minAvailable / CARD_NUM。
- 测试完成后是否删除 label / 任务。

所有打标、删标、创建任务、删除任务都是写操作，必须先确认。

### 2. 节点准备

先只读查询：

```bash
export KUBECONFIG=/root/kubeconfig
rayctl node get -A <ip-fragment-or-selector>
rayctl node check <node-name-or-ip>
```

### 3. 创建测试任务

任务模板和命令以 `tools/job-templates.md` 的 rayctl 任务模板为准。
优先使用 `--what-if` 预览，不直接提交。

```bash
export KUBECONFIG=/root/D/<实际 kubeconfig 文件名>
rayctl job create <template-name> ... --what-if
```

### 4. 坏节点排除

如果发现坏节点，可以动态排除，但必须同步调整：

- `minAvailable`。
- worker replicas。
- `CARD_NUM`。
- PodGroup / vcjob 模板中的节点数。

### 5. 输出格式

```text
场景：单机验收 / 多机平台纳管
结论：
证据：
- 原始 Avg bus bandwidth 行 ...
- rc=0 / 非 0 ...
- Out of bounds ...
风险 / 影响：
下一步：
```

## 禁止事项

- 不要把单机维修验收做成 vcjob。
- 不要为了单机测试打 label / 删 label。
- 不要为了单机测试先 uncordon。
- 不要在本地或开发机直接执行 D 集群机器 ansible。
- 不要擅自汇总用户要求保留原始输出的 MCCL 结果。
