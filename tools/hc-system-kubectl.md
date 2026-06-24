# hc-system-kubectl.md

本文件只保存 host cluster 系统组件操作相关 kubectl 命令模板。

## 1. 变量约定

```bash
VC_NAME='<vc-name>'
VC_UUID='<vc-uuid>'
VC_UID='<vc-uid>'
HC_NS='<hc-namespace>'
COMPONENT='<component-name>'
POD='<pod-name>'
POD_UID='<pod-uid>'
OWNER_KIND='<Deployment-or-StatefulSet>'
OWNER_NAME='<owner-name>'
VC_KUBECONFIG='<path-to-vc-kubeconfig>'
HC_KUBECONFIG='<path-to-hc-kubeconfig>'
VERIFY_NS='<namespace-in-vc>'
VERIFY_POD='<pod-name-in-vc>'
LOG_SINCE='3h'
LOG_KEYWORD='<keyword>'
UBUNTU_POD='ubuntu'
TARGET_FLAVOR='<minimum|small|medium|large|unlimit>'
OLD_FLAVOR='<old-flavor>'
DB_USER='lepton_service'
DB_HOST='pg-public.infra.svc.cluster.local'
DB_PORT='5432'
DB_NAME='lepton_service'
DB_PASSWORD='<runtime-password-from-configmap>'
```

所有用户输入都必须放入变量并加引号。
grep 用户输入时必须使用 `grep -F -- "$VAR"`。

## 2. 确认 kubeconfig / context

查看当前 context：

```bash
kubectl --kubeconfig "$HC_KUBECONFIG" config current-context
kubectl --kubeconfig "$VC_KUBECONFIG" config current-context
```

查看当前 server：

```bash
kubectl --kubeconfig "$HC_KUBECONFIG" config view --minify -o jsonpath='{.clusters[0].cluster.server}{"\n"}'
kubectl --kubeconfig "$VC_KUBECONFIG" config view --minify -o jsonpath='{.clusters[0].cluster.server}{"\n"}'
```

查看 namespace：

```bash
kubectl --kubeconfig "$HC_KUBECONFIG" config view --minify -o jsonpath='{.contexts[0].context.namespace}{"\n"}'
kubectl --kubeconfig "$VC_KUBECONFIG" config view --minify -o jsonpath='{.contexts[0].context.namespace}{"\n"}'
```

提醒：

- HC 系统组件操作使用 `"$HC_KUBECONFIG"`
- VC 内业务现象验证使用 `"$VC_KUBECONFIG"`
- kubeconfig 类型不明确时必须停止

## 3. VC kubeconfig 下验证 get/list 异常

```bash
kubectl --kubeconfig "$VC_KUBECONFIG" get pod -n "$VERIFY_NS" "$VERIFY_POD"
kubectl --kubeconfig "$VC_KUBECONFIG" get pod -n "$VERIFY_NS"
kubectl --kubeconfig "$VC_KUBECONFIG" auth can-i get pods -n "$VERIFY_NS"
kubectl --kubeconfig "$VC_KUBECONFIG" auth can-i list pods -n "$VERIFY_NS"
```

这是只读验证，不是 HC 操作。

## 4. HC namespace 下定位 VC 控制面组件

只在 `"$HC_NS"` 下查询：

```bash
kubectl --kubeconfig "$HC_KUBECONFIG" -n "$HC_NS" get pod -o wide
kubectl --kubeconfig "$HC_KUBECONFIG" -n "$HC_NS" get pod -o wide | grep -F -- "$COMPONENT"
kubectl --kubeconfig "$HC_KUBECONFIG" -n "$HC_NS" get pod "$POD" -o wide
kubectl --kubeconfig "$HC_KUBECONFIG" -n "$HC_NS" describe pod "$POD"
kubectl --kubeconfig "$HC_KUBECONFIG" -n "$HC_NS" get events --sort-by=.lastTimestamp | grep -F -- "$POD"
```

控制面日志只读模板：

```bash
for p in $(kubectl --kubeconfig "$HC_KUBECONFIG" -n "$HC_NS" get pod -o name | grep '^pod/vc-'); do
  echo
  echo "================ $p syncer 日志 ================"
  kubectl --kubeconfig "$HC_KUBECONFIG" -n "$HC_NS" logs "$p" -c syncer --since="$LOG_SINCE" 2>/dev/null \
    | grep -F -- "$LOG_KEYWORD" \
    | tail -120

  echo
  echo "================ $p agent 日志 ================"
  kubectl --kubeconfig "$HC_KUBECONFIG" -n "$HC_NS" logs "$p" -c agent --since="$LOG_SINCE" 2>/dev/null \
    | grep -F -- "$LOG_KEYWORD" \
    | tail -120
done
```

查 ownerReferences：

```bash
kubectl --kubeconfig "$HC_KUBECONFIG" -n "$HC_NS" get pod "$POD" -o jsonpath='{.metadata.ownerReferences[0].kind}{"\n"}{.metadata.ownerReferences[0].name}{"\n"}{.metadata.ownerReferences[0].uid}{"\n"}'
```

## 5. 查询 owner / replicas / selector

查询 Deployment：

```bash
kubectl --kubeconfig "$HC_KUBECONFIG" -n "$HC_NS" get deployment "$OWNER_NAME" -o wide
kubectl --kubeconfig "$HC_KUBECONFIG" -n "$HC_NS" describe deployment "$OWNER_NAME"
```

查询 StatefulSet：

```bash
kubectl --kubeconfig "$HC_KUBECONFIG" -n "$HC_NS" get statefulset "$OWNER_NAME" -o wide
kubectl --kubeconfig "$HC_KUBECONFIG" -n "$HC_NS" describe statefulset "$OWNER_NAME"
```

查询 replicas / readyReplicas / availableReplicas：

```bash
kubectl --kubeconfig "$HC_KUBECONFIG" -n "$HC_NS" get "$OWNER_KIND" "$OWNER_NAME" -o jsonpath='{.spec.replicas}{"\n"}{.status.replicas}{"\n"}{.status.readyReplicas}{"\n"}{.status.availableReplicas}{"\n"}'
```

查询 selector：

```bash
kubectl --kubeconfig "$HC_KUBECONFIG" -n "$HC_NS" get "$OWNER_KIND" "$OWNER_NAME" -o jsonpath='{.spec.selector.matchLabels}{"\n"}'
```

根据 selector 查关联 Pod：

```bash
SELECTOR='<key1=value1,key2=value2>'

kubectl --kubeconfig "$HC_KUBECONFIG" -n "$HC_NS" get pod -l "$SELECTOR" -o wide
```

## 6. 确认后删除控制面 Pod

删除前重新读取 Pod UID：

```bash
CURRENT_UID="$(kubectl --kubeconfig "$HC_KUBECONFIG" -n "$HC_NS" get pod "$POD" -o jsonpath='{.metadata.uid}')"
printf 'expected=%s\ncurrent=%s\n' "$POD_UID" "$CURRENT_UID"
```

UID 一致才允许继续。

```bash
# 确认后执行
kubectl --kubeconfig "$HC_KUBECONFIG" -n "$HC_NS" delete pod "$POD"
```

等待新 Pod Ready：

```bash
kubectl --kubeconfig "$HC_KUBECONFIG" -n "$HC_NS" get pod -o wide | grep -F -- "$COMPONENT"
kubectl --kubeconfig "$HC_KUBECONFIG" -n "$HC_NS" wait --for=condition=Ready pod -l "$SELECTOR" --timeout=180s
```

## 7. Flavor 调整相关模板

确认 `ubuntu` Pod：

```bash
HC_NS='prod-lepton'
UBUNTU_POD='ubuntu'

kubectl --kubeconfig "$HC_KUBECONFIG" -n "$HC_NS" get pod "$UBUNTU_POD" -o wide
```

查询数据库密码来源：

```bash
kubectl --kubeconfig "$HC_KUBECONFIG" -n "$HC_NS" get cm lepton-ecp-service -o yaml | grep -i password
```

进入 `ubuntu` Pod：

```bash
kubectl --kubeconfig "$HC_KUBECONFIG" -n "$HC_NS" exec -it "$UBUNTU_POD" -- bash
```

可选非交互模板：

```bash
kubectl --kubeconfig "$HC_KUBECONFIG" -n "$HC_NS" exec "$UBUNTU_POD" -- bash -lc 'env | grep -i PG'
```

连接数据库：

```bash
pgcli "postgres://${DB_USER}:${DB_PASSWORD}@${DB_HOST}:${DB_PORT}/${DB_NAME}"
```

只读查询当前 flavor：

```sql
select uid, flavor
from virtual_cluster
where not deleted
  and uid = '<VC_UID>';
```

允许值：

```text
minimum
small
medium
large
unlimit
```

`unlimit` 不建议使用；除非用户明确指定并二次确认，否则不要执行。

确认后执行 update：

```sql
-- 确认后执行
update virtual_cluster
set flavor = '<TARGET_FLAVOR>'
where not deleted
  and uid = '<VC_UID>'
returning uid, flavor;
```

回滚 SQL 模板：

```sql
-- 确认后执行
update virtual_cluster
set flavor = '<OLD_FLAVOR>'
where not deleted
  and uid = '<VC_UID>'
returning uid, flavor;
```

执行后复核：

```sql
select uid, flavor
from virtual_cluster
where not deleted
  and uid = '<VC_UID>';
```

## 8. 操作后复核

查新 Pod Ready：

```bash
kubectl --kubeconfig "$HC_KUBECONFIG" -n "$HC_NS" get pod -o wide | grep -F -- "$COMPONENT"
kubectl --kubeconfig "$HC_KUBECONFIG" -n "$HC_NS" get "$OWNER_KIND" "$OWNER_NAME" -o wide
```

查 events：

```bash
kubectl --kubeconfig "$HC_KUBECONFIG" -n "$HC_NS" get events --sort-by=.lastTimestamp | tail -50
kubectl --kubeconfig "$HC_KUBECONFIG" -n "$HC_NS" get events --sort-by=.lastTimestamp | grep -F -- "$OWNER_NAME"
```

用 VC kubeconfig 复测 get/list：

```bash
kubectl --kubeconfig "$VC_KUBECONFIG" get pod -n "$VERIFY_NS" "$VERIFY_POD"
kubectl --kubeconfig "$VC_KUBECONFIG" get pod -n "$VERIFY_NS"
```
