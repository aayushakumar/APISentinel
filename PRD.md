# APISentinel — Product Requirements Document

> **Version:** 0.1.0  
> **Status:** Implementation-ready draft  
> **Date:** 2026-10-17  
> **Owner:** Solo developer  
> **Product type:** Local-first developer tool / API contract and authorization testing  
> **Target:** Functional, demonstrable MVP by October 31, 2026  
> **Primary development environment:** Windows + Cursor + REA MCP  
> **Financial constraint:** $0 additional out-of-pocket spend; budget up to $125 in already-available Cursor credits, subject to account billing controls

---

## 0. Executive summary

**APISentinel** is an open-source, local-first API testing and security investigation workbench. It combines the everyday usability of an API client with automatic contract checks, controlled cross-identity authorization comparison, and evidence-backed diagnostics. Its core functionality works without any commercial AI API, account, cloud deployment, or subscription.

The project has two distinct engineering goals:

1. **Evidence-backed reverse engineering:** use [REA](https://github.com/morluto/rea) to analyze a locally available build of the Apache-2.0-licensed [Kong Insomnia](https://github.com/Kong/insomnia) Electron application, focusing on request-editor behavior, state management, local persistence, and request execution. Record observations and compare them to upstream source when appropriate. Reimplement capabilities independently; do not paste analyzed code or claim to have recovered inaccessible backend logic.
2. **Differentiated product:** build an API client with deterministic contract validation and a focused **authorization differential testing** workflow. The user can compare access to known private resources across two authorized test identities, observe discrepancies, and export a reproducible, redacted evidence report.

**The demo that matters:** Import a local demo API; run a normal request; run contract tests; select two test identities and a seeded resource; detect a planted object-level authorization bug; inspect the exact request/response evidence; export a report; optionally have a local LLM explain the finding.

### Nonnegotiables

- All core workflows run **locally** and remain functional with Ollama turned off.
- No requirement for Anthropic, OpenAI, Codex, Claude, paid hosted models, AWS, paid database, hosted queue, or rented server.
- No additional purchases. Optional tools that require payment are excluded from the MVP.
- **Security testing only on owned or explicitly authorized environments**, with automation restricted to an allowlist and conservative request methods.
- First build a **local web application**; desktop packaging is a stretch goal, not a blocker.
- Don't confuse Cursor's development credits with application inference credits. Cursor is a coding tool; APISentinel must not assume Cursor offers a production LLM endpoint.

---

## 1. Problem, audience, and opportunity

### 1.1 Problem

Developers regularly use separate tools to: send HTTP requests; inspect responses; import API definitions; validate contracts; compare staging and production behavior; troubleshoot authentication; and capture evidence of authorization failures. The context switches are slow, and many existing tools prioritize manual request construction over diagnosis.

A bare HTTP client makes requests but rarely answers the higher-value question: **“What is wrong with this endpoint, how can I reproduce it, and what evidence supports that conclusion?”**

### 1.2 Target personas

| Persona | Job to be done | Primary workflow |
| --- | --- | --- |
| Backend engineer | Diagnose unexpected API behavior quickly | Send request → view response → validate contract |
| QA / SDET engineer | Detect regressions before release | Import OpenAPI → generate/execute safe tests → review failures |
| Application security engineer | Reproduce and triage authorization issues in scoped environments | Select two test identities → compare access → inspect evidence |
| AI / platform engineer | Test service interfaces used by agents | Run API contract checks → export machine-readable results |

### 1.3 Positioning

**APISentinel is not “Postman, but free.”** It is a narrow, evidence-first tool for contract drift and authorization-difference analysis. Its differentiator is trustworthy, reproducible diagnostics—not sheer protocol coverage or AI-generated commentary.

### 1.4 Product principles

1. **Evidence over claims.** Every finding must link to the precise requests, responses, assertions, and expected policy.
2. **Deterministic before generative.** Use structured rules and validators for detection; use local AI only for optional explanations.
3. **Safe by default.** Scans are opt-in, scoped, rate-limited, non-destructive, and cancellable.
4. **Portable by design.** Local storage and exports; no hosted dependency.
5. **Vertical slices, not scaffolding theater.** Every milestone must have runnable functionality, tests, and a UI demonstration.
6. **Truthful attribution.** Distinguish observed REA findings from inference; distinguish intended metrics from measured metrics.

---

## 2. Goals, non-goals, and definition of done

### 2.1 MVP goals (P0)

- [ ] Create and save HTTP requests with method, URL, headers, query parameters, body, and timeout.
- [ ] Execute requests via backend HTTP client; display status, elapsed time, headers, and body.
- [ ] Organize requests under locally stored projects and collections.
- [ ] Define environments and interpolate non-secret variables such as `{{base_url}}`.
- [ ] Import an OpenAPI 3.x JSON/YAML document from disk; list operations and execute selected safe contract checks.
- [ ] Generate structured contract results with pass/fail, actual vs expected, and reproducible evidence.
- [ ] Define two test identities and a **known expected-denial policy** for a resource.
- [ ] Run safe, rate-limited cross-identity comparisons against an explicitly allowlisted test host.
- [ ] Label findings as `confirmed` only if an explicit expected-access policy is violated; otherwise `requires_review`.
- [ ] Export redacted JSON and Markdown reports with reproducible request templates.
- [ ] Include an intentionally vulnerable **local demo API** and a fixed demo walkthrough.
- [ ] Publish a REA investigation note with evidence, limits, and clean-room implementation decisions.
- [ ] Launch with simple Windows commands and no paid account or cloud service.

### 2.2 Nice-to-have goals (P1, only after P0)

- [ ] HAR import with redaction and request reconstruction.
- [ ] cURL import/export; selectable environment and collection runner.
- [ ] Schema-drift comparison between two OpenAPI specs.
- [ ] Local Ollama explanations of deterministic results.
- [ ] Export JUnit XML for CI use.
- [ ] Better response diff (semantic JSON comparison).
- [ ] Packaging into an Electron or Tauri desktop application.

### 2.3 Explicitly out of scope (MVP)

- Full Postman/Insomnia feature parity, gRPC, WebSockets, GraphQL editing, cloud sync, teams, RBAC, billing.
- Autonomous exploitation, bulk internet scanning, discovery of arbitrary remote hosts, stealth/evasion, credential acquisition.
- Blind endpoint fuzzing or unrestricted state-changing operations.
- Production-grade multi-user hosting, distributed agents, Kubernetes, CI SaaS, LLM fine-tuning.
- Automated code patching of real customer repositories.
- Reverse engineering proprietary commercial code outside an authorized scope.

### 2.4 Demo-ready definition of done

A fresh Windows checkout can launch the app and local demo API; the UI successfully executes a request, imports an OpenAPI file, produces contract results, detects at least one seeded authorization-policy violation with linked evidence, exports a safe report, and passes documented automated tests. REA findings and a clean-room design note are present in the repository.

---

## 3. User journeys and UX specification

### Journey A — Send and understand a request

1. User creates a project and collection.
2. User enters `GET {{base_url}}/api/v1/items/1` (or chooses an imported operation).
3. User selects an environment and executes the request.
4. UI shows status, elapsed time, response headers, formatted JSON/raw text, and size.
5. User saves the request, replays it, and views a **redacted** recent execution record.

**Acceptance criteria:** GET and POST with JSON work against the local demo; timeout and network errors are visible and actionable; variables resolve; secrets are not leaked into logs or exports.

### Journey B — Validate an API contract

1. User imports `demo/openapi.yaml`.
2. UI lists parsed operations and supported test types.
3. User selects a safe operation/target and clicks **Run Contract Tests**.
4. Engine runs bounded checks: status-code conformance, response content type, schema conformance; optional non-mutating negative cases if explicitly selected.
5. UI shows each result with expected/actual, response excerpt, and reproducibility metadata.

**Acceptance criteria:** A seeded schema mismatch fails with a precise field-level explanation; a conforming response passes; unsupported or unsafe operations are skipped with a reason, not silently executed.

### Journey C — Investigate object-level authorization

1. User chooses a scoped test host and adds two authorized test identities: **Owner A** and **Other user B**.
2. User selects a preexisting resource belonging to Owner A, plus an explicit policy such as `owner_only`.
3. Engine performs a baseline request as A, then the same **read-only** request as B, with identical method, path, and payload other than identity material.
4. It compares results using the expected policy. For example, if B receives private invoice content when policy says B must be denied, it raises a finding.
5. User opens the finding to see redacted request pairs, response status/body evidence, expected-access policy, confidence/limitation, and suggested regression assertion.
6. User exports the finding to Markdown or JSON.

**Acceptance criteria:** One known vulnerable demo endpoint yields a policy-violation finding; one appropriately protected endpoint yields no finding; two `200` responses alone do **not** imply a vulnerability absent resource/policy context.

### Journey D — Optional local AI explanation (P1)

1. User turns on **Local AI**, specifies an `http://127.0.0.1:11434` Ollama endpoint, and selects an installed model.
2. User chooses an already-detected finding and requests an explanation.
3. The backend sends only a sanitized finding object to Ollama and receives explanatory text, likely causes, and a human-readable test recommendation.
4. UI labels content **AI-generated explanation; not proof of a vulnerability**.

**Acceptance criteria:** With Ollama stopped, all P0 functions continue working; the app has a clear no-model fallback and makes no hidden external model calls.

### 3.1 Navigation and screen map

```text
APISentinel
├── Workspace sidebar
│   ├── Projects
│   │   ├── Collections
│   │   └── Saved requests
│   ├── Environments
│   └── Imports
├── Request workspace
│   ├── Method + URL + Send
│   ├── Params | Headers | Auth | Body
│   └── Response: Body | Headers | Timing | History
├── Testing
│   ├── Contract tests
│   ├── Authorization comparisons
│   └── Runs / results
├── Findings
│   ├── Evidence details
│   └── Export report
└── Settings
    ├── Target allowlist and safety limits
    ├── Data / redaction
    └── Local AI (optional)
```

**Design direction:** restrained developer-tool aesthetic; dark-mode-first with excellent light-mode support; three-pane layout with high-density navigation, editor, and response/evidence viewer. Optimize for readability, not decorative charts. Use accessible colors and keyboard shortcuts. No copied trademarks or pixel-perfect UI cloning.

---

## 4. Functional requirements and testability

| ID | Priority | Requirement | Acceptance test |
| --- | --- | --- | --- |
| FR-01 | P0 | Create/edit/delete projects, collections, requests | Restart preserves saved data |
| FR-02 | P0 | Execute HTTP GET/POST (manual only for POST) with headers/query/JSON body | Demo endpoint returns expected response |
| FR-03 | P0 | Inspect formatted/raw responses, status, timing, headers | Parse JSON and handle text, non-JSON, errors |
| FR-04 | P0 | Named environments, variable interpolation | Switch base URL without editing saved request |
| FR-05 | P0 | Import OpenAPI 3.x YAML/JSON file | Valid spec lists operations; invalid spec explains error |
| FR-06 | P0 | Contract checks over selected eligible operations | Seeded spec mismatch detected; conforming operation passes |
| FR-07 | P0 | Define test identities using user-provided credentials | Secret values never appear in DB export/logs |
| FR-08 | P0 | Scope authorization tests to allowlisted host and read-only paths | Unauthorized host/method rejected before network call |
| FR-09 | P0 | Compare expected vs actual access for known resource | Seeded BOLA-style case produces evidence-backed finding |
| FR-10 | P0 | Export redacted report | No access token, cookie, or password in export |
| FR-11 | P0 | Provide local demo service, fixtures and automated tests | End-to-end demo is runnable from clean checkout |
| FR-12 | P0 | Record honest REA research notes and traceability | Each claim has target version, evidence, observation/uncertainty |
| FR-13 | P1 | Import redacted HAR and reconstruct requests | Fixture HAR yields correct request metadata |
| FR-14 | P1 | Use Ollama for explanations only | Local AI failure never prevents tests or exports |
| FR-15 | P1 | Compare semantic JSON across response pairs | Ordering differences do not create misleading diffs |
| FR-16 | P1 | Desktop bundle | Package runs offline from signed/unsigned local artifact as documented |

**Requirement precedence:** security boundaries > data integrity > P0 functional requirements > UX polish > P1 extensions.

---

## 5. Technical architecture

### 5.1 High-level architecture

```text
                    Local browser (127.0.0.1)
  ┌─────────────────────────────────────────────────────┐
  │ React + TypeScript + Vite                           │
  │ Request builder | contract tester | findings viewer │
  └─────────────────────────┬───────────────────────────┘
                            │ localhost JSON API
  ┌─────────────────────────▼───────────────────────────┐
  │ FastAPI (bind to 127.0.0.1 only)                    │
  │                                                     │
  │  Project/collection service                         │
  │  HTTP executor (httpx)                              │
  │  OpenAPI parser + schema validator                  │
  │  Contract test orchestrator                         │
  │  AuthZ differential policy evaluator                │
  │  Finding + report generator                         │
  │  Target scope guard + redactor                      │
  │  Optional Ollama client (loopback only)             │
  └──────────┬─────────────────┬────────────────────────┘
             │                 │
        ┌────▼────┐      ┌─────▼─────────────────────────┐
        │ SQLite  │      │ Explicitly scoped target APIs │
        │ local DB│      │ Demo API on 127.0.0.1         │
        └─────────┘      └───────────────────────────────┘
```

### 5.2 Recommended stack

| Layer | Choice | Rationale |
| --- | --- | --- |
| Frontend | React, TypeScript, Vite | Fast local development, strong tooling |
| UI | Tailwind CSS + shadcn/ui + Monaco or CodeMirror (select one) | Polished functional request editor |
| Backend | Python 3.12+, FastAPI, Pydantic | Clear API contracts, Python test ecosystem |
| HTTP | httpx | Timeouts, headers, async, test transports |
| Persistence | SQLite + SQLModel or SQLAlchemy (select one) | Zero hosted infrastructure |
| OpenAPI | YAML parser + `openapi-spec-validator`, `jsonschema` | Validation and deterministic contract checks |
| Property testing | Schemathesis, behind safe execution policy (P1 for full fuzzing) | OpenAPI-driven test generation |
| Tests | pytest + HTTPX ASGITransport; Vitest; Playwright local smoke tests | Back-end, frontend, end-to-end coverage |
| Local AI | Ollama HTTP interface (optional, P1) | No paid API key |
| Reverse engineering | REA MCP + static Electron analysis | Evidence-backed investigation |
| Packaging | Browser/localhost first; Electron/Tauri later | Avoid two-platform debugging in MVP |

**Stack discipline:** no Redis, Celery, Kafka, ClickHouse, Kubernetes, cloud database, or external queue for v0.1. For bounded local test jobs, an in-process worker / asyncio task with persisted run state is sufficient; clearly document that interrupted jobs do not survive process termination until a later version.

### 5.3 Repository layout (target)

```text
apisentinel/
├── PRD.md
├── README.md
├── LICENSE
├── .gitignore
├── .env.example
├── docs/
│   ├── architecture.md
│   ├── rea-investigation.md
│   ├── security-model.md
│   ├── evaluation.md
│   └── demo-script.md
├── frontend/
│   ├── package.json
│   ├── src/
│   │   ├── app/
│   │   ├── components/
│   │   ├── features/
│   │   │   ├── requests/
│   │   │   ├── collections/
│   │   │   ├── contracts/
│   │   │   ├── authz/
│   │   │   └── findings/
│   │   ├── lib/
│   │   └── types/
│   └── tests/
├── backend/
│   ├── pyproject.toml
│   ├── app/
│   │   ├── main.py
│   │   ├── api/
│   │   ├── core/       # configuration, scope policy, redaction
│   │   ├── models/     # persistence + DTOs
│   │   ├── services/   # HTTP, contract, authz, reporting
│   │   └── db/
│   └── tests/
├── demo/
│   ├── vulnerable_api/
│   ├── fixtures/
│   └── openapi.yaml
├── scripts/
│   ├── dev.ps1
│   ├── test.ps1
│   └── seed_demo.py
└── .github/workflows/ci.yml
```

**Clean-room rule:** `docs/rea-investigation.md` describes observed behavior, not copied upstream modules. Do not commit Insomnia source bundles, extracted proprietary resources, credentials, captures containing user data, or third-party branding. If source from Apache-2.0 projects is reused intentionally later, follow the applicable attribution and notice requirements; independent implementation is preferred.

### 5.4 Core entities / suggested schema

| Entity | Important fields | Notes |
| --- | --- | --- |
| `Project` | `id`, `name`, `created_at` | Workspace root |
| `Collection` | `id`, `project_id`, `name` | Nested grouping optional P1 |
| `SavedRequest` | `id`, `collection_id`, `method`, `url_template`, `headers_template`, `body_template` | Secret references, not secret values |
| `Environment` | `id`, `project_id`, `name`, `variables_json` | Non-secret variables only |
| `Identity` | `id`, `project_id`, `name`, `credential_ref`, `auth_kind` | Credential lives in OS keyring or memory |
| `TargetScope` | `id`, `project_id`, `base_url`, `approved_at`, `policy_json` | Explicit user authorization attestation |
| `TestRun` | `id`, `project_id`, `kind`, `status`, `started_at`, `finished_at` | `contract` or `authorization` |
| `TestCase` | `id`, `run_id`, `method`, `path`, `expected_policy`, `result` | No inline secrets |
| `Evidence` | `id`, `test_case_id`, `request_redacted`, `response_redacted`, `assertions_json`, `timestamp` | Truncated and sanitized |
| `Finding` | `id`, `run_id`, `title`, `severity`, `confidence`, `status`, `evidence_ids` | Status: confirmed/requires_review/dismissed |

**Credential storage decision:** prefer Python `keyring` and Windows Credential Manager on Windows. Store only opaque references in SQLite. Provide session-memory-only mode if keyring setup fails; do not silently downgrade to unencrypted storage.

### 5.5 Backend API endpoints (proposed contracts)

| Endpoint | Method | Purpose |
| --- | --- | --- |
| `/api/v1/health` | GET | Local health and version |
| `/api/v1/projects` | GET/POST | List/create project |
| `/api/v1/projects/{id}` | GET/PATCH/DELETE | Project operations |
| `/api/v1/collections` | GET/POST | Manage collections |
| `/api/v1/requests` | GET/POST | Manage saved requests |
| `/api/v1/requests/execute` | POST | Execute a manually approved request |
| `/api/v1/import/openapi` | POST | Validate and import local spec |
| `/api/v1/tests/contract-runs` | POST | Start bounded contract run |
| `/api/v1/tests/authz-runs` | POST | Start bounded authorization comparison |
| `/api/v1/runs/{id}` | GET | Run status, counts, test results |
| `/api/v1/findings` | GET | Filter findings |
| `/api/v1/findings/{id}` | GET | Detailed evidence |
| `/api/v1/reports/{run_id}` | GET | Download redacted Markdown / JSON |
| `/api/v1/settings/scope` | GET/PUT | Manage explicit target allowlist |

**Illustrative authorization-run input:**

```json
{
  "project_id": "p_demo",
  "target_scope_id": "scope_local_demo",
  "method": "GET",
  "path_template": "/api/v1/invoices/{invoice_id}",
  "path_params": {"invoice_id": "inv_a_001"},
  "owner_identity_id": "identity_a",
  "other_identity_id": "identity_b",
  "expected_access_policy": "owner_only",
  "requests_per_second": 1,
  "max_requests": 2
}
```

**Illustrative finding output:**

```json
{
  "id": "finding_001",
  "type": "object_level_authorization_policy_violation",
  "status": "confirmed",
  "severity": "high",
  "title": "Other test identity received private invoice data",
  "expected_policy": "owner_only",
  "observed": {
    "owner_status": 200,
    "other_status": 200,
    "other_received_private_fields": true
  },
  "evidence_ids": ["evidence_owner", "evidence_other"],
  "limitations": ["Finding applies only to the configured test identities and seeded resource"]
}
```

These are **design examples**, not live API results. Final API contracts should be specified through Pydantic schemas and versioned OpenAPI documentation after the first vertical slice.

---

## 6. Engine behavior and algorithms

### 6.1 Request execution engine

1. Resolve non-secret environment variables and credential references separately.
2. Parse URL, method, headers, body, and timeout. Reject malformed URLs.
3. Apply allowed-scope checks to automated runs; manual requests still require explicit user action and dangerous-destination protections.
4. Make request via HTTPX with redirects **off by default**; if enabled, validate each hop against the scope policy and prevent redirects to blocked networks.
5. Capture status, elapsed time, bounded response body, redacted headers, and useful error details.
6. Save the minimum required redacted execution metadata; make full body retention opt-in for local demo data only.

### 6.2 Contract validation engine

- Import local OpenAPI 3.x JSON/YAML; validate parse and structure.
- Resolve supported local `$ref` with bounded resolution; reject uncontrolled network fetches.
- For a known endpoint, compare response status to declared response codes and body content type/schema to the matching definition.
- Store assertion-level results (e.g., `body.total: expected integer; got string`).
- Initially test safe GET/HEAD endpoints; don't auto-run POST/PUT/PATCH/DELETE even if present in spec.
- Expose deterministic tests without relying on a generative model. Schemathesis may power broader property-based checks later; scope, mutation policy, and current CLI behavior must be verified before enabling them.

### 6.3 Authorization differential test engine

**Inputs:** allowlisted target URL, safe endpoint template, known owner resource, two explicitly provided test identities, expected-access policy, and max-request/rate limits.

**Steps:**

1. Validate user authorization attestation, target hostname/IP, method (`GET` or `HEAD`), expected policy, and request budget.
2. Execute the owner baseline; if it does not establish expected resource access, stop and mark `inconclusive`.
3. Execute the other-identity request against the **same resource**.
4. Apply policy-aware assertions. Example for `owner_only`: other identity should receive denial (`401`, `403`, or appropriately `404`) **and** must not receive protected resource content.
5. Flag `confirmed` for a clearly violated explicit policy with observable private data. Flag `requires_review` for ambiguous status/body differences or insufficient policy evidence.
6. Generate a sanitized request-pair diff, assertion list, exact limitations, and a regression-test sketch.

**Not sufficient evidence:** two `200` responses by themselves, any response from public resources, inference about ownership without an explicit fixture/policy, or an LLM opinion.

**Safety:** no automatic identity harvesting, ID enumeration, destructive mutation, brute-force login, broad endpoint discovery, or live internet scans.

### 6.4 Finding pipeline

```text
Test request → Deterministic assertion → Redacted evidence
             → Expected-policy assessment → Finding status
             → Human review / export → Optional local explanation
```

All explanatory text must reference a structured finding. A local LLM may be used to summarize evidence, but must not create a `confirmed` finding by itself.

---

## 7. Reverse-engineering workstream: REA + Insomnia

### 7.1 Research question

> How does the Insomnia Electron client organize user-defined requests and environments, persist local project state, and route user interaction through its UI and application modules?

Investigating this provides practical architecture insight and evidence of systems-analysis capability. **It does not imply replicating hidden cloud backends.**

### 7.2 Legal and technical boundaries

- Target: a locally available, permitted installation/build of [Kong Insomnia](https://github.com/Kong/insomnia), which publishes its repository under Apache-2.0; check the exact target build's bundled third-party notices separately.
- Use static inspection of owned local app artifacts and test accounts. Avoid credential interception or third-party traffic capture.
- Make observations in `docs/rea-investigation.md`, with target version/hash, analysis commands, paths, evidence excerpts, and uncertainty.
- Keep design and implementation clean-room where possible; never represent a hypothesis as a discovered implementation detail.
- Do not analyze proprietary Postman assets or bypass paywalls/licensing controls.

### 7.3 Tool setup on Windows

Prerequisite: supported Node.js/npm version. REA's current docs specify Node.js **22.19+ (22.x), 24.11+ (24.x), or 26+**.

```powershell
node --version
npm --version
npx rea-agents@latest setup --client cursor --dry-run
# Review proposed changes and approve only what is needed:
npx rea-agents@latest setup --client cursor
# Restart Cursor before using REA MCP.
npx rea-agents@latest doctor --client cursor --json
```

For independent static inspection of an extracted Electron directory or ASAR (replace the path):

```powershell
npx -y rea-agents@latest analyze-javascript-application "C:/path/to/insomnia/app.asar" --json
```

**Fallback:** if the selected REA tool is unavailable in Cursor, use the CLI inspection and proceed with documented observations. Do not block product implementation on optional native binary tooling. REA's JavaScript/Electron inspection does not require Hopper/Ghidra/IDA.

### 7.4 REA investigation deliverables

| Investigation | Evidence to capture | Relevance to APISentinel |
| --- | --- | --- |
| UI → request state | Module names, state changes, observed behavior | Editor/workspace state model |
| Environment interpolation | Source paths or observable variable behavior | Environment engine |
| Save/load | Local data model and persistence boundary | SQLite design |
| Request dispatch | UI boundary, IPC/network events if accessible | HTTP executor separation |
| Error/result handling | Error cases and renderer behavior | Response workspace UX |

**Required note format:**

```md
### REA-F-001: [Finding title]
- Target version / artifact SHA-256:
- Tool and exact command:
- Observation:
- Evidence (file/module/reference, short excerpt, reproducible path):
- Interpretation:
- Confidence: high / medium / low
- What this does NOT establish:
- APISentinel design implication:
```

**Completion gate:** at least three substantive evidence-backed findings, including one frontend-to-runtime boundary, and a section listing what was *not* observable. If the tool cannot recover evidence, document the limitation and do not fabricate findings.

---

## 8. Safety, privacy, and threat model

### 8.1 Primary risks

| Threat | Mitigation |
| --- | --- |
| Accidental scan of unrelated systems | Explicit user-confirmed hostname allowlist for automated tests; default to loopback demo |
| SSRF / cloud metadata access | Block metadata IPs, nonapproved destinations, rebinding and disallowed redirect hops; recheck resolved IP on connection where feasible |
| Destructive testing | Automated authorization workflow allows GET/HEAD only; no active mutation; hard maximum request count |
| Credential leakage | OS keyring or memory; redact Authorization, Cookie, Set-Cookie, API keys and configured sensitive fields in logs/exports |
| Imported malicious OpenAPI/HAR payload | File-size limit, schema parsing bounds, no remote `$ref`, no executable templates |
| Unbounded response or workload | Timeouts, max response bytes, concurrency/rate caps, cancellation, bounded retention |
| Loopback service accessed by unrelated browser pages | Bind `127.0.0.1`; restrictive CORS + Origin validation and CSRF protection for mutation endpoints; avoid wildcard origins |
| AI explanation leakage | Ollama off by default; only redacted evidence passed to loopback endpoint |
| False security conclusions | Explicit policy + evidence + status (`confirmed`, `requires_review`, `inconclusive`); never infer from status alone |

### 8.2 MVP safeguards (hard requirements)

- Bind backend to loopback; no `0.0.0.0` default and no externally accessible deployment guide for v0.1.
- Automated security targets must be exactly allowlisted and explicitly approved.
- Reject automated redirects outside scope; all requests require configurable timeout and bounded body size.
- Default automated execution to one request per second, max ten requests per run, concurrency one. Higher limits are out of scope for MVP.
- Surface authorization and scope notice before tests; keep a local audit record of approvals, not secrets.
- Never store real credentials in code, test fixtures, screenshots, or git history.
- Safe mode is on by default; disabled security restrictions must **not** be an option in MVP.
- Reports redact all tokens/cookies and sensitive fields by default.

### 8.3 Data retention

- SQLite lives in user app-data, not inside the Git repository.
- Store response metadata and short sanitized excerpts by default, not unrestricted bodies.
- Provide **Clear local data** and **Delete project** actions.
- Define retention of execution history (suggested: most recent 100 runs per project) and make it configurable later.

---

## 9. Benchmark and evaluation plan

The portfolio story must include honest measured results. **All figures below are goals, not achieved performance.**

### 9.1 Demo fixture design

Create a local test API with two users (`owner_a`, `other_b`) and resources such as invoices or notes. Use **fake** fixtures only. Include at least:

- A conforming endpoint.
- An endpoint returning a deliberate schema mismatch.
- An owner-only endpoint that correctly denies the non-owner.
- An owner-only endpoint deliberately missing object-level authorization checks.
- A public endpoint that returns `200` to both identities (negative control).
- An endpoint with an ambiguous response requiring human review.

**Seeded defects must be visibly labeled demo-only and unavailable in production mode.** This benchmark demonstrates detection in a controlled fixture, not broad real-world vulnerability-finding capability.

### 9.2 Metrics

| Metric | How measured | MVP target |
| --- | --- | --- |
| Basic end-to-end functionality | Fresh-install manual workflow | All demo steps complete |
| Contract test correctness | Seeded conforming vs schema-mismatch fixtures | Both classified correctly |
| Authorization fixture correctness | Known vulnerable and properly protected seeded endpoints | Detect known violation; no false positive on public/control endpoints |
| Report redaction | Secret-canary tokens in headers/cookies/fixture bodies | Zero canary occurrences in exported report |
| Offline mode | Disconnect internet; run demo + tests | Full P0 demo works |
| CI test outcome | Backend + frontend + smoke tests | Green on main branch |
| REA research evidence | Independently check finding references | ≥3 verifiable findings or documented tool limitation |
| Core inference cost | Run with no Ollama and no API keys | $0 external inference |

### 9.3 Benchmark methodology

- Version-pin dependencies and record machine specs, date, test fixture hash, and number of runs.
- Run unit tests, contract checks, and seeded authorization fixtures independently.
- Report both true positives and false positives, including ambiguous cases.
- Do not publish unmeasured claims such as “90% vulnerability detection” or “10x faster than Postman.”
- A polished screenshot is not a replacement for a replayable benchmark.

---

## 10. Implementation phases and issue backlog

**Timebox:** October 17–31, 2026 (two-week **target**, not a guarantee). Keep scope constrained and ship vertical slices in order. Do not spend the first week perfecting architecture without running the product.

### Phase 0 — Project setup and first REA evidence (Day 1)

- [ ] Initialize Git repository, `PRD.md`, license choice, `.gitignore`, CI skeleton.
- [ ] Verify local Node/Python versions and Cursor usage limits; disable unapproved overage billing.
- [ ] Install/verify REA with Cursor; test static analysis of an authorized Electron artifact.
- [ ] Create `docs/rea-investigation.md`; record first genuine finding or blocker.
- [ ] Scaffold React + FastAPI with `/api/v1/health`; browser displays health status.

**Gate:** one-command local app startup and a committed REA note.

### Phase 1 — Functional API client (Days 2–5)

- [ ] SQLite migrations, Project/Collection/SavedRequest endpoints.
- [ ] Request executor with timeout, redirect rules, error handling, response truncation and redaction.
- [ ] Environment substitution and secret references.
- [ ] Usable request builder/response pane and persist/reload behavior.
- [ ] Unit and integration tests; seed demo requests.

**Gate:** save, send, inspect, reopen and replay a request in the UI.

### Phase 2 — OpenAPI contract testing (Days 6–8)

- [ ] Import and validate file-based OpenAPI JSON/YAML.
- [ ] Parse endpoints and select safe methods.
- [ ] Implement deterministic status/content-type/schema checks.
- [ ] Persist a bounded test run with result/evidence records.
- [ ] Display pass/fail details and run history.

**Gate:** seeded schema mismatch detected with field-level evidence.

### Phase 3 — Authorization differential testing (Days 9–11)

- [ ] Target scope approval/allowlist and strict automated-run policy.
- [ ] Test identities using Windows credential manager/memory fallback.
- [ ] Owner baseline + other-identity comparison runner.
- [ ] Explicit policy-aware classification and sanitized evidence.
- [ ] Findings browser and regression-test template.

**Gate:** known demo authorization violation flagged; protected endpoint and public endpoint not mislabeled.

### Phase 4 — Product release + evidence (Days 12–14)

- [ ] Markdown/JSON report export + secret-canary redaction tests.
- [ ] Demo seed script, README quick start, screenshots, architecture and demo walkthrough.
- [ ] Finalize REA investigation notes with evidence and limitations.
- [ ] CI (lint + tests), manual Windows clean-checkout verification.
- [ ] Record 2–3 minute screencast: import → test → detect → inspect → export.
- [ ] Optional local Ollama explanatory text **only if all P0 gates pass**.

**Gate:** complete repeatable offline demo and public repo ready.

### Work item template

```md
### TASK-XXX — concise title
- Priority: P0 / P1
- Owner: solo
- Depends on: TASK-...
- User outcome:
- Implementation:
- Acceptance criteria:
- Security considerations:
- Tests to add:
- Demo proof:
- Status: todo / doing / done
```

**Order of work:** build the smallest functional system first. Push after every testable vertical slice. Stop adding features when the demo is reliable.

---

## 11. Cursor working agreement: make credits last

### Suggested use of existing $125 Cursor credit (allocation, not a cost estimate)

| Stage | Credit budget |
| --- | ---: |
| REA research and architecture | $20 |
| Backend request engine and persistence | $30 |
| Frontend functionality and UI | $25 |
| Contract and AuthZ testing | $25 |
| Tests and release documentation | $15 |
| Unplanned debugging reserve | $10 |
| **Total available credit allocated** | **$125** |

These are planning ceilings; actual consumption depends on Cursor's account-specific credit terms, model choice, context length, and billing. **If Cursor cannot enforce no overage beyond the existing balance, disable on-demand/overage billing and stop when included credit runs out.** No task is authorized to incur a new charge.

### Credit rules

1. Use Cursor's existing **included/promo allowance only**. Check the exact expiration date, model-specific charges, usage meter, and whether on-demand usage can produce a bill. Disable or cap overages at $0 where supported.
2. Use a cost-efficient available model by default; reserve more expensive models for architectural decisions or unusually hard debugging. Model availability and billing vary by account—verify the actual Cursor UI.
3. Break work into small requests with clear acceptance tests. Avoid requests like “build the whole app” or repeatedly refeeding massive context.
4. Reference this file by path (e.g., `@PRD.md`) and relevant code files; ask for one milestone at a time.
5. Have Cursor run tests and show output before moving on. Do not accept “tests should pass” without execution.
6. Reuse local utilities, docs and deterministic scaffolding; do not use agent-loop/retry cycles that burn budget without producing a patch.
7. Keep an explicit credit reserve for debugging and documentation. Stop optional work before depleting the balance.
8. Never enter paid LLM API keys into the app or rely on free trials that might auto-bill.

### Cursor system instruction / first prompt

Paste this into Cursor Agent after placing `PRD.md` at the repository root:

```text
You are the lead engineer building APISentinel, a local-first, zero-additional-cost
API testing and evidence-backed authorization diagnostics tool.

Read @PRD.md fully. Treat P0 scope, safety requirements and the zero-spend
constraint as binding. Do not add hosted services, paid API keys, unapproved
network scans, or unnecessary infrastructure. Prefer a working vertical slice
over broad scaffolding.

FIRST TASK ONLY:
1. Inspect the repository and local tooling. List anything missing.
2. Write docs/architecture.md with a minimal implementable design; keep
   decisions aligned with the PRD.
3. Scaffold frontend (React + TypeScript + Vite) and backend (FastAPI + SQLite)
   with a health endpoint, safe local dev scripts, and one passing test each.
4. Set up a README with exact Windows PowerShell startup commands.
5. Verify the app starts and report the actual test output.
6. Explain the next single vertical slice you recommend.

If REA MCP is connected, inspect its available tools and propose a scoped,
authorized investigation of the local Insomnia Electron artifact. Do not
claim REA observations without executing analysis. If REA is unavailable,
write the blocker and continue building the core product.

Do not implement future phases yet. Never silently change the scope or
install any paid dependencies. Ask before any operation with a cost.
```

### Subsequent prompt pattern

```text
Read @PRD.md and the existing code. Implement only Phase [N] / TASK-[ID].
Before editing, summarize the minimum files to change and tests to add.
Follow the project's security model. Execute relevant tests and summarize
actual results, remaining limitations, and the next concrete milestone.
Do not expand scope, add cloud services, or claim unverified success.
```

---

## 12. Developer environment, quality gates, and release

### Prerequisites

- Windows 11 with PowerShell and Git.
- Node.js release compatible with REA (22.19+/24.11+/26+); choose an appropriate current stable runtime.
- Python 3.12+.
- Cursor account with existing credit; REA configured as MCP or usable via CLI.
- Optional: Ollama and compatible local model (P1; do not require download for P0).

### Desired dev commands (implement these during Phase 0)

```powershell
# From repository root; exact behavior documented after scaffolding.
./scripts/dev.ps1      # start local backend, frontend and optional demo API
./scripts/test.ps1     # run backend, frontend and smoke tests
```

No command should automatically install a paid product or contact a production scanning target. Dependency downloads from public package registries are acceptable, subject to the user's existing network access.

### Engineering standards

- Python: typing, Pydantic validation, ruff or equivalent lint, pytest, clear domain boundaries.
- TypeScript: strict typing, ESLint, Vitest, accessible components, no `any` in domain DTOs.
- Dependency version pinning / lockfiles on both sides; document license/notice obligations.
- Never commit `.env`, SQLite runtime DB, HAR captures with secrets, binary app bundles, test credentials, or local inference artifacts.
- Unit-test pure validators and redact-first code. Integration-test backend against mock transports and local fixture API.
- Prefer simple, composable functions over speculative abstractions.
- CI validates lint and tests without API keys.
- Bug triage requires reproducible input, expected output, actual output, and a failing test.

### Suggested GitHub release artifacts

- `README.md`: what it does, demo GIF/screenshot, stack, Windows quick start, limitations.
- `docs/architecture.md`: diagram, components, data flow, tradeoffs.
- `docs/rea-investigation.md`: versioned evidence and independent design implications.
- `docs/security-model.md`: scope controls, redaction, threat model.
- `docs/evaluation.md`: exact fixture, commands, test outcomes, limitations.
- `docs/demo-script.md`: scripted click path and backup screenshots.
- Open-source license selected after checking dependencies/attribution.

---

## 13. Risks, mitigations, and decision log

| Risk | Likelihood | Mitigation |
| --- | --- | --- |
| REA integration doesn't work on this Windows machine | Medium | CLI fallback; static JS/Electron only; don't block P0 |
| Insomnia installed artifact differs from source | Medium | Pin version/hash and note discrepancies |
| Scope creep into a Postman clone | High | P0 gates and hard P1 parking lot |
| Credential leakage in recordings/reports | High impact | Fake demo data, keyring/memory, canary redaction tests |
| False-positive security findings | Medium | Explicit expected policy, negative controls, `requires_review` classification |
| Cursor credits expire mid-build | Medium | Testable milestones, prioritize core, keep local deterministic toolchain |
| Local model too slow or insufficient RAM | Medium | Ollama strictly optional |
| Python/Node Windows integration friction | Medium | `dev.ps1`, pinned versions, no Docker requirement |
| Unintended network requests during testing | High impact | Allowlist, scope confirmation, redirect restrictions, local targets by default |

### Decisions already made

| ID | Decision | Reason |
| --- | --- | --- |
| ADR-001 | Local web UI before desktop packaging | Fastest demonstrable core with lowest complexity |
| ADR-002 | FastAPI + React + SQLite | Free and productive for solo engineer |
| ADR-003 | Security detection deterministic, AI optional | No inference charges; higher reliability and verifiability |
| ADR-004 | REA target is permitted Insomnia build, not proprietary Postman | Reproducible research and better licensing posture |
| ADR-005 | Authorization tests need explicit expected-access policy | Avoid treating benign 200/200 differences as vulnerabilities |
| ADR-006 | Conservative GET/HEAD automated requests only | Limit unintended state changes |

### Open technical choices to resolve during Phase 0

- Select SQLModel vs SQLAlchemy ORM; pick one, not both.
- Select Monaco vs CodeMirror editor; pick one.
- Decide exact local run worker implementation and interruption behavior.
- Choose secure Windows keyring integration and document its limitations.
- Pin specific release versions for all dependencies after scaffold and test.

---

## 14. Portfolio narrative and roadmap beyond MVP

### Evidence-backed README framing

> APISentinel is a local-first API intelligence workbench combining request execution, OpenAPI contract validation and policy-aware authorization comparisons. I investigated an existing open-source Electron API client using REA, documented evidence and limitations, and independently implemented the core workflows. An intentionally vulnerable local API provides reproducible tests and redacted findings; all essential functionality runs without paid model APIs.

**Do not claim:** equivalent Postman/Insomnia parity, autonomous zero-day discovery, customer adoption, commercial cost savings, benchmark numbers, or enterprise readiness unless separately substantiated.

### Post-MVP roadmap (only after v0.1 release)

- v0.2: HAR and cURL import/export; richer schema-drift comparisons; CLI mode for CI.
- v0.3: Automated policy inference from user-supplied access-control matrix; semantic response diffs; JUnit/SARIF exports.
- v0.4: Optional offline agent explanations and local RAG over OpenAPI + application logs.
- v0.5: Desktop packaging; plugin SDK; safe, explicitly scoped environment comparisons.

The best extension for an AI engineering résumé is **an evidence-grounded agent that proposes the next diagnostic test and justifies it using structured prior observations**, while requiring explicit user approval before making any network request. Ship this only after the deterministic workflows are excellent.

---

## 15. Primary references

- REA repository: https://github.com/morluto/rea
- REA installation and client integration: https://github.com/morluto/rea/blob/main/docs/installation.md
- Kong Insomnia repository and license: https://github.com/Kong/insomnia
- Schemathesis current documentation: https://schemathesis.readthedocs.io/en/latest/
- FastAPI: https://fastapi.tiangolo.com/
- HTTPX: https://www.python-httpx.org/
- OpenAPI Specification: https://spec.openapis.org/oas/latest.html
- Ollama: https://docs.ollama.com/

**Source-verification note (2026-10-17):** REA documentation describes a Cursor MCP setup flow and a Windows-compatible static JavaScript/Electron CLI workflow; Insomnia's public repository identifies Apache-2.0 licensing; Schemathesis documents API schema test checks. These are tool capabilities, not claims that this project has already implemented or verified them on the developer's machine.

---

## Appendix A — Release checklist

- [ ] Local-first; no authentication/account/cloud requirement
- [ ] No paid inference dependencies
- [ ] API client saves and executes manual requests
- [ ] OpenAPI contract run detects seeded schema mismatch
- [ ] Scope/allowlist and rate limits work
- [ ] AuthZ workflow detects seeded policy violation and respects controls
- [ ] Reports redact token/cookie/secret canaries
- [ ] App starts and tests pass from a clean Windows checkout
- [ ] Documentation describes results honestly, with measured numbers only
- [ ] REA evidence is real, attributable, and reproducible or its limitation is disclosed
- [ ] README screenshot / short demo / architecture diagram published

## Appendix B — What to build first, in one sentence

**Make `GET /api/v1/health` appear in a working React UI, then build one end-to-end request execution flow against the local demo API before implementing any AI or security test automation.**
