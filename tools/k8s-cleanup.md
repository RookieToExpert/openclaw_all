# k8s-cleanup.md - Kubernetes 清理类命令模板

详细流程和确认机制见 `skills/k8s-cleanup/SKILL.md`。
本文件只维护具体命令模板。

## 1. Failed Pod 只读预览

```bash
export KUBECONFIG=/root/D/<实际 kubeconfig 文件名>

kubectl get pod -A -o json | jq -r '
  .items[]
  | select(.status.phase=="Failed")
  | select(.metadata.creationTimestamp >= "<START-UTC>" and .metadata.creationTimestamp < "<END-UTC>")
  | [.metadata.namespace,.metadata.name,.metadata.creationTimestamp,.status.phase]
  | @tsv
' > /tmp/pods_to_delete.tsv

wc -l /tmp/pods_to_delete.tsv
head -20 /tmp/pods_to_delete.tsv
```

## 2. Failed Pod 删除，确认后执行

默认串行：

```bash
while IFS=$'\t' read -r ns pod ts phase; do
  kubectl delete pod "$pod" -n "$ns" --wait=false --ignore-not-found
done < /tmp/pods_to_delete.tsv
```

并发仅在用户确认并发数后使用：

```bash
cat /tmp/pods_to_delete.tsv | xargs -r -P 20 -n 4 bash -c '
  kubectl delete pod "$2" -n "$1" --wait=false --ignore-not-found
' _
```

---

## 3. 历史 vcjob TTL 清理：北京时间转 UTC

例如北京时间 2026-05-01 到 2026-06-01 对应：

```text
START_UTC=2026-04-30T16:00:00Z
END_UTC=2026-05-31T16:00:00Z
```

不得把北京时间日期直接写进 `creationTimestamp` 条件。

## 4. vcjob 只读预览、清单和快照

```bash
export KUBECONFIG=/root/D/<实际 kubeconfig 文件名>

START_UTC=<START-UTC>
END_UTC=<END-UTC>
RAW=/tmp/vcjobs-pre.json
LIST=/tmp/vcjobs-ttl.tsv

kubectl get vcjob -A -o json > "$RAW"

jq -r --arg start "$START_UTC" --arg end "$END_UTC" '
  .items[]
  | select(.metadata.creationTimestamp >= $start and .metadata.creationTimestamp < $end)
  | select(.status.state.phase != "Running" and .status.state.phase != "Pending")
  | [.metadata.namespace,.metadata.name,.status.state.phase,
     .metadata.creationTimestamp,(.spec.ttlSecondsAfterFinished // "UNSET")]
  | @tsv
' "$RAW" > "$LIST"

echo TOTAL; wc -l "$LIST"
echo PHASE_COUNTS; cut -f3 "$LIST" | sort | uniq -c | sort -nr
echo TTL_COUNTS; cut -f5 "$LIST" | sort | uniq -c | sort -nr
echo SAMPLE; head -20 "$LIST"
```

反向检查同一窗口内 Running / Pending：

```bash
jq --arg start "$START_UTC" --arg end "$END_UTC" '
  [.items[]
    | select(.metadata.creationTimestamp >= $start and .metadata.creationTimestamp < $end)
    | select(.status.state.phase == "Running" or .status.state.phase == "Pending")
  ] | length
' "$RAW"
```

候选对象审计备份：

```bash
jq --arg start "$START_UTC" --arg end "$END_UTC" '
  {apiVersion:"v1",kind:"List",items:[
    .items[]
    | select(.metadata.creationTimestamp >= $start and .metadata.creationTimestamp < $end)
    | select(.status.state.phase != "Running" and .status.state.phase != "Pending")
  ]}
' "$RAW" > /tmp/vcjobs-ttl-backup.json
```

备份用于审计，不保证可以直接恢复已删除对象。

## 5. 确认后串行 TTL patch

执行前必须重新生成实时清单，并确认数量和 phase 与用户确认时一致。

```bash
PATCH_LOG=/tmp/vcjobs-ttl-patch.log
: > "$PATCH_LOG"

TOTAL=$(wc -l < "$LIST" | tr -d ' ')
i=0; ok=0; fail=0

while IFS=$'\t' read -r NS JOB PHASE CREATED TTL; do
  CURRENT=$(kubectl -n "$NS" get vcjob "$JOB" \
    -o jsonpath='{.status.state.phase}' 2>/dev/null) || continue

  case "$CURRENT" in
    Running|Pending)
      echo "SKIP $NS/$JOB phase=$CURRENT" | tee -a "$PATCH_LOG"
      continue
      ;;
  esac

  i=$((i + 1))
  if kubectl -n "$NS" patch vcjob "$JOB" \
      --type=merge \
      -p '{"spec":{"ttlSecondsAfterFinished":60}}' \
      >> "$PATCH_LOG" 2>&1; then
    ok=$((ok + 1))
  else
    fail=$((fail + 1))
    echo "PATCH_FAILED $NS/$JOB" >> "$PATCH_LOG"
  fi

  if [ $((i % 100)) -eq 0 ] || [ "$i" -eq "$TOTAL" ]; then
    echo "PATCH_PROGRESS $i/$TOTAL ok=$ok fail=$fail $(date -Is)"
  fi
done < "$LIST"
```

历史任务设置 TTL=60 后可能立即被删除。大批量操作建议在 tmux 中执行：

```bash
tmux new -s vcjob-cleanup
# 执行已确认的 patch 脚本
# Ctrl-b d 保持后台运行
tmux attach -t vcjob-cleanup
```

## 6. SSH 中断后的续查

```bash
pgrep -af 'kubectl.*patch vcjob|vcjob-cleanup' || true
wc -l /tmp/vcjobs-ttl-patch.log
grep -c '^PATCH_FAILED ' /tmp/vcjobs-ttl-patch.log || true
grep -n -B4 -A2 '^PATCH_FAILED ' /tmp/vcjobs-ttl-patch.log
tail -20 /tmp/vcjobs-ttl-patch.log
```

不要直接重跑全量；先拿原始清单与当前对象做集合比对，只补仍存在且 TTL 未设置的对象。

## 7. TTL 后按原始清单核验 vcjob 和 Pod

```bash
sleep 120
kubectl get vcjob -A -o json > /tmp/vcjobs-post.json
kubectl get pod -A -o json > /tmp/pods-post.json
```

```bash
python3 - "$LIST" /tmp/vcjobs-post.json /tmp/pods-post.json <<'PY'
import json, sys
from collections import Counter

list_path, jobs_path, pods_path = sys.argv[1:]
targets = set()
names = set()
for line in open(list_path):
    ns, name, *_ = line.rstrip("\n").split("\t")
    targets.add((ns, name)); names.add(name)

jobs = json.load(open(jobs_path)).get("items", [])
remaining_jobs = [x for x in jobs if
    (x["metadata"]["namespace"], x["metadata"]["name"]) in targets]

pods = json.load(open(pods_path)).get("items", [])
remaining_pods = []
for x in pods:
    md = x.get("metadata", {})
    labels = md.get("labels", {}) or {}
    owners = [o.get("name") for o in md.get("ownerReferences", [])]
    if labels.get("volcano.sh/job-name") in names or any(o in names for o in owners):
        remaining_pods.append(x)

print("REMAINING_VCJOBS", len(remaining_jobs))
print("REMAINING_PHASES", Counter(
    ((x.get("status", {}).get("state") or {}).get("phase") or "Unknown")
    for x in remaining_jobs))
print("REMAINING_TTL", Counter(
    str(x.get("spec", {}).get("ttlSecondsAfterFinished"))
    for x in remaining_jobs))
print("REMAINING_RELATED_PODS", len(remaining_pods))
for x in remaining_jobs[:20]:
    print("VCJOB", x["metadata"]["namespace"], x["metadata"]["name"])
for x in remaining_pods[:20]:
    print("POD", x["metadata"]["namespace"], x["metadata"]["name"])
PY
```

必须使用原始清单比对，不能只重新按时间/状态筛选。

## 8. Aborted 未被 TTL 删除时检查

```bash
kubectl get vcjob <job-name> -n <namespace> -o json | jq '{
  name:.metadata.name,
  ttl:.spec.ttlSecondsAfterFinished,
  phase:.status.state.phase,
  state:.status.state,
  conditions:.status.conditions,
  completionTime:.status.completionTime,
  finishedAt:.status.finishedAt,
  deletionTimestamp:.metadata.deletionTimestamp,
  finalizers:.metadata.finalizers
}'
```

若对象为 `Aborted`、TTL 已设置，但没有 `completionTime` / `finishedAt` 且长期没有 deletionTimestamp，TTL controller 无法计算过期时间。不要反复 patch。

保存剩余 Aborted 清单：

```bash
jq -r '
  .items[]
  | select(.status.state.phase=="Aborted")
  | select(.spec.ttlSecondsAfterFinished==60)
  | [.metadata.namespace,.metadata.name,.status.state.phase,.metadata.creationTimestamp]
  | @tsv
' /tmp/vcjobs-post.json > /tmp/vcjobs-aborted-remaining.tsv

wc -l /tmp/vcjobs-aborted-remaining.tsv
head -20 /tmp/vcjobs-aborted-remaining.tsv
```

## 9. Aborted 直接删除，必须重新确认

确认后执行：

```bash
while IFS=$'\t' read -r NS JOB PHASE CREATED; do
  kubectl -n "$NS" delete vcjob "$JOB" --ignore-not-found --wait=false
done < /tmp/vcjobs-aborted-remaining.tsv
```

删除后再次按原始清单核验 vcjob 和关联 Pod。