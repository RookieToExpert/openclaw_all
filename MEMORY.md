# MEMORY.md - OpenClaw Core Memory

本文件只保存长期稳定规则、入口边界、安全红线和用户偏好。
不要保存长命令、脚本、节点列表、一次性查询结果、当天状态或临时故障结论。

---

## 1. 自主行为边界

OpenClaw 必须按 SOP 执行，不得自行扩展排障范围。

* SOP 没有定义下一步时，必须停止。
* 首选工具查不到时，不得自动扩大范围。
* 缺少 vcluster、namespace、job、pod、node、IP 等关键标识时，必须停止并返回缺失信息。
* 继续查询需要扩大范围时，必须先返回给用户，不得自行全量扫描。
* 不确定时只能做最小只读验证。

禁止：

* 禁止遍历所有 vcluster，除非用户明确要求分区级或全局概览。
* 禁止无依据执行 `kubectl get ... -A`。
* 禁止不知道 namespace 时全 namespace 扫描。
* 禁止不知道 vcluster 时遍历 `/root/D/*`。
* 禁止用 host cluster kubeconfig 查询 vcluster 内 vcjob / PodGroup。
* 禁止混用 Kubernetes 入口和物理机入口。
* 禁止执行 SOP 之外的高风险动作。
* 禁止凭记忆编造命令。任何集群操作前，必须通过 TOOLS.md 找到对应入口，命中 tools/*.md 或 skills/*/SKILL.md 后，先读取其内容，再按模板执行命令。

---

## 2. 入口路由

执行任何集群、任务、存储、节点或机器操作前，必须先判断对象类型，再选择入口。

### 2.1 Kubernetes / vcluster / AI 训练平台资源

入口：

```text
本地 → D 集群开发机 → rayctl 优先 → kubectl 只读兜底
```

适用对象：

* vcjob、Pod、PodGroup、PVC、PV、AFS。
* Service、Endpoint、Webhook。
* Kubernetes Node、vcluster、Volcano 任务。
* 查询某个节点属于哪个 vcluster。
* 创建、查询、删除训练 / 推理任务。

规则：

1. 优先使用 rayctl。
2. kubectl 默认只做只读兜底。
3. kubectl 兜底必须同时满足：

   * 当前 SOP 明确允许。
   * 已知 vcluster、namespace、resource name、UID、pod 或 node 等关键标识。
   * 已确认 kubeconfig 类型正确。
4. vcluster 内资源必须使用对应 vcluster kubeconfig。
5. 禁止用 host cluster kubeconfig 查询、创建或删除 vcluster 内 vcjob / PodGroup / Pod。
6. PVC Pending、关键标识缺失、工具查不到目标或 kubeconfig 不确定时，必须停止，不得继续创建任务或扩大查询范围。
7. vcluster 名称、IP 段、默认 label 只能作为初筛线索，最终必须实时验证。
8. kubectl 写操作只有在用户明确要求、当前 SOP 明确允许、kubeconfig 正确，并完成写操作确认后才允许。

### 2.2 D 集群物理机器 / Linux 系统级操作

入口：

```text
本地 → 堡垒机 → 跳板机 → sensetime 用户 → ansible 单 IP
```

适用对象：

* 物理 IP、hostname。
* 机器目录、文件、日志。
* 进程、磁盘、内存、CPU。
* NPU、mx-smi、网卡、路由、DNS。
* 机器本地脚本，例如 mccl.sh。

规则：

1. 不要从本地直接 SSH 到目标节点。
2. 不要从 D 集群开发机跳到目标节点。
3. 不要在本地 OpenClaw exec 环境直接执行 ansible。
4. 不要在 D 集群开发机执行物理机器类 ansible。
5. 单机查询默认使用跳板机 sensetime 用户下的 ansible 单 IP。
6. 改配置、删文件、重启服务、运行影响业务的脚本前必须确认。
7. 单机 MUXI MCCL 维修验收属于物理机操作，默认不创建 vcjob、不打 Kubernetes label、不先 uncordon。
8. 只有用户明确要求“测试通过后放回 / uncordon”时，才在 MCCL 通过后切回 Kubernetes 入口。

### 2.3 堡垒机内镜像制作 / 打驱动 / push 操作

入口：

```text
本地 → 堡垒机 10.140.3.216:5906 → 输入 3.216 → sudo -i
```

适用对象：

* `docker pull`
* `wget` 下载镜像压缩包
* `docker load`
* `docker tag`
* `docker push`
* MUXI 镜像打云脉网卡驱动

规则：

1. 这类操作不走 D 集群开发机。
2. 这类操作不走 `220.33` 跳板机 ansible 入口。
3. MUXI 镜像默认按 SOP 打驱动；非 MUXI 镜像不自动打驱动。
4. 非 MUXI 目标 ccr 必须由用户明确指定。
5. `wget` 下载固定在 `/data/xie` 下执行。
6. 临时下载链接、鉴权参数、密码不得写入知识库。

### 2.4 平台租户登录凭据

用户将平台租户账号密码保存在本地文件：

```text
~/.openclaw/workspace/platform.json
```

适用场景：

* `rayctl auth login` 需要登录 D / Dcloud 租户。
* 平台资源授权、租户登录态过期，且用户明确要求使用本地凭据文件。

规则：

1. 默认先检查现有 `rayctl auth token` 登录态；有效时不读取凭据文件。
2. 只有执行登录 / 授权链路确实需要密码时，才读取 `platform.json` 中对应集群和租户的字段。
3. 读取后只能临时用于本次登录，不得复制到聊天、日志、命令历史、memory、TOOLS、skill 或 tools 文件。
4. 不得输出 `platform.json` 的完整内容。
5. 不得把其中密码转存到其他文件。
6. 如果无法唯一确定集群或租户，必须停止并询问。

---

## 3. 写操作强制确认

任何写操作执行前，必须先说明：

1. 操作内容。
2. 影响范围。
3. 风险。
4. 回滚或停止方式。
5. 需要用户确认的问题。
6. 涉及时间窗口时，核对北京时间与 UTC 转换。

用户明确回复以下内容前，禁止执行真实修改：

* 确认
* 同意
* 可以执行
* 执行
* yes

写操作包括但不限于：

* `kubectl delete / patch / create / apply / label / annotate / scale / rollout restart`
* `rayctl job create`
* `rayctl pvc create`
* `rayctl node cordon / uncordon`
* 删除或修改文件。
* 修改 OpenClaw 配置或知识库文件。
* 创建临时脚本。
* 重启服务、关机、重启机器。
* 删除 PVC / PV / Pod / vcjob / Webhook / Node label。
* 执行会影响业务任务的脚本。

批量写操作必须先只读筛选、展示范围和样例、给出 dry-run 或预览命令，确认后再执行。

---

## 4. 记忆分层

* `MEMORY.md`：长期稳定规则，不保存动态状态和长命令。
* `memory/YYYY-MM-DD.md`：当天动态状态、一次性查询结果、临时观察；下次使用前必须重新验证。
* Deep Knowledge / Vector Memory：只作历史参考，不得覆盖 `MEMORY.md` 安全红线；实时状态必须重新查询。
* heartbeat 必须使用 light context，避免加载完整历史和完整 SOP。

---

## 5. 输出偏好

* 先给结论，再给依据。
* 少发散、少猜测、少全量扫。
* 优先给可执行、可验证、范围明确的步骤。
* 查不到就返回失败原因和下一步安全选项。
* 日志问题先解释含义，再给下一步命令。
* 结论要区分确定事实、推断和待验证点。
* 用户问“是不是某原因”时，不要直接肯定；要说明证据、反证和还需要看的点。
