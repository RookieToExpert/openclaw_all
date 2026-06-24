# tools/job-templates.md - rayctl 任务模板

本文件只保存 `rayctl job create` 命令模板和资源范围。
创建流程、确认机制和停止条件见 `skills/job-create/SKILL.md`。
入口边界、写操作确认和安全红线以 `MEMORY.md` 为准。

不适用场景：

* 新机器平台纳管 MCCL YAML 验收。
* 已纳管 MUXI 维修后宿主机 MCCL 验收。
* PVC 创建流程。

---

## 1. vcluster 到模板映射

| vcluster      | 机器类型           | 芯片 / 类型  | 单机模板                   | 多机模板                  |
| ------------- | -------------- | -------- | ---------------------- | --------------------- |
| `a2-*`        | `h1ls.rp.k60a` | 华为 910B  | `910b-single`          | `910b-multi`          |
| `ai4chem`     | `h1ls.rp.k60a` | 华为 910B  | `910b-single`          | `910b-multi`          |
| `a3-*`        | `h2ls.ru.k10`  | 华为 910C  | `910c-single`          | `910c-multi`          |
| `c550-jiaofu` | `x2ls.ri.i80`  | MUXI 风冷  | `c550-default-single`  | `c550-default-multi`  |
| `c550-ai4s`   | `x2ls.ri.i80`  | MUXI 风冷  | `c550-default-single`  | `c550-default-multi`  |
| `c550-h3c`    | `x2ls.ri.i70`  | MUXI 液冷  | `c550-h3c-single`      | `c550-h3c-multi`      |
| `c550-mohe`   | `x3ls.ri.i80`  | MUXI 超节点 | `c550-superpod-single` | `c550-superpod-multi` |

注意：

* vcluster 名称只作为初筛线索。
* 机器类型最终应结合 rayctl 输出、节点 label、allocatable 或用户提供信息确认。
* 不确定模板时，停止并返回需要确认的信息。

---

## 2. 模板资源范围

| 模板族             | 加速卡资源                  |  单机卡数范围 | CPU 上限 | Memory 上限 | 多机每节点固定值                | 备注                                     |
| --------------- | ---------------------- | ------: | -----: | --------: | ----------------------- | -------------------------------------- |
| `910c`          | `huawei.com/Ascend910` | 2-16，偶数 |    256 |    1920Gi | 16 卡 / 256 CPU / 1920Gi | `910c-multi` 支持 `--logical-supernodes` |
| `910b`          | `huawei.com/Ascend910` |     1-8 |    144 |    1920Gi | 8 卡 / 144 CPU / 1920Gi  | 一般不设置 `--logical-supernodes`           |
| `c550-default`  | `metax-tech.com/gpu`   |     1-8 |    224 |    1440Gi | 8 卡 / 224 CPU / 1440Gi  | 风冷，额外资源 `rdma-training/roce=1`         |
| `c550-h3c`      | `metax-tech.com/gpu`   |     1-8 |    224 |     640Gi | 8 卡 / 224 CPU / 640Gi   | 液冷，额外资源 `rdma-training/roce=1`         |
| `c550-superpod` | `metax-tech.com/gpu`   |     1-8 |    120 |    1440Gi | 8 卡 / 120 CPU / 1440Gi  | 超节点，额外资源 `rdma-training/roce=1`        |

固定默认值：

```text
queue: default
priorityClass: normal
host-arch: huawei-arm
shm-size: 64Gi
framework: PyTorch
```

---

## 3. 创建任务通用要求

创建任务属于写操作，必须先确认：

* 目标 vcluster。
* namespace。
* job name。
* template。
* image。
* command。
* CPU / memory / accelerator / nodes。
* 是否挂载 PVC。
* 可能影响。

预览不提交：

```bash
rayctl job create <template-name> ... --what-if
```

变量建议：

```bash
export KUBECONFIG=/root/D/<实际 kubeconfig 文件名>

JOB='<job-name>'
NS='<namespace>'
IMAGE='<image>'
CMD='<command>'
CPU='<cpu-core>'
MEM='<memory-gi>'
ACC='<accelerator-count>'
NODES='<node-count>'
```

---

## 4. 单机任务模板

910C：

```bash
export KUBECONFIG=/root/D/<实际 kubeconfig 文件名>

rayctl job create 910c-single \
  --name "$JOB" \
  --ns "$NS" \
  --image "$IMAGE" \
  --command "$CMD" \
  --cpu "$CPU" \
  --memory "$MEM" \
  --accelerators "$ACC" \
  --priority-class normal \
  --what-if
```

910B：

```bash
export KUBECONFIG=/root/D/<实际 kubeconfig 文件名>

rayctl job create 910b-single \
  --name "$JOB" \
  --ns "$NS" \
  --image "$IMAGE" \
  --command "$CMD" \
  --cpu "$CPU" \
  --memory "$MEM" \
  --accelerators "$ACC" \
  --priority-class normal \
  --what-if
```

C550 风冷：

```bash
export KUBECONFIG=/root/D/<实际 kubeconfig 文件名>

rayctl job create c550-default-single \
  --name "$JOB" \
  --ns "$NS" \
  --image "$IMAGE" \
  --command "$CMD" \
  --cpu "$CPU" \
  --memory "$MEM" \
  --accelerators "$ACC" \
  --what-if
```

C550 液冷 / 超节点只替换模板名：

```text
c550-h3c-single
c550-superpod-single
```

用户确认后，移除 `--what-if` 再执行真实创建。

---

## 5. 多机任务模板

通用多机：

```bash
export KUBECONFIG=/root/D/<实际 kubeconfig 文件名>

rayctl job create <template-name> \
  --name "$JOB" \
  --ns "$NS" \
  --nodes "$NODES" \
  --image "$IMAGE" \
  --command "$CMD" \
  --priority-class normal \
  --what-if
```

910C 多机如需逻辑超节点：

```bash
--logical-supernodes <count>
```

用户确认后，移除 `--what-if` 再执行真实创建。

---

## 6. 创建后检查

```bash
export KUBECONFIG=/root/D/<实际 kubeconfig 文件名>

rayctl -k /root/D/<实际 kubeconfig 文件名> job get "$JOB"
```

如果任务 Pending / Failed / PodGroup 异常，转入：

```text
skills/vcjob-debug/SKILL.md
tools/rayctl-kubectl.md → 3. 查询任务
```
