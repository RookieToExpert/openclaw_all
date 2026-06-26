# mccl-commands.md - MCCL 底层命令模板

场景区分和完整流程见 `skills/mccl-test/SKILL.md`。
本文件只维护具体命令。

## 1. 检查目标机是否已有脚本

必须在跳板机 sensetime 用户下执行。

```bash
ansible all -i '<目标IP>,' -m shell -a 'test -f ~sensetime/mccl.sh && ls -l ~sensetime/mccl.sh || echo MISSING'
```

---

## 2. 标准单机 mccl.sh

如果目标机没有脚本，写入属于写操作，必须先确认。

```bash
#!/bin/bash
export MACA_PATH=/opt/maca
export LD_LIBRARY_PATH=${MACA_PATH}/lib:${MACA_PATH}/ompi/lib
export FORCE_ACTIVE_WAIT=2
export MCCL_PCIE_BUFFER_MODE=0

GPU_NUM=4
if [[ $1 -gt 0 && $1 -lt 65 ]]; then
  GPU_NUM=$1
fi
TEST_DIR=${MACA_PATH}/samples/mccl_tests/perf/mccl_perf
BENCH_NAMES="all_reduce_perf all_gather_perf reduce_scatter_perf sendrecv_perf alltoall_perf"
MPI_PROCESS_NUM=${GPU_NUM}
MPI_RUN_OPT="--allow-run-as-root -mca pml ^ucx -mca osc ^ucx -mca btl ^openib"
for BENCH in ${BENCH_NAMES}; do
  echo -n "The test is ${BENCH}, the maca version is " && realpath ${MACA_PATH}
  ${MACA_PATH}/ompi/bin/mpirun -x MCCL_PCIE_BUFFER_MODE -np ${MPI_PROCESS_NUM} ${MPI_RUN_OPT} ${TEST_DIR}/${BENCH} -b 1K -e 1G -d bfloat16 -f 2 -g 1 -n 10
done
```

---

## 3. 下发 mccl.sh 到目标机

不要临时手写超长 `ansible -m shell -a 'cat ...'` 单行命令。
标准做法是：先在跳板机 `sensetime` 用户下生成临时文件，再用 `ansible copy` 下发到目标机。

### 3.1 在跳板机生成临时脚本

```bash
cat > /tmp/mccl.sh <<'EOF'
#!/bin/bash
export MACA_PATH=/opt/maca
export LD_LIBRARY_PATH=${MACA_PATH}/lib:${MACA_PATH}/ompi/lib
export FORCE_ACTIVE_WAIT=2
export MCCL_PCIE_BUFFER_MODE=0

GPU_NUM=4
if [[ $1 -gt 0 && $1 -lt 65 ]]; then
  GPU_NUM=$1
fi
TEST_DIR=${MACA_PATH}/samples/mccl_tests/perf/mccl_perf
BENCH_NAMES="all_reduce_perf all_gather_perf reduce_scatter_perf sendrecv_perf alltoall_perf"
MPI_PROCESS_NUM=${GPU_NUM}
MPI_RUN_OPT="--allow-run-as-root -mca pml ^ucx -mca osc ^ucx -mca btl ^openib"
for BENCH in ${BENCH_NAMES}; do
  echo -n "The test is ${BENCH}, the maca version is " && realpath ${MACA_PATH}
  ${MACA_PATH}/ompi/bin/mpirun -x MCCL_PCIE_BUFFER_MODE -np ${MPI_PROCESS_NUM} ${MPI_RUN_OPT} ${TEST_DIR}/${BENCH} -b 1K -e 1G -d bfloat16 -f 2 -g 1 -n 10
done
EOF
chmod 755 /tmp/mccl.sh
```

### 3.2 用 ansible copy 下发到目标机

```bash
ansible all -i '<目标IP>,' -m copy -a 'src=/tmp/mccl.sh dest=/home/sensetime/mccl.sh owner=sensetime group=sensetime mode=0755'
```

### 3.3 下发后核验

```bash
ansible all -i '<目标IP>,' -m shell -a 'ls -l /home/sensetime/mccl.sh && head -5 /home/sensetime/mccl.sh'
```

### 3.4 清理跳板机临时文件

```bash
rm -f /tmp/mccl.sh
```

如果需要一条命令串行执行，按以下顺序，不要改成内联大段脚本内容：

1. 在跳板机 `sensetime` 用户下 `cat > /tmp/mccl.sh <<'EOF' ... EOF`
2. `chmod 755 /tmp/mccl.sh`
3. `ansible ... -m copy -a 'src=/tmp/mccl.sh dest=/home/sensetime/mccl.sh ...'`
4. `ansible ... -m shell -a 'ls -l /home/sensetime/mccl.sh'`
5. `rm -f /tmp/mccl.sh`

写入后建议权限：

```bash
chmod 755 /home/sensetime/mccl.sh
chown sensetime:sensetime /home/sensetime/mccl.sh
```

---

## 4. 标准五轮八卡测试

确认后执行：

```bash
ansible all -i '<目标IP>,' -m shell -a 'cd ~sensetime && LOG=mccl-$(date +%Y%m%d-%H%M%S).log; for i in 1 2 3 4 5; do echo "===== MCCL RUN ${i} =====" | tee -a "$LOG"; sudo bash ./mccl.sh 8 2>&1 | tee -a "$LOG"; echo "rc=${PIPESTATUS[0]}" | tee -a "$LOG"; done; echo LOG=$LOG'
```

---

## 5. 查看 MCCL 日志

```bash
ansible all -i '<目标IP>,' -m shell -a 'cd ~sensetime && ls -lt mccl-*.log | head'
ansible all -i '<目标IP>,' -m shell -a 'cd ~sensetime && tail -n 200 <mccl-log-file>'
ansible all -i '<目标IP>,' -m shell -a "cd ~sensetime && grep -E 'Avg bus bandwidth|Out of bounds|error|failed|timeout|rc=' <mccl-log-file> | tail -200"
```

---

## 6. 判断重点

输出结果时优先保留原始关键行：

```text
# Avg bus bandwidth
# Out of bounds values : 0 OK
rc=0
error / failed / timeout / hang
```

---

## 7. MCCL 测试后残留进程清理

适用于“已纳管 MUXI 机器维修后验收”场景。

MCCL 测试无论成功、失败、超时、hang 还是被中断，放回集群前都必须执行残留进程核验。
如果发现残留，必须清理后再次核验。
只有核验输出 `MCCL_RESIDUAL_CLEAN`，才允许继续进入 `rayctl node uncordon` 确认流程。

`mx-smi --kill-all-process` 当前版本支持，但属于高风险清理动作，只能在确认目标节点是维修验收节点、且不承载业务任务后执行。

---

### 7.1 残留进程核验，只读

用于：

* MCCL 测试后检查。
* 清理后复查。
* 放回 / uncordon 前最终核验。

```bash
TARGET_IP='<目标IP>'

ansible all -i "${TARGET_IP}," -m shell -a '
set +e

PS_PROC_RE="all_reduce_perf|all_gather_perf|reduce_scatter_perf|sendrecv_perf|alltoall_perf|mpirun|orterun|orted|prted"
MX_PROC_RE="all_reduce_perf|all_gather_perf|reduce_scatter_perf|sendrecv_perf|alltoall_perf|mpirun|orterun|orted|prted|python3|python3.10"

echo "===== mx-smi normal ====="
mx-smi || true

echo "===== mx-smi show process ====="
mx-smi --show-process || true

echo "===== mx-smi show all process ====="
mx-smi --show-all-process || true

echo "===== MCCL / MPI linux processes ====="
LEFT_PS="$(
  ps -eo pid=,comm=,args= \
  | awk -v re="$PS_PROC_RE" '"'"'
      $0 ~ re && $2 !~ /^(bash|sh|zsh|awk|grep|pgrep|ansible|sudo)$/ {
        print
      }
    '"'"'
)"

if [ -n "$LEFT_PS" ]; then
  echo "MCCL_RESIDUAL_FOUND_IN_PS"
  echo "$LEFT_PS"
  exit 2
fi

if mx-smi --show-process 2>/dev/null | grep -E "$MX_PROC_RE"; then
  echo "MCCL_RESIDUAL_FOUND_IN_MX_SMI_SHOW_PROCESS"
  exit 3
fi

if mx-smi --show-all-process 2>/dev/null | grep -E "$MX_PROC_RE"; then
  echo "MCCL_RESIDUAL_FOUND_IN_MX_SMI_SHOW_ALL_PROCESS"
  exit 4
fi

echo "MCCL_RESIDUAL_CLEAN"
'
```

判断：

```text
MCCL_RESIDUAL_CLEAN
```

表示没有发现 MCCL / MPI / MUXI 设备占用残留。

如果输出以下任意结果，必须停止，不能放回集群：

```text
MCCL_RESIDUAL_FOUND_IN_PS
MCCL_RESIDUAL_FOUND_IN_MX_SMI_SHOW_PROCESS
MCCL_RESIDUAL_FOUND_IN_MX_SMI_SHOW_ALL_PROCESS
```

---

### 7.2 清理 MCCL / MUXI 设备残留进程

杀进程属于写操作 / 影响业务操作，必须确认：

* 目标节点是维修验收节点。
* 当前节点不承载业务任务。
* 用户同意清理 MUXI 设备占用进程。

确认后执行：

```bash
TARGET_IP='<目标IP>'

ansible all -i "${TARGET_IP}," -m shell -a '
set +e

PS_PROC_RE="all_reduce_perf|all_gather_perf|reduce_scatter_perf|sendrecv_perf|alltoall_perf|mpirun|orterun|orted|prted"

echo "===== before cleanup: mx-smi show process ====="
mx-smi --show-process || true

echo "===== before cleanup: mx-smi show all process ====="
mx-smi --show-all-process || true

echo "===== kill all MUXI device processes ====="
sudo mx-smi --kill-all-process
RC=$?
echo "mx-smi --kill-all-process rc=${RC}"

sleep 3

echo "===== after cleanup: mx-smi show process ====="
mx-smi --show-process || true

echo "===== after cleanup: mx-smi show all process ====="
mx-smi --show-all-process || true

echo "===== after cleanup: MCCL / MPI linux processes ====="
ps -eo pid=,comm=,args= \
| awk -v re="$PS_PROC_RE" '"'"'
    $0 ~ re && $2 !~ /^(bash|sh|zsh|awk|grep|pgrep|ansible|sudo)$/ {
      print
    }
  '"'"'

echo "CLEANUP_COMMAND_RC=${RC}"
'
```

清理后必须重新执行：

```text
7.1 残留进程核验，只读
```

只有重新核验输出：

```text
MCCL_RESIDUAL_CLEAN
```

才允许继续放回 / uncordon。

---

### 7.3 放回 / uncordon 前准入条件

放回集群前必须同时满足：

1. MCCL 测试通过。
2. 已执行测试后残留核验。
3. 如发现残留，已执行残留清理。
4. 清理后重新核验输出 `MCCL_RESIDUAL_CLEAN`。
5. 用户明确要求“放回 / uncordon”。
6. 已确认 Kubernetes node name。
7. 已完成 `rayctl node uncordon` 写操作确认。

如果残留核验不是 `MCCL_RESIDUAL_CLEAN`，必须停止，不得 uncordon。
