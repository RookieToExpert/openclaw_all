# job-templates.md - rayctl 任务模板

## 1. vcluster 到模板映射

| vcluster | 机器类型 | 芯片 / 类型 | 单机模板 | 多机模板 |
|---|---|---|---|---|
| `a2-*` | `h1ls.rp.k60a` | 华为 910B | `910b-single` | `910b-multi` |
| `ai4chem` | `h1ls.rp.k60a` | 华为 910B | `910b-single` | `910b-multi` |
| `a3-*` | `h2ls.ru.k10` | 华为 910C | `910c-single` | `910c-multi` |
| `c550-jiaofu` | `x2ls.ri.i80` | MUXI 风冷 | `c550-default-single` | `c550-default-multi` |
| `c550-ai4s` | `x2ls.ri.i80` | MUXI 风冷 | `c550-default-single` | `c550-default-multi` |
| `c550-h3c` | `x2ls.ri.i70` | MUXI 液冷 | `c550-h3c-single` | `c550-h3c-multi` |
| `c550-mohe` | `x3ls.ri.i80` | MUXI 超节点 | `c550-superpod-single` | `c550-superpod-multi` |

---

## 2. 模板资源范围

| 模板族 | 加速卡资源 | 单机卡数范围 | CPU 上限 | Memory 上限 | 多机每节点固定值 | 备注 |
|---|---|---:|---:|---:|---|---|
| `910c` | `huawei.com/Ascend910` | 2-16，偶数 | 256 | 1920Gi | 16 卡 / 256 CPU / 1920Gi | `910c-multi` 支持 `--logical-supernodes` |
| `910b` | `huawei.com/Ascend910` | 1-8 | 144 | 1920Gi | 8 卡 / 144 CPU / 1920Gi | 一般不设置 `--logical-supernodes` |
| `c550-default` | `metax-tech.com/gpu` | 1-8 | 224 | 1440Gi | 8 卡 / 224 CPU / 1440Gi | 风冷，额外资源 `rdma-training/roce=1` |
| `c550-h3c` | `metax-tech.com/gpu` | 1-8 | 224 | 640Gi | 8 卡 / 224 CPU / 640Gi | 液冷，额外资源 `rdma-training/roce=1` |
| `c550-superpod` | `metax-tech.com/gpu` | 1-8 | 120 | 1440Gi | 8 卡 / 120 CPU / 1440Gi | 超节点，额外资源 `rdma-training/roce=1` |

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

创建任务属于写操作，必须先向用户确认：

- 目标 vcluster。
- namespace。
- job name。
- template。
- image。
- command。
- CPU / memory / accelerator / nodes。
- 是否挂载 PVC。
- 可能影响。

预览不提交：

```bash
rayctl job create <template-name> ... --what-if
```

---

## 4. 单机任务模板

910C：

```bash
export KUBECONFIG=/root/D/<实际 kubeconfig 文件名>
rayctl job create 910c-single   --name <job-name>   --ns <namespace>   --image <image>   --command "<command>"   --cpu <cpu-core>   --memory <memory-gi>   --accelerators <accelerator-count>   --priority-class normal
```

910B：

```bash
export KUBECONFIG=/root/D/<实际 kubeconfig 文件名>
rayctl job create 910b-single   --name <job-name>   --ns <namespace>   --image <image>   --command "<command>"   --cpu <cpu-core>   --memory <memory-gi>   --accelerators <accelerator-count>   --priority-class normal
```

C550：

```bash
rayctl job create c550-default-single   --name <job-name>   --ns <namespace>   --image <image>   --command "<command>"   --cpu <cpu-core>   --memory <memory-gi>   --accelerators <accelerator-count>
```

液冷 / 超节点只替换模板名：

```text
c550-h3c-single
c550-superpod-single
```

---

## 5. 多机任务模板

```bash
export KUBECONFIG=/root/D/<实际 kubeconfig 文件名>

rayctl job create <template-name>   --name <job-name>   --ns <namespace>   --nodes <node-count>   --image <image>   --command "<command>"   --priority-class normal
```

910C 多机如需逻辑超节点：

```bash
--logical-supernodes <count>
```
