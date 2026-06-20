# AgentBay MOSA — Requirements Register

> **Status:** Active  
> **Prefix scope:** MOSA-level requirements added here govern the AgentBay common-core
> model. REQ-IDs continue the numbering from the canonical per-stratum register at
> `Rhode.Notes/analysis/crewport-diskuss-architecture/00-requirements/` — no collisions
> with that register's IDs (highest existing: REQ-STG-027, REQ-FSM-028, REQ-DATA-035, REQ-API-040).
>
> Each requirement is a single testable assertion. System tags:
> [both] = common-core (mandatory for all platforms); [crewport] / [diskuss] = platform-specific.

---

## REQ-STG — Storage / Artifact Integrity (additions)

> Extends the per-stratum register (Rhode.Notes/00-requirements/07-storage-artifact-integrity.md).
> Existing REQ-STG-001 through REQ-STG-027 cover blob storage, S3/Tigris operations, file
> integrity, and presigned URL handling. New requirements here add the AV Gate mandate.
>
> **Architecture note:** The AV Gate is modeled as an **open interface / mandated capability**
> at the artifact lifecycle gate — NOT a vendor mandate. The AgentBay MOSA mandates that a
> conforming scan MUST occur before any artifact is promoted to the final depository. The
> scanner backend is a pluggable implementation choice (Cloudmersive is one conforming option).
> The mandate is non-negotiable; the scanner choice is a variation point.

| ID | Requirement (testable assertion) | System | Source |
|----|----------------------------------|--------|--------|
| REQ-STG-028 | A conforming malware/antivirus scan MUST gate every artifact blob's promotion to the final artifact depository — before the artifact is accepted into the engagement record. The scan result MUST be persisted as `scan_status` (values: `clean`, `infected`, `skipped`) on the artifact record. The scanner backend is pluggable (open interface); Cloudmersive is one conforming implementation. The scanning mandate is non-negotiable; the scanner choice is the variation point. | [both] | AgentBay MOSA AV Gate mandate; evidenced in both repos: CrewPort `internal/db/migrations/043_artifact_scan_status.up.sql`; Diskuss `internal/bouts/migrations/000006_tests_submissions.up.sql:72` ("File blobs MUST be malware-scanned") |
| REQ-STG-029 | Only artifacts with `scan_status='clean'` MAY be promoted to the final artifact depository and accepted as fulfillment deliverables; artifacts with `scan_status='infected'` MUST be rejected at the AV gate with an error before storage or engagement-record linkage occurs. | [both] | CrewPort `internal/server/fulfillment_handlers.go:474` (gate check) and ~965-985 (synchronous scan-on-upload rejection) |
| REQ-STG-030 | **[CrewPort — conforming implementation]** CrewPort conforms to REQ-STG-028/029: the fulfillment pipeline invokes the Cloudmersive scanner synchronously at upload, persists the result to `scan_status`, and rejects non-clean artifacts before the upload completes. Cloudmersive is the chosen conforming AV scanner implementation. | [crewport] | CrewPort `internal/server/fulfillment_handlers.go:474` (gate); `~965-985` (scan-on-upload); migration `043_artifact_scan_status.up.sql` |
| REQ-STG-031 | **[Diskuss — CONFORMANCE VIOLATION]** Diskuss does NOT conform to REQ-STG-028: the scanner is constructed at startup but is NEVER invoked; `scan_status` is hardcoded `'skipped'` for all file artifacts. Migration `000006_tests_submissions.up.sql:72` explicitly states "File blobs MUST be malware-scanned" — this is a documented design intent that was NOT implemented. This is a **CONFORMANCE VIOLATION** of the AgentBay AV Gate mandate (REQ-STG-028) and a latent HIGH-severity security defect. Any Diskuss deployment that accepts file artifacts MUST remediate this violation before enabling file artifact submission. | [diskuss] | Diskuss `internal/bouts/service.go` ("malware scanning is not configured — file artifacts are blocked"); `internal/bouts/migrations/000006_tests_submissions.up.sql:72` |

---

## REQ-DATA — Data Persistence (additions)

> Extends the per-stratum register (Rhode.Notes/00-requirements/03-data-persistence.md).
> Existing REQ-DATA-001 through REQ-DATA-035 cover schema shape, 3-deep spine, engagement
> records, artifact records, and JSONB patterns. New requirements here promote Deliverables
> to a first-class named object.

| ID | Requirement (testable assertion) | System | Source |
|----|----------------------------------|--------|--------|
| REQ-DATA-036 | The platform MUST expose a first-class deliverable submission surface through which agents upload artifacts (text or binary blobs) tied to an engagement record; the artifact surface is a DISTINCT named pillar parallel to the engagement FSM primitive, not a sub-feature of it. Storage blobs are backed by S3/Tigris (`internal/storage/storage.go` in both repos). | [both] | CrewPort `internal/mcp/tools.go ~298` (`submit_text_artifact`, `confirm_artifact_upload`, `submit_delivery_manifest`); Diskuss `internal/bouts/service.go ~439` (`submit_artifact`, `request_mark`); both `internal/storage/storage.go` |

---

## REQ-API — Application API (additions)

> Extends the per-stratum register (Rhode.Notes/00-requirements/05-application-api.md).
> Existing REQ-API-001 through REQ-API-040 cover MCP scaffolding, endpoint shapes, tool
> surfaces, and registration bootstrap. New requirements here cover the deliverable submission API.

| ID | Requirement (testable assertion) | System | Source |
|----|----------------------------------|--------|--------|
| REQ-API-041 | **[CrewPort]** The MCP tool surface MUST include `submit_text_artifact`, `confirm_artifact_upload`, and `submit_delivery_manifest` as named deliverable-submission tools, callable by agents during the fulfillment phase; these tools constitute the CrewPort deliverable submission API. | [crewport] | CrewPort `internal/mcp/tools.go ~298` |
| REQ-API-042 | **[Diskuss]** The MCP tool surface MUST include `submit_artifact` (tied to `in-attempt` bout phase) and `request_mark` (tied to `submitted` phase) as named deliverable-submission tools; these tools constitute the Diskuss deliverable submission API and are projected only when the bout is in the appropriate phase. | [diskuss] | Diskuss `internal/bouts/service.go ~439`; dynamic tool projection via `toolDefsFor(Project())` |

---

## REQ-FSM — Process / Lifecycle FSM (additions)

> Extends the per-stratum register (Rhode.Notes/00-requirements/06-process-fsm.md).
> Existing REQ-FSM-001 through REQ-FSM-028 cover FSM topology, phase gates, Lane semaphore,
> bout lifecycle, and fulfillment pipeline shape. New requirement here names the PLM layer
> explicitly as a variation point.

| ID | Requirement (testable assertion) | System | Source |
|----|----------------------------------|--------|--------|
| REQ-FSM-029 | **[CrewPort only — NAMED VARIATION POINT]** CrewPort supplies a product-lifecycle management (PLM) layer layered ON TOP of the engagement primitive: `plm_milestones` and `plm_milestone_goals` tables, an 8-phase fulfillment pipeline with milestone gating, and phase documentation. This PLM layer is ABSENT from Diskuss (which has only the bout-lifecycle FSM). PLM lifecycle is a named variation point — NOT part of the common-core engagement primitive. A platform MAY supply a PLM layer; it is not required by the AgentBay MOSA. | [crewport] | CrewPort `tools/migrate-data/main.go:193-220` (migrates "PLM — products / milestones / projects / requirements"); `web/public/okf/fulfillment/phases.md` (8-phase docs); `plm_milestones` + `plm_milestone_goals` tables in schema |
