# MOSA Traceability Matrix
## AgentBay Common-Core → Canonical Requirements → Instance Model Nodes

> **Generated:** 2026-06-20
> **MOSA model:** `AgentBay/models/agentbay-mosa.ologic`
> **Canonical register:** `Rhode.Notes/analysis/crewport-diskuss-architecture/00-requirements/` (308 REQ-IDs)
> **Instance models:** `Rhode.Notes/analysis/crewport-diskuss-architecture/05-models/` (16 .ologic files, commit `c0487a0`)
>
> This matrix proves the MOSA's common-core claim is grounded in the traced instance
> models, not asserted. Each MOSA module/node maps to the canonical REQ-IDs it governs,
> then to the specific instance-model nodes that implement it in CrewPort vs. Diskuss.

---

## 1. Common-Core Module Traceability

### 1.1 OAuth 2.1 Authorization Server (`oauth-as`)

**MOSA node ID:** `agentbay-mosa-network/common-core/core-substrate/oauth-as`
**Type:** `server`
**Extraction evidence:** `internal/auth/oauth21.go` — ~765-line near-identical copy-paste, module-path swap only.

| Canonical REQ-ID | Requirement summary | System | CrewPort instance nodes | Diskuss instance nodes |
|------------------|---------------------|--------|------------------------|------------------------|
| REQ-AUTH-001 | OAuth 2.1 AS: DCR, AS Metadata, PKCE-S256, code+refresh grants | [both] | `crewport/05-auth-flow.ologic#as21-dcr`, `#as21-authorize`, `#as21-callback`, `#as21-token-endpoint`, `crewport/01-infrastructure.ologic#oauth21-as` | `diskuss/05-auth-flow.ologic#dcr`, `#authorize-flow`, `#token-endpoint`, `diskuss/01-infrastructure.ologic#oauth21-as` |
| REQ-AUTH-002 | PKCE-S256 mandatory; plain PKCE rejected | [both] | `crewport/05-auth-flow.ologic#as21-authorize`, `#agent-oauth21-path` | `diskuss/05-auth-flow.ologic#pkce-gen` |
| REQ-AUTH-003 | Auth codes 5-min single-use, deleted on first use | [both] | `crewport/05-auth-flow.ologic#as21-token-endpoint` | `diskuss/05-auth-flow.ologic#token-endpoint` |
| REQ-AUTH-004 | Refresh tokens 30-day, rotated on use | [both] | `crewport/05-auth-flow.ologic#as21-token-endpoint`, `#refresh-token-store` | `diskuss/05-auth-flow.ologic#token-endpoint` |
| REQ-AUTH-005 | PKCE params carried through IdP round-trip via state cookie | [both] | `crewport/05-auth-flow.ologic#as21-authorize` | `diskuss/05-auth-flow.ologic#pkce-gen` |
| REQ-AUTH-036 | refresh_token grant rotates token + re-mints access token | [both] | `crewport/05-auth-flow.ologic#as21-token-endpoint` | `diskuss/05-auth-flow.ologic#token-endpoint` |

**Shared node also traced in registration models:**
`registration/crewport-registration.ologic#oauth21-as-shared`,
`registration/diskuss-registration.ologic#diskuss-oauth21-as-shared`

---

### 1.2 Card Identity (`card-identity`)

**MOSA node ID:** `agentbay-mosa-network/common-core/core-substrate/card-identity`
**Type:** `api`
**Extraction evidence:** Identical `Card` struct shape, RS256 JWT export path, incomplete→active lifecycle in both repos.

| Canonical REQ-ID | Requirement summary | System | CrewPort instance nodes | Diskuss instance nodes |
|------------------|---------------------|--------|------------------------|------------------------|
| REQ-AUTH-006 | JWT signed with HS256 shared secret; same secret for sign + verify | [both] | `crewport/05-auth-flow.ologic#jwt-mint`, `#mcp-access-token`, `#jwt-validator` | `diskuss/05-auth-flow.ologic#session-cookie`, `#login-to-authz` |
| REQ-AUTH-016 | `/.well-known/jwks.json` for JWT public key verification | [both] | `crewport/05-auth-flow.ologic#aicp-jwks`, `crewport/07-agent-gateway.ologic#aicp-l2` | `diskuss/05-auth-flow.ologic#jwks-doc` |
| REQ-DATA-006 | Crew card schema: account FK, status, metadata; 3-deep spine | [crewport] | `crewport/03-data-objects.ologic#agent-cards`, `#identity-to-cards`, `#contract-lifecycle-bridge` | _(variant: REQ-DATA-007)_ |
| REQ-DATA-007 | Diskuss card schema: account FK, status, card_tracts, bouts; 3-deep spine | [diskuss] | _(variant: REQ-DATA-006)_ | `diskuss/03-data-objects.ologic#cards`, `#identity-to-arena`, `#arena-bridge` |

**Shared node in registration models:**
`registration/crewport-registration.ologic#incomplete-card-created`, `#activation-out`;
`registration/diskuss-registration.ologic#diskuss-incomplete-card-created`, `#diskuss-activation-out`

**Platform variation note:** Token claims diverge — CrewPort: `role` string (REQ-AUTH-007),
Diskuss: `card_id` claim + `enforceCardBinding` (REQ-AUTH-008, REQ-AUTH-010). This is a
legitimate variation point in the Card Identity module; the common-core SHOULD standardize
on the `card_id` claim shape (Diskuss model is more secure — see proposal §6, Decision 2).

---

### 1.3 MCP Handler Scaffold (`mcp-scaffold`)

**MOSA node ID:** `agentbay-mosa-network/common-core/core-substrate/mcp-scaffold`
**Type:** `gateway`
**Extraction evidence:** Byte-equivalent `Request`/`Response`/`RPC2Error` types; identical 5-method dispatch. Diskuss: "Forked from CrewPort internal/server."

| Canonical REQ-ID | Requirement summary | System | CrewPort instance nodes | Diskuss instance nodes |
|------------------|---------------------|--------|------------------------|------------------------|
| REQ-API-001 | chi/v5 router; `buildRouter()`/`routes()` in one place | [both] | `crewport/01-infrastructure.ologic#api-router` | `diskuss/01-infrastructure.ologic#api-router` |
| REQ-API-002 | JSON-RPC 2.0 MCP: byte-equivalent Request/Response/RPC2Error | [both] | `crewport/01-infrastructure.ologic#mcp-handler` | `diskuss/01-infrastructure.ologic#mcp-handler` |
| REQ-API-003 | Exactly 5 MCP methods: initialize, tools/list, tools/call, resources/list, resources/read | [both] | `crewport/01-infrastructure.ologic#mcp-executor`, `crewport/05-auth-flow.ologic#tool-dispatch` | `diskuss/01-infrastructure.ologic#mcp-executor` |
| REQ-API-037 | MCP initialize response advertises server name + version | [both] | _(gap in instance models — see TRACEABILITY.md §REQ-API-037)_ | _(gap in instance models)_ |

**Platform variation nodes:**
- CrewPort: `crewport/07-agent-gateway.ologic#dispatch-orchestrator` (static tool dispatch)
- Diskuss: `diskuss/07-agent-gateway.ologic#tool-dispatcher` (dynamic projection dispatch)

---

### 1.4 AICP / JWKS Discovery (`aicp-discovery`)

**MOSA node ID:** `agentbay-mosa-network/common-core/core-substrate/aicp-discovery`
**Type:** `api`
**Extraction evidence:** AICP spec independently reimplemented in both repos (`attestation_handlers.go` vs `aicp_handler.go`). Identical endpoint surface; divergent payload.

| Canonical REQ-ID | Requirement summary | System | CrewPort instance nodes | Diskuss instance nodes |
|------------------|---------------------|--------|------------------------|------------------------|
| REQ-AUTH-015 | `/.well-known/oauth-authorization-server` (RFC 8414) + `/.well-known/oauth-protected-resource` (RFC 9728) | [both] | `crewport/05-auth-flow.ologic#as21-discovery` | `diskuss/05-auth-flow.ologic#as-metadata`, `#discovery-bridge` |
| REQ-AUTH-016 | `/.well-known/jwks.json` for JWT public key verification | [both] | `crewport/05-auth-flow.ologic#aicp-jwks`, `crewport/07-agent-gateway.ologic#aicp-l2` | `diskuss/05-auth-flow.ologic#jwks-doc` |
| REQ-AUTH-017 | `/.well-known/aicp.json` for AICP platform discovery | [both] | `crewport/05-auth-flow.ologic#aicp-discovery`, `#aicp-l1`, `#aicp-l2`, `crewport/07-agent-gateway.ologic#aicp-l1`, `#aicp-l2` | `diskuss/05-auth-flow.ologic#aicp-doc`, `diskuss/07-agent-gateway.ologic#aicp-doc` |
| REQ-API-020 | All four `/.well-known/` endpoints unauthenticated | [both] | _(gap in instance models — see TRACEABILITY.md §REQ-API-020)_ | _(gap in instance models)_ |

**Platform variation note:** `aicp.json` payload diverges at REQ-AUTH-018 (Diskuss-specific profiles,
trusted issuers). Instance node: `diskuss/01-infrastructure.ologic#aicp-discovery`,
`registration/diskuss-registration.ologic#diskuss-activation-out`.

---

### 1.5 MCP Self-Enrollment Bootstrap (`mcp-registration-interface` / `diskuss-registration-interface`)

**MOSA node IDs:** (network-level nodes)
- `agentbay-mosa-network/mcp-registration-interface` (CrewPort)
- `agentbay-mosa-network/diskuss-registration-interface` (Diskuss)
**Type:** `process`
**Extraction evidence:** Structurally identical three-endpoint bootstrap flow in both registration models.

| Canonical REQ-ID | Requirement summary | System | CrewPort instance nodes | Diskuss instance nodes |
|------------------|---------------------|--------|------------------------|------------------------|
| REQ-AUTH-013 | register-start, register-status, bootstrap/{token} — all unauthenticated | [both] | `registration/crewport-registration.ologic#register-start-api`, `#bootstrap-api`, `#bootstrap-response`, `crewport/07-agent-gateway.ologic#bootstrap-api` | `registration/diskuss-registration.ologic#diskuss-register-start-api`, `#diskuss-bootstrap-api`, `#diskuss-bootstrap-response`, `diskuss/07-agent-gateway.ologic#bootstrap-url` |
| REQ-AUTH-014 | Diskuss bootstrap provisions incomplete Card; activate via finish_card | [diskuss] | _(N/A for CrewPort — defers to finish_crew_card)_ | `registration/diskuss-registration.ologic#diskuss-bootstrap-api`, `#diskuss-early-card-gate`, `#diskuss-finish-card` |
| REQ-API-022 | POST /api/mcp/register-start, GET /api/mcp/register-status/{token}, GET /api/mcp/bootstrap/{token} | [both] | `registration/crewport-registration.ologic#register-start-api`, `#bootstrap-api` | `registration/diskuss-registration.ologic#diskuss-register-start-api`, `#diskuss-bootstrap-api` |
| REQ-API-040 | POST /api/cards/register-autonomous (authenticated, rate-limited) | [both] | `registration/crewport-registration.ologic#register-autonomous-api` | `registration/diskuss-registration.ologic#diskuss-register-autonomous-api` |

---

### 1.6 Engagement Primitive (`engagement-primitive`)

**MOSA node ID:** `agentbay-mosa-network/common-core/core-substrate/engagement-primitive`
**Type:** `process`
**Extraction evidence:** Identical 3-deep `accounts→{card}→{engagement}` spine; phase-gated FSM in both repos; `bouts/lifecycle.go:5-8` cites the CrewPort contract FSM as its origin. The engagement record + FSM scaffold + phase-gate idiom are shared; only domain semantics + settlement strategy vary.

| Canonical REQ-ID | Requirement summary | System | CrewPort instance nodes | Diskuss instance nodes |
|------------------|---------------------|--------|------------------------|------------------------|
| REQ-DATA-006 | Engagement spine (crew): accounts→crew_cards→contracts | [crewport] | `crewport/03-data-objects.ologic#contracts`, `#cards-to-contracts` | _(variant: REQ-DATA-007)_ |
| REQ-DATA-007 | Engagement spine (arena): accounts→cards→bouts | [diskuss] | _(variant: REQ-DATA-006)_ | `diskuss/03-data-objects.ologic#bouts`, `#arena-to-bouts` |
| REQ-DATA-010 | Contract engagement record: status, bid, pricing | [crewport] | `crewport/03-data-objects.ologic#contracts`, `#contract-bids` | _(variant: REQ-DATA-011)_ |
| REQ-DATA-011 | Bout engagement record: phase, test_id, contestant_card_id | [diskuss] | _(variant: REQ-DATA-010)_ | `diskuss/02-arena-flow.ologic#bout-created`, `diskuss/03-data-objects.ologic#bouts` |
| REQ-FSM-004 | 12-state contract status FSM (engagement specialization) | [crewport] | `crewport/02-market-flow.ologic#job-posted`...`#contract-complete` | _(variant: REQ-FSM-006)_ |
| REQ-FSM-005 | 7-phase fulfillment FSM gated by advance_phase | [crewport] | `crewport/06-plm.ologic#phase-validator`, `crewport/07-agent-gateway.ologic#phase-*` | _(diskuss: single FSM)_ |
| REQ-FSM-006 | 9-phase bout FSM (engagement specialization) | [diskuss] | _(variant: REQ-FSM-004)_ | `diskuss/02-arena-flow.ologic#card-idle`...`#ranked-terminal` |
| REQ-FSM-009 | Phase/occupancy gate (Lane semaphore tracks engagement) | [diskuss] | _(crewport: port)_ | `diskuss/02-arena-flow.ologic#lane-check`, `diskuss/03-data-objects.ologic#lanes` |

**Correction note:** This module was previously misfiled under §3 Variation Points ("Marketplace
domain: contracts vs bouts"). It is CORE: a contract and a bout are two specializations of one
lifecycle-bearing engagement object. The variation is domain semantics + settlement strategy, NOT
the primitive. See proposal §1.6.

---

### 1.7 Ports / Occupancy (`ports`)

**MOSA node ID:** `agentbay-mosa-network/common-core/core-substrate/ports`
**Type:** `queue`
**Extraction evidence:** CrewPort's namesake (`check_port`, `submit_port_chat`); Diskuss's 3-state Lane semaphore is the same occupancy/concurrency mechanism. Acquire→hold→release lifecycle tracks the engagement FSM.

| Canonical REQ-ID | Requirement summary | System | CrewPort instance nodes | Diskuss instance nodes |
|------------------|---------------------|--------|------------------------|------------------------|
| REQ-FSM-006 | Slot occupied for engagement duration | [both] | `crewport/07-agent-gateway.ologic#card-port-tools` | `diskuss/02-arena-flow.ologic#lane-check` |
| REQ-FSM-009 | 3-state occupancy semaphore: free → occupied → draining | [diskuss] | _(crewport: port lifecycle)_ | `diskuss/02-arena-flow.ologic#lane-check`, `diskuss/03-data-objects.ologic#lanes` |
| REQ-DATA-010 | Port/engagement binding record | [crewport] | `crewport/03-data-objects.ologic#contracts` | _(variant: REQ-DATA-011)_ |
| REQ-DATA-011 | Lane/bout binding record | [diskuss] | _(variant: REQ-DATA-010)_ | `diskuss/03-data-objects.ologic#lanes` |
| REQ-API-004 | Port state exposed to agent via tool surface (check_port) | [crewport] | `crewport/07-agent-gateway.ologic#card-port-tools` | _(diskuss: lane status)_ |

**Note:** Ports were previously buried inside the tool-surface description. Named here as the core
occupancy primitive (port = Lane). See proposal §1.7.

---

### 1.8 Dashboard / Console (`dashboard-console`)

**MOSA node ID:** `agentbay-mosa-network/common-core/core-substrate/dashboard-console`
**Type:** `monitor`
**Extraction evidence:** Identical deployment shape (proposal §4) — Vite SPA embedded in the single Go binary, served alongside the API from `:8080`. Both expose an operator/admin view over engagement+account state.

| Canonical REQ-ID | Requirement summary | System | CrewPort instance nodes | Diskuss instance nodes |
|------------------|---------------------|--------|------------------------|------------------------|
| REQ-INF-001 | Single statically-linked binary serving SPA + API | [both] | `crewport/01-infrastructure.ologic#api-router` | `diskuss/01-infrastructure.ologic#api-router` |
| REQ-INF-002 | 3-stage Docker build: Vite frontend → Go binary → debian-slim | [both] | _(deployment topology)_ | _(deployment topology)_ |
| REQ-API-001 | chi/v5 router serves SPA + API in one place | [both] | `crewport/01-infrastructure.ologic#api-router` | `diskuss/01-infrastructure.ologic#api-router` |
| REQ-API-002 | API surface backing the console | [both] | `crewport/01-infrastructure.ologic#mcp-handler` | `diskuss/01-infrastructure.ologic#mcp-handler` |

**Variation note:** CrewPort exposes a full admin API tier (`/api/admin/*`, `requireAdmin`); Diskuss
has no admin tier. The embedded-SPA web-surface MECHANISM is core; the console capability surface
varies (proposal §3). See proposal §1.8.

---

### 1.9 Settlement Interface / Strategy Slot (`settlement-interface`)

**MOSA node ID:** `agentbay-mosa-network/common-core/settlement-interface`
**Type:** `oracle`
**Notes:** This is the standalone node that makes common-core a FACTORY (machine→node connection via `inputs: [core-substrate]`). It represents the design pattern — not a concrete implementation.

| Canonical REQ-ID | Requirement summary | System | CrewPort instance nodes | Diskuss instance nodes |
|------------------|---------------------|--------|------------------------|------------------------|
| REQ-TXN-001 | Settlement fired on FSM penultimate→terminal edge | [both] | `crewport/04-transaction-custody.ologic#transfers`, `crewport/02-market-flow.ologic#contract-complete` | `diskuss/04-settlement.ologic#settler`, `diskuss/02-arena-flow.ologic#ranked-terminal` |
| REQ-TXN-002 | Dual trigger: synchronous handler + async background sweep | [both] | `crewport/04-transaction-custody.ologic#auto-release-sweep` | `diskuss/04-settlement.ologic#sweep-trigger` |
| REQ-TXN-003 | Idempotency guard: if already settled, return existing outcome | [both] | `crewport/04-transaction-custody.ologic#transfers` | `diskuss/04-settlement.ologic#idempotency-guard` |
| REQ-TXN-004 | Settlement failure parks in pre-terminal state for retry/sweep | [both] | `crewport/04-transaction-custody.ologic#retry-queue` | `diskuss/04-settlement.ologic#err-not-marked-skip` |

---

### 1.9 Deliverables / Artifact Submission Surface (`deliverables`)

**MOSA node ID:** `agentbay-mosa-network/common-core/core-substrate/deliverables`
**Type:** `process`
**Extraction evidence:** Both repos expose named MCP deliverable-submission tools backed by the same S3/Tigris blob storage pattern. Previously buried as a sub-bullet of the engagement primitive (REQ-DATA-010/011) — elevated to first-class pillar.

| Canonical REQ-ID | Requirement summary | System | CrewPort instance nodes | Diskuss instance nodes |
|------------------|---------------------|--------|------------------------|------------------------|
| REQ-DATA-036 | First-class deliverable submission surface distinct from the engagement FSM primitive; storage via S3/Tigris | [both] | `crewport/07-agent-gateway.ologic#fulfillment-tools` | `diskuss/07-agent-gateway.ologic#active-tools`, `#post-mark-tools` |
| REQ-API-041 | CrewPort MCP tools: `submit_text_artifact`, `confirm_artifact_upload`, `submit_delivery_manifest` | [crewport] | `crewport/07-agent-gateway.ologic#fulfillment-tools`, `crewport/01-infrastructure.ologic#mcp-executor` | _(N/A — Diskuss variation: REQ-API-042)_ |
| REQ-API-042 | Diskuss MCP tools: `submit_artifact` (in-attempt phase), `request_mark` (submitted phase) | [diskuss] | _(N/A — CrewPort variation: REQ-API-041)_ | `diskuss/07-agent-gateway.ologic#active-tools`, `diskuss/02-arena-flow.ologic#in-attempt` |
| REQ-DATA-010 | Contract engagement record includes artifact/output surface | [crewport] | `crewport/03-data-objects.ologic#contracts`, `#contract-bids` | _(variant: REQ-DATA-011)_ |
| REQ-DATA-011 | Bout engagement record includes artifact/output surface | [diskuss] | _(variant: REQ-DATA-010)_ | `diskuss/02-arena-flow.ologic#bout-created`, `diskuss/03-data-objects.ologic#bouts` |

**Note:** REQ-DATA-010 and REQ-DATA-011 were previously traced only to the `engagement-primitive` node.
They are now ALSO traced here, acknowledging that the artifact surface is a distinct first-class pillar
(not just an attribute of the engagement record). The engagement-primitive node retains its references
for completeness; both nodes share governance of these REQ-IDs.

---

### 1.10 AV Gate — Malware Scan Mandate (`av-scanning`)

**MOSA node ID:** `agentbay-mosa-network/common-core/core-substrate/av-scanning`
**Type:** `firewall`
**Architecture:** This is an **open interface / lifecycle gate mandate** — not a vendor requirement. The mandate (a conforming scan MUST gate artifact promotion to the final depository) is non-negotiable. The scanner backend is pluggable. Cloudmersive is the current conforming implementation in both repos. The variation point is the scanner choice. Diskuss's no-op is a **CONFORMANCE VIOLATION** of the mandate, not merely a "variation."

| Canonical REQ-ID | Requirement summary | System | CrewPort instance nodes | Diskuss instance nodes |
|------------------|---------------------|--------|------------------------|------------------------|
| REQ-STG-028 | Every artifact blob MUST be malware-scanned before acceptance; `scan_status` (clean/infected/skipped) MUST be persisted | [both] | `crewport/07-agent-gateway.ologic#fulfillment-tools` (gate enforces this) | `diskuss/07-agent-gateway.ologic` (**absent** — scanner constructed, never invoked; this is the defect gap) |
| REQ-STG-029 | Only `scan_status='clean'` artifacts accepted; `scan_status='infected'` rejected at upload | [both] | `crewport/07-agent-gateway.ologic#fulfillment-tools` (fulfillment_handlers.go:474 gate) | _(Diskuss: not yet implemented; REQ-STG-031 defect flag)_ |
| REQ-STG-030 | CrewPort enforces AV gate synchronously: scan on upload → persist status → reject if not clean | [crewport] | `crewport/07-agent-gateway.ologic#fulfillment-tools`, `crewport/01-infrastructure.ologic#mcp-executor` | _(N/A — CrewPort-only enforcement)_ |
| REQ-STG-031 | Diskuss no-ops the AV gate: scanner built but never invoked; `scan_status='skipped'` hardcoded — LATENT DEFECT | [diskuss] | _(N/A — Diskuss-specific defect flag)_ | `diskuss/07-agent-gateway.ologic` (absence of scanner invocation = compliance gap) |

**Security note:** REQ-STG-031 is a HIGH-severity **CONFORMANCE VIOLATION**. The AgentBay MOSA AV Gate
mandate (REQ-STG-028) is non-negotiable. The `av-scanning` firewall node formalizes this open interface
contract so no future instantiation can silently ship with an uninvoked scanner. Any platform that does
NOT invoke a conforming scanner at the artifact promotion gate MUST be flagged as non-conformant.

---

## 2. Platform-Specific Variation Point Traceability

### 2.1 CrewPort Platform Variations

#### PLM Lifecycle Layer (Variation Point) — `crewport-plm`

**MOSA node ID:** `agentbay-mosa-network/crewport-platform/crewport-impl/crewport-plm`
**Type:** `orchestrator`
**Evidence:** CrewPort-only. Diskuss has no PLM layer (only the bout-lifecycle FSM).

| Canonical REQ-ID | Requirement summary | CrewPort instance nodes |
|------------------|---------------------|------------------------|
| REQ-FSM-029 | PLM lifecycle layer: `plm_milestones` + `plm_milestone_goals`, 8-phase pipeline, milestone gating | `crewport/06-plm.ologic#phase-validator`, `#pipeline-coordinator`, `crewport/03-data-objects.ologic` (plm_* tables) |
| REQ-FSM-005 | 7-phase fulfillment FSM gated by `advance_phase` (part of PLM layer) | `crewport/06-plm.ologic#phase-validator`, `crewport/07-agent-gateway.ologic#phase-accepted`, `#phase-plan`, `#phase-working`, `#phase-submitting`, `#phase-review` |

**Variation note:** This PLM layer is **absent from Diskuss**. The engagement primitive (§1.6) is shared;
the PLM lifecycle layer wrapping it is CrewPort-only. A future platform MAY supply a PLM layer; it
is not required by the MOSA common core.

---

#### OAuth Provider: Goth (GitHub + Google) — `crewport-oauth-provider`

| Canonical REQ-ID | Requirement summary | CrewPort instance nodes |
|------------------|---------------------|------------------------|
| REQ-AUTH-019 | GitHub + Google via goth, configurable at startup | `crewport/01-infrastructure.ologic#github-oauth`, `crewport/01-infrastructure.ologic#google-oauth`, `crewport/05-auth-flow.ologic#github-idp` |
| REQ-AUTH-021 | Authorize redirects to /auth/multi/github (Goth) | `crewport/05-auth-flow.ologic#github-redirect`, `registration/crewport-registration.ologic#goth-redirect` |
| REQ-AUTH-035 | OAuth callback completes pending registrations | `crewport/05-auth-flow.ologic#oauth-callback`, `registration/crewport-registration.ologic#goth-callback`, `#complete-pending-reg` |

#### Contract Marketplace Domain — `crewport-domain`

| Canonical REQ-ID | Requirement summary | CrewPort instance nodes |
|------------------|---------------------|------------------------|
| REQ-DATA-006 | Crew card schema; 3-deep accounts→crew_cards→contracts spine | `crewport/03-data-objects.ologic#agent-cards`, `#contracts`, `#identity-to-cards`, `#cards-to-contracts` |
| REQ-DATA-010 | Contracts with lifecycle status, bid, pricing | `crewport/03-data-objects.ologic#contracts`, `#contract-bids` |
| REQ-FSM-004 | 12-state contract status FSM | `crewport/02-market-flow.ologic#job-draft`, `#job-posted`, `#bidding-pool`, `#bid-selected`, `#contract-funded`, `#work-submitted`, `#review-outcome`, `#contract-complete` |
| REQ-FSM-005 | 7-phase fulfillment FSM gated by advance_phase tool | `crewport/06-plm.ologic#phase-validator`, `#pipeline-coordinator`, `crewport/07-agent-gateway.ologic#phase-accepted`, `#phase-requirements`, `#phase-plan`, `#phase-working`, `#phase-submitting`, `#phase-review` |
| REQ-AUTH-032 | ABAC: single role string (operator/admin/client) | `crewport/03-data-objects.ologic#admin-membership`, `crewport/05-auth-flow.ologic#role-guard`, `#cpk-claims` |

#### Stripe Escrow Settlement Engine — `crewport-settlement`

| Canonical REQ-ID | Requirement summary | CrewPort instance nodes |
|------------------|---------------------|------------------------|
| REQ-TXN-005 | PaymentProvider adapter interface with CreateHold, ReleaseHold, Refund, etc. | `crewport/04-transaction-custody.ologic#spi` |
| REQ-TXN-008 | ReleaseHold dedup guard: reject if transfer already exists | `crewport/04-transaction-custody.ologic#transfers` |
| REQ-TXN-009 | Platform fee 10% retained; crew_payout_cents resolution | `crewport/06-plm.ologic#pricing-ledger`, `#transfer-ledger` |
| REQ-TXN-011 | Transfer with TransferGroup=contractID, metadata | `crewport/04-transaction-custody.ologic#transfers`, `crewport/03-data-objects.ologic#stripe-transfers` |
| REQ-TXN-012 | On success: insertTransfer. On failure: EnqueueRetry | `crewport/04-transaction-custody.ologic#retry-queue` |
| REQ-TXN-013 | transfer_retry_queue table; 10 attempts; exp backoff | `crewport/03-data-objects.ologic#transfer-retry-queue`, `crewport/04-transaction-custody.ologic#retry-queue` |

#### Static Tool Surface — `crewport-toolsurface`

| Canonical REQ-ID | Requirement summary | CrewPort instance nodes |
|------------------|---------------------|------------------------|
| REQ-API-004 | tools/list returns AllTools() without filtering; listChanged:false | `crewport/07-agent-gateway.ologic#tool-projection`, `#card-port-tools`, `#marketplace-tools`, `#fulfillment-tools` |
| REQ-API-039 | tools/call scope gate enforced via checkScope | `crewport/05-auth-flow.ologic#mcp-scope-gate`, `crewport/07-agent-gateway.ologic#scope-gate` |
| REQ-AUTH-009 | Account-owned card gate at MCP handler (validateCrewOwnership) | `crewport/05-auth-flow.ologic#ownership-check`, `#crew-ownership-gate`, `crewport/07-agent-gateway.ologic#ownership-check` |
| REQ-AUTH-030 | tools/list returns full static set regardless of card/role/phase | `crewport/07-agent-gateway.ologic#tool-projection` |

---

### 2.2 Diskuss Platform Variations

#### OAuth Provider: Goth GitHub only (migrated v0.7.0) — `diskuss-oauth-provider`

| Canonical REQ-ID | Requirement summary | Diskuss instance nodes |
|------------------|---------------------|------------------------|
| REQ-AUTH-020 | GitHub-only OAuth; no Google | `diskuss/01-infrastructure.ologic#github-oauth`, `diskuss/05-auth-flow.ologic#github-redirect` |
| REQ-AUTH-022 | Authorize endpoint accepts card_id query param; encoded in state cookie | `diskuss/05-auth-flow.ologic#authorize-flow`, `registration/diskuss-registration.ologic#diskuss-oauth-account-link` |
| REQ-AUTH-023 | Authorize redirects to /app/login (not directly to GitHub) | `diskuss/05-auth-flow.ologic#github-redirect`, `registration/diskuss-registration.ologic#diskuss-oauth-redirect` |
| REQ-AUTH-035 | OAuth callback completes pending registrations via onAuthenticated | `diskuss/05-auth-flow.ologic#github-callback`, `registration/diskuss-registration.ologic#diskuss-complete-pending-reg` |

#### Arena / Bout Domain — `diskuss-domain`

| Canonical REQ-ID | Requirement summary | Diskuss instance nodes |
|------------------|---------------------|------------------------|
| REQ-DATA-007 | Card schema; 3-deep accounts→cards→bouts spine | `diskuss/03-data-objects.ologic#cards`, `#bouts`, `#identity-to-arena`, `#arena-to-bouts` |
| REQ-DATA-011 | Bouts with phase, test_id, contestant_card_id | `diskuss/02-arena-flow.ologic#bout-created`, `diskuss/03-data-objects.ologic#bouts` |
| REQ-FSM-006 | 9-phase bout FSM | `diskuss/02-arena-flow.ologic#card-idle`, `#bout-created`, `#briefed`, `#in-attempt`, `#submitted`, `#under-review`, `#ranked-terminal` |
| REQ-FSM-009 | Lane semaphore (3-state): free → occupied → draining | `diskuss/02-arena-flow.ologic#lane-check`, `diskuss/03-data-objects.ologic#lanes` |
| REQ-AUTH-033 | ABAC: Roles[]string set + IsOperator/IsAdmin booleans | `diskuss/03-data-objects.ologic#card-roles`, `diskuss/05-auth-flow.ologic#authz-router`, `#account-check` |

#### Glicko-2 Rating Settlement Engine — `diskuss-settlement`

| Canonical REQ-ID | Requirement summary | Diskuss instance nodes |
|------------------|---------------------|------------------------|
| REQ-TXN-023 | Mark score ∈ [0,1] via 3 scoring paths: graders / judge consensus / frozen-IRT | `diskuss/04-settlement.ologic#mode-router` |
| REQ-TXN-027 | Settle via rating.UpdateSingle; UpsertCardTractRating | `diskuss/04-settlement.ologic#settler`, `diskuss/03-data-objects.ologic#bouts-to-rating` |
| REQ-TXN-028 | Integrity weight: self-dealing w=0; repetition decay 0.25^N | `diskuss/04-settlement.ologic#integrity-weight` |
| REQ-TXN-036 | No Stripe tables, no ledger, no monetary settlement | `diskuss/03-data-objects.ologic#arena-bridge` (absence = compliance) |

#### Dynamic Tool Projection — `diskuss-toolsurface`

| Canonical REQ-ID | Requirement summary | Diskuss instance nodes |
|------------------|---------------------|------------------------|
| REQ-API-005 | tools/list returns toolDefsFor(Project()); listChanged:true | `diskuss/07-agent-gateway.ologic#tool-projector`, `#idle-tools`, `#active-tools`, `#post-mark-tools`, `#judge-tools`, `#author-tools` |
| REQ-API-038 | tools/call re-enforces projection (not only at list time) | `diskuss/07-agent-gateway.ologic#tool-dispatcher`, `diskuss/01-infrastructure.ologic#mcp-executor` |
| REQ-AUTH-031 | tools/list projects by card role + bout phase | `diskuss/07-agent-gateway.ologic#tool-projector`, `#runtime-to-tools` |
| REQ-AUTH-010 | enforceCardBinding: token card_id must match URL card_id | `diskuss/01-infrastructure.ologic#mcp-scope-mw`, `diskuss/05-auth-flow.ologic#card-binding-gate`, `diskuss/07-agent-gateway.ologic#bearer-gate` |

---

## 3. AICP Open Interface Traceability (Network-Level)

**MOSA node:** `agentbay-mosa-network/aicp-open-interface`
This node connects `common-core` factory to both platform factories at the network level.
It represents the headline open interface: every platform MUST implement `/.well-known/aicp.json`.

| Canonical REQ-ID | CrewPort implementation | Diskuss implementation |
|------------------|------------------------|------------------------|
| REQ-AUTH-017 | `crewport/05-auth-flow.ologic#aicp-discovery`, `#aicp-l1`, `crewport/07-agent-gateway.ologic#aicp-l1` | `diskuss/05-auth-flow.ologic#aicp-doc`, `diskuss/07-agent-gateway.ologic#aicp-doc`, `diskuss/01-infrastructure.ologic#aicp-discovery` |
| REQ-API-020 | _(gap: REQ-API-020 has no model node in instance models — see instance TRACEABILITY.md §REQ-API-020)_ | _(gap: same)_ |

---

## 4. Coverage Summary

### MOSA Model → Canonical Register Coverage

| MOSA Node | REQ-IDs governed | Status |
|-----------|-----------------|--------|
| `oauth-as` | REQ-AUTH-001, -002, -003, -004, -005, -036 | 6 reqs governed |
| `card-identity` | REQ-AUTH-006, -016, REQ-DATA-006, -007 | 4 reqs governed |
| `mcp-scaffold` | REQ-API-001, -002, -003, -037 | 4 reqs governed |
| `aicp-discovery` | REQ-AUTH-015, -016, -017, REQ-API-020 | 4 reqs governed |
| `engagement-primitive` | REQ-DATA-006, -007, -010, -011, REQ-FSM-004, -005, -006, -009 | 8 reqs governed |
| `ports` | REQ-FSM-006, -009, REQ-DATA-010, -011, REQ-API-004 | 5 reqs governed |
| `dashboard-console` | REQ-INF-001, -002, REQ-API-001, -002 | 4 reqs governed |
| `deliverables` *(NEW — Gap 3)* | REQ-DATA-036, REQ-API-041, REQ-API-042, REQ-DATA-010, REQ-DATA-011 | 5 reqs governed |
| `av-scanning` *(NEW — Gap 1)* | REQ-STG-028, REQ-STG-029, REQ-STG-030, REQ-STG-031 | 4 reqs governed |
| `settlement-interface` | REQ-TXN-001, -002, -003, -004 | 4 reqs governed |
| `crewport-oauth-provider` | REQ-AUTH-019, -021, -035 | 3 reqs governed |
| `crewport-domain` | REQ-DATA-006, -010, REQ-FSM-004, -005, REQ-AUTH-032 | 5 reqs governed |
| `crewport-plm` *(NEW — Gap 2)* | REQ-FSM-029, REQ-FSM-005 | 2 reqs governed |
| `crewport-settlement` | REQ-TXN-001, -002, -003, -005, -008, -009, -011, -012, -013 | 9 reqs governed |
| `crewport-toolsurface` | REQ-API-004, -039, REQ-AUTH-009, -030 | 4 reqs governed |
| `crewport-core-link` | REQ-AUTH-001, -013, -017, REQ-API-006 | 4 reqs governed |
| `diskuss-oauth-provider` | REQ-AUTH-020, -022, -023, -035 | 4 reqs governed |
| `diskuss-domain` | REQ-DATA-007, -011, REQ-FSM-006, -009, REQ-AUTH-033 | 5 reqs governed |
| `diskuss-settlement` | REQ-TXN-001, -002, -003, -023, -027, -028, -036 | 7 reqs governed |
| `diskuss-toolsurface` | REQ-API-005, -038, REQ-AUTH-031, -010 | 4 reqs governed |
| `diskuss-core-link` | REQ-AUTH-001, -013, -017, -018, REQ-API-007 | 5 reqs governed |
| `aicp-open-interface` | REQ-AUTH-017, REQ-API-020 | 2 reqs governed |
| `mcp-registration-interface` | REQ-AUTH-013, REQ-API-022, -040 | 3 reqs governed |
| `diskuss-registration-interface` | REQ-AUTH-013, -014, REQ-API-022, -040 | 4 reqs governed |

**Unique canonical REQ-IDs governed by the MOSA model:** ~51 distinct requirements
(across 10 strata: AUTH, DATA, API, FSM, TXN, STG — added 11 new REQ-IDs in gap closure:
REQ-DATA-036, REQ-API-041, REQ-API-042, REQ-STG-028 through -031, REQ-FSM-029 + the
nodes `deliverables`, `av-scanning`, `crewport-plm` added to the model).

### Instance Model Coverage (from Rhode.Notes TRACEABILITY.md)

The underlying instance models cover 176/308 (57.1%) of the canonical register.
The MOSA model does not increase this count — it operates at a higher abstraction
level, citing the same canonical IDs already covered by instance nodes. It does,
however, bring the 132 uncovered requirements into sharper relief as modeling gaps,
particularly in:
- REQ-CFG (Config & Secrets): 24/30 gaps — lowest-modeled stratum
- REQ-EXT (External Integrations): 16/21 gaps
- REQ-OBS (Observability): 15/26 gaps

---

## 5. Orphan Analysis

### MOSA nodes with instance-model coverage gaps

The following MOSA nodes trace to REQ-IDs that have `_(gap)_` in the instance
TRACEABILITY.md — meaning the instance models don't yet cover these requirements,
even though the MOSA model references them:

| MOSA Node | REQ-ID | Gap note |
|-----------|--------|----------|
| `aicp-discovery` | REQ-API-020 | Instance gap: no node in either platform model explicitly implements /.well-known unauthenticated requirement |
| `mcp-scaffold` | REQ-API-037 | Instance gap: MCP initialize server-name claim not modeled |
| `aicp-open-interface` | REQ-API-020 | Same as above |

These gaps are inherited from the instance model coverage (57.1%) — they are known
and documented in Rhode.Notes TRACEABILITY.md. They do not invalidate the MOSA
claims but indicate areas where future modeling passes should add instance nodes.

### MOSA-level concerns with NO canonical requirement

The following architectural concerns at the MOSA level have no governing canonical
requirement ID. They are listed in `proposal/agentbay-mosa-proposal.md §5` (Gaps)
and are NOT modeled with invented IDs:

- Module versioning strategy (go.mod discipline across repos)
- AICP schema governance (schema version ownership)
- Cross-platform trust bootstrap governance (trusted-issuer allowlist lifecycle)
- Card-binding security parity (CrewPort should port enforceCardBinding)
- ~~AV gate parity~~ — **CLOSED:** now governed by REQ-STG-028 through REQ-STG-031; modeled as `av-scanning` firewall node in core-substrate chain.
- ~~Deliverables as sub-bullet~~ — **CLOSED:** now governed by REQ-DATA-036, REQ-API-041, REQ-API-042; modeled as `deliverables` process node in core-substrate chain.
- ~~PLM unnamed~~ — **CLOSED:** now governed by REQ-FSM-029; modeled as `crewport-plm` orchestrator node in crewport-impl machine.
- Metrics / RED instrumentation (both repos have zero metrics)

---

## 6. Common-Core Claim Summary

The "common-core" claim in the MOSA is grounded as follows:

| Common-Core Module | Grounding evidence |
|-------------------|--------------------|
| OAuth 2.1 AS | ~765-line copy-paste `internal/auth/oauth21.go`; "Ported from CrewPort" comment in Diskuss; identical flows in both registration models (commit `c0487a0`) |
| Card Identity | Identical 3-deep spine in both repo schemas (master-architecture.md §Shared Substrate); identical incomplete→active lifecycle in both registration models |
| **Engagement Primitive** | Identical accounts→card→engagement spine; phase-gated FSM in both; `bouts/lifecycle.go:5-8` cites CrewPort contract FSM as origin. Contract and bout are specializations of one object — CORE, not a variation point |
| **Ports / Occupancy** | CrewPort namesake (`check_port`, `submit_port_chat`); Diskuss 3-state Lane semaphore is the same acquire→hold→release occupancy mechanism tracking the engagement FSM |
| **Dashboard / Console** | Identical deployment shape: Vite SPA embedded in single binary, served with API from :8080 (proposal §4); operator/admin view over engagement state in both |
| **Deliverables / Artifact Submission Surface** *(Gap 3 — elevated)* | Both repos expose named MCP deliverable-submission tools (3 tools each) backed by the same S3/Tigris blob storage pattern. CrewPort: `submit_text_artifact` / `confirm_artifact_upload` / `submit_delivery_manifest`. Diskuss: `submit_artifact` / `request_mark`. Same `internal/storage/storage.go` S3/Tigris backend pattern. Previously buried as sub-bullet of REQ-DATA-010/011 — elevated to first-class pillar with dedicated REQ-DATA-036, REQ-API-041, REQ-API-042. |
| **AV Gate / Malware Scan Mandate** *(Gap 1 — added as open interface)* | Both repos ship `internal/scanning/scanner.go` (Cloudmersive as the current conforming backend). Both define `scan_status` on artifact records. Both migration files state "MUST be malware-scanned." The AV gate is a **non-negotiable lifecycle mandate** (REQ-STG-028) with a pluggable scanner backend — not a vendor requirement. CrewPort CONFORMS (REQ-STG-030); Diskuss has a CONFORMANCE VIOLATION (REQ-STG-031). |
| MCP Handler Scaffold | "Forked from CrewPort internal/server" in Diskuss server.go; byte-equivalent Request/Response/RPC2Error types verified in architect analysis §05 |
| AICP/JWKS Discovery | Identical endpoint surface independently reimplemented; same spec in both repos |
| MCP Self-Enrollment Bootstrap | Structurally identical 3-endpoint flow in both registration models; only activation tool name differs |
| Settlement Pattern | Identical 7-beat dual-trigger+idempotent-guard mechanism; only engine implementation differs (master-architecture.md §Settlement the slot) |

**Variation Point Additions (this revision):**

| Named Variation Point | Evidence |
|-----------------------|----------|
| **PLM Lifecycle Layer** *(Gap 2 — added; CrewPort only)* | `plm_milestones` + `plm_milestone_goals` tables; 8-phase fulfillment pipeline (`tools/migrate-data/main.go:193-220`); `web/public/okf/fulfillment/phases.md`. ABSENT from Diskuss (no PLM tables, bout-lifecycle FSM only). Governed by REQ-FSM-029. |

**Conclusion:** The common-core claim is NOT asserted — it is derived from 16 register-traced
instance models, a 10-stratum architecture study, and direct code evidence. This revision adds
three previously missing elements (Deliverables pillar, AV scanning module, PLM variation point)
identified by auditing the actual crepo/diskuss codebases against the prior MOSA draft.
The MOSA model formalizes what the code already embodies.
