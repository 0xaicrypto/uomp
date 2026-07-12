---
title: 'UOMP Core Architecture Design'
description: 'Core architecture concepts of UOMP: agent declaration, user authorization, capability token, and memory guard'
---

# UOMP Core Architecture Design

This document provides a **high-level architectural overview** of UOMP's core design concepts. It is consistent with the full protocol [Spec](/en/spec/) but focuses on overall flow and key components. For formal protocol definitions, see the Spec.

---

## 1. Overall Flow

UOMP authorization flows in three layers:

```
Agent declares requested scopes → User explicitly authorizes → Short-lived session token takes effect → Guard enforces access control
```

Agents never hold user passwords or long-term credentials. They declare required data scopes via `uom.json`, receive a short-lived JWT Capability Token after user authorization through the CLI, and then access the user's local Guard service with that token.

---

## 2. Agent Declaration (`uom.json`)

An Agent declares what it wants to access in `uom.json` and has no inherent permissions:

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

- `tags`: request access by tag, e.g. `preference`.
- `keys`: request access by specific key, e.g. `preference.theme`.
- `deny_tags` / `deny_keys`: explicitly exclude scopes.

---

## 3. User Authorization (CLI)

`uomp run ./my-agent` performs the following:

1. Read and parse `uom.json`.
2. Optionally verify Agent identity (DID / GPG).
3. Interactively ask the user which tags to authorize.
4. Create a session and issue a JWT.
5. Inject `UOM_TOKEN` into the Agent process.

This reflects UOMP's core principle: **data access is always actively granted by the user, with least privilege by default.**

---

## 4. Capability Token

After authorization, AuthService issues a short-lived session token:

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

The token is a locally issued **JWT EdDSA** token, valid for 30 minutes by default. It contains authorized tags/keys, deny lists, query limits, and audience.

---

## 5. Memory Guard Enforcement

Guard is the gatekeeper for data access. Every request must include:

```http
Authorization: Bearer <UOM_TOKEN>
```

Guard will:

1. Verify the JWT signature and expiry.
2. Check whether the tag/key is authorized according to the token's `scopes`.
3. Deny anything in `deny_tags` / `deny_keys`.
4. Require explicit `keys` authorization for `sensitivity: high` data.
5. Log every access and reject write operations in the MVP.

---

## 6. Identity Verification (Optional)

UOMP supports DID (`did:ethr` / `did:web`) and GPG as Agent identity verification methods, but identity verification is **not mandatory** in the MVP. Agents without an `identity` field receive a yellow warning from the CLI but can still run, lowering the barrier for examples and development.

---

## 7. Design Principles

| Principle | Implementation |
|-----------|----------------|
| User sovereignty | Data lives locally; tokens are issued locally |
| Least privilege | Token contains only user-authorized tags/keys |
| Short-lived | 30 minutes by default; supports revoke/close |
| Auditable | Guard logs every access |
| Read-write separation | MVP only allows reads; writes pending staging mechanism |
