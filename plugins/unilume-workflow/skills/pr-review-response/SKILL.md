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

- 全部未解决线程都已取到并分桶
- 每条都已通过代码改动或书面理由处置
- 每条都已单独回复
- 已处置且未被拒绝的线程都已 resolve
- 重新执行第 1 节的查询，输出为空

## 来源

工作流的次序与措辞改编自 github/gh-aw 的 `copilot-review` skill（MIT）。
未采用其中两处：按 `authorAssociation` 过滤外部贡献者（本组织仓库全私有，无外部贡献者），
以及依赖 `pr-finisher` 缓存快照的取数分支（该 skill 未引入本仓）。
