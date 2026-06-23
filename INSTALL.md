# INSTALL.md - OpenClaw Optimized Split Pack

这个版本把 `TOOLS.md` 真正拆成了 `tools/*.md`：

```text
openclaw_optimized_split_v2/
├── MEMORY.md
├── TOOLS.md                  # 只保留索引
├── tools/                    # 具体工具和命令模板
│   ├── README.md
│   ├── environment-entry.md
│   ├── rayctl-kubectl.md
│   ├── job-templates.md
│   ├── dcluster-ansible.md
│   ├── mccl-commands.md
│   ├── k8s-cleanup.md
│   └── fault-records.md
└── skills/                   # 专项 SOP
    ├── README.md
    ├── dcluster-machine-op/SKILL.md
    ├── k8s-cleanup/SKILL.md
    ├── mccl-test/SKILL.md
    ├── pvc-afs/SKILL.md
    └── vcjob-debug/SKILL.md
```

建议替换方式：

1. 备份当前 OpenClaw 配置目录。
2. 用本包的 `MEMORY.md` 替换原 `MEMORY.md`。
3. 用本包的 `TOOLS.md` 替换原 `TOOLS.md`。
4. 新增整个 `tools/` 目录。
5. 新增或替换整个 `skills/` 目录。

设计原则：

- `MEMORY.md`：长期原则、入口路由、安全红线。
- `TOOLS.md`：只做索引，不放长命令。
- `tools/*.md`：具体命令模板。
- `skills/*/SKILL.md`：专项流程。
- `memory/YYYY-MM-DD.md`：当天动态状态和查询结果。
