# UOMP × Starknet — One Pager

## What is UOMP?

**UOMP (User-Owned Memory Protocol)** is an open protocol that lets users keep their personal memory data on their own device while granting AI Agents temporary, scoped, and auditable access through short-lived Capability Tokens.

- **Local-first**: Memory never leaves the user’s device unless explicitly authorized.
- **Least privilege**: Access is scoped by tags and keys, not broad permissions.
- **Session-based**: Every grant is tied to a session; revocation is instant.
- **Auditable**: All access is logged so users can see exactly what Agents read.

## Why now?

AI Agents need rich personal context to be useful. Today that context is either:

1. Uploaded to centralized clouds, or
2. Accessed via long-lived, over-scoped API keys.

Both break user sovereignty. UOMP proposes a third way: **user-owned memory with cryptographic authorization**.

## The Starknet opportunity

Starknet is building a leading stack for **verifiable, decentralized AI Agents**.
UOMP can slot in as the user-memory and authorization layer:

| Starknet primitive | How UOMP uses it |
|---|---|
| Account Abstraction | User identity, signatures, and authorization prompts |
| Agent Registry (ERC-8004-style) | Discover and verify Agent manifests |
| Cheap on-chain storage | Anchor session roots, token hashes, revocations |
| Cairo / zk proofs | Prove access compliance without revealing memory content |
| Sub-cent fees | Make per-session anchoring economically viable |

## Current status

- **Draft-00 spec** published at https://www.uomp.org/spec/
- **Reference implementation** in TypeScript with SQLite Memory Store, JWT Capability Tokens, and a CLI.
- **Working example**: calendar-agent that reads user preferences after explicit authorization.
- **Open source**: https://github.com/0xaicrypto/uomp-core

## What we’re looking for

1. **Feedback** from Agent builders on the protocol surface.
2. **Collaborators** to prototype a Starknet-native integration.
3. **Support** from the Starknet Foundation to align UOMP with the AI Agent roadmap.

## Contact

- Website: https://www.uomp.org
- GitHub Discussions: https://github.com/0xaicrypto/uomp
- Reference implementation: https://github.com/0xaicrypto/uomp-core
