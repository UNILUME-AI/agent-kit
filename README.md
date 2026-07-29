# agent-kit

UNILUME 跨仓共享的 AI agent 资产，以 Claude Code plugin marketplace 形式分发。

本仓存在的理由是**消除复制**：收录的内容在各业务仓里一份都不存，因此不存在漂移。
改这里一处，所有消费方下次加载即生效。

## 收录判据

三条同时成立才收：

1. 各仓逐字相同，无仓库特异性
2. 由 agent 按需加载（不是 CI 工具链，不是常驻上下文）
3. 移走后消费方仍读得到（不依赖某个仓内的相对路径）

已明确**不**收录的，以及原因：

| 资产 | 不收录的原因 |
| --- | --- |
| `AGENTS.md` | Codex 只读仓根的 `AGENTS.md`，移走它就失去评审基准；且 `AGENTS.md` 常驻上下文，plugin skill 按需加载，语义不等价 |
| `speckit-*` skills | 真源是 spec-kit CLI 的 core_pack 与 scrum 的 override 层，收录等于建第二个分发通道 |
| `doc-style` / `doc-style-feishu` | 引用 scrum 仓内 `governance/*.md` 三个文件，移出后路径失效 |
| `terraform-skill`、`unilume-iac-lessons` | 只有 IaC 仓需要 |
| `constitution-core.md` | 已有 `sync-constitution.sh` 分发 |
| `conventions` 包 | CI 工具链，不由 agent 加载，已有 npm 分发 |
| `claude.yml`、`cross-review.yml` | GitHub 只运行仓库内的事件 workflow，触发器必须留在本仓 |

## 消费方

### 本机 Claude Code

```
/plugin marketplace add UNILUME-AI/agent-kit
/plugin install unilume-workflow@agent-kit
```

之后 `/plugin update` 统一升级，各业务仓无需任何改动。

### CI 里的 @claude

在 `UNILUME-AI/.github` 的 `reusable-claude.yml` 里给 `claude-code-action` 加两个输入：

```yaml
plugin_marketplaces: https://github.com/UNILUME-AI/agent-kit.git
plugins: unilume-workflow@agent-kit
```

7 个业务仓的 `claude.yml` 都委派到该 reusable，改这一处即全部生效。

本仓当前为私有仓。`actions/checkout` 配置的 git 凭据只对当前仓库生效，
因此 CI 侧克隆本仓是否需要额外令牌，须以一次真实的 @claude 运行为准。

## 结构

```
.claude-plugin/marketplace.json        marketplace 清单
plugins/unilume-workflow/
├── .claude-plugin/plugin.json         plugin 清单
└── skills/
    └── pr-review-response/            处理 PR 评审意见
        ├── SKILL.md
        └── LICENSE                    工作流次序改编自 github/gh-aw（MIT）
```
