# Skills

可复用的操作指南。每个 skill 是一个目录，含 `SKILL.md`（带 YAML frontmatter）。

| Skill | 说明 |
|---|---|
| [k3s-ax-deploy](k3s-ax-deploy/SKILL.md) | 在单机 k3s 上部署 google/ax 及其硬依赖 Agent Substrate。覆盖 IPv6-only 集群、本地 registry、feature gate 缺失等场景。 |

## 目录结构

```
skills/
└── <skill-name>/
    ├── SKILL.md          # 必需。frontmatter: name, description
    ├── TROUBLESHOOTING.md # 可选。按报错关键字排错
    └── assets.md         # 可选。配置文件与来源
```

`SKILL.md` 的 frontmatter 里 `description` 决定何时被选中，要写清触发条件
（做什么、以及什么报错时该用它），不要只写标题。
