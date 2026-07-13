# Proposal: UOMP as a user-owned memory layer for Starknet AI Agents

Hi Starknet community,

I’m building **UOMP (User-Owned Memory Protocol)** — an open protocol draft that lets users keep their personal memory data locally and grant AI Agents temporary, scoped, auditable access via short-lived Capability Tokens.

- Website: https://www.uomp.org
- Spec: https://www.uomp.org/spec/
- Reference implementation: https://github.com/0xaicrypto/uomp-core
- Demo: 30-second CLI walkthrough on the homepage

## Why this matters for AI Agents

AI Agents need rich personal context to be useful. Today that context is usually obtained in one of two ways:

1. **Upload everything to a cloud provider** — the provider owns your memory.
2. **Hand over long-lived, over-scoped API keys** — the Agent owns the access.

Neither gives the user real control. UOMP proposes a third path:

> **Your memory stays on your device. Agents receive only the minimum, temporary access you explicitly authorize.**

## How UOMP works (today)

1. **Agent declares scope** in a `uom.json` manifest — tags, keys, read/write actions.
2. **User reviews and authorizes** through a local CLI/GUI.
3. **Auth Service issues a JWT Capability Token** bound to a session.
4. **Memory Guard** filters every request against the granted scope.
5. **Session ends or is revoked** → the token becomes invalid immediately.

All access is logged, so users can audit exactly what an Agent read.

## Why Starknet?

Starknet is one of the few ecosystems seriously investing in **verifiable, decentralized AI Agents** — with native Account Abstraction, sub-cent transaction costs, and a growing agent stack (Daydreams, Giza, Starknet.agent, etc.).

I think UOMP fits naturally as the **off-chain user-memory + authorization layer**, while Starknet provides the on-chain primitives:

| Layer | Starknet primitive | UOMP role |
|---|---|---|
| Identity | Account Abstraction / DID | Verify the user and sign authorization |
| Discovery | ERC-8004-style Agent Registry | Validate Agent manifests |
| Anchoring | Cheap on-chain storage | Root session tokens, revocations, audits |
| Proof | Cairo / zk proofs | Prove access compliance without revealing data |

## A concrete integration idea

Here’s one possible end-to-end flow I’d love feedback on:

1. User has a local **Memory Store** (SQLite / file system) plus a **Memory Guard**.
2. Agent publishes a `uom.json` on an **ERC-8004 Registry** contract on Starknet.
3. User approves the scope; the local Auth Service issues a JWT.
4. The JWT hash is anchored to Starknet as a session record.
5. Agent calls the local Guard with the JWT; Guard verifies signature, checks Starknet for revocation, and returns filtered data.
6. Session ends → revocation transaction on Starknet invalidates the session.

This keeps user data local while making authorization and revocation globally verifiable.

## Current status

- **Draft-00 spec** is published and open for comments.
- **Reference implementation** in TypeScript is working end-to-end, including a calendar-agent example.
- The protocol is intentionally framework-agnostic — the goal is a shared standard, not a single product.

## What I’m asking the community

1. **Does this problem resonate** with what you’re building in the Starknet AI space?
2. **Which integration point is most valuable first** — identity, registry, revocation anchoring, or zk proofs?
3. **Are there existing Starknet standards or working groups** we should align with?
4. **Who else should see this?** Tag them in the replies.

I’d love to collaborate with Agent builders, identity folks, and anyone thinking about privacy in the agent stack. Let’s design a Starknet-native version of UOMP together.

Thanks for reading!
