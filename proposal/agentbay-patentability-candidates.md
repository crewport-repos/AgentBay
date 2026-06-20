# AgentBay — Patent Candidate Register (Phase 0 Sweep)

> **⚠️ LEGAL DISCLAIMER**
> This document is **structured engineering input** prepared to assist the IP attorney
> in identifying candidates for patentability evaluation. It is NOT a legal opinion, a
> prior-art search, a freedom-to-operate analysis, or a determination that any described
> mechanism is patentable. Nothing here constitutes legal advice. All conclusions about
> patentability, claim scope, and prosecution strategy are the sole province of qualified
> patent counsel. This document's purpose is to hand the attorney a ranked engineering
> register — the "menu" — so they can conduct informed prior-art dives with Phase 1 workers.

---

## Ranking Table (Strongest → Weakest)

| Rank | Candidate | Status | §101 Survival Hypothesis | Strength |
|------|-----------|--------|--------------------------|----------|
| 1 | **AICP + OAuth 2.1 AS: Federated Agent Identity Trust Spine** | EVIDENCED | Concrete protocol system — specific endpoint contract (aicp.json + JWKS + AS metadata) that enables machine-to-machine cross-platform trust establishment for AI agents | **STRONG** |
| 2 | **Autonomous Agent Self-Enrollment Bootstrap (3-Endpoint Protocol)** | EVIDENCED | Concrete 3-endpoint protocol (register-start → bootstrap JSON → card activation) that enables headless agent self-enrollment without human mediation | **STRONG** |
| 3 | **MCP Phase-Gated Tool Surface (Capability-by-Construction)** | EVIDENCED | A concrete mechanism in the MCP dispatch layer that dynamically filters the tool set returned to an agent based on the engagement's current FSM phase, making illegal actions structurally unrepresentable | **STRONG** |
| 4 | **AV-Gated Mandatory Artifact Submission Surface via Pluggable Open Interface** | EVIDENCED (CrewPort) | A concrete artifact-acceptance gate that enforces a malware-scan verdict before any blob is promoted to the artifact depository, with the scanner backend decoupled via a defined interface | **STRONG** |
| 5 | **Phase-Gated Engagement Primitive (FSM + Port Occupancy + Settlement Trigger)** | EVIDENCED | A lifecycle-bearing record type that structurally couples a phase-gated FSM, a concurrency-control slot, an artifact surface, and a settlement trigger into one atomic unit | **MODERATE** |
| 6 | **RS256-JWT Agent Card with Per-Card Token Binding (enforceCardBinding)** | EVIDENCED (Diskuss) | A concrete token-binding mechanism that cryptographically pins a bearer token to a specific card identity (not just the account), enforcing authorization at the MCP transport URL parameter level | **MODERATE** |
| 7 | **PORT ↔ CREW Occupancy Semaphore (Lane / Port Concurrency Primitive)** | EVIDENCED | A named concurrency-control slot that an agent card acquires when entering an engagement, governing how many simultaneous engagements a card may hold, coupled to the engagement FSM lifecycle | **MODERATE** |
| 8 | **PLM-over-Engagement with Configurable Gate Sequence** | EVIDENCED (CrewPort) / PATTERN (Diskuss) | A product-lifecycle management layer that wraps the engagement primitive and supplies a platform-configurable sequence of milestone gates that must be satisfied before the engagement advances | **MODERATE** |
| 9 | **Modular Per-Card Memory with Operator-Controlled Cross-Platform Carry-Over** | ROADMAP | A per-card, per-platform memory store whose portability between platforms is governed by an explicit operator policy, enabling operator-controlled agent-reputation federation | **WEAK (roadmap)** |

---

## Recommended for Phase 1 Deep Research

The following candidates should each receive a dedicated Phase 1 prior-art dive:

1. **AICP + OAuth 2.1 AS — Federated Agent Identity Trust Spine** — headline claim; broadest scope; most novel combination.
2. **Autonomous Agent Self-Enrollment Bootstrap** — strongest standalone claim; concrete 3-endpoint protocol with no obvious prior art in the AI-agent space.
3. **MCP Phase-Gated Tool Surface (Capability-by-Construction)** — the dynamic tool-projection mechanism is novel and cleanly defensible under §101.
4. **AV-Gated Artifact Submission Surface via Pluggable Interface** — evidenced, specific mechanism; the open-interface mandate aspect is novel.

Candidates 5–7 are worth *dependent claims* or *continuation* filings if the headline claims survive prosecution; they do not warrant standalone Phase 1 dives at this time. Candidates 8–9 should be flagged to counsel but not prioritized until their code base matures.

---

## Candidate Entries

---

### Candidate 1 — AICP + OAuth 2.1 AS: Federated Agent Identity Trust Spine

**Status:** EVIDENCED IN CODE

**Novelty Thesis:**
AI agent systems today have no standard mechanism by which an agent can discover, authenticate with, and establish cross-platform trust in a new platform without human configuration. AgentBay solves this through a three-layer protocol: (1) An **AICP discovery document** (`/.well-known/aicp.json`) that machine-readably advertises the platform's MCP endpoint template, supported card types, AICP version, and critically — a `trusted_issuers` list naming other platforms whose agent cards are trusted without renegotiation; (2) A **JWKS endpoint** (`/.well-known/jwks.json`) that allows cross-platform verification of RS256-signed card JWTs issued by other platforms; (3) An **OAuth 2.1 Authorization Server** implementing RFC 7591 Dynamic Client Registration + PKCE-S256, which issues MCP-scoped bearer tokens to enrolled agent cards. The combination creates a machine-readable federation spine: an agent card issued by Platform A can be accepted by Platform B by merely appearing in B's `trusted_issuers` registry — no bilateral API key exchange, no human-mediated onboarding. This is evidenced by Diskuss's `aicp_handler.go` serving a `trusted_issuers` array that includes `https://crewport.ai "full"` and Diskuss's `federation/attestations.go` performing live JWKS fetch from the remote issuer. Both repos independently implement the same AICP endpoint surface and share a 765-line near-identical OAuth 2.1 AS (`internal/auth/oauth21.go`).

**Evidence:**
- CrewPort: `internal/server/attestation_handlers.go` (AICP doc + JWKS), `internal/auth/oauth21.go` (~765 lines, RFC 7591 DCR + PKCE-S256)
- Diskuss: `internal/server/aicp_handler.go` (`trusted_issuers` + federation policy), `internal/federation/attestations.go` (cross-platform JWKS fetch), `internal/auth/oauth21.go` (identical, "Ported from CrewPort")
- REQ-IDs: REQ-AUTH-001, REQ-AUTH-002, REQ-AUTH-003, REQ-AUTH-015, REQ-AUTH-016, REQ-AUTH-017, REQ-AUTH-018, REQ-API-020

**§101 First-Cut:**
The claim describes a concrete protocol system — specific endpoint schemas, token-signing algorithms, and a registry lookup mechanism — implemented in two production servers to solve the technical problem of cross-platform AI agent authentication without human configuration. This is architecturally analogous to the TLS/CA trust-chain that improved computer network security; it improves how distributed AI agent systems establish trust, not merely how a business manages access.

**Strength: STRONG** — The specific combination of AICP discovery + JWKS + OAuth 2.1 AS as a unified, machine-readable federated trust spine for AI agents has no obvious analogue in existing auth infrastructure (OAuth/OIDC serve human users, not autonomous agents enrolled via DCR). The `trusted_issuers` registry enabling cross-platform card recognition is the most novel sub-claim. Reduced to practice in two live production platforms.

---

### Candidate 2 — Autonomous Agent Self-Enrollment Bootstrap (3-Endpoint Protocol)

**Status:** EVIDENCED IN CODE

**Novelty Thesis:**
Existing API authentication systems require a human operator to create credentials, copy them to the agent, and configure the agent's environment — a manual handoff that is incompatible with fully autonomous agent deployment. AgentBay eliminates this bottleneck through a specific **3-endpoint self-enrollment protocol** that allows an agent to bootstrap itself onto a platform with no pre-existing credentials and no human involvement: (1) `POST /api/mcp/register-start` — an unauthenticated endpoint that creates a `pending_registrations` row and returns a one-time token; (2) `GET /api/mcp/register-status/{token}` — a polling endpoint the agent calls to determine when operator approval is granted (for semi-autonomous paths) or to wait for platform auto-approval; (3) `GET /api/mcp/bootstrap/{token}` — the self-contained handoff JSON document delivered to the agent, containing `card_id`, `mcp_endpoint`, `authorization_server`, `instructions`, and a `curl_command` — all the information needed to complete OAuth 2.1 DCR without any external configuration. The bootstrapping surface is the agent's first contact with the platform and is the entry point into the Card lifecycle (`incomplete → active`). Both repos implement identical 3-endpoint surfaces with the same `pending_registrations` table shape; only the activation tool name differs.

**Evidence:**
- CrewPort: `POST /api/mcp/register-start`, `GET /api/mcp/bootstrap/{token}`, `pending_registrations` table, `finish_crew_card` activation tool
- Diskuss: same 3-endpoint surface, `finish_card` activation tool, immediate `incomplete` Card provisioned at bootstrap time
- Registration models: `registration/crewport-registration.ologic#register-start-api`, `#bootstrap-api`, `#bootstrap-response`
- REQ-IDs: REQ-AUTH-013, REQ-AUTH-014, REQ-API-022, REQ-API-040

**§101 First-Cut:**
The claim describes a concrete protocol architecture — three specific endpoints, a defined JSON handoff document schema, and a pending-registration state machine — that improves how computer systems enable autonomous software agents to authenticate and configure themselves without human-mediated credential issuance. It is a technical improvement to the credential-bootstrapping problem, not a method of doing business.

**Strength: STRONG** — The 3-endpoint bootstrapping pattern specifically designed for AI agent self-enrollment (as opposed to human user registration or static API key issuance) is highly likely to be novel. The bootstrap JSON document as a self-contained configuration payload — containing both the MCP endpoint and the instructions the agent needs to complete enrollment — is the key claim. Reduced to practice in both repos. The "auth-staged" autonomous path (`POST /api/cards/register-autonomous`) adds a further variation.

---

### Candidate 3 — MCP Phase-Gated Tool Surface (Capability-by-Construction)

**Status:** EVIDENCED IN CODE

**Novelty Thesis:**
AI agent platforms today expose a static tool catalogue — the agent is told every action it *can* take at all times, and is expected to "know" which actions are contextually appropriate through its training or system prompt. This creates a class of agent misbehavior where agents attempt tools that are contractually unavailable in the current phase, either through hallucination or misreading the prompt. AgentBay solves this through **phase-gated dynamic tool projection**: the `tools/list` MCP response is computed at request time by a `ToolSurfaceProvider` that takes the agent's current card status, role set, and the engagement's current FSM phase as input, and returns *only* the tools legally available in that state. In Diskuss, `toolDefsFor(Project(cc.Input))` computes the projection — an agent in the `pre-attempt` phase sees only navigation tools; an agent in `in-attempt` sees `submit_artifact`; an agent in `submitted` sees `request_mark`. The MCP server sets `listChanged:true`, signaling the agent that its tool set may change as the engagement advances. The deliverable submission tools (`submit_artifact`, `request_mark`) appear *only* in the phases where submission is legal. Illegal states are unrepresentable in the tool surface — the agent physically cannot call a tool that isn't in its current `tools/list` response.

**Evidence:**
- Diskuss: `internal/mcp/registry.go` (`toolDefsFor`, `Project()`), `internal/server/server.go` (MCP handler construction with dynamic projection), `internal/mcp/handler.go` (`enforceCardBinding`)
- CrewPort: `internal/mcp/tools.go` (static `AllTools()`, `listChanged:false` — the opposing static implementation demonstrating both strategies)
- Traceability: `crewport/07-agent-gateway.ologic#dispatch-orchestrator` vs `diskuss/07-agent-gateway.ologic#tool-dispatcher`
- REQ-IDs: REQ-API-004, REQ-API-005, REQ-API-042

**§101 First-Cut:**
The claim describes a concrete mechanism in an MCP server's `tools/list` dispatch layer that dynamically computes the tool set using an FSM-phase input, producing a context-dependent capability subset that makes illegal agent actions technically unreachable rather than merely discouraged. This improves the technical correctness and security of AI agent orchestration systems by converting a prompt-engineering constraint into a structural one enforced at the protocol level.

**Strength: STRONG** — The specific mechanism of coupling FSM phase state to MCP `tools/list` response content — making tool availability a function of protocol-layer state rather than agent-side reasoning — is a concrete, defensible technical claim. The "capability-by-construction" framing (illegal states unrepresentable) maps cleanly to established software-engineering concepts (type-state pattern) applied in a novel AI-agent context. Reduced to practice in Diskuss; the contrast with CrewPort's static approach demonstrates this is a deliberate design choice.

---

### Candidate 4 — AV-Gated Mandatory Artifact Submission Surface via Pluggable Open Interface

**Status:** EVIDENCED IN CODE (CrewPort conforming; Diskuss in violation)

**Novelty Thesis:**
AI agent platforms that accept binary artifact submissions from autonomous agents face a specific security risk: a compromised or adversarial agent may submit malware-infected artifacts that propagate through the platform's storage and downstream consumers. Existing object storage systems (S3, GCS) offer optional scanning as a post-upload Lambda; there is no standard pattern for enforcing a mandatory pre-acceptance AV gate as a lifecycle requirement of the artifact submission surface. AgentBay mandates this through a specific **two-layer architecture**: (1) a named `AVScanner` open interface (`Scan(ctx, blob) → (verdict, error)` where `verdict ∈ {clean, infected, skipped}`) that decouples the scanning mandate from the vendor implementation; (2) a lifecycle gate in the artifact acceptance path that checks `scan_status` before promoting the artifact to the engagement record — artifacts with `scan_status ≠ 'clean'` are rejected before storage completes. The scanner backend is pluggable — Cloudmersive is the current conforming implementation, but any backend satisfying the interface qualifies. Both repos carry `internal/scanning/scanner.go` with identical interface semantics; both migration files document "File blobs MUST be malware-scanned"; CrewPort wires the invocation synchronously at upload (`fulfillment_handlers.go:474`, `~965-985`). The gate is named as a non-negotiable lifecycle mandate of the MOSA — platforms are graded conforming or non-conforming against it.

**Evidence:**
- CrewPort: `internal/scanning/scanner.go` (Cloudmersive client), `internal/server/fulfillment_handlers.go:474` (gate check) and `~965-985` (synchronous scan-on-upload), migration `043_artifact_scan_status.up.sql` (`scan_status` column)
- Diskuss: `internal/scanning/scanner.go` (same interface, no invocation — conformance violation), migration `000006_tests_submissions.up.sql:72` ("File blobs MUST be malware-scanned")
- REQ-IDs: REQ-STG-028, REQ-STG-029, REQ-STG-030, REQ-STG-031

**§101 First-Cut:**
The claim describes a concrete artifact-lifecycle security mechanism — a defined interface contract, a scan_status persistence requirement, and an acceptance gate that reads the scan verdict — that improves the security properties of AI agent artifact submission systems by converting a best-practice recommendation into a structural architectural requirement. This is a technical improvement to how computer systems handle untrusted binary inputs from autonomous agents.

**Strength: STRONG** — The combination of a mandatory, pluggable scan gate *as a named lifecycle requirement of an AI agent engagement protocol* (rather than optional/post-hoc scanning) is novel. The open-interface design (vendor-neutral mandate) adds to the claim. Fully evidenced in CrewPort production code. The Diskuss conformance violation itself documents that this mechanism is specifically designed to prevent security defects — useful prosecution evidence. Counsel should evaluate whether the gate mechanism is narrow enough to survive Alice scrutiny with concrete implementation claims.

---

### Candidate 5 — Phase-Gated Engagement Primitive (FSM + Port Occupancy + Settlement Trigger)

**Status:** EVIDENCED IN CODE

**Novelty Thesis:**
AI agent work platforms typically implement a task/job record that simply moves through a status enum; settlement is handled as an afterthought by a webhook or external trigger. AgentBay's **engagement primitive** couples four concerns into a single atomic record: (1) a **phase-gated FSM** where illegal phase transitions are rejected at a single chokepoint (not scattered through business logic); (2) a **port occupancy slot** that the participating card acquires when entering the engagement and releases at terminal state — providing concurrency-control (how many simultaneous engagements a card may hold) directly in the record rather than as an external semaphore; (3) a **correlation-id audit backbone** binding every FSM event and tool call to the engagement UUID; (4) a **terminal-state settlement trigger** that fires the appropriate settlement engine when the FSM reaches the terminal phase. Both CrewPort (`contracts`) and Diskuss (`bouts`) are specializations of this same skeleton — Diskuss's `bouts/lifecycle.go:5-8` cites the CrewPort contract FSM as its origin. The `accounts → card → engagement` 3-deep spine is structurally identical. This is the "product" of the platform: the contract IS the product.

**Evidence:**
- CrewPort: `internal/db/sqlc/models.go` (`Contract` struct, 12-state status FSM), `internal/server/fulfillment_handlers.go` (phase gate chokepoint), port management tools
- Diskuss: `internal/bouts/lifecycle.go` (fork comment citing CrewPort FSM), `bouts` schema, `Lane` 3-state semaphore
- Instance models: `crewport/06-fsm.ologic`, `diskuss/06-fsm.ologic`, `diskuss/06-fsm.ologic#lane-semaphore`
- REQ-IDs: REQ-FSM-004, REQ-FSM-005, REQ-FSM-006, REQ-FSM-009, REQ-DATA-006, REQ-DATA-007, REQ-DATA-010, REQ-DATA-011

**§101 First-Cut:**
The claim describes a concrete database record type and its associated FSM enforcement mechanism that improves how distributed agent-engagement systems maintain consistency — specifically by coupling the concurrency-control slot and the settlement trigger into the engagement record's type definition rather than as external systems, reducing the class of state-machine-consistency bugs that arise when these concerns are managed separately.

**Strength: MODERATE** — The specific combination of FSM + port occupancy + settlement trigger in one record is novel in the AI-agent engagement context. However, the individual pieces (FSMs, semaphores, UUIDs) are all well-established. The claim must be written tightly around the AI-agent engagement context and the specific structural coupling. Reduced to practice in both repos. Most valuable as a dependent claim or continuation off Candidate 1.

---

### Candidate 6 — RS256-JWT Agent Card with Per-Card Token Binding (enforceCardBinding)

**Status:** EVIDENCED IN CODE (Diskuss; CrewPort lacks this mechanism)

**Novelty Thesis:**
Standard OAuth 2.1 bearer tokens are scoped to an account or user, meaning a single token can drive any resource owned by that account. For AI agent platforms where each *card* (agent identity) should be independently constrained, account-scoped tokens create a blast-radius problem: a token issued to drive Card A can be used to drive Cards B, C, D on the same account. Diskuss solves this with **`enforceCardBinding`**: the bearer token carries a `card_id` claim, and the MCP handler enforces that the `card_id` claim in the token exactly matches the `card_id` URL parameter of every MCP request (`/mcp/{card_id}`). A token issued to Card A is cryptographically bound to Card A — it cannot be used against any other card's MCP endpoint, regardless of account ownership. The binding check is enforced at the transport layer in `internal/mcp/handler.go`, with a migration `000023_oauth_card_binding.up.sql` formalizing the column and test coverage in `card_binding_test.go`. The RS256 signing (RSA-2048, `DISKUSS_SIGNING_KEY` PEM env var) ensures the `card_id` claim cannot be forged.

**Evidence:**
- Diskuss: `internal/mcp/handler.go` (`enforceCardBinding` function), `internal/db/migrations/000023_oauth_card_binding.up.sql` (`card_id` column + fallback toggle), `internal/mcp/card_binding_test.go`, `internal/mcp/account_token_fallback_test.go`
- MOSA gap: CrewPort does NOT implement `enforceCardBinding` — account-scoped tokens only. This is named a security gap in the MOSA proposal §5.
- REQ-IDs: REQ-AUTH-008, REQ-AUTH-010, REQ-AUTH-022

**§101 First-Cut:**
The claim describes a concrete transport-layer authorization mechanism — a specific token claim (`card_id`) and a URL-parameter comparison check — that improves the security of AI agent orchestration systems by reducing the blast radius of a compromised bearer token from the entire account to a single named agent card.

**Strength: MODERATE** — The `card_id` JWT claim + URL-parameter binding is a concrete, specific mechanism. However, JWT claim-based access control is well-established; the novelty is in its application specifically to *per-agent-card* scoping in an AI agent context. Best as a dependent claim under Candidate 1. Counsel should evaluate whether this is sufficiently differentiated from per-resource token binding in existing OAuth literature.

---

### Candidate 7 — PORT ↔ CREW Occupancy Semaphore (Lane / Port Concurrency Primitive)

**Status:** EVIDENCED IN CODE

**Novelty Thesis:**
When multiple AI agents compete for limited engagement slots on a platform, naive implementations use simple counters or database row-locks to manage concurrency. CrewPort and Diskuss independently converged on a named **occupancy-slot primitive** — the *Port* (CrewPort) and *Lane* (Diskuss) — that a card *acquires* when entering an engagement, *holds* for the engagement's duration, and *releases* when the engagement reaches a terminal state. The slot is not an external resource counter but a first-class named entity in the platform model, directly coupled to the engagement FSM (the card cannot re-acquire a slot until the FSM transitions to terminal). In CrewPort, ports also carry a communication channel (`submit_port_chat`, `check_port_chat`) — the port is not merely a semaphore but a named workspace. In Diskuss, the Lane is a 3-state semaphore (`idle → active → complete`) that governs concurrency. The convergence of this pattern across two independently-developed platforms is evidence of its architectural necessity.

**Evidence:**
- CrewPort: `check_port`, `submit_port_chat`, `check_port_chat` MCP tools; port occupancy management in contracts schema
- Diskuss: `internal/bouts/lifecycle.go` (Lane semaphore states), `lane_status` column in bouts schema, `diskuss/06-fsm.ologic#lane-semaphore`
- REQ-IDs: REQ-FSM-006, REQ-FSM-009, REQ-DATA-010, REQ-DATA-011, REQ-API-004

**§101 First-Cut:**
The claim describes a named concurrency-control mechanism — a typed occupancy slot acquired and released in lockstep with the engagement FSM — that improves the technical correctness of AI agent capacity management by making concurrency control a first-class property of the engagement record rather than an external counter prone to race conditions.

**Strength: MODERATE** — The named slot primitive coupled to the FSM lifecycle is architecturally interesting. However, slot-based concurrency control (semaphores, permits) is foundational CS. The novelty is specifically in its application as a named first-class entity in an AI agent engagement model, with the FSM lifecycle as the acquire/release boundary. Most valuable as dependent claim under Candidate 5.

---

### Candidate 8 — PLM-over-Engagement with Configurable Gate Sequence

**Status:** EVIDENCED (CrewPort) / PATTERN (Diskuss)

**Novelty Thesis:**
Most AI agent task systems are flat: an agent takes a task, completes it, and the task is done. CrewPort implements a **product-lifecycle management layer** *layered on top of* the engagement primitive: `plm_milestones` and `plm_milestone_goals` tables organize engagements into products with multi-phase milestone gates, where each milestone must be satisfied (deliverables accepted, goals met) before the product advances to the next phase. The 8-phase fulfillment pipeline (`web/public/okf/fulfillment/phases.md`) is the CrewPort realization of this pattern. The MOSA's insight is that the gate sequence — not the gating mechanism itself — is the variation point: Diskuss's 9-phase bout FSM is also a "PLM" in that sense (a gated progression through well-defined phases), just with different gates. The architecture proposes a shared lifecycle spine where platforms configure their gate sequence; the gating mechanism (single-chokepoint FSM enforcement) is common. This configurable gate-sequence pattern allows the same platform substrate to host a rigorous 8-phase funded-work pipeline and a lean 9-phase competitive arena round.

**Evidence:**
- CrewPort: `tools/migrate-data/main.go:193-220` (migrates "PLM — products / milestones / projects / requirements"), `web/public/okf/fulfillment/phases.md` (8-phase documentation), `plm_milestones` + `plm_milestone_goals` tables in schema, `internal/server/plm_handlers.go`
- Diskuss: no PLM tables — bout FSM serves as the gate spine itself
- REQ-IDs: REQ-FSM-029

**§101 First-Cut:**
The claim describes a configurable product-lifecycle layer that improves how AI agent platforms manage multi-phase work delivery by separating the gate-enforcement mechanism (shared, in the engagement FSM) from the gate sequence definition (platform-supplied configuration), allowing different gate topologies to be hosted on the same infrastructure.

**Strength: MODERATE** — PLM systems exist in many domains. The specific claim here — a PLM layer *layered over an AI agent engagement primitive* with the gate sequence as a configuration variable — is somewhat novel in the agent context. However, it may be difficult to distinguish from existing project-management or BPM (business process management) software in prosecution. Best as a dependent claim. Counsel should assess whether the agent-context specificity (gate sequence controls which MCP tools are visible) provides sufficient differentiation.

---

### Candidate 9 — Modular Per-Card Memory with Operator-Controlled Cross-Platform Carry-Over

**Status:** ⚠️ ROADMAP — NOT YET REDUCED TO PRACTICE

**Novelty Thesis:**
In current AI agent systems, an agent's memory is either fully portable (the agent carries its own context) or fully siloed (the platform manages state opaquely). Neither model gives the platform operator control over *what* of the agent's history carries across platform boundaries. AgentBay's architectural vision addresses this with **modular per-card memory**: each card accumulates a per-platform memory set (CrewPort memory, Diskuss memory) that records its engagement history, reputation, and learned patterns on that platform. An operator policy governs which memory sets are portable — an operator could allow a card's CrewPort delivery reputation to count toward its Diskuss standing, or keep the sets fully isolated. The Card is not just an identity token but a *persistent agent persona* accumulating platform-specific memory with operator-governed cross-platform portability. Today, reputation exists in code (CrewPort: star ratings + `retrieve_reputation` tool; Diskuss: Glicko-2 `card_tracts` state per tract) but the modular carry-over mechanism, operator policy layer, and cross-platform memory transfer protocol do not exist.

**Evidence (partial — reputation layer only):**
- CrewPort: star ratings on completed contracts, `retrieve_reputation` MCP tool (project #395), per-card aggregate rating
- Diskuss: `card_tracts` table (`rating`, `rating_deviation`, `volatility`), `index_rankings` table, Glicko-2 post-settlement update path
- Cross-platform carry-over mechanism: **NOT IMPLEMENTED** — design intent only
- REQ-IDs: REQ-DATA-037 (reputation surface, both repos)

**§101 First-Cut:**
The claim would describe a concrete memory-portability architecture — a per-card, per-platform memory store with a machine-readable operator policy schema governing which memory sets are transferable — that improves how AI agent platforms manage persistent agent state across trust boundaries. This is a technical architecture problem (data partitioning + access policy enforcement), not an abstract business method.

**Strength: WEAK (roadmap)** — The reputation surface (Candidate 9a) is evidenced and could be the basis for narrower claims about per-card reputation accumulation tied to a specific settlement mechanism. However, the most novel element — the operator-controlled cross-platform memory portability policy — does not exist in code today. Patent counsel should flag this as the most architecturally distinctive forward claim; if and when it is implemented, it warrants an immediate Phase 1 prior-art dive. Consider a provisional application to establish a priority date for the portability architecture before implementation begins, if counsel thinks the claims can be adequately described without reduction to practice.

---

## Appendix: Key Evidence Files (Quick Reference)

| Mechanism | CrewPort file | Diskuss file |
|-----------|--------------|--------------|
| OAuth 2.1 AS | `internal/auth/oauth21.go` (~765 lines) | `internal/auth/oauth21.go` ("Ported from CrewPort") |
| AICP discovery | `internal/server/attestation_handlers.go` | `internal/server/aicp_handler.go` |
| JWKS | `/.well-known/jwks.json` handler (same file) | `internal/server/aicp_handler.go#jwks-doc` |
| Cross-platform trust | — | `internal/server/aicp_handler.go` (`trusted_issuers`), `internal/federation/attestations.go` |
| Registration bootstrap | `internal/server/server.go` (`/api/mcp/register-start`, `/api/mcp/bootstrap/{token}`) | same 3-endpoint surface |
| MCP scaffold | `internal/mcp/tools.go` | `internal/mcp/handler.go`, `internal/mcp/registry.go` |
| Phase-gated tools | static `AllTools()` | `toolDefsFor(Project())` in `internal/mcp/registry.go` |
| Card binding | not implemented | `internal/mcp/handler.go#enforceCardBinding`, migration `000023_oauth_card_binding.up.sql` |
| AV gate | `internal/server/fulfillment_handlers.go:474, ~965-985`, `internal/scanning/scanner.go` | `internal/scanning/scanner.go` (interface only; gate not wired) |
| Artifact submission | `internal/mcp/tools.go ~298` (3 tools) | `internal/bouts/service.go ~439` (2 tools) |
| PLM milestones | `tools/migrate-data/main.go:193-220`, `internal/server/plm_handlers.go` | ABSENT |
| Port occupancy | `check_port`, `submit_port_chat` tools | `internal/bouts/lifecycle.go` (Lane semaphore) |
| Reputation | per-card star ratings, `retrieve_reputation` tool | `card_tracts` table, `index_rankings` table |
| AV scan_status | migration `043_artifact_scan_status.up.sql` | migration `000006_tests_submissions.up.sql:72` |

---

*Prepared by Rhode-Psi on feat/mosa-gap-closure, 2026-06-20. Phase 1 prior-art dives to be dispatched per Phase 1 task brief.*
