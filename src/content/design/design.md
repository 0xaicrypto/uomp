---
title: 'UOMP 实现设计'
description: 'UOMP 参考实现 uomp-mvp 的架构与实现说明：组件职责、数据流与认证授权落地细节'
---

# UOMP 实现设计

本文档说明 UOMP 的**参考实现** [`uomp-mvp`](https://github.com/0xcrypto2024/uomp-core) 如何将[协议规范](/spec/)落地为可运行的代码。它面向希望理解或扩展实现的人。

---

## 1. 实现概览

`uomp-mvp` 是一个 TypeScript monorepo，核心组件与 Spec 中的角色一一对应：

| 包 / 应用 | Spec 角色 | 职责 |
|-----------|-----------|------|
| `packages/core` | — | 共享类型、常量、工具函数 |
| `packages/store` | Memory Store | SQLite 持久化、tag/key 查询 |
| `packages/token` | — | EdDSA JWT 的签发与校验 |
| `packages/auth` | Auth Service | Session 创建/授权/关闭/撤销 |
| `packages/guard` | Memory Guard | Token 校验、Scope 过滤、审计日志 |
| `packages/identity` | Identity Verification | DID / GPG 验证入口 |
| `packages/sdk` | Agent SDK（示例级） | 当前为示例 Agent 提供的简单 HTTP 调用封装；面向 GUI 应用集成的完整 TypeScript SDK 见 [路线图](/roadmap/) Milestone 2 |
| `packages/cli` | User UI | 交互式授权、启动 Agent |
| `apps/server` | — | Auth + Guard 组合 HTTP 服务 |

---

## 2. 标准架构流程

在 UOMP 的标准模型中，**Agent 是独立进程**，uomp CLI 是**用户侧的授权代理**，运行在 Memory Store / Guard 所在机器：

<div class="mermaid">
sequenceDiagram
    participant Agent as Agent（独立进程）
    participant CLI as uomp CLI（用户侧）
    participant Auth as Auth Service
    participant Guard as Memory Guard
    participant Store as Memory Store

    Agent->>CLI: 1. 提供 uom.json
    CLI->>CLI: 2. 身份验证（可选）
    CLI->>CLI: 3. 用户授权面板
    CLI->>Auth: 4. createSession(status=created)
    CLI->>Auth: 5. grantSession(grantedScopes)
    Auth-->>CLI: UOM_TOKEN
    CLI-->>Agent: 6. 交付 UOM_TOKEN
    Agent->>Guard: 7. HTTP + Authorization
    Guard->>Guard: 校验 Token & Scope
    Guard->>Store: 按 scope 读取
    Store-->>Guard: Memory Items
    Guard-->>Agent: 返回过滤后数据
</div>

关键点：

- **Agent 与 CLI 是独立进程**，Agent 不依赖 CLI 启动。
- **身份验证、授权面板、Token 签发都发生在 CLI（用户侧，即 Memory 所在机器）。**
- **Agent 只接收 Token 并使用它读取数据**，不参与授权决策。

### 2.1 本地开发便利模式

MVP 中的 `pnpm cli run ./examples/calendar-agent` 是为了降低上手门槛提供的快捷方式：

<div class="mermaid">
sequenceDiagram
    participant CLI as uomp CLI
    participant Auth as Auth Service
    participant Guard as Memory Guard
    participant Agent as Agent（子进程）

    CLI->>CLI: 读取 uom.json
    CLI->>Auth: createSession & grantSession
    Auth-->>CLI: UOM_TOKEN
    CLI->>Guard: 启动本地 Guard
    CLI->>Agent: 子进程启动，注入 UOM_TOKEN
    Agent->>Guard: 使用 Token 访问数据
    Agent-->>CLI: 进程退出
    CLI->>Auth: closeSession
</div>

这种模式把「授权代理」和「Agent 启动器」合并了，仅适用于本机开发测试，不是生产架构。

对应代码入口：

- `packages/cli/src/commands/run.ts`：本地开发模式的编排
- `packages/auth/src/index.ts`：`AuthService`
- `packages/guard/src/index.ts`：`MemoryGuard`
- `packages/token/src/index.ts`：`JWTTokenIssuer`

---

## 3. 认证与授权实现

### 3.1 Agent 声明

`uom.json` 中的 `requested_scopes` 使用 snake_case，但内部 `AgentManifest` 类型使用 camelCase。CLI 在 `loadManifest()` 中通过 `normalizeManifest()` 完成转换：

```ts
// packages/cli/src/commands/run.ts
const raw = JSON.parse(content);
return this.normalizeManifest(raw);
```

### 3.2 会话创建与授权

`AuthService.createSession()` 将请求写入 SQLite `sessions` 表，状态为 `created`。

`AuthService.grantSession()`：

1. 检查 Session 状态为 `created`
2. 构造 `CapabilityTokenPayload`
3. 调用 `JWTTokenIssuer.issue()` 生成 JWT
4. 计算 token hash 并存入 `sessions.token_hash`
5. 更新 Session 状态为 `active`

```ts
const payload: CapabilityTokenPayload = {
  version: '1.0',
  sessionId,
  agentId: row.agent_id,
  issuedAt: new Date().toISOString(),
  expiresAt: expiresAt.toISOString(),
  scopes: grantedScopes,
  profile: 'local',
  audience: 'http://127.0.0.1:9374',
  limits: { maxReadQueries: 100, maxWriteQueries: 0 },
};
```

### 3.3 JWT 实现细节

`JWTTokenIssuer` 使用 `jose` 库：

- 算法：`EdDSA`（曲线 `Ed25519`）
- 私钥/公钥通过 `generateKeyPair('EdDSA', { crv: 'Ed25519' })` 生成
- 内部 payload 为 camelCase，JWT claims 为 snake_case
- 同时设置标准 JWT `iat` 和 `exp` claim
- Header 包含 `kid: 'uomp-auth-key-1'`

```ts
private payloadToJWT(payload) {
  return {
    version: payload.version,
    session_id: payload.sessionId,
    agent_id: payload.agentId,
    issued_at: payload.issuedAt,
    expires_at: payload.expiresAt,
    scopes: payload.scopes,
    limits: payload.limits,
    profile: payload.profile,
    audience: payload.audience,
    allowed_endpoints: payload.allowedEndpoints,
  };
}
```

### 3.4 Guard 鉴权

`MemoryGuard.validateRequest()` 按顺序校验：

1. `Authorization` 头格式为 `Bearer <token>`
2. JWT 签名有效
3. Token 未过期
4. Session 未被吊销（通过 `token_blacklist` 表）

然后按请求类型分别调用：

- `GET /v1/memory/:key` → `isKeyAllowed()`
- `GET /v1/memory?tag=xxx` → `isTagAllowed()`，再对每个结果调用 `isKeyAllowed()`

`isKeyAllowed()` 的判定顺序：

1. 如果 key 在 `denyKeys` 中 → 拒绝
2. 如果 key 在 `keys` 中 → 允许
3. 如果 item 的 `sensitivity === 'high'` → 必须命中 `keys`，否则拒绝
4. 如果 item 的任意 tag 在 `tags` 中且不在 `denyTags` 中 → 允许
5. 否则拒绝

### 3.5 身份验证（可选）

身份验证由 **CLI 在用户本机** 执行，不在 Agent 进程内执行。`IdentityVerifier` 当前实现：

- **DID**：使用 `did-resolver` + `ethr-did-resolver` + `web-did-resolver`。MVP 仅验证 DID 文档可解析，不强求签名绑定。
- **GPG**：已引入 `openpgp`，但 `verifyGpg()` 目前是 placeholder，仅检查 `proof.proofValue` 是否存在。
- **无 identity**：返回 `valid=false`，CLI 在用户本机打印黄色警告，但仍可继续运行。

这是为了降低示例 Agent 的上手门槛；生产环境应在用户主机上强制身份验证，未通过验证的 Agent 不应获得 Token。

---

## 4. Memory Store 实现

`MemoryStore` 基于 `better-sqlite3`：

- `memory_items` 表存储 key、value（JSON 字符串）、tags（JSON 数组字符串）、sensitivity、source 等
- `getByTag()` 使用 SQLite JSON1 扩展：

```sql
SELECT * FROM memory_items
WHERE EXISTS (SELECT 1 FROM json_each(tags) WHERE value = ?)
```

- `set()` 使用 `INSERT ... ON CONFLICT(key) DO UPDATE` 实现 upsert

---

## 5. 审计日志

`MemoryGuard` 在每次请求后写入 `audit_logs`：

- 请求成功或失败都会记录
- 包含 `session_id`、`agent_id`、`action`、`key`、`tags`、`allowed`、`reason`
- MVP 不限制读取次数，但 `limits` 字段已预留，未来可在此实现配额扣除

---

## 6. 本地配置文件

CLI 初始化后生成：

- `~/.uomp/config.json` — 服务端口、数据目录
- `~/.uomp/uomp.sqlite` — Memory Store
- `~/.uomp/auth.sqlite` — Session 与黑名单
- `~/.uomp/audit.sqlite` — 审计日志
- `~/.uomp/.secrets/` — Ed25519 密钥对（MVP 每次运行重新生成，生产应持久化）

---

## 7. MVP 限制与未来扩展

| 能力 | MVP 状态 | 说明 |
|------|---------|------|
| Agent 读取 | ✅ 已实现 | 按 tag/key 授权 |
| Agent 写入 | ❌ 未开放 | Guard 返回 `503 WRITE_NOT_AVAILABLE` |
| Agent 删除 | ❌ 未开放 | 同上 |
| 远程 Profile | ⚠️ 部分预留 | `profile: 'remote'`、`audience`、`allowed_endpoints` 已定义，但 TLS/mTLS 未实现 |
| 身份验证 | ⚠️ 可选 | DID/GPG 框架存在，但验证强度较弱 |
| 查询限额 | ⚠️ 预留 | `limits` 已写入 Token，但未强制执行 |

---

## 8. 相关链接

- [协议规范](/spec/)
- [参考实现仓库](https://github.com/0xcrypto2024/uomp-core)
- [示例 Agent](https://github.com/0xcrypto2024/uomp-core/tree/main/examples/calendar-agent)
