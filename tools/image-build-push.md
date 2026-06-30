# image-build-push.md

本文件只保存镜像制作 / 打标 / 打驱动 / push 命令模板。
对应 skill：`skills/image-build-push/SKILL.md`

## 1. 变量约定

```bash
SOURCE_IMAGE='<source-image>'
SOURCE_FILE='<downloaded-file>'
DOWNLOAD_URL='<download-url>'
DOWNLOAD_DIR='/data/xie'
IMAGE_NAME_TAG='<image-name-and-tag>'
TARGET_IMAGE='registry2.d.pjlab.org.cn/ccr-ailabdev/<image-name-and-tag>'
TARGET_IMAGE_DRIVER='registry2.d.pjlab.org.cn/ccr-ailabdev/<image-name-and-tag>-driver'
TARGET_CCR='<user-specified-ccr-prefix>/<image-name-and-tag>'
DRIVER_BUILD_SCRIPT='/home/test6/tool/mcr-xsc-providers/build.sh'
DRIVER_SOURCE_DIR='/home/test6/tool/mcr-xsc-providers/xsc-providers'
```

所有用户输入都必须放入变量并加引号。

## 2. 入口与基础准备

先进入堡垒机：

```bash
ssh -p 5906 test6@10.140.3.216
```

进入堡垒机内镜像操作入口：

```text
3.216
```

进入 root：

```bash
sudo -i
```

## 3. MUXI 镜像：`docker pull` 路径

拉取镜像：

```bash
# 确认后执行
docker pull "$SOURCE_IMAGE"
```

重新 tag 到固定仓库：

```bash
# 确认后执行
docker tag "$SOURCE_IMAGE" "$TARGET_IMAGE"
```

打云脉网卡驱动：

```bash
# 确认后执行
"$DRIVER_BUILD_SCRIPT" \
  "$TARGET_IMAGE" \
  "$DRIVER_SOURCE_DIR" \
  "$TARGET_IMAGE_DRIVER"
```

push 带驱动镜像：

```bash
# 确认后执行
docker push "$TARGET_IMAGE_DRIVER"
```

## 4. MUXI 镜像：`wget + docker load` 路径

进入下载目录：

```bash
cd "$DOWNLOAD_DIR"
```

下载：

```bash
# 确认后执行
sudo wget -c -O "$SOURCE_FILE" "$DOWNLOAD_URL"
```

load 镜像：

```bash
# 确认后执行
sudo docker load -i "$SOURCE_FILE"
```

重新 tag 到固定仓库：

```bash
# 确认后执行
docker tag "$SOURCE_IMAGE" "$TARGET_IMAGE"
```

打云脉网卡驱动：

```bash
# 确认后执行
"$DRIVER_BUILD_SCRIPT" \
  "$TARGET_IMAGE" \
  "$DRIVER_SOURCE_DIR" \
  "$TARGET_IMAGE_DRIVER"
```

push 带驱动镜像：

```bash
# 确认后执行
docker push "$TARGET_IMAGE_DRIVER"
```

## 5. 非 MUXI 镜像

如果来源是 `docker pull`：

```bash
# 确认后执行
docker pull "$SOURCE_IMAGE"
```

如果来源是本地 tar：

```bash
cd "$DOWNLOAD_DIR"

# 确认后执行
sudo docker load -i "$SOURCE_FILE"
```

重新 tag 到用户指定 ccr：

```bash
# 确认后执行
docker tag "$SOURCE_IMAGE" "$TARGET_CCR"
```

push：

```bash
# 确认后执行
docker push "$TARGET_CCR"
```
