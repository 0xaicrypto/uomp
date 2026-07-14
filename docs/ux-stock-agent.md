# UOMP 股票分析 Agent：CLI / SDK / 数据源设计文档

> 目标：以股票分析 Agent 为牵引，把 UOMP 的 CLI 和 SDK 从“能用”打磨成“好懂、可控、可扩展”。
> 本设计同时面向两个角色：
> - **Agent User（投资者 / 终端用户）**：使用 CLI 发现 Agent、连接、授权、管理会话、查看审计。
> - **Agent Developer（Agent 开发者）**：使用 SDK 开发股票分析 Agent，调用 Memory Guard 读取用户数据。
>
> 关键边界：**Agent User 的 CLI 不负责启动 Agent 进程**。用户只做发现、连接、授权；Agent 由用户自己或独立启动器运行，凭 Token 访问 Memory Guard。

---

## 1. 设计目标

1. **用户敢用**：投资者能清楚知道 Agent 会读什么、读到多细、多久失效。
2. **开发者好写**：Agent 开发者几行代码就能接入 UOMP，不用理解 JWT、DID、Session 细节。
3. **用户与 Agent 解耦**：用户不因为“授权”就不得不运行一段外部代码；Agent 可以独立运行、独立分发。
4. **场景闭环**：从导入持仓、发现 Agent、连接、授权、Agent 独立运行、获取分析、撤销会话，全程可控。
5. **可扩展**：数据模型和 SDK 接口要能支持后续更多金融场景（基金、债券、加密资产）。

---

## 2. 角色与视角

### 2.1 Agent User（投资者）

- 不是开发者，不会写代码。
- 关心三件事：我的数据在哪、Agent 能看什么、怎么让它停下来。
- 使用 CLI 完成所有操作，但 **CLI 不帮用户启动 Agent**。
- 用户自己决定在哪里、以什么方式运行 Agent；CLI 只负责签发 Token 并安全交付。

### 2.2 Agent Developer（Agent 作者）

- 会写 TypeScript/Python，想快速做一个股票分析 Agent。
- 关心：怎么声明权限、怎么读数据、怎么上报审计、怎么在本地跑通。
- 使用 SDK + `uom.json`。
- 需要一个开发者专用的 CLI 命令来本地调试和启动 Agent（例如 `uomp dev run` 或 `uomp agent run`）。

---

## 3. 数据模型（Memory Schema）

股票分析 Agent 需要的数据分为两类：**用户私有数据**（存在本地 Memory Store）和**公开/半公开数据**（可由 Agent 外部获取，也可存入本地）。

### 3.1 用户私有数据

| 数据项 | tag | sensitivity | 说明 |
|--------|-----|-------------|------|
| 当前持仓 | `portfolio:holdings` | high | 股票代码、数量、成本价、市值等 |
| 自选股/关注列表 | `portfolio:watchlist` | medium | 用户关注的标的，不含金额 |
| 风险偏好 | `profile:risk` | medium | 保守/稳健/激进，可承受回撤等 |
| 投资目标 | `profile:goal` | medium | 长期增值、分红、养老等 |
| 交易限制 | `profile:constraints` | medium | 不做空、不投某行业等 |
| 历史分析记录 | `analysis:history` | low | Agent 过去生成的报告摘要 |

### 3.2 公开/半公开数据

| 数据项 | tag | sensitivity | 说明 |
|--------|-----|-------------|------|
| 实时行情 | `market:quote` | low | 最新价、涨跌幅、成交量 |
| 历史 K 线 | `market:history` | low | 日线/周线/月线价格 |
| 公司基本面 | `market:fundamental` | low | PE、PB、ROE、营收等 |
| 宏观经济 | `market:macro` | low | 利率、CPI、GDP 等 |
| 新闻情绪 | `market:news` | low | 公开新闻、研报摘要 |

### 3.3 输出数据

| 数据项 | tag | sensitivity | 说明 |
|--------|-----|-------------|------|
| 分析报告 | `analysis:report` | low | Agent 生成的文本/HTML/Markdown 报告 |

> 注：按 UOMP 当前规范，`sensitivity=high` 的 Memory Item 不能通过 tag 泛化授权，需要用户显式确认或 key 级授权。`portfolio:holdings` 应标记为 high。

---

## 4. 数据源设计

股票分析 Agent 需要的数据分为“用户本地已有数据”和“Agent 运行时需要抓取的数据”。

### 4.1 用户本地数据来源

| 来源 | 数据 | 导入方式 | 说明 |
|------|------|----------|------|
| 用户手动输入 | 持仓、自选股、风险偏好 | CLI 交互式问卷 | 最简单，适合 demo |
| CSV/Excel 导入 | 持仓、交易记录 | `uomp import holdings.csv` | 支持常见券商导出格式 |
| 券商 API / 文件同步 | 实时持仓 | 后续扩展，MVP 不做 | 需要合规考虑 |

### 4.2 Agent 运行时抓取的数据源

Agent 不依赖本地 Memory Store 获取公开市场数据，可以自行调用公开 API。推荐数据源：

#### 美股

| 数据源 | 覆盖范围 | 认证 | 限制 | 备注 |
|--------|----------|------|------|------|
| Yahoo Finance API（unofficial） | 实时行情、历史 K 线 | 免费 | 不稳定，适合 demo | 可用 `yfinance` Python 库 |
| Alpha Vantage | 行情、基本面、技术指标 | API Key | 免费版 25 次/天 | 适合基本面分析 |
| Finnhub | 实时行情、新闻、基本面 | API Key | 免费版 60 次/分钟 | 功能较全 |
| Polygon.io | 美股行情、新闻 | API Key | 付费 | 高质量，生产推荐 |
| SEC EDGAR | 财报、公告 | 免费 | 原始数据 | 适合基本面深度分析 |

#### A股/港股

| 数据源 | 覆盖范围 | 认证 | 限制 | 备注 |
|--------|----------|------|------|------|
| Tushare | A股行情、基本面、宏观 | Token | 积分制 | 国内最常用 |
| AKShare | A股/港股/基金数据 | 免费 | 接口不稳定 | 纯免费，适合 demo |
| 东方财富 Choice / Wind | 全市场 | 付费 | 商业授权 | 生产级 |
| 新浪财经接口 | 实时行情 | 免费 | 不稳定 | 简单 demo 可用 |

#### 新闻与情绪

| 数据源 | 覆盖范围 | 认证 | 备注 |
|--------|----------|------|------|
| NewsAPI | 全球新闻 | API Key | 适合英文新闻 |
| Bing News API | 新闻搜索 | API Key | 微软生态 |
| 聚宽/宽客社区 | 中文研报 | 部分免费 | A股研报 |
| Twitter/X API | 情绪 | API Key | 需谨慎使用 |

#### 宏观经济

| 数据源 | 覆盖范围 | 认证 | 备注 |
|--------|----------|------|------|
| FRED（美联储） | 美国宏观数据 | 免费 | 全球经济指标 |
| World Bank Open Data | 全球宏观 | 免费 | 长期趋势分析 |
| 国家统计局 | 中国宏观 | 免费 | A股宏观分析 |

### 4.3 数据源使用原则

1. **用户私有数据必须走 Memory Guard**：持仓、风险偏好等只能由 Agent 通过 UOMP Token 读取。
2. **公开数据 Agent 可自行获取**：但建议在 `uom.json` 中声明会用到的外部数据源，让用户知情。
3. **避免 Agent 把私有数据传给外部 API**：Agent 不能把用户持仓列表作为参数发给第三方 LLM 或数据源。分析逻辑应尽量本地完成。
4. **LLM 调用默认本地或用户可控**：如果必须调用云端 LLM，应先对持仓做脱敏（如只传代码和权重，不传成本价）。

---

## 5. CLI 设计：Agent User 视角

CLI 是投资者的“授权管理器”，不是 Agent 启动器。设计要点：

- **只做发现、连接、授权、会话管理、审计**。
- 所有危险操作都有确认。
- 授权前必须展示“数据暴露摘要”。
- 会话状态一目了然。
- 错误信息告诉用户“为什么”和“怎么办”。

### 5.1 命令总览

#### Agent User 命令

| 命令 | 作用 |
|------|------|
| `uomp import <file>` | 导入持仓/自选股/风险偏好 |
| `uomp data` | 查看本地 Memory Store 中的数据 |
| `uomp discover <path-or-registry>` | 发现 Agent，显示清单信息 |
| `uomp connect <agent>` | 连接 Agent，验证身份，预览权限请求 |
| `uomp authorize <agent>` | 创建 Session 并签发 Token |
| `uomp sessions` | 查看活跃会话 |
| `uomp revoke <session-id>` | 撤销会话 |
| `uomp audit` | 查看访问审计日志 |
| `uomp config` | 配置默认风险偏好、数据源偏好 |
| `uomp dry-run <agent>` | 模拟授权，不读真实数据 |
| `uomp registry search <keyword>` | 从 Registry 搜索 Agent |

#### Agent Developer 命令

| 命令 | 作用 |
|------|------|
| `uomp agent init <name>` | 初始化一个 Agent 项目 |
| `uomp agent validate` | 验证 `uom.json` 和文件结构 |
| `uomp agent test` | 使用测试数据本地调试 Agent |
| `uomp agent run <agent>` | 开发者本地启动 Agent（用于测试） |
| `uomp agent publish` | 打包 Agent 供发布 |

> 注意：`uomp agent run` 属于开发者调试工具，不是给普通用户的命令。

### 5.2 核心流程 wireflow

#### 5.2.1 导入持仓

```bash
$ uomp import holdings.csv --tag portfolio:holdings
```

输出：

```text
已导入持仓数据:
  文件: holdings.csv
  标的数量: 8
  总市值: $124,500
  标签: portfolio:holdings
  敏感度: high
  存储位置: ~/.uomp/memory/
```

#### 5.2.2 发现 Agent

```bash
$ uomp discover ./examples/stock-analyst
```

或从 Registry：

```bash
$ uomp registry search stock
$ uomp discover registry://stock-analyst
```

输出：

```text
Agent: stock-analyst v0.1
发布者: example-org  [DID 已验证]
描述: 基于持仓和市场公开信息生成投资策略分析

外部数据源:
  - yahoo-finance
  - alpha-vantage

权限请求:
  [高敏感] portfolio:holdings   - 当前持仓
  [中敏感] portfolio:watchlist - 自选股
  [中敏感] profile:risk        - 风险偏好
  [低敏感] market:public       - 公开市场数据（Agent 将自行获取）

写入权限: 无
```

#### 5.2.3 连接 Agent

```bash
$ uomp connect ./examples/stock-analyst
```

输出：

```text
已连接 Agent: stock-analyst v0.1
身份验证: DID 已验证
发布者: example-org

本次连接未授权任何数据，Agent 无法访问 Memory Guard。
请运行 `uomp authorize stock-analyst` 进行授权。
```

“连接”的含义是：

- CLI 读取并验证 Agent 的 `uom.json`。
- 确认 Agent 身份（DID / GPG / Registry）。
- 在本地建立一个“已连接”记录，方便后续授权。
- **不启动 Agent，也不签发 Token**。

#### 5.2.4 授权 Agent

```bash
$ uomp authorize ./examples/stock-analyst
```

输出：

```text
授权请求: stock-analyst v0.1
发布者: example-org  [DID 已验证]

权限请求:
  [高敏感] portfolio:holdings   - 当前持仓（8 条记录）
  [中敏感] portfolio:watchlist - 自选股（15 条）
  [中敏感] profile:risk        - 风险偏好
  [低敏感] market:public       - 公开市场数据（Agent 将自行获取）

写入权限: 无
默认会话时长: 10 分钟

本次将暴露:
  - 你的 8 条持仓记录（含成本价和市值）
  - 你的自选股列表
  - 你的风险偏好

[y] 确认授权  [n] 取消  [e] 编辑范围  [d] 模拟运行
```

用户选 `y` 后：

```text
已创建会话: sess_abc123
已签发 Capability Token（有效期至 10:30）

请把以下环境变量设置到你运行 Agent 的终端或启动器中：

  export UOM_TOKEN="eyJhbG..."
  export UOMP_BASE_URL="http://127.0.0.1:9374"

或者保存到文件:
  uomp authorize ./examples/stock-analyst --output ~/.uomp/tokens/sess_abc123.env

Agent 启动后可以通过 Memory Guard 访问已授权数据。
你可以随时运行 `uomp revoke sess_abc123` 撤销授权。
```

#### 5.2.5 编辑范围

用户选 `e` 后进入交互：

```text
选择本次要授权的数据:
  [x] portfolio:holdings   （当前持仓）
  [ ] portfolio:watchlist  （不授权）
  [x] profile:risk         （风险偏好）
  [x] market:public        （公开数据）

高敏感数据选项:
  [ ] 暴露成本价和具体股数
  [x] 仅暴露持仓代码和权重（脱敏模式）
```

#### 5.2.6 查看会话

```bash
$ uomp sessions
```

输出：

```text
活跃会话:
  sess_abc123  stock-analyst  剩余 7 分钟  [portfolio:holdings, profile:risk]
  sess_def456  news-agent     剩余 23 分钟 [market:news]

已关闭会话（最近 5 条）:
  sess_ghi789  stock-analyst  10:30 已过期
```

#### 5.2.7 撤销会话

```bash
$ uomp revoke sess_abc123
```

输出：

```text
已撤销会话 sess_abc123。
对应 Capability Token 已立即失效。
正在运行的 Agent 将在下一次访问 Memory Guard 时被拒绝。
```

#### 5.2.8 审计日志

```bash
$ uomp audit --agent stock-analyst --today
```

输出：

```text
2026-07-14 10:00:01  stock-analyst  READ  portfolio:holdings  8 items
2026-07-14 10:00:02  stock-analyst  READ  profile:risk        1 item
2026-07-14 10:00:03  stock-analyst  FETCH market:public       AAPL,TSLA,NVDA
2026-07-14 10:00:10  stock-analyst  SAVE  analysis:report     1 item
```

### 5.3 CLI 配置

```bash
$ uomp config set risk_profile conservative
$ uomp config set default_holdings_file ~/portfolio.csv
$ uomp config set data_source.market.primary yahoo
$ uomp config set data_source.market.cn tushare
```

配置保存在 `~/.uomp/config.json`，敏感度为 low，可被 Agent 读取以调整分析策略。

### 5.4 错误信息设计

| 场景 | 旧错误 | 新错误 |
|------|--------|--------|
| Token 未授权某 tag | `ACCESS_DENIED` | `Agent 请求读取 "portfolio:holdings"，但当前会话未授权。请让用户运行: uomp authorize <agent> --include portfolio:holdings` |
| Agent 请求写入 | `WRITE_NOT_AVAILABLE` | `当前 Agent 请求写入数据，但 UOMP MVP 禁止 Agent 写入。如需保存报告，请让 Agent 输出到本地文件。` |
| 会话已过期 | `TOKEN_EXPIRED` | `会话 sess_abc123 已过期（10:30）。请重新运行: uomp authorize <agent>` |
| 高敏感未确认 | `ACCESS_DENIED` | `"portfolio:holdings" 为高敏感数据，需要用户在授权时显式确认。请使用 --sensitive 参数或交互式授权。` |

---

## 6. CLI 设计：Agent Developer 视角

开发者也需要 CLI 来调试、验证和发布 Agent。

### 6.1 初始化 Agent

```bash
$ uomp agent init stock-analyst --template typescript
```

生成目录结构：

```text
stock-analyst/
  uom.json
  src/
    index.ts
  package.json
  README.md
```

### 6.2 验证 Agent

```bash
$ uomp agent validate
```

检查项：

- `uom.json` 格式是否正确
- `requested_scopes` 是否合理
- 必填文件是否存在
- identity / proof 是否可验证
- 是否声明了外部数据源

输出示例：

```text
验证通过:
  Agent ID: stock-analyst
  版本: 0.1.0
  发布者: example-org
  权限请求: 4 个 tag（1 个 high, 2 个 medium, 1 个 low）
  外部数据源: yahoo-finance, alpha-vantage
  风险: 无写入权限，符合 MVP 规范
```

### 6.3 本地调试

```bash
$ uomp agent test
```

自动完成：

1. 使用测试数据填充本地 Memory Store
2. 签发一个测试 Token
3. 启动 Agent
4. 输出审计日志

### 6.4 开发者本地启动 Agent

```bash
$ uomp agent run ./examples/stock-analyst
```

开发者测试时使用，等价于：

```bash
$ uomp authorize ./examples/stock-analyst --output /tmp/uomp.env
$ source /tmp/uomp.env
$ node ./examples/stock-analyst/dist/index.js
```

> 这个命令只对开发者暴露，普通用户不需要也不应该使用。

---

## 7. SDK 设计：Agent User 视角

普通用户其实不需要 SDK，但“用户侧 SDK”可以指：

- CLI 内部调用的库（`@uomp/cli-core`）
- 未来 GUI 应用集成的 SDK（如 Electron/Tauri App）

这里先定义未来 GUI 会用到的**用户侧 SDK**：

```ts
import { UompClient } from '@uomp/client';

const client = new UompClient({ dataDir: '~/.uomp' });

// 导入持仓
await client.memory.import({
  file: '~/holdings.csv',
  tag: 'portfolio:holdings',
  sensitivity: 'high',
});

// 查看数据
const holdings = await client.memory.query({ tags: ['portfolio:holdings'] });

// 发现 Agent
const manifest = await client.discover('./examples/stock-analyst');

// 连接 Agent（验证身份）
const connection = await client.connect('./examples/stock-analyst');

// 授权 Agent，返回 Token
const session = await client.authorize({
  agentPath: './examples/stock-analyst',
  includeSensitive: true,
  durationMinutes: 10,
});

// 监听会话事件
session.on('access', (event) => {
  console.log(`Agent 读取了 ${event.tag}`);
});

// 撤销
await session.revoke();
```

---

## 8. SDK 设计：Agent Developer 视角

这是本次设计的重点。Agent 开发者 SDK 要足够薄，让开发者专注于分析逻辑。

### 8.1 核心类

```ts
import { UompAgent } from '@uomp/sdk';

const agent = await UompAgent.fromEnv();

// 读取用户授权的数据
const holdings = await agent.memory.read({ tags: ['portfolio:holdings'] });
const risk = await agent.memory.read({ tags: ['profile:risk'] });

// 读取公开数据（SDK 提供基础封装，但调用外部 API）
const quotes = await agent.market.quotes(['AAPL', 'TSLA']);
const fundamentals = await agent.market.fundamentals(['AAPL']);

// 生成分析（开发者自己的逻辑）
const report = analyze({ holdings, risk, quotes, fundamentals });

// 保存报告到本地文件（不写入 Memory Store）
await agent.output.save('./output/report.md', report);

// 也可以把报告摘要写入 Memory Store（如果用户授权）
await agent.memory.write({
  tag: 'analysis:report',
  key: 'stock-analysis-20260714',
  value: { summary: report.summary },
});
```

### 8.2 SDK API 清单

#### `UompAgent`

| 方法 | 作用 |
|------|------|
| `fromEnv()` | 从 `UOM_TOKEN` / `UOMP_BASE_URL` 初始化 |
| `whoami()` | 返回当前 Agent 的 manifest 和已授权 scope |
| `memory.read(opts)` | 读取 Memory Guard 数据 |
| `memory.write(opts)` | 写入 Memory Store（需授权，MVP 建议禁用） |
| `memory.query(opts)` | 复杂查询 |
| `market.quotes(symbols)` | 获取行情（调用外部数据源） |
| `market.fundamentals(symbols)` | 获取基本面 |
| `market.news(symbols)` | 获取新闻 |
| `market.macro(indicators)` | 获取宏观数据 |
| `output.save(path, content)` | 保存报告到本地文件 |
| `audit.log(event)` | 上报自定义审计事件 |

#### `UompAgentConfig`

```ts
interface UompAgentConfig {
  token?: string;           // UOM_TOKEN
  baseUrl?: string;         // UOMP_BASE_URL
  manifestPath?: string;    // uom.json 路径
  dataSource?: {
    market?: 'yahoo' | 'alpha-vantage' | 'tushare' | 'akshare' | 'custom';
    apiKey?: string;
  };
}
```

### 8.3 错误处理

SDK 应抛出结构化错误，方便开发者区分：

```ts
try {
  await agent.memory.read({ tags: ['portfolio:holdings'] });
} catch (err) {
  if (err.code === 'SCOPE_DENIED') {
    console.log('请要求用户授权 portfolio:holdings');
  }
  if (err.code === 'TOKEN_EXPIRED') {
    console.log('会话已过期，请重新授权');
  }
}
```

### 8.4 数据脱敏辅助

SDK 可以提供辅助函数，帮助开发者避免把敏感数据传给外部 LLM：

```ts
import { redactHoldings } from '@uomp/sdk/utils';

const safe = redactHoldings(holdings, { keep: ['symbol', 'weight'] });
// safe = [{ symbol: 'AAPL', weight: 0.25 }, ...]
```

---

## 9. 股票分析 Agent 的完整用户旅程

```text
[投资者]
   |
   v
导入持仓 CSV  ---->  数据进入本地 Memory Store (portfolio:holdings, high)
   |
   v
uomp discover ./stock-analyst
   |
   v
uomp connect ./stock-analyst  ---->  验证 Agent 身份，建立连接记录
   |
   v
uomp authorize ./stock-analyst
   |
   v
CLI 展示数据暴露摘要，用户确认 / 编辑范围 / 脱敏
   |
   v
CLI 创建 Session，签发 Token
   |
   v
CLI 输出环境变量或 Token 文件给用户
   |
   v
[用户在另一个终端 / 启动器中运行 Agent]
   |
   v
Agent 读取 UOM_TOKEN，访问 Memory Guard
   |
   v
SDK 读取 portfolio:holdings, profile:risk
   |
   v
SDK 调用 Yahoo Finance / Alpha Vantage 获取公开数据
   |
   v
Agent 本地生成分析报告
   |
   v
报告保存到 ./output/report.md
   |
   v
会话超时 / 用户撤销 ----> Token 失效
```

---

## 10. 安全与隐私要点

1. **持仓默认高敏感**：`portfolio:holdings` 必须标记为 high，不能 tag 泛化授权。
2. **公开数据不敏感**：`market:*` 可设为 low，Agent 可自行获取。
3. **用户 CLI 不启动 Agent**：避免“授权即执行”的安全风险，Agent 必须由用户独立启动。
4. **Token 交付要安全**：默认输出到终端，由用户手动复制；也支持 `--output` 保存到用户指定文件。不推荐自动注入外部进程。
5. **LLM 调用要脱敏**：如果 Agent 调用外部 LLM，应先去掉成本价、股数等敏感字段。
6. **报告本地保存**：分析结论默认写到用户本地文件，不写入 Memory Store，除非用户授权 `analysis:report`。
7. **会话短期**：股票分析通常 5-10 分钟足够，默认 Token 有效期不超过 10 分钟。
8. **审计完整**：每次 `memory.read`、每次外部 API 调用、每次报告生成都应记录。

---

## 11. 实现阶段

### Phase 1：MVP Demo（1-2 周）

- 股票 Agent 能读取 `portfolio:holdings` 和 `profile:risk`
- Agent 从 Yahoo Finance 获取行情
- 生成 Markdown 报告保存到本地
- CLI 支持 `uomp discover`、`uomp connect`、`uomp authorize` 和数据暴露摘要
- Token 以环境变量形式交付给用户

### Phase 2：体验打磨（2-3 周）

- `uomp import` CSV 导入
- `uomp dry-run` 模拟授权
- `uomp config` 用户配置
- SDK 的数据脱敏辅助函数
- 更友好的错误信息
- 开发者命令 `uomp agent run` / `uomp agent test`

### Phase 3：生产准备（后续）

- 支持券商 API / 文件同步
- 多数据源适配器（Tushare / Alpha Vantage / Polygon）
- 本地 LLM 支持（Ollama）
- GUI 应用

---

## 12. 待决策问题

1. **Agent 获取公开数据时，是否也需要 Token？**
   - 建议：公开数据不走 Memory Guard，但 Agent 应在 `uom.json` 声明会用哪些外部数据源。
2. **报告是否允许 Agent 写回 Memory Store？**
   - 建议：MVP 禁止；报告保存到本地文件。后续可通过 `analysis:report` tag 授权写入。
3. **高敏感数据是否支持“脱敏授权”？**
   - 建议：支持。CLI 提供脱敏选项，底层通过 key 级授权实现。
4. **Token 交付方式偏好？**
   - 选项 A：终端打印 `export` 命令，用户手动复制。
   - 选项 B：保存到 `~/.uomp/tokens/<session>.env` 文件，用户 source。
   - 选项 C：通过本地 socket/IPC 传给 Agent（需要 Agent 监听，增加复杂度）。
   - 建议 Phase 1 用 A + B，C 后续考虑。
5. **CLI 是否内置 TUI（如 Ink/blessed）？**
   - 建议：Phase 2 考虑，Phase 1 用简单文本交互即可。
6. **是否需要为 Agent 开发者提供 Python SDK？**
   - 建议：先做好 TypeScript SDK，Python SDK 后续跟进（金融圈 Python 开发者很多）。

---

## 13. 下一步行动

1. 确认“用户 CLI 不启动 Agent”的边界和 Token 交付方式（A / B / C）。
2. 确认授权前的“数据暴露摘要”和“编辑范围”交互细节。
3. 确认 SDK 的最小 API 集合。
4. 之后即可进入 Phase 1 实现：先做一个最小 CLI（discover/connect/authorize）+ 一个股票 Agent demo。
