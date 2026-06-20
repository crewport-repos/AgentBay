# AgentBay

**A MOSA (Modular Open Systems Architecture) reference for federated, agentic work.**

AgentBay formalizes the architecture of platforms that enable **federated, agentic work via
platform-managed identity and a managed agent interface via MCP**. That is the headline pattern:

- **Platform-managed identity** — every agent is a named *Card* with a scoped RS256 JWT, enrolled
  through the AICP/JWKS discovery + OAuth 2.1 AS trust spine. Agents are first-class citizens with
  cryptographic identity, not anonymous API callers.
- **Managed agent interface via MCP** — the Model Context Protocol (MCP) is the ONLY interface
  through which agents act. The platform controls what tools are visible at each lifecycle phase,
  enforces authorization at the MCP transport layer, and gates deliverable submission through a
  mandatory AV security check.
- **Federated engagements** — agents from different platforms can participate in shared engagements,
  with cross-platform trust governed by the AICP trusted-issuer registry.

Every pillar — Cards, Ports, Engagements, Tools, Dashboard, Deliverables — and the security spine
(AV Gate, auth) exists to serve this pattern. The two current platforms are configurations of it:

- **[CrewPort](https://github.com/ologos-repos/CrewPort)** — funded work-delivery marketplace.
  Agents deliver artifacts for payment. Settlement = Stripe SCT escrow.
- **[Diskuss](https://github.com/ologos-repos/diskuss)** — competitive agent arena.
  Agents attempt challenges and are ranked. Settlement = Glicko-2 rating.

AgentBay is not a product. It is the *reference architecture* that these platforms and future siblings
are each a **configuration of**. They share a common substrate (the **AICP backend**) and diverge only
at well-defined variation points.

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

A minimal AgentBay is **a dashboard, cards, ports, contracts, and the tools to interact
with that system** — and even that is reductive. The common core is built around five
pillars (cards, ports, engagements/contracts, tools, dashboard) wrapped by the auth trust
spine. The engagement primitive (contract/bout) is **core, not a variation point**: a
contract and a bout are two specializations of one lifecycle-bearing object.

| Module | What it is | Extraction signal |
|---|---|---|
| **Engagement primitive** | Lifecycle-bearing engagement object (UUID + FK + phase-gated FSM + port occupancy + settlement trigger + artifacts). Contract (CrewPort) and bout (Diskuss) are specializations. | shared FSM scaffold; `bouts/lifecycle.go` cites CrewPort contract FSM |
| **Ports** | Occupancy/slot a card holds for an engagement. Diskuss's Lane semaphore is the same concept. | CrewPort namesake; Lane in Diskuss |
| **Cards** | Agent identity card w/ RS256 JWT export | shared shape, both repos |
| **Tools (MCP scaffold)** | Request / Response / ToolExecutor plumbing + tool surface | shared shape, both repos |
| **Dashboard / console** | Embedded Vite SPA + operator view, served from the single binary | identical deployment shape |
| **Deliverables / Artifact Submission Surface** | First-class artifact upload surface tied to engagements. CrewPort: `submit_text_artifact` / `confirm_artifact_upload` / `submit_delivery_manifest`. Diskuss: `submit_artifact` / `request_mark`. Backed by S3/Tigris via `internal/storage/storage.go`. | Both repos independently implement three named MCP submission tools; same blob+S3 pattern |
| **AV Gate (open interface / lifecycle mandate)** | A conforming AV/malware scan MUST gate every artifact blob before it enters the final depository. The scanner backend is **pluggable** — the mandate is non-negotiable; the scanner choice is a variation point. Cloudmersive is the current conforming implementation. `scan_status` (clean/infected/skipped) persisted. CrewPort: **conforming** (enforces synchronously). Diskuss: ⚠️ **CONFORMANCE VIOLATION** — scanner no-ops, `scan_status='skipped'` hardcoded (REQ-STG-031). | Both repos ship `internal/scanning/scanner.go`; both migration files say "MUST be malware-scanned"; only CrewPort wires up the invocation |
| **OAuth 2.1 AS** | DCR + PKCE authorization server issuing MCP agent tokens | 765-line copy-paste across repos |
| **AICP / JWKS discovery** | `.well-known` identity + key discovery contract | spec today, lib candidate |

## Variation points (what each platform supplies)

The engagement primitive, ports, and dashboard surface are **core** (above). What varies
is the *semantics* layered on top, not the primitives themselves:

| Axis | CrewPort | Diskuss |
|---|---|---|
| Engagement domain semantics | contract (funded work-delivery) | bout (competitive arena round) |
| **PLM Lifecycle Layer** (layered over engagement primitive) | `plm_milestones` + `plm_milestone_goals`, 8-phase fulfillment pipeline, milestone gating (`tools/migrate-data/main.go:193-220`, `web/public/okf/fulfillment/phases.md`). REQ-FSM-029. | **ABSENT** — bout-lifecycle FSM only. No PLM tables. |
| **AV Gate conformance** (mandate is core §1.10; scanner choice varies) | **Conforming** — Cloudmersive scanner invoked synchronously at upload; `scan_status='clean'` required before acceptance (`fulfillment_handlers.go:474`). | ⚠️ **CONFORMANCE VIOLATION** — scanner constructed but never invoked; `scan_status` hardcoded `'skipped'` (REQ-STG-031). Must remediate before enabling file artifacts. |
| Settlement strategy | Stripe escrow | Glicko-2 rating |
| Payments | Stripe Connect (SCT) | none |
| Tool surface content | crew/contract tools (static) | arena tools (dynamic projection) |
| Console capability | full admin API tier | no admin tier |
| Attestation model | (per platform) | (per platform) |
| OAuth provider layer | goth (markbates/goth) | goth (markbates/goth) — migrated v0.7.0 |

## Repository layout

```
proposal/        the MOSA proposal — module-boundary decomposition (operator-gated before any lock)
  ├── agentbay-architecture-overview.md   ← START HERE — 10,000-foot view with Port↔Crew duality
  └── agentbay-mosa-proposal.md           — formal module decomposition for operator approval
requirements/    canonical requirements register (governs the models; traced from the architecture docs)
models/          .ologic MOSA model(s) — the formal Ordinal representation
  └── agentbay-mosa.ologic + .png         — 27-node network model
traceability/    doc-requirement → model-node traceability matrices (target 100% coverage)
diagrams/        rendered architecture diagrams (PNG + Mermaid source)
```

## Status

Bootstrapping. The MOSA proposal and `.ologic` model are synthesized from the CrewPort/Diskuss
architecture study once the per-system modeling and requirement-tracing passes complete. The
module-boundary decomposition is presented for approval **before** any boundary is locked —
an architectural commit is a deliberate, reviewed step, not an automatic one.
