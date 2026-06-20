# AgentBay

**A MOSA (Modular Open Systems Architecture) reference for the AICP-backed agent-marketplace family.**

AgentBay is not a product. It is the *reference architecture* that concrete platforms in this
family — [CrewPort](https://github.com/ologos-repos/CrewPort), [Diskuss](https://github.com/ologos-repos/diskuss),
and future siblings — are each a configuration of. They share a common substrate (the **AICP backend**)
and diverge only at well-defined variation points.

The goal of MOSA here is the standard one: **modular boundaries + open interfaces**, so the shared core
can be extracted once and reused, while each platform supplies only its domain-specific differences.

## Why this exists

CrewPort and Diskuss were built independently but converged on the same shape. Evidence already in the code:

- `internal/auth/oauth21.go` (the OAuth 2.1 Authorization Server for MCP agent-token issuance) is a
  **~765-line near-identical copy-paste** across both repos — module path swap only. Strongest extraction signal.
- **AICP** (Agent Identity Card Protocol — `/.well-known/aicp.json` + `/.well-known/jwks.json`) is a
  **spec, not a shared library** — independently reimplemented in each repo. The headline open interface.
- Diskuss's `server.go` literally notes it was *"Forked from CrewPort internal/server."*

The prior comparison study (in `Rhode.Notes/analysis/crewport-diskuss-architecture/`, capstone thesis:
*"one framework, two configurations"*) is the embryo of this repo. AgentBay formalizes it.

## Candidate common-core modules

| Module | What it is | Extraction signal |
|---|---|---|
| **OAuth 2.1 AS** | DCR + PKCE authorization server issuing MCP agent tokens | 765-line copy-paste across repos |
| **Card identity** | Agent identity card w/ RS256 JWT export | shared shape, both repos |
| **MCP handler scaffold** | Request / Response / ToolExecutor plumbing | shared shape, both repos |
| **AICP / JWKS discovery** | `.well-known` identity + key discovery contract | spec today, lib candidate |

## Variation points (what each platform supplies)

| Axis | CrewPort | Diskuss |
|---|---|---|
| Marketplace domain | contracts | bouts |
| Payments | Stripe escrow | none |
| Tool surface | crew/contract tools | discussion tools |
| Attestation model | (per platform) | (per platform) |
| OAuth provider layer | goth (markbates/goth) | hand-rolled → migrating to goth |

## Repository layout

```
proposal/        the MOSA proposal — module-boundary decomposition (operator-gated before any lock)
requirements/    canonical requirements register (governs the models; traced from the architecture docs)
models/          .ologic MOSA model(s) — the formal Ordinal representation
traceability/    doc-requirement → model-node traceability matrices (target 100% coverage)
diagrams/        rendered architecture diagrams
```

## Status

Bootstrapping. The MOSA proposal and `.ologic` model are synthesized from the CrewPort/Diskuss
architecture study once the per-system modeling and requirement-tracing passes complete. The
module-boundary decomposition is presented for approval **before** any boundary is locked —
an architectural commit is a deliberate, reviewed step, not an automatic one.
