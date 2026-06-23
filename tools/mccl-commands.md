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
