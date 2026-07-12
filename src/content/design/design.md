---
title: 'UOMP 核心架构设计'
description: 'UOMP 核心架构设计概念：Agent 声明、用户授权、Capability Token 与 Memory Guard'
---

# UOMP 核心架构设计

本文档从**架构视角**简要介绍 UOMP 的核心设计概念。它与大协议规范（Spec）保持一致，但更侧重整体流程和关键组件的理解。详细协议定义请参阅[规范](/spec/)。

---

## 1. 总体流程

UOMP 的授权流分为三层：

```
Agent 声明请求范围 → 用户显式授权 → 短时会话令牌生效 → Guard 强制鉴权
```

Agent 本身不持有用户密码或长期凭证。它通过 `uom.json` 声明所需的数据范围，由用户在 CLI 中授权后，获得一个短时的 JWT Capability Token，最后凭此令牌访问用户本地 Guard 服务。

---

## 2. Agent 声明（`uom.json`）

Agent 通过 `uom.json` 声明它想访问的范围，不自带任何权限：

```json
{
  "uomp_version": "1.0",
  "agent": {
    "id": "calendar_agent",
    "name": "Calendar Assistant"
  },
  "requested_scopes": {
    "read": {
      "tags": ["preference"],
      "keys": [],
      "deny_tags": ["private"],
      "deny_keys": []
    }
  },
  "required_capabilities": ["memory.read"]
}
```

- `tags`：按标签批量请求，例如 `preference`。
- `keys`：按具体 key 请求，例如 `preference.theme`。
- `deny_tags` / `deny_keys`：显式排除某些范围。

---

## 3. 用户授权（CLI）

`uomp run ./my-agent` 会：

1. 读取并解析 `uom.json`。
2. 可选验证 Agent 身份（DID / GPG）。
3. 交互式询问用户授权哪些 tag。
4. 创建 Session 并签发 JWT。
5. 把 `UOM_TOKEN` 注入 Agent 进程。

这是 UOMP 的关键原则：**数据访问权始终由用户主动授予，且默认最小权限。**

---

## 4. Capability Token

授权后，AuthService 签发一个短时会话令牌：

```ts
{
  sessionId: 'sess_xxx',
  agentId: 'calendar-agent',
  scopes: {
    read: { tags: ['preference'], keys: [], denyTags: ['private'], denyKeys: [] }
  },
  audience: 'http://127.0.0.1:9374',
  limits: { maxReadQueries: 100, maxWriteQueries: 0 }
}
```

Token 是本地签发的 **JWT EdDSA**，默认 30 分钟有效，包含被授权的 tags/keys、deny 列表、查询限额和 audience。

---

## 5. Memory Guard 鉴权

Guard 是数据访问的守门人。Agent 每次请求都必须携带：

```http
Authorization: Bearer <UOM_TOKEN>
```

Guard 会：

1. 校验 JWT 签名与过期时间。
2. 按 token 中的 `scopes` 判断 tag/key 是否被授权。
3. 拒绝 `deny_tags` / `deny_keys` 中的范围。
4. `sensitivity: high` 的数据必须显式通过 `keys` 授权。
5. 记录审计日志，MVP 阶段写操作统一拒绝。

---

## 6. 身份验证（可选）

UOMP 支持 DID（`did:ethr` / `did:web`）和 GPG 作为 Agent 身份验证方式，但 MVP 中**不强制**。未配置 identity 的 Agent 会收到 CLI 的黄色警告，但仍可运行，以降低示例和开发阶段的上手门槛。

---

## 7. 设计原则总结

| 原则 | 实现方式 |
|------|---------|
| 用户主权 | 数据存在本地，Token 由本地签发 |
| 最小权限 | Token 只包含用户授权的 tags/keys |
| 短时效 | 默认 30 分钟，支持 revoke/close |
| 可审计 | Guard 记录每次访问 |
| 读写分离 | MVP 仅开放读取，写入待 staging 机制 |
