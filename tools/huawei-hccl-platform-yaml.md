# Huawei 910C HCCL 平台任务工具

本文件用于 Huawei Ascend 910C 通过平台任务 / YAML / vcjob / PodGroup 运行 HCCL allreduce / allgather / alltoall 压测。

对应 skill：

```text
skills/mccl-test/SKILL.md → 场景 C：Huawei 910C HCCL 平台任务压测
```

适用：

* Huawei 910C 新机器纳管前 HCCL 压测。
* Huawei 910C SuperPod HCCL allreduce / allgather / alltoall。
* Huawei 910C 指定节点 HCCL allreduce / allgather / alltoall。
* Huawei 910C HCCL 二分法排坏节点时的 YAML 生成和字段同步。
* HCCL 平台任务创建后 Pod / PodGroup / Event / master log 检查。
* Pod Running 后进入 master 手动补跑。

不适用：

* MUXI 已纳管机器维修后宿主机 MCCL 验收。
* MUXI 新机器平台纳管前 MCCL YAML。
* D 集群物理机 ansible 单 IP 测试。
* 普通训练 / 推理任务创建。
* Huawei 910B HCCL，除非后续单独补充 910B SOP 和模板。

---

## 1. 固定入口

从本地进入 D 集群开发机：

```bash
ssh -p 32222 root@10.140.158.149
```

切换到 Huawei 910C 测试 vcluster：

```bash
export KUBECONFIG=~/D/a3-huawei-test
```

默认 namespace：

```bash
NS=default
```

建议确认当前 kubeconfig：

```bash
kubectl config current-context
kubectl get node --no-headers | head
```

如果当前环境里 `k` 是 `kubectl` alias，可以使用：

```bash
k config current-context
k get node --no-headers | head
```

---

## 2. 参数计算规则

Huawei 910C 当前按每节点 16 卡计算。

```bash
CARD_PER_NODE=16
NODE_COUNT="<节点数>"
TOTAL_CARDS=$((NODE_COUNT * CARD_PER_NODE))
WORKER_REPLICAS=$((NODE_COUNT - 1))
```

YAML 字段必须同步：

```text
metadata.annotations.sp-block = TOTAL_CARDS
spec.minAvailable = NODE_COUNT
master.replicas = 1
worker.replicas = NODE_COUNT - 1
mpirun 总卡数 = TOTAL_CARDS
```

当前完整模板中，`mpirun` 总卡数由 hostfile 动态计算：

```bash
NODE_COUNT=$(wc -l < "$HOST_FILE")
TOTAL_NPUS=$((NODE_COUNT * 16))
```

因此必须保证：

```text
实际调度节点数 = hostfile 行数 = spec.minAvailable = master.replicas + worker.replicas
```

常用示例：

| 节点数 | sp-block | minAvailable | master replicas | worker replicas | mpirun 总卡数 |
| --: | -------: | -----------: | --------------: | --------------: | ---------: |
|   2 |       32 |            2 |               1 |               1 |         32 |
|   4 |       64 |            4 |               1 |               3 |         64 |
|   6 |       96 |            6 |               1 |               5 |         96 |
|  12 |      192 |           12 |               1 |              11 |        192 |
|  24 |      384 |           24 |               1 |              23 |        384 |
|  45 |      720 |           45 |               1 |              44 |        720 |
|  48 |      768 |           48 |               1 |              47 |        768 |

---

## 3. 测试方法

优先使用用户指定镜像内实际存在的 CANN / HCCL Test 路径。若镜像同时有多个 CANN 版本，先在 master pod 内只读确认：

```bash
ls -d /usr/local/Ascend/cann-* /usr/local/Ascend/ascend-toolkit/* 2>/dev/null
find /usr/local/Ascend -path '*/tools/hccl_test/bin/*_test' -maxdepth 8 2>/dev/null
```

已验证过的 CANN 8.5.2 镜像路径：

```bash
CANN_ENV=/usr/local/Ascend/cann-8.5.2/set_env.sh
HCCL_TEST_DIR=/usr/local/Ascend/cann-8.5.2/tools/hccl_test/bin

ALL_REDUCE_TEST=$HCCL_TEST_DIR/all_reduce_test
ALL_GATHER_TEST=$HCCL_TEST_DIR/all_gather_test
ALLTOALL_TEST=$HCCL_TEST_DIR/alltoall_test
```

旧模板默认路径如下，仅在镜像确实使用 ascend-toolkit 8.1.RC1 时使用：

allreduce：

```bash
TEST_BIN=/usr/local/Ascend/ascend-toolkit/8.1.RC1/tools/hccl_test/bin/all_reduce_test
```

allgather：

```bash
TEST_BIN=/usr/local/Ascend/ascend-toolkit/8.1.RC1/tools/hccl_test/bin/all_gather_test
```

alltoall：

```bash
TEST_BIN=/usr/local/Ascend/ascend-toolkit/8.1.RC1/tools/hccl_test/bin/alltoall_test
```

历史默认数据范围：

```bash
-b 1G -e 16G -f 2
```

对应：

```text
1G, 2G, 4G, 8G, 16G
```

默认测试参数：

```bash
-p 16 \
-b 1G \
-e 16G \
-f 2 \
-w 5 \
-n 20 \
-c 1
```

---

## 4. SSH / MPI 端口规则

Volcano MPI plugin 保持 22：

```yaml
plugins:
  mpi:
    - "--master=master"
    - "--worker=worker"
    - "--port=22"
```

容器内 HCCL mpirun 使用 2222：

```bash
/usr/sbin/sshd -p 2222
```

HCCL mpirun 必须使用：

```bash
export HYDRA_LAUNCHER_EXTRA_ARGS="-p 2222 -o StrictHostKeyChecking=no"
```

在 master 内通过 SSH 查询 worker 的 Pod ID 时，也要使用 2222：

```bash
ssh -p 2222 -o StrictHostKeyChecking=no "$host" "<command>"
```

禁止把 HCCL mpirun 的 Hydra 端口改成 22。当前验证通过路径是 2222。

---

## 5. 测试范围输入规则

Huawei 910C HCCL 测试范围不在工具文件中固定。

每次由用户提供以下一种或多种范围：

```text
固定节点 IP 列表
SuperPod ID
节点数量
测试方法 allreduce / allgather / alltoall
完整测试矩阵
二分法当前轮次节点集合
```

工具文件只负责把用户提供的范围转换为 YAML 字段，不负责决定测试矩阵。

禁止：

* 固定写死某批 IP。
* 固定写死 SuperPod 4 / SuperPod 8。
* 默认生成完整测试矩阵。
* 自动扩大用户指定的节点范围。
* 用户只要求一组测试时，不要自动扩展成 allreduce + allgather + alltoall。
* 用户只要求一个 SuperPod 时，不要自动扩展到其他 SuperPod。

---

## 6. YAML 文件命名

所有临时 YAML 放在 `/tmp`。

普通测试建议：

```bash
YAML="/tmp/hccltest-910c-${NODE_COUNT}n-${TEST_KIND}.yaml"
```

二分法建议带轮次：

```bash
YAML="/tmp/hccltest-910c-r${ROUND}-${NODE_COUNT}n-${TEST_KIND}.yaml"
```

其中：

```text
TEST_KIND = allreduce、allgather 或 alltoall
ROUND = 二分法轮次，例如 1、2、3
```

`generateName` 建议：

```text
hccltest-910c-<节点数>n-<allreduce|allgather|alltoall>-
```

二分法建议：

```text
hccltest-910c-r<轮次>-<节点数>n-<allreduce|allgather|alltoall>-
```

示例：

```text
hccltest-910c-24n-allreduce-
hccltest-910c-24n-alltoall-
hccltest-910c-r1-24n-allreduce-
hccltest-910c-r2-12n-allreduce-
```

注意：

* `generateName` 必须以字母开头。
* 不要用 `910c-hccltest-` 开头，避免 DNS-1035 名称错误。
* 每一轮二分法任务建议带轮次和节点数，避免混淆。

---

## 7. 调度策略

SuperPod 调度和指定节点调度是互斥分支。

同一个 YAML 中不要同时使用：

```text
resource.sensecore.cn/a3-super-pod-id
kubernetes.io/hostname In [...]
```

除非用户明确要求且你已经确认这是平台允许的更窄 AND 过滤方式。默认情况下二者二选一。

---

### 7.1 SuperPod 调度

如果用户要求使用 SuperPod X 的机器压测，master 和 worker 都要在 `nodeSelector` 中设置：

```yaml
nodeSelector:
  host-arch: huawei-arm
  accelerator-type: module-910c-8
  resource.sensecore.cn/a3-super-pod-id: "<X>"
```

注意：

* master 和 worker 必须同步修改。
* 不得只改 master 或只改 worker。
* 如果后续用二分法排坏节点，每一轮都必须同步检查 SuperPod selector 是否符合当前节点范围。
* SuperPod ID 由用户每次指定，不在工具文件中固定。

---

### 7.2 指定节点调度

如果用户要求指定某几个节点，则必须：

1. 删除 master 和 worker 的 `nodeSelector.resource.sensecore.cn/a3-super-pod-id`。
2. 保留基础 nodeSelector。
3. 在 master 和 worker 的 affinity 中，同一个 `nodeSelectorTerm.matchExpressions` 下增加 `kubernetes.io/hostname In [...]`。

基础 nodeSelector：

```yaml
nodeSelector:
  host-arch: huawei-arm
  accelerator-type: module-910c-8
```

affinity 示例：

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: resource.compute.sensecore.cn/machine-type
              operator: In
              values:
                - h2ls.ru.k10
            - key: kubernetes.io/hostname
              operator: In
              values:
                - "<node-ip-1>"
                - "<node-ip-2>"
```

注意：

* `kubernetes.io/hostname` 必须加在同一个 `matchExpressions` 列表里，和 machine-type 形成 AND 关系。
* 不要新建一个平级 `nodeSelectorTerms`，否则多个 `nodeSelectorTerms` 是 OR 关系，可能扩大调度范围。
* master 和 worker 必须同步修改。
* values 中填写用户要求的节点 IP，前提是当前平台的 `kubernetes.io/hostname` 标签值就是节点 IP。

如果不确定标签值，先只读确认：

```bash
NODE_PATTERN="<节点IP1|节点IP2>"
k get node --show-labels | grep -E "$NODE_PATTERN"
```

如果节点列表很多，建议写到临时文件后检查：

```bash
cat > /tmp/hccl-910c-nodes.txt <<'EOF'
<node-ip-1>
<node-ip-2>
<node-ip-3>
EOF

while read -r NODE; do
  k get node "$NODE" --show-labels 2>/dev/null || true
done < /tmp/hccl-910c-nodes.txt
```

---

## 8. 完整 YAML 模板

说明：

* 这是 Huawei 910C HCCL 平台任务模板。
* 模板使用占位符，创建前必须替换。
* 默认结构为 1 个 master + N-1 个 worker。
* master 和 worker 都启动 `sshd -p 2222`。
* master 生成 `/root/mpi_env.sh` 和 `/root/hccl.sh`。
* worker 只生成 `/root/mpi_env.sh`，启动 sshd 后 `sleep inf`。
* master 默认等待 worker 镜像拉取、sshd 启动和 DNS 就绪。
* master 执行完 HCCL 后 `sleep inf`，方便进入容器补跑。
* 当前平台模板中保留 `ring-controller.atlas: ascend-910b`。不要擅自改成 `ascend-910c`，除非确认平台调度要求已经变化。
* 如果使用 SuperPod 调度，保留并替换 `<SUPERPOD_ID>`。
* 如果使用指定节点调度，删除 master 和 worker 的 `resource.sensecore.cn/a3-super-pod-id`，并按第 7.2 节添加 hostname affinity。
* 如果改成 alltoall，只修改 master command 中的 `TEST_BIN`。
* 如果改节点数，必须同步修改 `sp-block`、`minAvailable`、worker replicas。
* `mpirun` 总卡数在容器内由 hostfile 行数动态计算。

占位符：

```text
<NAMESPACE>
<GENERATE_NAME>
<TOTAL_CARDS>
<NODE_COUNT>
<WORKER_REPLICAS>
<IMAGE>
<IMAGE_PULL_SECRET>
<TEST_BIN>
<START_DELAY_SECONDS>
<SUPERPOD_ID>
```

完整模板：

```yaml
apiVersion: batch.volcano.sh/v1alpha1
kind: Job
metadata:
  generateName: <GENERATE_NAME>
  namespace: <NAMESPACE>
  labels:
    lepton.sensetime.com/submitter: ailabdev
    lepton.sensetime.com/framework-type: MPI
    ring-controller.atlas: ascend-910b
  annotations:
    sp-block: "<TOTAL_CARDS>"

spec:
  priorityClassName: normal
  queue: default
  schedulerName: volcano

  policies:
    - event: PodEvicted
      action: RestartJob

  minAvailable: <NODE_COUNT>

  plugins:
    ssh: []
    svc: []
    mpi:
      - "--master=master"
      - "--worker=worker"
      - "--port=22"
    hcclrank: []

  maxRetry: 1

  tasks:
    - name: master
      replicas: 1
      policies:
        - event: PodEvicted
          action: RestartJob
        - event: TaskCompleted
          action: CompleteJob
      template:
        metadata:
          labels:
            lepton.sensetime.com/submitter: ailabdev
            ring-controller.atlas: ascend-910b
        spec:
          imagePullSecrets:
            - name: <IMAGE_PULL_SECRET>
          containers:
            - name: master
              image: <IMAGE>
              imagePullPolicy: IfNotPresent
              command:
                - bash
                - -lc
              args:
                - |
                  cat > /root/mpi_env.sh <<'EOF'
                  #!/bin/bash
                  . /usr/local/Ascend/ascend-toolkit/set_env.sh

                  export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/Ascend/driver/lib64/common:/usr/local/Ascend/driver/lib64/driver:/usr/local/mpich-3.2.1/lib
                  export CPU_AFFINITY_CONF=1,npu0:12-25,npu1:26-39,npu2:52-65,npu3:66-79,npu4:92-105,npu5:106-119,npu6:132-145,npu7:146-159,npu8:172-185,npu9:186-199,npu10:212-225,npu11:226-239,npu12:252-265,npu13:266-279,npu14:292-305,npu15:306-319
                  export HCCL_BUFFERSIZE=8192
                  export HCCL_SUPERPOD_MODE=1
                  export HCCL_LOGIC_SUPERPOD_ID=$(npu-smi info -t spod-info -i 0 -c 0 | grep -i "Pod ID" | awk '{print $5}')
                  export HCCL_IF_IP=$(hostname -i | awk '{print $1}')

                  exec "$@"
                  EOF

                  chmod +x /root/mpi_env.sh

                  cat > /root/hccl.sh <<'EOF'
                  #!/bin/bash
                  set -e

                  RAW_HOST_LIST=$(echo "$VC_MASTER_HOSTS,$VC_WORKER_HOSTS,$MPI_HOST" | tr ',' '\n' | grep -v '^$' | sort -u)

                  echo "Step 1: Detecting Pod IDs and sorting hosts..."
                  TEMP_SORT_FILE=$(mktemp)

                  for host in $RAW_HOST_LIST; do
                      POD_ID=$(ssh -p 2222 -o StrictHostKeyChecking=no "$host" "npu-smi info -t spod-info -i 0 -c 0 | grep -i 'Pod ID' | awk '{print \$5}'" 2>/dev/null || true)
                      POD_ID=${POD_ID:-0}
                      echo "$POD_ID $host" >> "$TEMP_SORT_FILE"
                      echo "Host $host is in Physical Pod $POD_ID"
                  done

                  SORTED_HOSTS=$(sort -n "$TEMP_SORT_FILE" | awk '{print $2}')
                  rm -f "$TEMP_SORT_FILE"

                  echo "Step 2: Resolving IPs in sorted order..."
                  HOST_FILE="/root/hostfile"
                  rm -f "$HOST_FILE"
                  TEMP_DIR=$(mktemp -d)

                  resolve_host() {
                      local host=$1
                      local out_file=$2
                      local IP

                      IP=$(getent hosts "$host" | awk '{print $1}' | head -n1)

                      if [ -z "$IP" ]; then
                          local SHORT_HOST
                          SHORT_HOST=$(echo "$host" | cut -d'.' -f1)
                          IP=$(getent hosts "$SHORT_HOST" | awk '{print $1}' | head -n1)
                      fi

                      if [ -n "$IP" ]; then
                          echo "${IP}:16" > "$out_file"
                      else
                          echo "${host}:16" > "$out_file"
                      fi
                  }

                  index=0
                  declare -a task_files

                  for host in $SORTED_HOSTS; do
                      task_file="$TEMP_DIR/$index"
                      task_files+=("$task_file")
                      resolve_host "$host" "$task_file" &
                      index=$((index + 1))
                  done

                  wait

                  for f in "${task_files[@]}"; do
                      cat "$f" >> "$HOST_FILE"
                  done

                  rm -rf "$TEMP_DIR"

                  echo "------------------- Sorted Hostfile --------------------"
                  cat "$HOST_FILE"
                  echo "--------------------------------------------------------"

                  NODE_COUNT=$(wc -l < "$HOST_FILE")
                  TOTAL_NPUS=$((NODE_COUNT * 16))

                  MPI_BIN=/usr/local/mpich-3.2.1/bin/mpirun
                  ENV_WRAPPER=/root/mpi_env.sh
                  TEST_BIN=<TEST_BIN>

                  export HYDRA_LAUNCHER_EXTRA_ARGS="-p 2222 -o StrictHostKeyChecking=no"

                  echo "Launching HCCL test on $TOTAL_NPUS NPUs..."
                  echo "TEST_BIN=$TEST_BIN"

                  $MPI_BIN -f "$HOST_FILE" -n "$TOTAL_NPUS" \
                      "$ENV_WRAPPER" "$TEST_BIN" \
                      -p 16 \
                      -b 1G \
                      -e 16G \
                      -f 2 \
                      -w 5 \
                      -n 20 \
                      -c 1
                  EOF

                  chmod +x /root/hccl.sh

                  mkdir -p /var/run/sshd

                  if ! pgrep -x sshd >/dev/null 2>&1; then
                      /usr/sbin/sshd -p 2222
                  fi

                  START_DELAY_SECONDS=${START_DELAY_SECONDS:-<START_DELAY_SECONDS>}
                  echo "Waiting ${START_DELAY_SECONDS}s for worker pods image pull, sshd startup and DNS readiness..."
                  sleep "$START_DELAY_SECONDS"

                  bash /root/hccl.sh

                  sleep inf
              env: []
              volumeMounts:
                - name: shm-data
                  mountPath: /dev/shm
              resources:
                requests:
                  cpu: "256"
                  memory: 1920Gi
                  huawei.com/Ascend910: "16"
                limits:
                  cpu: "256"
                  memory: 1920Gi
                  huawei.com/Ascend910: "16"
              securityContext:
                capabilities:
                  add:
                    - IPC_LOCK
          restartPolicy: Never
          volumes:
            - name: shm-data
              emptyDir:
                medium: Memory
                sizeLimit: 64Gi
          affinity:
            nodeAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                nodeSelectorTerms:
                  - matchExpressions:
                      - key: resource.compute.sensecore.cn/machine-type
                        operator: In
                        values:
                          - h2ls.ru.k10
          nodeSelector:
            host-arch: huawei-arm
            accelerator-type: module-910c-8
            resource.sensecore.cn/a3-super-pod-id: "<SUPERPOD_ID>"

    - name: worker
      replicas: <WORKER_REPLICAS>
      policies:
        - event: PodEvicted
          action: RestartJob
      template:
        metadata:
          labels:
            lepton.sensetime.com/submitter: ailabdev
            ring-controller.atlas: ascend-910b
        spec:
          imagePullSecrets:
            - name: <IMAGE_PULL_SECRET>
          containers:
            - name: worker
              image: <IMAGE>
              imagePullPolicy: IfNotPresent
              command:
                - bash
                - -lc
              args:
                - |
                  cat > /root/mpi_env.sh <<'EOF'
                  #!/bin/bash
                  . /usr/local/Ascend/ascend-toolkit/set_env.sh

                  export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/Ascend/driver/lib64/common:/usr/local/Ascend/driver/lib64/driver:/usr/local/mpich-3.2.1/lib
                  export CPU_AFFINITY_CONF=1,npu0:12-25,npu1:26-39,npu2:52-65,npu3:66-79,npu4:92-105,npu5:106-119,npu6:132-145,npu7:146-159,npu8:172-185,npu9:186-199,npu10:212-225,npu11:226-239,npu12:252-265,npu13:266-279,npu14:292-305,npu15:306-319
                  export HCCL_BUFFERSIZE=8192
                  export HCCL_SUPERPOD_MODE=1
                  export HCCL_LOGIC_SUPERPOD_ID=$(npu-smi info -t spod-info -i 0 -c 0 | grep -i "Pod ID" | awk '{print $5}')
                  export HCCL_IF_IP=$(hostname -i | awk '{print $1}')

                  exec "$@"
                  EOF

                  chmod +x /root/mpi_env.sh

                  mkdir -p /var/run/sshd

                  if ! pgrep -x sshd >/dev/null 2>&1; then
                      /usr/sbin/sshd -p 2222
                  fi

                  sleep inf
              env: []
              volumeMounts:
                - name: shm-data
                  mountPath: /dev/shm
              resources:
                requests:
                  cpu: "256"
                  memory: 1920Gi
                  huawei.com/Ascend910: "16"
                limits:
                  cpu: "256"
                  memory: 1920Gi
                  huawei.com/Ascend910: "16"
              securityContext:
                capabilities:
                  add:
                    - IPC_LOCK
          restartPolicy: Never
          volumes:
            - name: shm-data
              emptyDir:
                medium: Memory
                sizeLimit: 64Gi
          affinity:
            nodeAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                nodeSelectorTerms:
                  - matchExpressions:
                      - key: resource.compute.sensecore.cn/machine-type
                        operator: In
                        values:
                          - h2ls.ru.k10
          nodeSelector:
            host-arch: huawei-arm
            accelerator-type: module-910c-8
            resource.sensecore.cn/a3-super-pod-id: "<SUPERPOD_ID>"
```

---

## 9. 生成 YAML 时的替换示例

### 9.1 24 节点 allreduce SuperPod 示例

```bash
NS=default
NODE_COUNT=24
CARD_PER_NODE=16
TOTAL_CARDS=$((NODE_COUNT * CARD_PER_NODE))
WORKER_REPLICAS=$((NODE_COUNT - 1))
TEST_KIND=allreduce
ROUND=1
SUPERPOD_ID="<用户指定的 SuperPod ID>"
IMAGE="registry2.d.pjlab.org.cn/ccr-lepton-official-images/mindspeed-llm:can8.1-910c-arm"
IMAGE_PULL_SECRET="ccr-tangrui"
START_DELAY_SECONDS=300
TEST_BIN="/usr/local/Ascend/ascend-toolkit/8.1.RC1/tools/hccl_test/bin/all_reduce_test"

YAML="/tmp/hccltest-910c-r${ROUND}-${NODE_COUNT}n-${TEST_KIND}.yaml"
GENERATE_NAME="hccltest-910c-r${ROUND}-${NODE_COUNT}n-${TEST_KIND}-"
```

需要替换：

```text
<NAMESPACE>           → default
<GENERATE_NAME>       → hccltest-910c-r1-24n-allreduce-
<TOTAL_CARDS>         → 384
<NODE_COUNT>          → 24
<WORKER_REPLICAS>     → 23
<IMAGE>               → registry2.d.pjlab.org.cn/ccr-lepton-official-images/mindspeed-llm:can8.1-910c-arm
<IMAGE_PULL_SECRET>   → ccr-tangrui
<TEST_BIN>            → /usr/local/Ascend/ascend-toolkit/8.1.RC1/tools/hccl_test/bin/all_reduce_test
<START_DELAY_SECONDS> → 300
<SUPERPOD_ID>         → 用户指定的 SuperPod ID
```

### 9.2 24 节点 alltoall SuperPod 示例

```bash
NS=default
NODE_COUNT=24
CARD_PER_NODE=16
TOTAL_CARDS=$((NODE_COUNT * CARD_PER_NODE))
WORKER_REPLICAS=$((NODE_COUNT - 1))
TEST_KIND=alltoall
ROUND=1
SUPERPOD_ID="<用户指定的 SuperPod ID>"
IMAGE="registry2.d.pjlab.org.cn/ccr-lepton-official-images/mindspeed-llm:can8.1-910c-arm"
IMAGE_PULL_SECRET="ccr-tangrui"
START_DELAY_SECONDS=300
TEST_BIN="/usr/local/Ascend/ascend-toolkit/8.1.RC1/tools/hccl_test/bin/alltoall_test"

YAML="/tmp/hccltest-910c-r${ROUND}-${NODE_COUNT}n-${TEST_KIND}.yaml"
GENERATE_NAME="hccltest-910c-r${ROUND}-${NODE_COUNT}n-${TEST_KIND}-"
```

---

## 10. allreduce / allgather / alltoall 切换

allreduce 使用：

```bash
TEST_BIN=/usr/local/Ascend/ascend-toolkit/8.1.RC1/tools/hccl_test/bin/all_reduce_test
```

allgather 使用：

```bash
TEST_BIN=/usr/local/Ascend/ascend-toolkit/8.1.RC1/tools/hccl_test/bin/all_gather_test
```

alltoall 使用：

```bash
TEST_BIN=/usr/local/Ascend/ascend-toolkit/8.1.RC1/tools/hccl_test/bin/alltoall_test
```

如果镜像使用 CANN 8.5.2，把上面的前缀替换为：

```bash
/usr/local/Ascend/cann-8.5.2/tools/hccl_test/bin/
```

修改测试方法时，只改 master `/root/hccl.sh` 内的 `TEST_BIN`。

不要修改 worker command。worker 不主动执行 HCCL 测试。

---

## 11. 节点数调整 checklist

如果节点数从原始模板改为 N，必须同步修改：

```text
metadata.annotations.sp-block = N × 16
spec.minAvailable = N
master.replicas = 1
worker.replicas = N - 1
```

当前模板里 `mpirun` 总卡数由 hostfile 行数动态计算：

```bash
NODE_COUNT=$(wc -l < "$HOST_FILE")
TOTAL_NPUS=$((NODE_COUNT * 16))
```

所以只要 hostfile 与实际调度节点一致，mpirun 总卡数会自动跟随 hostfile。

但 YAML 的以下字段仍然必须人工同步：

```text
sp-block
minAvailable
worker.replicas
selector / affinity
```

---

## 12. 指定节点模板修改方式

如果当前任务是指定节点列表，而不是 SuperPod：

### 12.1 删除 SuperPod selector

master 和 worker 都删除：

```yaml
resource.sensecore.cn/a3-super-pod-id: "<SUPERPOD_ID>"
```

保留：

```yaml
nodeSelector:
  host-arch: huawei-arm
  accelerator-type: module-910c-8
```

### 12.2 增加 hostname affinity

master 和 worker 都在同一个 `matchExpressions` 下增加：

```yaml
- key: kubernetes.io/hostname
  operator: In
  values:
    - "<node-ip-1>"
    - "<node-ip-2>"
    - "<node-ip-3>"
```

完整结构示例：

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: resource.compute.sensecore.cn/machine-type
              operator: In
              values:
                - h2ls.ru.k10
            - key: kubernetes.io/hostname
              operator: In
              values:
                - "<node-ip-1>"
                - "<node-ip-2>"
                - "<node-ip-3>"
```

不要写成：

```yaml
nodeSelectorTerms:
  - matchExpressions:
      - key: resource.compute.sensecore.cn/machine-type
        operator: In
        values:
          - h2ls.ru.k10
  - matchExpressions:
      - key: kubernetes.io/hostname
        operator: In
        values:
          - "<node-ip-1>"
```

因为多个 `nodeSelectorTerms` 是 OR 关系，可能扩大调度范围。

---

## 13. 创建前预览

创建前至少预览：

```bash
grep -nE "generateName:|namespace:|sp-block:|minAvailable:|replicas:|image:|TEST_BIN=|HYDRA_LAUNCHER_EXTRA_ARGS|a3-super-pod-id|kubernetes.io/hostname|START_DELAY_SECONDS" "$YAML"
```

建议同时检查 `sshd -p 2222`：

```bash
grep -nE "sshd -p 2222|ssh -p 2222|HYDRA_LAUNCHER_EXTRA_ARGS" "$YAML"
```

必须确认：

```text
generateName 正确
namespace 正确
sp-block = 节点数 × 16
minAvailable = 节点数
master replicas = 1
worker replicas = 节点数 - 1
image 正确
TEST_BIN 对应 allreduce / allgather / alltoall
HYDRA_LAUNCHER_EXTRA_ARGS 使用 -p 2222
master 和 worker 都启动 sshd -p 2222
master 和 worker 的 nodeSelector / affinity 一致
SuperPod 和指定节点调度没有混用
```

---

## 14. dry-run 与创建

确认 YAML 结构：

```bash
k apply --dry-run=server -f "$YAML" -o yaml >/tmp/hccltest-dryrun.yaml
```

查看 dry-run 关键字段：

```bash
grep -nE "generateName:|namespace:|sp-block:|minAvailable:|replicas:|image:|TEST_BIN=|HYDRA_LAUNCHER_EXTRA_ARGS|a3-super-pod-id|kubernetes.io/hostname" /tmp/hccltest-dryrun.yaml
```

确认后创建：

```bash
k create -f "$YAML"
```

创建是写操作，必须用户确认后执行。

---

## 15. 创建后检查

创建后等待 worker 镜像拉取、sshd 启动和 DNS 就绪。

默认等待：

```bash
sleep 300
```

查看任务：

```bash
NS=default
JOB="<job-name>"

k get vcjob -n "$NS" | grep -F -- "$JOB"
k get pod -n "$NS" -o wide | grep -F -- "$JOB"
k get podgroup -n "$NS" | grep -F -- "$JOB"
k get events -n "$NS" --sort-by=.lastTimestamp | grep -F -- "$JOB"
```

查看 master pod：

```bash
MASTER_POD=$(k get pod -n "$NS" -o name \
  | grep -F -- "$JOB" \
  | grep -F -- "master" \
  | head -n1 \
  | sed 's#pod/##')

echo "$MASTER_POD"
```

查看 master 日志：

```bash
k logs -n "$NS" "$MASTER_POD" --tail=300
```

持续观察日志：

```bash
k logs -f -n "$NS" "$MASTER_POD"
```

成功日志通常包含：

```text
data_size(Bytes):
aveg_time(us):
alg_bandwidth(GB/s):
check_result:
success
```

失败关键字：

```text
HcclGetRootInfo failed
received invalid data from root process 0
timeout
failed
error
hang
Could not resolve hostname
No route to host
Connection refused
Permission denied
```

---

## 16. master 内检查

进入 master：

```bash
NS=default
JOB="<job-name>"

MASTER_POD=$(k get pod -n "$NS" -o name \
  | grep -F -- "$JOB" \
  | grep -F -- "master" \
  | head -n1 \
  | sed 's#pod/##')

k exec -it -n "$NS" "$MASTER_POD" -- bash
```

进入后检查：

```bash
cat /root/hostfile
wc -l /root/hostfile
grep -n "HYDRA_LAUNCHER_EXTRA_ARGS" /root/hccl.sh
grep -n "TEST_BIN=" /root/hccl.sh
pgrep -a sshd
```

检查 hostfile 和总卡数：

```bash
NODE_COUNT=$(wc -l < /root/hostfile)
TOTAL_NPUS=$((NODE_COUNT * 16))
echo "NODE_COUNT=$NODE_COUNT TOTAL_NPUS=$TOTAL_NPUS"
```

---

## 17. 手动 exec 补跑

如果自动日志没有测试结果，但 Pod 都 Running，可以进入 master 手动跑。

手动补跑 allreduce：

```bash
MPI_BIN=/usr/local/mpich-3.2.1/bin/mpirun
ENV_WRAPPER=/root/mpi_env.sh
TEST_BIN=/usr/local/Ascend/ascend-toolkit/8.1.RC1/tools/hccl_test/bin/all_reduce_test

NODE_COUNT=$(wc -l < /root/hostfile)
TOTAL_NPUS=$((NODE_COUNT * 16))

export HYDRA_LAUNCHER_EXTRA_ARGS="-p 2222 -o StrictHostKeyChecking=no"

$MPI_BIN -f /root/hostfile -n "$TOTAL_NPUS" \
  "$ENV_WRAPPER" "$TEST_BIN" \
  -p 16 \
  -b 1G \
  -e 16G \
  -f 2 \
  -w 5 \
  -n 20 \
  -c 1
```

已验证过的 910C SuperPod allgather / alltoall 审计矩阵常用数据范围：

```bash
-b 512M -e 4G -f 2
```

对应：

```text
512M, 1G, 2G, 4G
```

allgather 零拷贝：

```bash
# 关闭零拷贝
all_gather_test ... -z 0

# 开启零拷贝
all_gather_test ... -z 1
```

alltoall 不做零拷贝开 / 关对比，不加 `-z`。

指定 Device 执行 HCCL Test：

```bash
# 单卡 / 单 die 场景示例：每台固定使用 Device 4
export HCCL_TEST_USE_DEVS="4"

# 多卡示例：只允许使用 4/5/6/7 号卡
export HCCL_TEST_USE_DEVS="4,5,6,7"
```

每机只用 1 卡时：

```text
hostfile 每个节点写 :1
mpirun -n <节点数>
HCCL test 参数使用 -p 1
```

手动补跑 alltoall：

```bash
MPI_BIN=/usr/local/mpich-3.2.1/bin/mpirun
ENV_WRAPPER=/root/mpi_env.sh
TEST_BIN=/usr/local/Ascend/ascend-toolkit/8.1.RC1/tools/hccl_test/bin/alltoall_test

NODE_COUNT=$(wc -l < /root/hostfile)
TOTAL_NPUS=$((NODE_COUNT * 16))

export HYDRA_LAUNCHER_EXTRA_ARGS="-p 2222 -o StrictHostKeyChecking=no"

$MPI_BIN -f /root/hostfile -n "$TOTAL_NPUS" \
  "$ENV_WRAPPER" "$TEST_BIN" \
  -p 16 \
  -b 1G \
  -e 16G \
  -f 2 \
  -w 5 \
  -n 20 \
  -c 1
```

必须确认：

```text
/root/hostfile 行数 = 当前测试节点数
TOTAL_NPUS = 当前测试节点数 × 16
TEST_BIN 对应 allreduce / allgather / alltoall
HYDRA_LAUNCHER_EXTRA_ARGS 使用 -p 2222
```

---

## 18. 临时两机测试

如果已有大规模 Pod Running，但只想临时测两机，不要执行：

```bash
bash /root/hccl.sh
```

原因：

```text
/root/hccl.sh 每次执行都会重新基于 VC_MASTER_HOSTS、VC_WORKER_HOSTS、MPI_HOST 生成全量 /root/hostfile。
手动修改 /root/hostfile 后再执行 bash /root/hccl.sh，会被覆盖。
```

应手动写 hostfile，并直接执行 mpirun：

```bash
cat > /root/hostfile <<'EOF'
<master-ip>:16
<worker-ip>:16
EOF

MPI_BIN=/usr/local/mpich-3.2.1/bin/mpirun
ENV_WRAPPER=/root/mpi_env.sh
TEST_BIN=/usr/local/Ascend/ascend-toolkit/8.1.RC1/tools/hccl_test/bin/all_reduce_test

export HYDRA_LAUNCHER_EXTRA_ARGS="-p 2222 -o StrictHostKeyChecking=no"

$MPI_BIN -f /root/hostfile -n 32 \
  "$ENV_WRAPPER" "$TEST_BIN" \
  -p 16 \
  -b 1G \
  -e 16G \
  -f 2 \
  -w 5 \
  -n 20 \
  -c 1
```

如果临时两机测 alltoall，只改：

```bash
TEST_BIN=/usr/local/Ascend/ascend-toolkit/8.1.RC1/tools/hccl_test/bin/alltoall_test
```

---

## 18.1 CANN 8.5.2 手动补跑与审计留存模板

适用于已有 910C HCCL 任务 Pod 全部 Running 后，在 master pod 内补跑 allgather / alltoall 矩阵并保留审计文件。

公共变量：

```bash
AUDIT_DIR="/tmp/hccl_<test>_<matrix>_$(date +%Y%m%d_%H%M%S)"
mkdir -p "$AUDIT_DIR"/{scripts,hostfiles,logs}

MPI_BIN=/usr/local/mpich-3.2.1/bin/mpirun
ENV_WRAPPER=/tmp/mpi_env_cann852.sh
HCCL_TEST_DIR=/usr/local/Ascend/cann-8.5.2/tools/hccl_test/bin

export HYDRA_LAUNCHER_EXTRA_ARGS="-p 2222 -o StrictHostKeyChecking=no"
```

审计目录必须至少保留：

```text
scripts/       启动脚本
hostfiles/     每个场景实际使用的 hostfile
logs/          每个场景原始日志
summary.tsv    场景、节点数、每节点卡数、总卡数、Device、rc、日志路径
*_results.md   汇总结果，按用户要求保留两位小数
*_results.tsv  汇总结果原始表格
```

allgather full16 模板：

```bash
TEST_BIN=$HCCL_TEST_DIR/all_gather_test

NODE_COUNT=$(wc -l < "$HOSTFILE")
NPUS_PER_NODE=$(awk -F: 'NR==1{print $2}' "$HOSTFILE")
TOTAL_NPUS=$(awk -F: '{s+=$2} END{print s}' "$HOSTFILE")

# ZERO_COPY=0 表示关闭零拷贝；ZERO_COPY=1 表示开启零拷贝
ZERO_COPY=0

$MPI_BIN -f "$HOSTFILE" -n "$TOTAL_NPUS" \
  "$ENV_WRAPPER" "$TEST_BIN" \
  -p "$NPUS_PER_NODE" \
  -b 512M \
  -e 4G \
  -f 2 \
  -w 5 \
  -n 20 \
  -c 1 \
  -z "$ZERO_COPY"
```

allgather 已验证场景命名建议：

```text
pod4_pod8_32n_full16
pod4_16n_full16
pod8_16n_full16
pod4_2n_full16
pod8_2n_full16
pod4_1n_full16
pod8_1n_full16
```

alltoall full16 模板：

```bash
TEST_BIN=$HCCL_TEST_DIR/alltoall_test

NODE_COUNT=$(wc -l < "$HOSTFILE")
NPUS_PER_NODE=$(awk -F: 'NR==1{print $2}' "$HOSTFILE")
TOTAL_NPUS=$(awk -F: '{s+=$2} END{print s}' "$HOSTFILE")

$MPI_BIN -f "$HOSTFILE" -n "$TOTAL_NPUS" \
  "$ENV_WRAPPER" "$TEST_BIN" \
  -p "$NPUS_PER_NODE" \
  -b 512M \
  -e 4G \
  -f 2 \
  -w 5 \
  -n 20 \
  -c 1
```

alltoall 每机 1 卡 / 1 die 模板：

```bash
TEST_BIN=$HCCL_TEST_DIR/alltoall_test
DEVICE_ID=4
export HCCL_TEST_USE_DEVS="$DEVICE_ID"

# hostfile 每个节点只写 1 个 slot
cat > "$HOSTFILE" <<'EOF'
<pod-ip-1>:1
<pod-ip-2>:1
<pod-ip-3>:1
<pod-ip-4>:1
EOF

TOTAL_NPUS=$(awk -F: '{s+=$2} END{print s}' "$HOSTFILE")

HCCL_TEST_USE_DEVS="$DEVICE_ID" \
$MPI_BIN -f "$HOSTFILE" -n "$TOTAL_NPUS" \
  "$ENV_WRAPPER" "$TEST_BIN" \
  -p 1 \
  -b 512M \
  -e 4G \
  -f 2 \
  -w 5 \
  -n 20 \
  -c 1
```

alltoall 每机 1 卡已验证场景：

```text
pod4_4n_1card_dev4
pod8_4n_1card_dev4
pod4_8n_1card_dev4
pod8_8n_1card_dev4
pod4_16n_1card_dev4
pod8_16n_1card_dev4
```

注意：

* alltoall 不加 `-z`。
* allgather 必须显式写明 `-z 0` 或 `-z 1`。
* 每机 1 卡场景必须同时满足 hostfile `:1`、`mpirun -n <节点数>`、HCCL Test `-p 1`。
* `HCCL_TEST_USE_DEVS` 应在 `mpirun` 前导出，并可在命令前再加一次，确保远端进程继承。
* 如果 master pod 无法通过 pod 名解析自身，需要在所有 Pod 的 `/etc/hosts` 中补 master pod name 到 pod IP 的映射；这是临时补跑可用的容器内操作，不应写入 YAML 模板默认行为。

## 19. 二分法 YAML 字段同步规则

二分法策略放在：

```text
skills/mccl-test/SKILL.md → 场景 C → 二分法排坏节点
```

本工具文件只规定每一轮 YAML 字段如何同步。

二分法每一轮都会生成新的 YAML。

假设当前轮次节点数为 `N`：

```text
sp-block = N × 16
minAvailable = N
master.replicas = 1
worker.replicas = N - 1
mpirun 总卡数 = N × 16
```

当前模板中 `mpirun` 总卡数由 hostfile 动态计算：

```bash
NODE_COUNT=$(wc -l < "$HOST_FILE")
TOTAL_NPUS=$((NODE_COUNT * 16))
```

因此每轮必须保证：

```text
实际调度出来的 Pod 节点数 = hostfile 行数 = minAvailable = master + worker replicas
```

否则可能出现：

```text
YAML 申请 N 个节点
hostfile 生成 M 个节点
mpirun 实际跑 M × 16 卡
sp-block / minAvailable / replicas 与实际运行不一致
```

---

## 20. 二分法每轮 YAML checklist

每一轮生成 YAML 后，创建前必须检查：

```text
metadata.generateName
metadata.namespace
metadata.annotations.sp-block
spec.minAvailable
master.replicas
worker.replicas
master image
worker image
master TEST_BIN
HYDRA_LAUNCHER_EXTRA_ARGS
master nodeSelector / affinity
worker nodeSelector / affinity
SuperPod ID 或 kubernetes.io/hostname 节点列表
```

预览命令：

```bash
grep -nE "generateName:|namespace:|sp-block:|minAvailable:|replicas:|image:|TEST_BIN=|HYDRA_LAUNCHER_EXTRA_ARGS|a3-super-pod-id|kubernetes.io/hostname" "$YAML"
```

确认 YAML 结构：

```bash
k apply --dry-run=server -f "$YAML" -o yaml >/tmp/hccltest-dryrun.yaml
```

确认后创建：

```bash
k create -f "$YAML"
```

创建任务是写操作，必须用户确认后执行。

---

## 21. 二分法任务命名建议

为了避免多轮排查混乱，建议 generateName 带上轮次、节点数和测试方法：

```text
hccltest-910c-r<轮次>-<节点数>n-<allreduce|alltoall>-
```

示例：

```text
hccltest-910c-r1-24n-allreduce-
hccltest-910c-r1-24n-alltoall-
hccltest-910c-r2-12n-allreduce-
hccltest-910c-r3-6n-allreduce-
```

YAML 文件建议放在 `/tmp`：

```bash
YAML="/tmp/hccltest-910c-r${ROUND}-${NODE_COUNT}n-${TEST_KIND}.yaml"
```

---

## 22. 清理任务

删除任务属于写操作，必须用户确认后执行。

清理前预览：

```bash
YAML="/tmp/<yaml-file>.yaml"

k delete -f "$YAML" --dry-run=server -o yaml
```

确认资源：

```bash
NS=default
JOB="<job-name>"

k get vcjob -n "$NS" | grep -F -- "$JOB"
k get pod -n "$NS" -o wide | grep -F -- "$JOB"
k get podgroup -n "$NS" | grep -F -- "$JOB"
```

用户确认后删除：

```bash
k delete -f "$YAML"
```

删除后核验：

```bash
k get vcjob -n "$NS" | grep -F -- "$JOB" || true
k get pod -n "$NS" -o wide | grep -F -- "$JOB" || true
k get podgroup -n "$NS" | grep -F -- "$JOB" || true
```

---

## 23. 常见问题

### 23.1 DNS-1035 generateName 错误

错误原因：

```text
generateName 以数字开头，例如 910c-hccltest-
```

修复：

```text
使用 hccltest-910c- 开头。
```

### 23.2 master 能启动，但没有测试结果

检查：

```bash
k logs -n "$NS" "$MASTER_POD" --tail=300
k exec -it -n "$NS" "$MASTER_POD" -- bash
```

进入 master 后检查：

```bash
pgrep -a sshd
cat /root/hostfile
wc -l /root/hostfile
grep -n "TEST_BIN=" /root/hccl.sh
grep -n "HYDRA_LAUNCHER_EXTRA_ARGS" /root/hccl.sh
```

### 23.3 Could not resolve hostname

检查：

```bash
cat /root/hostfile
getent hosts <host>
getent hosts <short-host>
```

必要时确认：

```bash
echo "$VC_MASTER_HOSTS"
echo "$VC_WORKER_HOSTS"
echo "$MPI_HOST"
```

### 23.4 Connection refused

重点检查 worker 是否启动 sshd 2222：

```bash
pgrep -a sshd
ss -lntp | grep 2222
```

### 23.5 HYDRA 使用了 22

错误配置：

```bash
export HYDRA_LAUNCHER_EXTRA_ARGS="-p 22 -o StrictHostKeyChecking=no"
```

正确配置：

```bash
export HYDRA_LAUNCHER_EXTRA_ARGS="-p 2222 -o StrictHostKeyChecking=no"
```

### 23.6 手动 hostfile 被覆盖

原因：

```text
再次执行 bash /root/hccl.sh 会重新生成 /root/hostfile。
```

解决：

```text
临时两机测试时，不要执行 bash /root/hccl.sh。
手动写 /root/hostfile 后，直接执行 mpirun。
```

---

## 24. 结果关键字

成功关键字：

```text
data_size(Bytes)
aveg_time(us)
alg_bandwidth(GB/s)
check_result
success
```

失败关键字：

```text
HcclGetRootInfo failed
received invalid data from root process 0
timeout
failed
error
hang
Could not resolve hostname
No route to host
Connection refused
Permission denied
```

如果用户未提供带宽阈值，只能判断基础通信是否成功，不能判断是否满足最终纳管带宽标准。
