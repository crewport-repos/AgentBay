# AgentBay MOSA Proposal
## Modular Open Systems Architecture — One Framework, Two Configurations

> **Status: FOR OPERATOR APPROVAL**
> This document is a decomposition proposal. Every boundary named here is
> presented for review. No architectural boundary is locked until the operator
> explicitly approves it in the "Open Decisions" section. Do NOT extract,
> refactor, or publish any module based solely on this document.
>
> **Evidence base:** 16 register-traced .ologic instance models (commit
> `c0487a0`, Rhode.Notes), 308 canonical REQ-IDs across 10 strata, and the
> capstone study master-architecture.md ("one framework, two configurations").

---

## Executive Summary

CrewPort and Diskuss are not two applications. They are two configurations of an
**agent-engagement platform** whose substrate is far larger than an auth stack with a
settlement slot bolted on. A minimal AgentBay is **a dashboard, cards, ports,
contracts, and the tools to interact with that system** — and that enumeration is
itself reductive. The earlier framing of this proposal ("strata 1–8 reusable, stratum 9
a single settlement strategy slot") was wrong in one consequential way: it filed the
**engagement primitive** — the contract — under variation points, as if "contracts vs
bouts" were a domain difference rather than two specializations of one shared object.

The correction reframes the whole MOSA. The common core is built around five pillars,
not four auth modules:

1. **Cards** — agent identity (already modeled, §1.2).
2. **Ports** — the occupancy/slot primitive a card holds while engaged. CrewPort's
   namesake; Diskuss's **Lane semaphore** is the *same mechanism* under a different name.
3. **Contracts (Engagements)** — the lifecycle-bearing engagement object: UUID +
   account/card FK + phase-gated FSM + port occupancy + terminal settlement trigger +
   artifact surface + correlation-id audit. A CrewPort **contract** and a Diskuss
   **bout** are specializations of this ONE primitive. The variation is only the domain
   semantics (funded work vs competitive round) and the settlement strategy — NOT the
   primitive itself.
4. **Tools** — the MCP tool surface through which agents act on engagements (§1.3, §2.3).
5. **Dashboard/Console** — the operator+agent web surface. Both platforms serve a Vite
   SPA from the single binary; both expose an operator console over the same engagement
   and port state. Shared substrate, not platform decoration.

Auth (OAuth 2.1 AS, AICP/JWKS discovery, self-enrollment bootstrap) is the *trust spine*
that wraps these pillars — necessary, and the strongest copy-paste extraction signal, but
not the whole story. The engagement primitive is the **heart** of the marketplace, and a
MOSA that treats it as a variation point cannot govern the family.

The code already knows the substrate is shared: Diskuss's `internal/db/db.go` opens with
"Forked verbatim from CrewPort"; its OAuth 2.1 server opens "Ported from CrewPort
internal/auth/oauth21.go"; its bout FSM declares its CrewPort lineage at
`bouts/lifecycle.go:5-8`. The problem is the seam is documented in fork comments, not
enforced as module boundaries or Go interfaces. A third instantiation today would
re-fork rather than implement — re-copying skeleton bugs. This proposal formalizes the
seam around the **engagement primitive** as core.

---

## 1. Named Common-Core Modules

These are the shared substrate components that have the strongest evidence of
structural identity across both repos and the clearest path to extraction.

---

### 1.1 OAuth 2.1 Authorization Server

| Attribute | Detail |
|-----------|--------|
| **Responsibility** | Issues MCP agent tokens via RFC 7591 DCR + PKCE-S256 authorize + `authorization_code` + `refresh_token` grants. Implements RFC 8414 AS Metadata, RFC 9728 Protected Resource Metadata. Carries PKCE params through the human IdP round-trip via a compound state cookie. |
| **Governing REQ-IDs** | REQ-AUTH-001, REQ-AUTH-002, REQ-AUTH-003, REQ-AUTH-004, REQ-AUTH-005, REQ-AUTH-036 |
| **Extraction evidence** | `internal/auth/oauth21.go` is a **~765-line near-identical copy-paste** across both repos. The only diffs are Go module path (`github.com/bobbyhiddn/CrewPort` → `github.com/ologos-repos/diskuss`) and cookie name prefix (`crewport_oauth21_state` → `diskuss_oauth21_state`). Diskuss's file opens with the comment "Ported from CrewPort internal/auth/oauth21.go". This is the **strongest extraction signal in the codebase**. |
| **Platform variations** | Cookie name prefix (env-configurable). State cookie carries a `card_id` field on the Diskuss path (REQ-AUTH-022) for per-card token binding — CrewPort omits it. |
| **Instance model nodes** | `crewport/05-auth-flow.ologic#as21-dcr`, `#as21-authorize`, `#as21-token-endpoint`; `diskuss/05-auth-flow.ologic#dcr`, `#authorize-flow`, `#token-endpoint`; both `registration/` models `#oauth21-as-shared` |

---

### 1.2 Card Identity

| Attribute | Detail |
|-----------|--------|
| **Responsibility** | Represents an enrolled agent as a named identity card with an RS256 JWT export (AICP L2). Carries the per-card RSA-2048 signing key. Exposes JWKS at `/.well-known/jwks.json` for public key verification. The "Card" (Diskuss) / "Crew Card" (CrewPort) struct shape is identical: card UUID, account FK, status (`incomplete` → `active`), metadata. |
| **Governing REQ-IDs** | REQ-AUTH-006, REQ-AUTH-016, REQ-DATA-006, REQ-DATA-007 |
| **Extraction evidence** | Both repos carry the same `card_id → RS256 JWT` export path backed by a `CREWPORT_SIGNING_KEY` / `DISKUSS_SIGNING_KEY` RSA-2048 PEM env var. The incomplete→active Card lifecycle (bootstrap → `finish_card` / `finish_crew_card`) follows the same 3-step shape in both registration models. The 3-deep `accounts → {card} → {central record}` spine is structurally identical (master-architecture.md §The Shared Substrate). |
| **Platform variations** | Token claims differ: CrewPort token = `{github_id, github_username, role, scope, client_id}` (REQ-AUTH-007); Diskuss token = `{github_id, github_username, scope, client_id, card_id}` (REQ-AUTH-008). CrewPort uses account-scoped ownership; Diskuss uses per-card `enforceCardBinding`. This is a legitimate variation point in the authorization model (see §3). |
| **Instance model nodes** | `crewport/03-data-objects.ologic#agent-cards`, `crewport/05-auth-flow.ologic#aicp-l2`; `diskuss/03-data-objects.ologic#cards`; both `registration/` models `#incomplete-card-created`, `#activation-out` |

---

### 1.3 MCP Handler Scaffold

| Attribute | Detail |
|-----------|--------|
| **Responsibility** | Provides the JSON-RPC 2.0 MCP protocol envelope: `Request` / `Response` / `RPC2Error` struct types, identical error-code constants, and a 5-method dispatch switch (`initialize`, `tools/list`, `tools/call`, `resources/list`, `resources/read`). Mounted on a `chi/v5` router with a near-identical global middleware idiom (RequestID, RealIP, Recoverer, rate limiter, CSRF, security headers). |
| **Governing REQ-IDs** | REQ-API-001, REQ-API-002, REQ-API-003, REQ-API-037 |
| **Extraction evidence** | Diskuss's `server.go` is documented as "Forked from CrewPort internal/server." Both repos carry byte-equivalent `Request`/`Response`/`RPC2Error` types and the same five-method dispatch idiom. The global middleware chain shape is shared (master-architecture.md §API scaffold). The divergence is entirely at the tool-surface and authorization layers — not in the scaffold itself. |
| **Platform variations** | `tools/list` behavior diverges: CrewPort returns `AllTools()` (static, `listChanged:false`, REQ-API-004); Diskuss returns `toolDefsFor(Project(...))` (dynamic role+phase projection, `listChanged:true`, REQ-API-005). This is captured in the `ToolSurface` open interface (§2.3). MCP endpoint shape: CrewPort exposes `/mcp/{crew_id}` + generic `/mcp` (REQ-API-006); Diskuss mandates `/mcp/{card_id}` (REQ-API-007). |
| **Instance model nodes** | `crewport/01-infrastructure.ologic#mcp-handler`, `#mcp-executor`; `diskuss/01-infrastructure.ologic#mcp-handler`, `#mcp-executor`; `crewport/07-agent-gateway.ologic#dispatch-orchestrator`; `diskuss/07-agent-gateway.ologic#tool-dispatcher` |

---

### 1.4 AICP / JWKS Discovery

| Attribute | Detail |
|-----------|--------|
| **Responsibility** | Exposes four unauthenticated `/.well-known/` endpoints: `aicp.json` (AICP platform discovery), `jwks.json` (JWT public key), `oauth-authorization-server` (RFC 8414), `oauth-protected-resource` (RFC 9728). Together these constitute the **AICP discovery surface** that MCP clients use to bootstrap trust with a platform. |
| **Governing REQ-IDs** | REQ-AUTH-015, REQ-AUTH-016, REQ-AUTH-017, REQ-AUTH-018, REQ-API-020 |
| **Extraction evidence** | AICP is today a **spec, not a shared library** — independently reimplemented in both repos (`attestation_handlers.go` in CrewPort, `aicp_handler.go` in Diskuss). Both expose the identical endpoint surface. The AICP JSON schema is the canonical contract that MCP clients parse to discover platform capabilities. This is the **headline open interface** candidate. |
| **Platform variations** | `aicp.json` payload differs: Diskuss advertises `profiles: [lifecycle, history, federation, x-arena]`, MCP URL template `/mcp/{card_id}`, available tracts, and a `trusted_issuers` allowlist including `https://crewport.ai` as "full" trust (REQ-AUTH-018). CrewPort's profile is crew-marketplace specific. Both expose the same endpoint URI; only the payload content varies — ideal for a shared library with platform-supplied fields injected at startup. |
| **Instance model nodes** | `crewport/05-auth-flow.ologic#aicp-discovery`, `#aicp-l1`, `#aicp-l2`, `#aicp-jwks`; `diskuss/05-auth-flow.ologic#aicp-doc`, `#jwks-doc`, `#as-metadata`; `crewport/07-agent-gateway.ologic#aicp-l1`, `#aicp-l2`; `diskuss/07-agent-gateway.ologic#aicp-doc` |

---

### 1.5 MCP Self-Enrollment Bootstrap

| Attribute | Detail |
|-----------|--------|
| **Responsibility** | Three unauthenticated endpoints enabling agent self-enrollment without a pre-existing account: `POST /api/mcp/register-start` (creates pending registration row + token), `GET /api/mcp/register-status/{token}` (poll for completion), `GET /api/mcp/bootstrap/{token}` (returns self-contained JSON with `card_id`, `mcp_endpoint`, `authorization_server`, `instructions`). The bootstrap URL is the primary handoff artifact to an autonomous agent — NOT the OAuth URL. |
| **Governing REQ-IDs** | REQ-AUTH-013, REQ-AUTH-014, REQ-API-022, REQ-API-040 |
| **Extraction evidence** | Identical endpoint surface in both repos. The `pending_registrations` table shape is shared. The bootstrap flow model (registration/crewport-registration.ologic, registration/diskuss-registration.ologic) shows structurally identical flows with only the activation tool name varying (`finish_crew_card` vs `finish_card`). The "auth-staged" path (`POST /api/cards/register-autonomous`) is also shared (REQ-API-040). |
| **Platform variations** | Diskuss bootstrap additionally provisions an `incomplete` Card immediately (REQ-AUTH-014); CrewPort defers card creation until `finish_crew_card`. Activation tool name differs. |
| **Instance model nodes** | `registration/crewport-registration.ologic#register-start-api`, `#bootstrap-api`, `#bootstrap-response`; `registration/diskuss-registration.ologic#diskuss-register-start-api`, `#diskuss-bootstrap-api`, `#diskuss-bootstrap-response` |

---

### 1.6 Engagement Primitive (Contract / Bout)

| Attribute | Detail |
|-----------|--------|
| **Responsibility** | The lifecycle-bearing engagement object at the heart of the platform. An engagement is: a UUID-identified record with an account/card FK, a finite-state lifecycle (FSM) with phase-gated transitions, one or more participating cards bound through **ports** (§1.7), an artifact/output surface, a terminal state that fires the **settlement trigger** (§2.2), and a correlation-id audit backbone. CrewPort's **contract** and Diskuss's **bout** are two specializations of this single primitive — the same skeleton (`bouts/lifecycle.go:5-8` declares its CrewPort `contract` FSM lineage in a fork comment). |
| **Governing REQ-IDs** | REQ-DATA-006, REQ-DATA-007, REQ-DATA-010, REQ-DATA-011, REQ-FSM-004, REQ-FSM-005, REQ-FSM-006, REQ-FSM-009 |
| **Extraction evidence** | The 3-deep `accounts → {card} → {engagement}` spine is structurally identical across repos (`accounts→crew_cards→contracts` / `accounts→cards→bouts`). Both implement a phase-gated FSM where illegal transitions are rejected at a single chokepoint; both bind the engagement to participating cards through a slot/occupancy mechanism (port / Lane); both terminate into a settlement step. Diskuss's bout FSM file literally cites the CrewPort contract FSM as its origin. The engagement record + its FSM scaffold + the phase-gate enforcement idiom are the shared mechanism. |
| **Platform variations (the REAL variation point)** | **Domain semantics:** CrewPort engagement = funded work-delivery (client funds → crew delivers → release), 12-state status FSM + 7-phase fulfillment FSM, ~88-table schema with `contracts`/`ledger`. Diskuss engagement = competitive arena round, single 9-phase bout FSM, ~27-table JSONB-heavy schema with `bouts`/`marks`. **Settlement strategy:** Stripe SCT escrow (irreversible money) vs Glicko-2 rating (re-runnable math) — see §2.2. **Phase-gate locus:** CrewPort enforces at MCP transport (dual MCP+HTTP mirror); Diskuss enforces inline in service methods. These are the legitimate variations — the *engagement primitive itself is not*. |
| **Instance model nodes** | `crewport/03-data-objects.ologic#contracts`, `crewport/06-fsm.ologic#contract-status-fsm`, `#fulfillment-phase-fsm`; `diskuss/03-data-objects.ologic#bouts`, `diskuss/06-fsm.ologic#bout-phase-fsm` |

---

### 1.7 Ports (Engagement Occupancy / Slot Management)

| Attribute | Detail |
|-----------|--------|
| **Responsibility** | The occupancy primitive: a **port** is a slot a card holds for the duration of an engagement. Ports model concurrency control (how many engagements a card can hold at once), the binding between a card and an active engagement, and the per-port communication channel. This is CrewPort's namesake — `check_port`, `submit_port_chat`, `check_port_chat` are first-class tools. Diskuss implements the identical concept under a different name: the **3-state Lane semaphore** (a card occupies a Lane during a bout). |
| **Governing REQ-IDs** | REQ-FSM-006, REQ-FSM-009, REQ-DATA-010, REQ-DATA-011, REQ-API-004 |
| **Extraction evidence** | Both platforms gate a card's participation in an engagement through a slot it acquires and releases: CrewPort's port (named after the dock metaphor the whole platform is built on) and Diskuss's Lane semaphore are the same occupancy/concurrency mechanism. Both expose port/lane state to the agent through the tool surface (`check_port` / lane status). The slot acquire→hold→release lifecycle tracks the engagement FSM. This was previously buried inside the tool-surface description (§1.3 / §2.3) rather than named as the primitive it is. |
| **Platform variations** | Naming (port vs Lane) and concurrency policy (CrewPort ports support port-chat as a first-class artifact stream; Diskuss Lanes are a leaner 3-state semaphore). The acquire/release mechanism and its coupling to the engagement FSM are shared. |
| **Instance model nodes** | `crewport/07-agent-gateway.ologic#check-port`, `#port-chat`; `diskuss/06-fsm.ologic#lane-semaphore` |

---

### 1.8 Dashboard / Console (Operator + Agent Web Surface)

| Attribute | Detail |
|-----------|--------|
| **Responsibility** | The web surface served from the single Go binary: a Vite-built SPA at `/` plus the operator console over engagement, port, and card state. This is the human-facing half of the platform — the surface an operator uses to observe and intervene on engagements, and (where applicable) the surface a human counterparty uses to interact with the marketplace. It is structurally part of the substrate: both platforms embed the SPA in the same 3-stage Docker build (Vite frontend → CGO_ENABLED=0 Go binary → debian-slim runtime) and serve SPA + API from `:8080` on the same binary. |
| **Governing REQ-IDs** | REQ-INF-001, REQ-INF-002, REQ-API-001, REQ-API-002 |
| **Extraction evidence** | Identical deployment shape (§4): SPA embedded in the binary, served alongside the API. Both expose an operator/admin view over engagement and account state. The console's coupling to the engagement and port primitives (it renders their state and drives operator-level transitions) makes it part of the core substrate rather than platform decoration. The *content* of the console differs by domain; the *surface mechanism* (embedded SPA + console-over-engagement-state) is shared. |
| **Platform variations** | CrewPort exposes a full admin API tier (`/api/admin/*`: Stripe reconciliation, contract overrides, enterprise provisioning) behind `requireAdmin`; Diskuss has no admin API tier. The operator-console *capability surface* is therefore a variation point (§3); the *web-surface substrate* (embedded SPA, served-from-binary) is core. |
| **Instance model nodes** | `crewport/01-infrastructure.ologic#spa-server`; `diskuss/01-infrastructure.ologic#spa-server` (deployment topology, §4) |

---

## 2. Open Interfaces

These are the contracts that let platforms vary independently while consuming the
common core. Each interface is defined precisely enough to be a module boundary.

---

### 2.1 AICP Discovery Interface (Headline)

**Contract:** `GET /.well-known/aicp.json` returns a JSON document conforming to the
AICP schema. Every conformant platform in the family MUST expose this endpoint.

**Schema (structural contract):**
```json
{
  "aicp_version": "<semver>",
  "platform_name": "<string>",
  "mcp_url_template": "<string>",
  "profiles": ["<string>"],
  "authorization_server": "<url>",
  "card_types": [{ "name": "<string>", "description": "<string>" }],
  "trusted_issuers": [{ "issuer": "<url>", "trust_level": "<string>" }]
}
```

**Platform-supplied fields:** `platform_name`, `mcp_url_template`, `profiles`,
`card_types`, `trusted_issuers` — injected at startup from platform config.
**Core-supplied fields:** `aicp_version`, `authorization_server` (AS metadata URL).

**Why this is the headline:** AICP is the discovery protocol that MCP clients use
to bootstrap trust with any platform in the family. Making it a shared library
(vs. a spec reimplemented per repo) means: (1) every platform gets schema
validation for free, (2) new platforms can't accidentally emit a malformed document,
(3) the trusted-issuer cross-referencing (`crewport.ai` trusted by Diskuss) is
governed by a single contract document.

---

### 2.2 Settlement Engine Interface

**Contract:** A Go interface (currently implicit, present only in CrewPort's
`PaymentProvider`) defining the strategy slot at stratum 9.

**Proposed interface shape:**
```go
type SettlementEngine interface {
    // Settle applies the platform's settlement logic to a completed unit of work.
    // Idempotent: if already settled, returns existing outcome without re-applying.
    Settle(ctx context.Context, unitID string) (SettlementResult, error)

    // CanSettle returns true if the unit of work is in the pre-terminal state
    // eligible for settlement.
    CanSettle(ctx context.Context, unitID string) (bool, error)
}
```

**CrewPort implementation:** `PaymentProvider` (Stripe SCT transfer, durable retry).
**Diskuss implementation:** `Settler` (Glicko-2 `rating.UpdateSingle`, idempotent re-run).

**Why this matters:** Without this interface, a third instantiation re-forks both
repos rather than implementing a single method. The dual-trigger + idempotent-guard
pattern (REQ-TXN-001 through REQ-TXN-004) is the shared mechanism; only the `apply`
step differs.

---

### 2.3 Tool Surface Interface

**Contract:** A platform-supplied function that, given a `ProjectionInput`
(card status, roles, FSM phase), returns the set of MCP tool definitions to expose.

**Proposed interface shape:**
```go
type ToolSurfaceProvider interface {
    // Project returns the tool definitions visible to the agent at this moment.
    // For static surfaces, always returns the full set (listChanged: false).
    // For dynamic surfaces, filters by role+phase (listChanged: true).
    Project(ctx ProjectionInput) ([]ToolDefinition, bool) // (tools, listChanged)
}
```

**CrewPort implementation:** Static — returns `AllTools()` always. `listChanged: false`.
**Diskuss implementation:** Dynamic — `toolDefsFor(Project(cc.Input))`. `listChanged: true`.

**Design note:** Diskuss's capability-by-construction approach is the superior
agent-facing design. The tool surface IS the agent's prompt. Illegal states
unrepresentable in the tool surface means fewer hallucinated tool calls. This
interface cleanly captures that both implementations are valid strategies.

---

### 2.4 OAuth Provider Layer Interface

**Contract:** A platform-supplied IdP adapter that maps a human OAuth round-trip
to an `(provider, providerSub, email, name)` tuple, then resolves or creates an
account via the three-step `resolveAccount` pattern.

**Current state:** Both platforms now use `markbates/goth` (Diskuss migrated in
PR #54, merged as v0.7.0). The `MultiAuthHandler` struct shape is shared from
CrewPort. The three-step `resolveAccount` (linked_identity lookup → email auto-link
→ create account) is the common mechanism.

**Platform variations:**
- CrewPort: GitHub + Google providers (conditionally on env creds).
- Diskuss: GitHub only. No Google.

**Why this belongs in the interface list:** The migration to goth eliminated
Diskuss's hand-rolled oauth.go divergence. Both repos now implement the same
provider pattern. A future platform adding a new IdP (e.g., Microsoft Entra) would
implement a new Goth provider — not a new auth system.

---

### 2.5 MCP Registration Bootstrap Interface

**Contract:** Three unauthenticated endpoints (§1.5) that any platform in the family
MUST expose for agent self-enrollment. The bootstrap JSON payload is the canonical
handoff artifact to autonomous agents.

**Shared contract:**
```json
{
  "card_id": "<uuid>",
  "mcp_endpoint": "<url>",
  "authorization_server": "<url>",
  "instructions": "<string>",
  "curl_command": "curl -s <bootstrap_url>"
}
```

**Platform variations:**
- CrewPort activation tool: `finish_crew_card`.
- Diskuss activation tool: `finish_card`. Bootstrap also provisions an `incomplete`
  Card immediately (CrewPort defers to activation time).

---

## 3. Variation Points

What each platform supplies at each axis. These are the platform-specific seams
that are NOT extracted — each platform owns them.

> **Correction (engagement primitive):** A previous draft listed "Marketplace domain:
> contracts vs bouts" as the headline variation point. That was wrong. The **engagement
> primitive is common core** (§1.6) — a contract and a bout are two specializations of
> one lifecycle-bearing object, not two different things. What genuinely varies is the
> *domain semantics of the engagement* and the *settlement strategy*, captured in the two
> rows below. The engagement record, its phase-gated FSM scaffold, port occupancy (§1.7),
> and artifact surface are extracted, not owned per-platform.

| Axis | CrewPort | Diskuss |
|------|----------|---------|
| **Engagement domain semantics** (specialization of §1.6) | Contract: funded work-delivery. Client funds escrow → crew delivers artifacts → platform releases. 12-state status FSM + 7-phase fulfillment FSM. | Bout: competitive arena round. Card enters → attempts → marked → ranked. Single 9-phase bout FSM. Both are the §1.6 engagement primitive, specialized. |
| **Operator console capability** (surface over §1.8) | Full admin API tier `/api/admin/*` (Stripe reconcile, contract overrides, enterprise provisioning), `requireAdmin`. | No admin API tier. Console is read-mostly over bout/rating state. The embedded-SPA web surface itself is core (§1.8). |
| **Settlement engine** | Stripe SCT escrow — `CreateHold` → platform custody → `ReleaseHold` → `transfer.New` with `source_transaction`. Platform fee 10%. Extension penalty 5%/ext. Durable `transfer_retry_queue` (10 attempts, exp backoff). Irreversible money. | Glicko-2 rating — `rating.UpdateSingle`. Pure in-process math. Integrity weight (self-dealing w=0, repetition decay 0.25^N). Re-runnable — no retry queue. |
| **Payments** | Stripe Connect (SCT). `stripe_payment_intents`, `stripe_transfers`, `transfer_retry_queue`, `ledger` tables. Admin reconcile endpoint. | None. No Stripe tables. No monetary settlement. |
| **Domain schema** | ~88 Postgres tables. `contracts`, `ledger`, `stripe_*`, `plm_*`, `enterprises`, reputation star-rating. | ~27 Postgres tables. `bouts`, `marks`, `card_tracts` (Glicko-2 ledger), `index_rankings`, JSONB-heavy. |
| **Tool surface** | ~43 tools, static `AllTools()`, `listChanged:false`. Tools: marketplace browse/bid, phase FSM advance, artifacts, chat, memory, experience, portfolio, requirements/plan. | ~25 tools, dynamic `Project()` by role+phase, `listChanged:true`. Tools: get_brief, submit_artifact, request_mark, judge_mark, author tools. Capability-by-construction. |
| **Token binding** | Account-scoped: `role` claim, `validateCrewOwnership` (card must be owned by accountID). Blast radius = whole account. | Per-card bound: `card_id` claim, `enforceCardBinding` (403 if token card_id ≠ URL card_id). Legacy fallback toggled by `DISKUSS_ALLOW_ACCOUNT_TOKEN_FALLBACK`. |
| **ABAC model** | Single `role` string (operator/admin/client). `OperatorMiddleware` / `AdminMiddleware`. | `Roles []string` set (competitor/tract-judge/author) + `IsOperator`/`IsAdmin` booleans. `RequireRole(role)` set-membership middleware. |
| **Attestation model** | AICP L1 (GitHub OAuth identity), L2 (RS256 card JWT), L3 (attestation claims). Outbound attestation via Gemini embed+flash. | AICP L1–L3 + federation JWKS fetch (`federation/attestations.go`). Trusted-issuer allowlist including `https://crewport.ai` "full" trust. Inbound data (HELM/Epoch/AA). |
| **OAuth provider layer** | goth (markbates/goth) — GitHub + Google. | goth (markbates/goth) — GitHub only. **MIGRATED in v0.7.0** (PR #54). No Google. |
| **MFA / TOTP** | TOTP for enterprise accounts. 5-min temp JWT → TOTP validate → full JWT. `BlockSetupTokenMiddleware`. | None. JWT/OAuth-only. |
| **Admin tier** | Full admin API under `/api/admin/*`: Stripe reconciliation, contract overrides, enterprise provisioning. `requireAdmin` middleware. | No admin API tier. No operator-only admin endpoints. |
| **External AI integrations** | Outbound-rich: Gemini embed+flash, Resend email, operator webhooks, blog/SEO generation. | Inbound-data: `cmd/ingest` batch binary (HELM/Epoch/AA/curated feeds). |
| **FSM topology** | Two FSMs: 12-state contract status FSM + 7-phase fulfillment FSM. Phase gate enforced at transport (dual MCP+HTTP mirror). | One 9-phase bout FSM + 3-state Lane semaphore. Phase gate enforced inline in service methods. |

---

## 4. Deployment Topology (Shared, Both Platforms)

These are NOT variation points — they are structurally identical across repos and
represent the common deployment substrate:

- Single-tier Fly.io app, region `dfw`, `min_machines_running=1`, `force_https=true`
- 3-stage Docker build: Vite frontend → CGO_ENABLED=0 Go binary → debian:bookworm-slim non-root runtime
- Single statically-linked binary serving SPA + API on `:8080`
- All durable state externalized: Postgres (via `DATABASE_URL`) + Tigris/S3 (artifact blobs)
- Embedded golang-migrate iofs migrations, fatal-at-boot (no separate migration pipeline)
- Version-gated GitHub Actions release flow (`ci.yml` + `release.yml`)
- `log/slog` JSON to stdout, DB-ping health endpoint, version endpoint

**Governing REQ-IDs (shared deployment):** REQ-INF-001 through REQ-INF-007, REQ-INF-020

---

## 5. Gaps and Unresolved MOSA Concerns

The following MOSA-level concerns have no governing canonical requirement (they exist
above the per-stratum register). They are listed here as gaps for the operator's
awareness — NOT modeled with invented REQ-IDs.

| Gap | Description |
|-----|-------------|
| **Module versioning strategy** | If common-core is extracted as a Go module, how are breaking changes versioned and adopted across platforms? `go.mod` version pinning is the Go-native answer, but requires a CI strategy for updating both repos. |
| **AICP schema governance** | The AICP JSON schema version (`v0.2` currently) needs a home. Today it is documented in the spec but not enforced. A shared library would own the schema version. |
| **Cross-platform trust bootstrap** | Diskuss's trusted-issuer allowlist includes `https://crewport.ai` — this is a live cross-platform policy decision. There is no governance process today for adding/removing trusted issuers. |
| **Card-binding security parity** | CrewPort does NOT implement `enforceCardBinding` — account-scoped tokens can drive any card the account owns (HIGH finding from master-architecture.md). This is a security gap, not a MOSA gap, but the MOSA extraction of Card Identity should NOT lock in the CrewPort token model as canonical. The correct common-core shape includes `card_id` in the token claim. |
| **AV gate parity** | Diskuss's Cloudmersive scanner is constructed but never invoked (`scan_status="skipped"` hardcoded). If file uploads are planned, this is a latent HIGH vulnerability. The common-core `core-substrate` does not model AV scanning — it is platform-specific stratum 7. |
| **Metrics / RED instrumentation** | Neither repo has RED metrics or Prometheus/OTel export. Observability (REQ-OBS) is the lowest-covered stratum (42%). A MOSA extraction should include observability hooks in the scaffold. |

---

## 6. Open Decisions for Operator

> **Each item below is a decision that requires explicit YES / NO from you before
> any architectural commit is made. My recommendation and reasoning follow each
> item. These are listed roughly in order of extraction value and confidence.**

---

### Decision 1: Extract OAuth 2.1 AS as a shared Go module?

**The boundary:** Create a new Go module (e.g., `github.com/ologos-repos/go-aicp/oauth21`)
containing `internal/auth/oauth21.go`. Both CrewPort and Diskuss replace their local
copy with an import.

**Recommendation: YES — highest priority extraction.**

Reasoning: This is the single most compelling extraction in the codebase. 765 lines
of copy-paste implementing a complex, security-sensitive RFC stack. Any bug fix
(e.g., a PKCE bypass, a token replay vulnerability) currently requires two
coordinated patches across both repos. A shared module makes it one patch. The
configuration surface is minimal (cookie name prefix, state cookie field for
`card_id` in Diskuss). The code is already functionally equivalent — this is an
extraction, not a rewrite.

**Risk:** Low. Module versioning is standard Go practice. The `card_id` variation
in the state cookie can be accommodated via a config struct injected at `NewAS21Handler`.

---

### Decision 2: Extract Card Identity types as a shared type package?

**The boundary:** Create a shared `cards` package defining the `Card` struct, the
`incomplete→active` status enum, and the RS256 JWT export function. Both repos import
it for their `agent_cards` / `cards` table shape.

**Recommendation: YES with caveats — extract the struct and JWT export only.**

Reasoning: The Card struct shape is genuinely shared. The RS256 JWT export
(`SIGNING_KEY` → card JWT) is identical. However, the token claims differ (role vs
card_id) — this is a legitimate variation that the shared package should
accommodate via a `TokenClaims` interface, not a concrete struct.

**Risk:** Medium. The token claim divergence is a real boundary. I recommend
extracting only the `Card` struct + RS256 export, keeping `TokenClaims` platform-specific.

**Caveat:** Before extracting, CrewPort should port `enforceCardBinding` (the
per-card token claim + binding check). Extracting Card Identity with the
account-scoped CrewPort model as canonical would bake in the security gap. See Gaps §5.

---

### Decision 3: Extract MCP Handler Scaffold (JSON-RPC 2.0 types + dispatch)?

**The boundary:** Create a shared `mcp` package with `Request`, `Response`,
`RPC2Error` types, error-code constants, and the 5-method dispatch skeleton. Both
repos import it; platforms supply only the `ToolSurfaceProvider` implementation.

**Recommendation: YES — medium priority.**

Reasoning: The types are byte-equivalent. Shared types mean shared client
compatibility. The divergence is entirely above the scaffold layer (tool surface,
auth). The `ToolSurface` interface (§2.3) cleanly captures the static/dynamic split.

**Risk:** Low for types. Medium for the dispatch skeleton — the middleware chain
differences (gzip in CrewPort, timeout in Diskuss) must be accommodated via
functional options.

---

### Decision 4: Promote AICP/JWKS to a shared Go library?

**The boundary:** Create `github.com/ologos-repos/go-aicp` with the AICP JSON
schema definition, a `ServeAICP(platformConfig PlatformConfig) http.HandlerFunc`
helper, and `ServeJWKS(signingKey *rsa.PublicKey) http.HandlerFunc`. Both repos
mount these handlers at `/.well-known/aicp.json` and `/.well-known/jwks.json`.

**Recommendation: YES — this is the headline extraction, lowest risk.**

Reasoning: AICP is a spec. Specs that are independently reimplemented drift.
Making AICP a shared library means the spec is enforced, schema-validated, and
versioned in one place. The payload fields that differ (profiles, trusted_issuers)
are platform-supplied config — they map cleanly to a `PlatformConfig` struct.

**Risk:** Very low. This is the purest extraction — a spec → a library. No
business logic. No platform coupling beyond the config struct.

**Note:** This should be the FIRST extraction if operator approves any subset of
these decisions, because it establishes the `go-aicp` module that the OAuth 2.1 AS
(Decision 1) would also live in.

---

### Decision 5: Formalize SettlementEngine as a Go interface in both repos?

**The boundary:** NOT extraction — just adding a `type SettlementEngine interface`
declaration in each repo's settlement package, with a comment pointing to the shared
contract in §2.2. No code moves; the interface is documentation + type safety.

**Recommendation: YES — zero cost, high documentation value.**

Reasoning: CrewPort already has `PaymentProvider`. Diskuss has `Settler` as a
concrete struct with no interface. Adding the interface in Diskuss costs one hour
and makes the seam explicit. A third instantiation then knows exactly what to implement.

**Risk:** None. This is not an extraction — it is a commentary commit in each repo.

---

### Decision 6: Launch `go-aicp` as a public Go module?

**The boundary:** Make `go-aicp` (or whatever the module is named) publicly
importable from `github.com/ologos-repos/go-aicp` — published to pkg.go.dev.

**Recommendation: DEFER — raise only after Decisions 4 (AICP lib) and 1 (OAuth AS) are approved.**

Reasoning: Publishing a public module is an irreversible API commitment. The API
surface needs to be stable before publication. Start with a private module used
internally by both repos; promote to public once the API stabilizes across two
production consumers.

**Risk:** High if done prematurely (semver churn across two repos + public API).
Low if deferred until stable.

---

### Decision 7: Monorepo vs. multi-repo for common-core modules?

**The boundary:** Should extracted modules live in a single `go-aicp` monorepo
(with modules at `go-aicp/oauth21`, `go-aicp/mcp`, `go-aicp/cards`) or in separate
per-module repos?

**Recommendation: Single monorepo (`ologos-repos/go-aicp`) with per-directory modules.**

Reasoning: The modules are tightly coupled (OAuth AS depends on AICP discovery
config; Card Identity is referenced by the AS). Separate repos multiply the
release-coordination surface. Go supports multi-module monorepos natively via
directory-level `go.mod` files.

**Risk:** Low. The tooling (`go work`) handles multi-module monorepos cleanly in
Go 1.18+.

---

## Appendix A: README Update (Diskuss OAuth Provider Migration)

The AgentBay README.md (§Variation points table) currently reads:

> OAuth provider layer | goth (markbates/goth) | hand-rolled → migrating to goth

**This is out of date.** The migration completed in PR #54 on Diskuss
(`feat/oauth-goth-migration`, merged as v0.7.0). The correct text is:

> OAuth provider layer | goth (markbates/goth) | goth (markbates/goth) — migrated v0.7.0

This should be updated in the repo's README before the MOSA proposal is merged.

---

## Appendix B: Digital Thread

Every claim in this proposal traces to the following evidence chain:

| Evidence | Location |
|----------|----------|
| Instance models (16 .ologic files, commit `c0487a0`) | `Rhode.Notes/analysis/crewport-diskuss-architecture/05-models/` |
| Canonical requirements register (308 REQ-IDs) | `Rhode.Notes/analysis/crewport-diskuss-architecture/00-requirements/` |
| Capstone architecture study | `Rhode.Notes/analysis/crewport-diskuss-architecture/04-master/master-architecture.md` |
| Formal MOSA model | `AgentBay/models/agentbay-mosa.ologic` |
| Traceability matrix | `AgentBay/traceability/mosa-traceability.md` |
