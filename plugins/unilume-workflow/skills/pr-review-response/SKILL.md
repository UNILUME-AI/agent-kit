---
name: pr-review-response
description: 处理 PR 上的评审意见时使用，包括 Codex、@claude 或人类留下的 review thread。覆盖收集、分桶、修复、回复、解决线程的完整次序，以及本仓实测过的 gh 命令。
metadata:
  scope: 本仓自有，工作流部分改编自 github/gh-aw 的 copilot-review skill（MIT，commit 4fedbacec289）
  verified: 2026-07-29
---

# 处理 PR 评审意见

本仓的评审由组织级 Codex connector 自动发起，修复由 Claude 完成。
本文件规定修复方的次序与收尾动作。未实测的推断不要写入本文件。

## 1. 先收集齐再动手

**在看懂全部诉求之前逐条回应，会导致回复描述的是中间状态。**

一次性取全部未解决的 review thread：

```bash
gh api graphql -f query='
query($o:String!,$r:String!,$n:Int!){repository(owner:$o,name:$r){pullRequest(number:$n){
  reviewThreads(first:100){nodes{
    id isResolved isOutdated path line
    comments(first:10){nodes{author{login} body}}}}}}}' \
  -f o=UNILUME-AI -f r=<repo> -F n=<pr> \
  --jq '.data.repository.pullRequest.reviewThreads.nodes[] | select(.isResolved|not)'
```

`gh pr view --json` **没有** `reviewThreads` 字段（gh 2.96.0 实测报 `Unknown JSON field`），
review thread 只能经 GraphQL 取得。线程标识形如 `PRRT_...`，与评论标识 `PRRC_...` 不是同一个。

组织级 GitHub MCP server 也提供 `resolve_review_thread`，但本机 `plugin:github` 缺
`GITHUB_PERSONAL_ACCESS_TOKEN`、连接失败，因此以上面的 `gh` 路径为准。

## 2. 分桶后再定计划

把线程归入：正确性 / 测试 / 文档 / 表达 / CI / 重复 / 不予修改。
每桶只有三种处置：照做、部分采纳、给出理由后拒绝。
**不允许对范围内的意见默不作声**——不改可以，不吭声不行。

## 2.5 打标入账本，评估是否熔断（分桶之后、动手修复之前）

固定动作，每轮必做。

**运行时作用域与真源位置**（评审修正：真源必须对执行环境可达）——本节的语义层只覆盖
**本机开发会话**（设计 §11：无人值守 CI 不跑长评审循环，硬层另有 pre-push 兜底）：

| 真源 | 位置 |
|---|---|
| 设计文档（阈值、窗口、流程） | <https://github.com/UNILUME-AI/scrum/blob/main/governance/review-fuse-breaker-design-2026-08-16.md> |
| taxonomy（class id 与「设计类」列） | 本机 `~/Documents/unilume-claude-palantir/harness优化/github代码review经验/taxonomy.md` |
| 账本 schema | 同目录 `templates/round-ledger.md` |

任一真源不可达（目录不存在、链接 404）→ 跳过语义层并在汇报中写明「语义熔断本轮未生效：
<原因>」——门没跑与门通过必须可区分，不许静默降级。

四步：

1. **打标**：每条发现标三项——`class`（taxonomy.md 的 id，单一真源勿造新值）、
   `form`（判据形态粗桶，只有四个值：文本／运行时／流程／判断）、`severity`
   （应修／建议／误报）。误报也打标入账，按打标时点定格，证伪后不追溯改轮。
2. **写入账本（只增，CG-8）**：每轮新建一条评论，首行 `<!-- review-round v1
   head:<sha> -->`，字段 schema 真源：沉淀流程目录 `templates/round-ledger.md`。
   不维护单条可变总账——GitHub 评论没有条件写原语，「先读再写回」不构成互斥；
   只增评论让不同轮永不共享一条评论，丢失更新结构上不存在。重复处理同轮：已有
   **自己发的**同 head 评论就地编辑之，否则新建。读方聚合规则（2026-08-16 评审修正）：
   同 head 多条评论取 **findings 并集**（按 thread URL 去重），不取最新——两会话各持
   不同快照写同一轮时，取最新会丢掉另一条里的发现，并集单调不丢证据；轮数计数不受
   影响（head 去重后仍是一轮）。创建失败重试一次，仍失败按「真源不可达」降级——
   跳过语义层并显式声明。评估 R2/R3 前先聚合全部轮评论为账本视图。
3. **评估 R2/R3**——窗口与阈值**以设计 §5 为准，本文不复写数值**（改一处就漂）；
   判定按「近 N 轮累计」不要求连续，A-B-A-B 交错正是要堵的辩解句式：
   - R2：同一 `form` 粗桶反复——修实例没有缩小类；
   - R3：taxonomy「设计类」列成员反复——问题在结构不在位置。
4. **命中任一 → 跳过修复，直接熔断**：停掉本 PR 的 /loop（CronDelete）；PR 转草稿；
   当场写 `status: fused` 复盘到沉淀流程 `prs/` 目录；发形态重估请求（证据表须**并列
   硬计数与账本轮数两个数字并注明口径**，两数不一致时口径差列为必答项；必答题：
   「反复出现的这类缺陷，判的是文本的性质还是运行时的性质」；三条路 A 继续修／B 换
   判据形态／C 撤回，各附一句代价）；然后等待人工裁决。该评论同时作为对未回复评审
   意见的集中回应——「不吭声不行」的义务经此转移到裁决之后。

R1（硬轮数阈值见设计 §5.1，含棘轮规则）不归本 skill 判——pre-push 与 merge 点的确定性拦截自会生效；
被拦时照 deny 文案执行，不要试图绕过（清除标签只能由人在网页加）。

## 3. 拒绝要有证据，不要为迎合而加代码

**Codex 会跨轮重复提同一条误报，包括虚构不存在的测试。**

- 据 grep 或实跑结果终结该条，把证据写进回复。
- 不要为了让告警消失而加死代码或包装层。
- `verdict` 不是必需检查，不会拦合并，误报不必让步。

判据：如果这条意见成立，能否给出一个会失败的具体输入或命令。给不出就是误报。

## 4. 验证之后才回复

改完先重看 diff 并跑相应验证，确保回复描述的是最终状态而非打算。

## 5. 逐条回复，即使一个修复覆盖多条

每条范围内的评论都要一条直接回复，说明下列之一：改了什么、修复落在哪里、
为什么没改、为什么已被另一处改动覆盖。

多条意见由同一个 commit 一并解决时，仍然逐条回复，不要只在其中一条下说明。

## 6. 回复之后才解决线程

**不要在没有回答之前解决线程。**先 push 修复，再回复，最后 resolve——
顺序颠倒等于对评审人声称已修而实际未推。

一次调用完成回复与解决：

```bash
gh api graphql -f query='
mutation($t:ID!,$b:String!){
  addPullRequestReviewThreadReply(input:{pullRequestReviewThreadId:$t,body:$b}){clientMutationId}
  resolveReviewThread(input:{threadId:$t}){thread{isResolved}}}' \
  -f t=PRRT_xxx -f b='已修复：<改法> · <commit sha>'
```

需要仓库写权限或本人是 PR 作者。被拒绝的意见**只回复、不解决**，把线程留给人判。

## 完成判据

- 本轮已打标并写入轮次账本（§2.5 只增评论）；**真源不可达而按 §2.5 跳过语义层的轮次豁免本条**，其完成判据改为：汇报中已写明「语义熔断本轮未生效：<原因>」——显式声明的降级是完成，静默漏做才是未完成。
- **熔断命中时（§2.5 第 4 步）下列各条全部豁免**：逐条回复、resolve 线程、清零查询——
  「不吭声不行」的义务由形态重估请求集中承接并转移到裁决之后。熔断路径的完成判据只有
  三件：账本已写、复盘已落（status: fused）、形态重估请求已发。此后停下等人，继续修复
  反而算违规。

- 全部未解决线程都已取到并分桶
- 每条都已通过代码改动或书面理由处置
- 每条都已单独回复
- 已处置且未被拒绝的线程都已 resolve
- 重新执行第 1 节的查询，输出为空

## 来源

工作流的次序与措辞改编自 github/gh-aw 的 `copilot-review` skill（MIT）。
未采用其中两处：按 `authorAssociation` 过滤外部贡献者（本组织仓库全私有，无外部贡献者），
以及依赖 `pr-finisher` 缓存快照的取数分支（该 skill 未引入本仓）。
