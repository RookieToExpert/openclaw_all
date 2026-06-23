# update-openclaw-memory Skill

## 触发条件

当用户说以下任一触发语时使用本 skill：

- 更新 openclaw memory
- 更新 OpenClaw 知识库
- 同步 OpenClaw 配置
- git pull openclaw_all

## 目标

同步 OpenClaw 本地知识库 / tools / skills / 配置仓库，并把本次更新涉及的文件返回给用户。

## 执行步骤

### 1. 进入仓库

```bash
cd ~/openclaw_all
```

### 2. 更新前检查工作区状态

```bash
git status
```

如果存在未提交修改，先把状态返回给用户，再决定是否继续执行 `git pull --ff-only`。

### 3. 拉取最新内容

```bash
git pull --ff-only
```

如果失败，原样返回失败信息，不要改用 `git pull`、`git reset` 或其他更激进的方式。

### 4. 更新后检查变更

```bash
git status --short
```

必要时补充：

```bash
git diff --name-only HEAD@{1} HEAD
```

用于告诉用户本次更新涉及了哪些文件。

### 5. 向用户汇报

输出至少包含：

- pull 是否成功
- 更新了哪些文件
- 是否存在本地未提交修改

### 6. 重启 gateway

```bash
openclaw gateway restart
```

## 输出格式

```text
结论：
更新结果：
- git pull ...
- 更新文件 ...
当前工作区状态：
下一步：
```

## 禁止事项

- 不要在不是 `~/openclaw_all` 的目录执行该流程。
- 不要把 `git pull --ff-only` 擅自改成普通 `git pull`。
- 不要使用 `git reset --hard`、`git checkout --` 之类破坏性命令。
- 不要省略 `openclaw gateway restart`。
