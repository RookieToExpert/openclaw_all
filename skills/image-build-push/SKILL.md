# image-build-push Skill

## 触发条件

用于以下镜像制作 / 打标 / 打驱动 / push 场景：

- 打 MUXI 相关镜像。
- 从远端镜像仓库 `docker pull` 后重新 tag / push。
- 从下载链接 `wget` 镜像压缩包后 `docker load`、重新 tag / push。
- 给 MUXI 镜像打云脉网卡驱动。
- 非 MUXI 镜像按用户指定 ccr 重新 tag / push。

不适用场景：

- Kubernetes / vcluster / rayctl / kubectl 查询与写操作。
- 物理机 ansible 单 IP 查询。
- VC 控制面重置、VC flavor、HC policy 更新。

## 相关工具文件

- `tools/image-build-push.md`
- `tools/environment-entry.md`

## 前置原则

- 这类操作走堡垒机内 `3.216` 入口，不走开发机，不走 `220.33` 跳板机 ansible。
- 镜像制作、`docker pull`、`docker load`、`docker tag`、`docker push`、执行打驱动脚本都属于写操作，必须确认。
- 不明确镜像来源、目标 tag、目标 ccr、是否为 MUXI 时必须停止。
- 非 MUXI 镜像的目标 ccr 必须由用户明确指定，不要擅自猜。
- MUXI 镜像默认需要打云脉网卡驱动，产物镜像名为原镜像后缀 `-driver`。
- `wget` 下载必须在 `/data/xie` 下执行。
- 不要把下载 URL 中的临时鉴权参数、密码、token 写入知识库。

## 场景选择

### 场景 A：MUXI 镜像，来源为 `docker pull`

### 1. 收集必要信息

必须明确：

- 源镜像完整地址。
- 镜像名和 tag。
- 目标仓库固定为 `registry2.d.pjlab.org.cn/ccr-ailabdev/`。

### 2. 只读确认

先确认：

- 这是 MUXI 镜像。
- 将使用堡垒机内 `3.216` 入口。
- 打驱动脚本路径存在。

### 3. 写操作确认

确认前必须输出：

- 源镜像地址。
- 中间 tag 地址。
- 最终带 `-driver` 的目标地址。
- 将执行 `docker pull`、`docker tag`、驱动脚本、`docker push`。
- 风险：拉取大镜像、覆盖同名 tag、push 到错误仓库。
- 回滚方式：删除错误 tag，停止后不要继续 push。

### 4. 确认后执行

按 `tools/image-build-push.md` → 3. MUXI 镜像：`docker pull` 路径执行。

---

### 场景 B：MUXI 镜像，来源为 `wget + docker load`

### 1. 收集必要信息

必须明确：

- 下载 URL。
- 下载文件名。
- `docker load -i` 使用的实际文件名。
- load 后得到的原始镜像名和 tag。

### 2. 只读确认

先确认：

- 下载目录固定为 `/data/xie`。
- 这是 MUXI 镜像。
- 后续仍需打云脉网卡驱动并 push 到 `registry2.d.pjlab.org.cn/ccr-ailabdev/`。

### 3. 写操作确认

确认前必须输出：

- 下载 URL 和保存文件名。
- `docker load` 输入文件。
- 中间 tag 地址。
- 最终带 `-driver` 的目标地址。
- 风险：下载大文件、load 后镜像名与预期不一致、push 到错误仓库。

### 4. 确认后执行

按 `tools/image-build-push.md` → 4. MUXI 镜像：`wget + docker load` 路径执行。

---

### 场景 C：非 MUXI 镜像

### 1. 收集必要信息

必须明确：

- 源镜像完整地址，或 `docker load` 后镜像名。
- 目标 ccr 完整前缀。
- 目标镜像名和 tag。

如果用户没有明确目标 ccr，必须停止。

### 2. 只读确认

先确认：

- 这是非 MUXI 镜像。
- 不需要打云脉网卡驱动。

### 3. 写操作确认

确认前必须输出：

- 源镜像地址。
- 目标镜像地址。
- 将执行 `docker pull` 或 `docker load`、`docker tag`、`docker push`。
- 风险：push 到错误 ccr、覆盖错误 tag。

### 4. 确认后执行

按 `tools/image-build-push.md` → 5. 非 MUXI 镜像路径执行。

## 输出要求

- 先给结论，再给依据。
- 明确区分源镜像、中间镜像、最终镜像。
- 明确区分已确认事实、待确认信息、待执行写操作。

## 禁止事项

- 不要把这类镜像操作发到开发机。
- 不要把这类镜像操作发到 `220.33` 跳板机 ansible 入口。
- 不要在未确认是 MUXI 还是非 MUXI 前生成命令。
- 不要在非 MUXI 场景下自动打驱动。
- 不要在 `wget` 场景下离开 `/data/xie` 下载。
- 不要把临时下载 URL、鉴权参数、密码写入知识库。
- 当前 SOP 未定义下一步时必须停止。
