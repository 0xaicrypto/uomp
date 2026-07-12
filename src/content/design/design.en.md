---
title: 'UOMP Implementation Design'
description: 'Architecture and implementation notes for the uomp-mvp reference implementation: component responsibilities, data flow, and auth details'
---

# UOMP Implementation Design

This document explains how the UOMP **reference implementation** [`uomp-mvp`](https://github.com/0xcrypto2024/uomp-core) turns the [protocol specification](/en/spec/) into runnable code. It is intended for people who want to understand or extend the implementation.

---

## 1. Implementation Overview

`uomp-mvp` is a TypeScript monorepo. Each package maps to a role in the Spec:

| Package / App | Spec Role | Responsibility |
|---------------|-----------|----------------|
| `packages/core` | — | Shared types, constants, utilities |
| `packages/store` | Memory Store | SQLite persistence, tag/key queries |
| `packages/token` | — | EdDSA JWT issuance and verification |
| `packages/auth` | Auth Service | Session create/grant/close/revoke |
| `packages/guard` | Memory Guard | Token validation, scope filtering, audit logging |
| `packages/identity` | Identity Verification | DID / GPG verification entry point |
| `packages/sdk` | Agent SDK | TypeScript SDK for Agents |
| `packages/cli` | User UI | Interactive authorization, Agent launcher |
| `apps/server` | — | Combined Auth + Guard HTTP service |

---

## 2. Complete Data Flow

Using `pnpm cli run ./examples/calendar-agent` as an example:

```
1. CLI reads examples/calendar-agent/uom.json
2. CLI calls IdentityVerifier to verify Agent identity (optional), showing a warning to the user
3. CLI prompts the user to select allowed tags
4. CLI calls AuthService.createSession() (status=created)
5. CLI calls AuthService.grantSession(sessionId, grantedScopes) to issue JWT
6. CLI starts local MemoryGuard (listening on 127.0.0.1:9374)
7. CLI spawns the Agent process, only injecting UOM_TOKEN and UOMP_BASE_URL
8. Agent calls Guard API via SDK; Guard validates token and returns scoped data
9. After Agent exits, CLI closes the Session (status=closed)
```

Key point: **Identity verification, authorization prompt, and Token issuance all happen in the CLI (on the user's side, where Memory lives); the Agent process only uses the Token to read data.**

Key code entry points:

- `packages/cli/src/commands/run.ts`: orchestrates the whole flow
- `packages/auth/src/index.ts`: `AuthService`
- `packages/guard/src/index.ts`: `MemoryGuard`
- `packages/token/src/index.ts`: `JWTTokenIssuer`

---

## 3. Authentication & Authorization Implementation

### 3.1 Agent Declaration

`uom.json` uses `snake_case` for `requested_scopes`, while the internal `AgentManifest` type uses `camelCase`. The CLI converts between them in `loadManifest()` via `normalizeManifest()`:

```ts
// packages/cli/src/commands/run.ts
const raw = JSON.parse(content);
return this.normalizeManifest(raw);
```

### 3.2 Session Creation & Granting

`AuthService.createSession()` writes the request into the SQLite `sessions` table with status `created`.

`AuthService.grantSession()`:

1. Checks the Session status is `created`
2. Constructs the `CapabilityTokenPayload`
3. Calls `JWTTokenIssuer.issue()` to generate the JWT
4. Computes a token hash and stores it in `sessions.token_hash`
5. Updates the Session status to `active`

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

### 3.3 JWT Implementation Details

`JWTTokenIssuer` uses the `jose` library:

- Algorithm: `EdDSA` (curve `Ed25519`)
- Keys generated via `generateKeyPair('EdDSA', { crv: 'Ed25519' })`
- Internal payload is camelCase; JWT claims are snake_case
- Standard JWT `iat` and `exp` claims are also set
- Header includes `kid: 'uomp-auth-key-1'`

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

### 3.4 Guard Enforcement

`MemoryGuard.validateRequest()` validates in order:

1. `Authorization` header is `Bearer <token>`
2. JWT signature is valid
3. Token is not expired
4. Session is not revoked (via `token_blacklist` table)

Then it dispatches by request type:

- `GET /v1/memory/:key` → `isKeyAllowed()`
- `GET /v1/memory?tag=xxx` → `isTagAllowed()`, then `isKeyAllowed()` for each result

`isKeyAllowed()` decision order:

1. If key is in `denyKeys` → deny
2. If key is in `keys` → allow
3. If item `sensitivity === 'high'` → must match `keys`, else deny
4. If any item tag is in `tags` and not in `denyTags` → allow
5. Otherwise deny

### 3.5 Identity Verification (Optional)

Identity verification is performed by the **CLI on the user's machine**, not inside the Agent process. `IdentityVerifier` current implementation:

- **DID**: uses `did-resolver`, `ethr-did-resolver`, and `web-did-resolver`. In MVP it only checks that the DID document is resolvable; signature binding is not enforced.
- **GPG**: `openpgp` is imported, but `verifyGpg()` is currently a placeholder that only checks whether `proof.proofValue` exists.
- **No identity**: returns `valid=false`; the CLI on the user's machine prints a yellow warning but still allows execution.

This lowers the barrier for example Agents; production deployments should enforce identity verification on the user's host, and Agents that fail verification must not receive a Token.

---

## 4. Memory Store Implementation

`MemoryStore` is based on `better-sqlite3`:

- `memory_items` stores key, value (JSON string), tags (JSON array string), sensitivity, source, etc.
- `getByTag()` uses the SQLite JSON1 extension:

```sql
SELECT * FROM memory_items
WHERE EXISTS (SELECT 1 FROM json_each(tags) WHERE value = ?)
```

- `set()` uses `INSERT ... ON CONFLICT(key) DO UPDATE` for upsert

---

## 5. Audit Logging

`MemoryGuard` writes to `audit_logs` after every request:

- Both successful and failed requests are logged
- Includes `session_id`, `agent_id`, `action`, `key`, `tags`, `allowed`, `reason`
- MVP does not enforce read quotas, but the `limits` field is already reserved for future quota deduction

---

## 6. Local Configuration Files

After `uomp init`, the following are generated in `~/.uomp`:

- `config.json` — service port, data directory
- `uomp.sqlite` — Memory Store
- `auth.sqlite` — Sessions and blacklist
- `audit.sqlite` — Audit logs
- `.secrets/` — Ed25519 key pair (MVP regenerates on each run; production should persist)

---

## 7. MVP Limitations & Future Extensions

| Capability | MVP Status | Notes |
|------------|------------|-------|
| Agent read | ✅ Implemented | Authorized by tag/key |
| Agent write | ❌ Not open | Guard returns `503 WRITE_NOT_AVAILABLE` |
| Agent delete | ❌ Not open | Same as write |
| Remote Profile | ⚠️ Partially reserved | `profile: 'remote'`, `audience`, `allowed_endpoints` defined, but TLS/mTLS not implemented |
| Identity verification | ⚠️ Optional | DID/GPG framework present, but verification is weak |
| Query quotas | ⚠️ Reserved | `limits` written into token, but not enforced |

---

## 8. Related Links

- [Protocol Specification](/en/spec/)
- [Reference Implementation Repository](https://github.com/0xcrypto2024/uomp-core)
- [Example Agent](https://github.com/0xcrypto2024/uomp-core/tree/main/examples/calendar-agent)
