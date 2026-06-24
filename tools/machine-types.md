# machine-types.md - 机器类型、芯片、vcluster、IP 段映射

本文件保存机器类型、芯片类型、vcluster 族、资源名、label、IP 段的统一参考信息。  
本文件用于初筛和判断方向，不直接作为最终事实。  
最终结论必须结合 rayctl 或 Kubernetes 实时信息验证。

---

## 1. 判断原则

- 每个 vcluster 通常只对应一种主要机器类型，不要默认一个 vcluster 内混合多种机器类型。
- vcluster 名称、IP 段、历史经验只作为初筛线索。
- 不要仅凭 `ring-controller.atlas: ascend-910b` 判断机器类型，该标签可能是默认标签。
- 机器类型最终应结合以下信息判断：
  - `resource.compute.sensecore.cn/machine-type`
  - 节点 allocatable
  - rayctl 输出
  - Kubernetes node label
  - vcluster / 模板映射
- 查询具体 IP / 节点类型时，必须优先用 `rayctl node get -A <ip>` 或 `rayctl node check <ip>` 实时验证。
- 查询“某类机器有哪些 IP 段”时，可以先返回本文件中的静态映射，并说明需要实时验证。

---

## 2. vcluster 族与芯片初筛

| vcluster 族 | 芯片 / 类型 | 常见机器类型 | 加速卡资源 |
|---|---|---|---|
| `a2-*` | 华为 910B | `h1ls.rp.k60a` | `huawei.com/Ascend910` |
| `ai4chem` | 华为 910B | `h1ls.rp.k60a` | `huawei.com/Ascend910` |
| `a3-*` | 华为 910C | `h2ls.ru.k10` | `huawei.com/Ascend910` |
| `c550-jiaofu` | MUXI / 沐曦风冷 | `x2ls.ri.i80` | `metax-tech.com/gpu` |
| `c550-ai4s` | MUXI / 沐曦风冷 | `x2ls.ri.i80` | `metax-tech.com/gpu` |
| `c550-h3c` | MUXI / 沐曦液冷 | `x2ls.ri.i70` | `metax-tech.com/gpu` |
| `c550-mohe` | MUXI / 沐曦超节点 | `x3ls.ri.i80` | `metax-tech.com/gpu` |

---

## 3. 机器类型到芯片映射

| machine-type | 芯片 / 类型 | 说明 |
|---|---|---|
| `h1ls.rp.k60a` | 华为 910B | 常见于 `a2-*` / `ai4chem` |
| `h2ls.ru.k10` | 华为 910C | 常见于 `a3-*` |
| `x2ls.ri.i80` | MUXI / 沐曦风冷 | 常见于 `c550-jiaofu` / `c550-ai4s` |
| `x2ls.ri.i70` | MUXI / 沐曦液冷 | 常见于 `c550-h3c` |
| `x3ls.ri.i80` | MUXI / 沐曦超节点 | 常见于 `c550-mohe` |

---

## 4. 芯片类型初筛规则

### 4.1 MUXI / 沐曦 / C550

以下任一条件可作为 MUXI 初筛线索：

- vcluster 名称包含 `c550`。
- machine-type 为：
  - `x2ls.ri.i80`
  - `x2ls.ri.i70`
  - `x3ls.ri.i80`
- 节点 allocatable 包含：
  - `metax-tech.com/gpu`
- 用户明确提到：
  - MUXI
  - 沐曦
  - C550

最终结论必须用 rayctl 或 Kubernetes 实时查询验证。

### 4.2 华为 910B

以下任一条件可作为 910B 初筛线索：

- vcluster 名称包含 `a2`。
- vcluster 名称为 `ai4chem`。
- machine-type 为 `h1ls.rp.k60a`。
- 节点 allocatable 包含 `huawei.com/Ascend910`，并结合 machine-type 验证。

不要只凭 `ring-controller.atlas: ascend-910b` 判断 910B。

### 4.3 华为 910C

以下任一条件可作为 910C 初筛线索：

- vcluster 名称包含 `a3`。
- machine-type 为 `h2ls.ru.k10`。
- 节点 allocatable 包含 `huawei.com/Ascend910`，并结合 machine-type 验证。

---

## 5. IP 段初筛映射

IP 段只能作为初筛线索，不能作为最终结论。  
实际判断某个 IP 的机器类型时，必须用 rayctl 或 Kubernetes 实时验证。

| IP 段 | 初筛机器类型 | 说明 |
|---|---|---|
| `<待补充>` | `<待补充>` | 后续根据真实环境补充 |