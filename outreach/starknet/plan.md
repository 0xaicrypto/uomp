# UOMP × Starknet 社区推广计划

## 定位

UOMP（User-Owned Memory Protocol）是一个**用户主权的 Agent 记忆授权协议**。
在 Starknet 正在大力推进 "Verifiable AI Agents" 的背景下，UOMP 可以作为：

- **链下用户记忆的安全守门员**：记忆保留在用户本地，Agent 通过短期 Capability Token 访问。
- **与 Starknet 链上身份/注册表互补**：用 Starknet 账户抽象做身份验证，用 ERC-8004 Registry 做 Agent 发现，用 Cairo 可验证计算做访问审计证明。
- **AI Agent 生态的通用授权层**：不绑定具体框架，可与 Daydreams、Giza、Starknet.agent 等项目的 Agent 集成。

## 目标

1. 在 Starknet 社区建立 UOMP 的知名度。
2. 找到 2–3 个潜在合作/集成对象（AI Agent 项目或基础设施团队）。
3. 收集反馈，完善协议 Draft-00 的 Starknet 适配章节。
4. 争取进入 Starknet AI Portal 或申请 Seed Grant / Growth Grant。

## 阶段与行动

### Phase 1：调研与准备（1–2 天）

- [ ] 完整阅读 [Starknet AI Portal](https://www.starknet.io/verifiable-ai-agents/)，记录已有的 Agent 框架与工具。
- [ ] 列出重点目标项目：Daydreams、Giza、Starknet.agent、Audit Agent by Nethermind、Swarmzero、Toolblox 等。
- [ ] 注册/加入社区：Discord、Telegram、Community Forum。
- [ ] 准备一份面向 Starknet 的 UOMP 一句话介绍（elevator pitch）。

### Phase 2：软性曝光（3–5 天）

- [ ] 在 [Starknet Community Forum](https://community.starknet.io/) 的 **Development Proposals** 或 **All-Purpose Hangout** 发布介绍帖（见 `forum-post.md`）。
- [ ] 在 Discord 的 `#ai-agents` / `#developers` 频道以“分享一个开放协议草案”的方式介绍，不直接硬广。
- [ ] 发布 X/Twitter Thread（见 `x-thread.md`），@ 相关账号：@Starknet、@gizatechxyz、@daydreamsagents 等。
- [ ] 在 Reddit r/Starknet 或相关中文社区同步一篇中文摘要。

### Phase 3：精准 outreach（1–2 周）

- [ ] 向 5–10 个目标项目发送 DM/邮件（见 `dm-templates.md`）。
- [ ] 通过 [Starknet AI Portal Talk to Us](https://forms.reform.app/starkware/talk-to-us-ai-portal-starknet/3miubw) 表单联系基金会，表达合作意愿。
- [ ] 回复 Forum 帖子中的评论，迭代协议设计。
- [ ] 参与一次 Starknet 线上 AMA 或 Hacker House 的相关讨论。

### Phase 4：可演示集成（2–4 周）

- [ ] 基于 `uomp-core` 做一个 **Starknet 适配示例**：
  - 用户用 Starknet 钱包签名生成 DID。
  - Agent 在 `uom.json` 中声明 scope。
  - 会话Token的哈希/撤销根记录在 Starknet 链上（可选）。
- [ ] 写一篇文章：《UOMP + Starknet：让用户记忆成为可验证的 Agent 资产》。
- [ ] 制作 60 秒 demo 视频，替换/补充官网首页视频。
- [ ] 准备 [Starknet Seed Grant](https://www.starknet.io/grants/seed-grants/) 或 Growth Grant 申请材料。

## 关键渠道

| 渠道 | 链接 | 用途 |
|---|---|---|
| Community Forum | https://community.starknet.io/ | 发布提案、收集深度反馈 |
| Discord | https://discord.gg/starknet-community | 日常讨论、快速反馈 |
| Telegram | https://t.me/starknet_ecosystem | 生态动态 |
| X / Twitter | https://x.com/Starknet | 公开传播 |
| AI Portal 联系表单 | https://forms.reform.app/starkware/talk-to-us-ai-portal-starknet/3miubw | 联系基金会 |
| Grants | https://www.starknet.io/grants/seed-grants/ | 申请资助 |

## 衡量指标

- Forum 帖子浏览量、回复数。
- Discord/Telegram 互动次数。
- X Thread 的 impressions、likes、retweets。
- 收到的合作/集成意向数量。
- GitHub `0xaicrypto/uomp` 的 stars / discussions 增长。
