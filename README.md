# agent-kit

UNILUME 跨仓共享的 AI agent 资产，以 Claude Code plugin marketplace 形式分发。

本仓存在的理由是**消除复制**：收录的内容在各业务仓里一份都不存，因此不存在漂移。

生效方式两条路不同，别混：本机 Claude Code 下次 `/plugin update` 即生效；**CI 侧不会自动跟随**——
`reusable-claude.yml` 把本仓钉在一个 commit SHA 上，改这里不 bump 那个 SHA，@claude 读到的仍是旧内容。
这是有意的，理由见下面「消费方 · CI 里的 @claude」。

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

`UNILUME-AI/.github` 的 `reusable-claude.yml` 先按 SHA 把本仓取到 `RUNNER_TEMP`，再把**本地路径**
交给 `claude-code-action`：

```yaml
- name: Fetch shared agent skills (pinned)
  env:
    AGENT_KIT_SHA: <40 位 SHA> # main YYYY-MM-DD
  run: |
    git init -q "$RUNNER_TEMP/agent-kit"
    git -C "$RUNNER_TEMP/agent-kit" remote add origin https://github.com/UNILUME-AI/agent-kit.git
    git -C "$RUNNER_TEMP/agent-kit" fetch -q --depth 1 origin "$AGENT_KIT_SHA"
    git -C "$RUNNER_TEMP/agent-kit" checkout -q FETCH_HEAD

- uses: anthropics/claude-code-action@<sha>
  with:
    plugin_marketplaces: ${{ runner.temp }}/agent-kit
    plugins: unilume-workflow@agent-kit
```

**为什么不直接传 git URL。** action 的 `install-plugins.ts` 只是把输入原样交给
`claude plugin marketplace add`，对 URL 形式不接受任何 ref/tag（校验正则要求以 `.git` 结尾），
因此 URL 形式无法钉版本。而本仓的内容会被注入一个持有 `contents: write`、放行
`Bash(git:*)` / `Bash(gh:*)` 的特权 job——skill 是能左右 agent 行为的文本，引用可变分支
等于让本仓的任何改动未经消费方评审就进入那个 job。本地路径形式由上一步按 SHA 取好，是不可变的。

**升级步骤**（两步，缺一不可）：

1. 本仓改动合入 main，取新 SHA：`gh api repos/UNILUME-AI/agent-kit/commits/main --jq .sha`
2. 改 `reusable-claude.yml` 里的 `AGENT_KIT_SHA`（走该仓评审），各业务仓再 bump 自己钉的
   reusable SHA

16 个业务仓的 `claude.yml` 按 SHA 跨仓引用该 reusable，`.github` 自身走 `./` 本地路径引用。

**令牌问题已结**：本仓为私有仓，曾存疑 CI 侧克隆是否需要额外令牌。2026-08-02 在 platform 上
三次真实 @claude 运行中，`Fetch shared agent skills` 均在 1 秒内成功，无需额外配置——
job 默认的 `GITHUB_TOKEN` 对同 org 私有仓的 fetch 是够的。

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
