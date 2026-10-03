<div align="center">

# Autonomous Outreach Intelligence & Campaign Orchestrator

**An n8n orchestration layer for controlled B2B outreach: evidence-aware AI drafting, centralized lifecycle state, human review, fail-closed sending, event processing, and bounded autonomous campaign execution.**

<br>

![n8n](https://img.shields.io/badge/n8n-Workflow_Orchestration-EA4B71?style=flat-square&logo=n8n&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-State_%26_Persistence-4169E1?style=flat-square&logo=postgresql&logoColor=white)
![JavaScript](https://img.shields.io/badge/JavaScript-Controller_Logic-F7DF1E?style=flat-square&logo=javascript&logoColor=black)
![HTTP](https://img.shields.io/badge/REST%2FHTTP-Provider_Boundaries-0F766E?style=flat-square)
![State Machine](https://img.shields.io/badge/Lifecycle-Central_State_Machine-2563EB?style=flat-square)
![Human in the Loop](https://img.shields.io/badge/Human--in--the--Loop-Review_Gate-7C3AED?style=flat-square)
![Audit](https://img.shields.io/badge/Audit-Event_Logging-475569?style=flat-square)

`B2B Outreach` &nbsp;·&nbsp; `Evidence-Aware Drafting` &nbsp;·&nbsp; `Fail-Closed Send Gate` &nbsp;·&nbsp; `Bounded Autonomy` &nbsp;·&nbsp; `Reconciliation`

</div>

<br>

This repository documents a single n8n workflow export that coordinates the full outreach lifecycle around a PostgreSQL state store and a set of configurable external providers. It is an orchestration and control system, not a mail-merge sender: every lifecycle change is routed through one authoritative transition controller, every outbound message passes a layered send gate, and every uncertain provider outcome is held for reconciliation instead of being resent.

> [!NOTE]
> This README is derived from static inspection of the exported workflow JSON. It describes configured behavior. It makes no claim of production deployment, runtime success, delivery performance, or business outcomes. The exported workflow is saved with `active: false`.

---

## 🧩 Project Snapshot

| Layer | Implementation |
|:------|:---------------|
| **Automation** | One n8n workflow with 40 nodes: 18 authenticated POST webhooks, 3 schedule triggers, 6 JavaScript controller nodes, 1 routing switch, 1 PostgreSQL gateway, 10 HTTP provider nodes, 1 shared response node |
| **State Model** | Central lifecycle graph of 54 legal edges across 16 lead states, enforced inside a single parameterized SQL statement per transition |
| **Data Layer** | PostgreSQL through one `DB Gateway` node; schema is expected to be provisioned externally |
| **AI Layer** | External AI provider for target understanding, draft generation, and claim verification; outputs are validated by deterministic code |
| **Research** | External research provider plus authenticated ingestion endpoints; evidence is stored as structured claims |
| **Verification** | External email verification; only an explicit `VALID` result continues |
| **Human Review** | Dedicated decision endpoint tied to the `REVIEW` lifecycle state and a review queue |
| **Sending** | Queue, per-lead lock, daily reservation, dual gate evaluation, idempotency key, external send provider |
| **Recovery** | Ambiguous-outcome hold, optional reconciliation provider, circuit breaker with health-gated recovery, bounded autonomous recovery sweeps |
| **Observability** | `audit_events` writes on lifecycle, send, recovery, and autonomous events; read-only status, feedback, and roadmap audit endpoints |

---

## 🖼️ Workflow Overview

<img width="835" height="527" alt="02 Autonomous Outreach Intelligence   Campaign Orchestrator" src="https://github.com/user-attachments/assets/5b37d05d-cd47-4790-9b63-9596e5d7a2ae" />


| Canvas Region | Nodes | Role |
|:--------------|:------|:-----|
| **Entry points** | 18 webhook nodes and 3 schedule triggers | Receive requests, events, and timed maintenance |
| **Controllers** | Lead & State, Research & Draft, Send Safety, Event & Reporting, Autonomous Runtime, Provider & DB Result | Deterministic decision logic written as JavaScript |
| **Route bus** | `Orchestrator Router` | Switch on a `route` value with 17 named outputs and a `FAIL_CLOSED` fallback |
| **Execution nodes** | `DB Gateway` and 10 HTTP nodes | Run SQL or call external providers |
| **Exit** | `Shared Webhook Response` | Returns JSON with a computed status code |

---

## 🎯 What the System Does

The workflow coordinates a lead through a governed lifecycle. Each capability below is implemented in the exported controllers; external services supply the data they depend on.

| Capability | What the workflow does |
|:-----------|:-----------------------|
| **Intake** | Accepts leads from authenticated endpoints, enforces campaign state, suppression, duplicate rules, and field validity |
| **Verification** | Records external email verification outcomes and routes to verified, hold, or rejected |
| **Research** | Accepts finalized research as structured evidence claims and assigns research quality |
| **Drafting** | Builds a constrained AI input from supported claims, validates the returned draft, and persists it |
| **Claim verification** | Requires an external verifier to classify every claim in the draft against supplied evidence |
| **QA and review** | Decides pass, review, regenerate, or stop; supports a human approval or rejection step |
| **Queue and send** | Queues approved leads and sends only after a layered gate, atomic reservation, and a second provider-time recheck |
| **Outcome handling** | Records confirmed sends, holds ambiguous outcomes, and supports external reconciliation |
| **Events** | Processes unsubscribe, bounce, and reply events with event-ID de-duplication and suppression |
| **Autonomy** | Runs bounded discovery runs from a target definition with durable run accounting and recovery sweeps |
| **Reporting** | Provides run status, feedback counts, a roadmap audit snapshot, and a scheduled analytics audit entry |

### Responsibility Split

| Concern | Owner | Examples in this workflow |
|:--------|:------|:--------------------------|
| **AI** | External AI provider through HTTP | Target understanding from free text, draft generation, claim classification |
| **Deterministic state logic** | JavaScript controllers | Lifecycle graph, QA decision rules, gate evaluation, autonomous counters and stop rules |
| **Provider API work** | HTTP Request nodes | Discovery, contact discovery, email discovery, verification, research, send, reconcile, alert |
| **Database logic** | `DB Gateway` with parameterized SQL | Row locks, compare-and-set updates, counters, queue rows, audit writes |
| **Safety controls** | Send Safety and Lead & State controllers | Suppression, duplicate checks, send locks, flags, dependency freshness |
| **Human actions** | Review endpoint and admin endpoint | Approve or reject a reviewed draft, reset the circuit breaker flag |

---

## 🏗️ Architecture Overview

All entry points feed one of six controllers. Controllers emit an item carrying a `route` value; the `Orchestrator Router` dispatches that item to a database call, a provider call, another controller, or the shared response. Provider and database results return through the `Provider & DB Result Controller`, which normalizes them and re-enters the router.

| Layer | Responsibility | Main Implementation |
|:------|:---------------|:--------------------|
| **Intake** | Lead acceptance, verification results, circuit-breaker reset | `Lead & State Controller` |
| **State** | Authoritative lifecycle transitions | `Lead & State Controller` transition builder, `DB Gateway` |
| **Research** | Research request, text sanitization, evidence finalization | `Research & Draft Controller` |
| **AI** | Draft generation, claim verification, target understanding | `AI Provider`, `AI Claim Verification Provider` |
| **QA** | Draft validation and decision | `Research & Draft Controller` |
| **Review** | Human decision and event handling | `Event & Reporting Controller` |
| **Send Safety** | Queue gate, final gate, locks, reservations, reconciliation logic | `Send Safety Controller` |
| **Provider** | Discovery, contact, email discovery, verification, research, send, reconcile, alert | 10 HTTP nodes |
| **Events** | Unsubscribe, bounce, reply | `Event & Reporting Controller` |
| **Autonomous Runtime** | Target intake, discovery loop, counters, recovery sweeps | `Autonomous Runtime Controller` |
| **Persistence** | SQL execution | `DB Gateway` (PostgreSQL node) |
| **Reporting** | Status, feedback, roadmap audit, analytics audit entry | `Event & Reporting Controller`, `Autonomous Runtime Controller` |

<details>
<summary><b>Route bus: router outputs</b></summary>

<br>

| Route value | Dispatches to | Purpose |
|:------------|:--------------|:--------|
| `DB` | `DB Gateway` | Execute controller-generated parameterized SQL |
| `AI` | `AI Provider` | Target understanding and draft generation |
| `AI_CLAIM` | `AI Claim Verification Provider` | Claim classification of a draft |
| `DISCOVERY` | `Discovery Provider` | Candidate discovery request |
| `CONTACT` | `Contact Discovery Provider` | Contact lookup for a candidate company |
| `EMAIL_DISCOVERY` | `Email Discovery Provider` | Email lookup |
| `VERIFICATION` | `Email Verification Provider` | Email verification request |
| `RESEARCH` | `Research Provider` | Multi-source company research request |
| `EMAIL_SEND` | `Email Provider` | Outbound send call |
| `RECONCILE` | `Reconcile Provider` | Provider-side outcome lookup |
| `ALERT` | `Alert Webhook` | Circuit-breaker alert message |
| `STATE` | `Lead & State Controller` | Lifecycle transition requests |
| `CONTENT` | `Research & Draft Controller` | Draft and QA stages |
| `SEND` | `Send Safety Controller` | Queue and send stages |
| `EVENT` | `Event & Reporting Controller` | Event and reporting stages |
| `AUTO` | `Autonomous Runtime Controller` | Autonomous run stages |
| `FINAL` | `Shared Webhook Response` | Return the response |
| `FAIL_CLOSED` (fallback) | `Event & Reporting Controller` | Unmatched route values leave the main path |

</details>

### How It Works

| Stage | System Responsibility | Primary Control |
|:-----:|:----------------------|:----------------|
| **01** | Target or lead intake | Campaign ACTIVE check, suppression, duplicate rules, advisory lock |
| **02** | Email verification | External verifier, explicit `VALID` required |
| **03** | Research | External research, structured evidence claims, quality tiers |
| **04** | AI drafting | Constrained input, JSON output validation, claim ID allow-list |
| **05** | Claim verification | External classification with per-claim coverage checks |
| **06** | QA | Placeholder, recipient, opt-out, and claim checks |
| **07** | Review | `REVIEW` state, review queue, human decision endpoint |
| **08** | Queue | Idempotency key, queue gate, autonomous send-slot cap |
| **09** | Final send gate | Layered fail-closed checks, send lock, atomic reservation |
| **10** | Provider send | Idempotency key sent with request, single attempt, no node retries |
| **11** | Outcome reconciliation | Confirmed send, ambiguous hold, optional provider reconciliation |
| **12** | Audit and reporting | `audit_events`, status and report endpoints |

---

## 🧬 Central Lifecycle State Machine

Lifecycle ownership is centralized. Controllers never write `current_status` directly; they build a transition request and the `Lead & State Controller` converts it into one SQL statement that reads the locked row, validates the transition, updates the row, runs optional embedded SQL fragments, and writes the audit event.

### Lifecycle States

| State | Meaning | Typical Next States |
|:------|:--------|:--------------------|
| `DISCOVERED` | Lead accepted at intake | `VERIFIED`, `HOLD`, `REJECTED`, `SUPPRESSED`, `FAILED` |
| `VERIFIED` | Email verification returned `VALID` | `RESEARCHED`, `HOLD`, `REJECTED`, `SUPPRESSED`, `FAILED` |
| `RESEARCHED` | Research finalized | `DRAFTED`, `HOLD`, `REJECTED`, `SUPPRESSED`, `FAILED` |
| `DRAFTED` | Draft and QA record persisted | `QA`, `DRAFTED`, `HOLD`, `REJECTED`, `SUPPRESSED`, `FAILED` |
| `QA` | QA decision evaluated | `APPROVED`, `REVIEW`, `DRAFTED`, `HOLD`, `REJECTED`, `SUPPRESSED`, `FAILED` |
| `REVIEW` | Awaiting a human decision | `APPROVED`, `DRAFTED`, `HOLD`, `REJECTED`, `SUPPRESSED` |
| `APPROVED` | Automatically or human approved | `QUEUED`, `HOLD`, `REJECTED`, `SUPPRESSED` |
| `QUEUED` | Send queue row exists | `SENT`, `HOLD`, `FAILED`, `SUPPRESSED` |
| `SENT` | Provider send confirmed | `BOUNCED`, `REPLIED`, `UNSUBSCRIBED`, `SUPPRESSED` |
| `REPLIED` | Reply event recorded | `UNSUBSCRIBED`, `SUPPRESSED` |
| `HOLD` | Stopped pending intervention | `DISCOVERED`, `REVIEW`, `REJECTED`, `SUPPRESSED` |
| `FAILED` | Legal failure target | `HOLD`, `REJECTED`, `SUPPRESSED` |
| `BOUNCED` | Hard bounce processed | No outgoing edges in the graph |
| `UNSUBSCRIBED` | Unsubscribe recorded for a sent or replied lead | No outgoing edges in the graph |
| `SUPPRESSED` | Suppression applied | No outgoing edges in the graph |
| `REJECTED` | Rejected by verification or human decision | No outgoing edges in the graph |

> [!NOTE]
> `FAILED` is a legal target in the graph. The controllers reviewed do not contain a call that requests it for a lead; queue-level `FAILED` is a separate send queue state.

### Transition Mechanics

| Mechanism | Behavior in the SQL builder |
|:----------|:----------------------------|
| **Locked current-state read** | Reads `lead_id`, `current_status`, and `campaign_id` with `FOR UPDATE` |
| **Legal transition check** | Joins the current status against a 54-row edge list; an unchanged status counts as legal |
| **Expected-state match** | Optional `expected_status`; mismatch yields `EXPECTED_STATE_MISMATCH` |
| **Idempotent replay** | A desired status equal to the current status is flagged idempotent; event replay modes (`hard_bounce`, `unsubscribe`, `suppression`, `positive_reply`) resolve the effective status against the locked row |
| **CAS-style update** | The update requires the row status to still equal the status that was read |
| **Preconditions** | Optional `pre_sql` fragment executed in the same statement; soft-fail mode is limited to event de-duplication |
| **Guards** | Optional `guard_sql` fragment; an empty result yields `TRANSITION_GUARD_FAILED` |
| **Postconditions** | Optional `post_sql` fragment; an empty result after an update yields `POSTCONDITION_NOT_APPLIED` |
| **Atomic abort** | When a required mutation did not produce its row, a deliberately failing cast aborts the whole statement so partial mutations are not committed |
| **Audit insertion** | Writes an `audit_events` row with previous state, new state, source, and metadata for every applied transition |
| **Error codes** | `UNKNOWN_LEAD_ID`, `ILLEGAL_TRANSITION`, `EXPECTED_STATE_MISMATCH`, `PRECONDITION_NOT_APPLIED`, `TRANSITION_GUARD_FAILED`, `CAS_FAILED`, `POSTCONDITION_NOT_APPLIED` |

**Why centralize.** Every path that changes lifecycle state (verification, research, QA, review, queueing, sending, events, autonomy) goes through the same legality, expected-state, and audit logic. That removes the possibility of one controller writing a status another controller considers impossible, and it makes each mutation attributable through a single audit shape. This is an engineering control built from SQL and compare-and-set semantics; it is not a formally verified model.

---

## 📡 Endpoint Map

All webhook nodes use `POST`, header authentication, and response-node mode. These are configured workflow endpoints, not claims of a publicly deployed service.

| Endpoint | Purpose | Authentication | Notes |
|:---------|:--------|:--------------:|:------|
| `ingest-lead` | Lead intake | Header auth | Same intake pipeline as `discovery-candidate` |
| `discovery-candidate` | Candidate intake from an external discovery service | Header auth | Flat lead fields in the request body |
| `verification-result` | Record an email verification outcome | Header auth | Maps status to `VERIFIED`, `HOLD`, or `REJECTED` |
| `research-request` | Mark a lead as awaiting external research | Header auth | Idempotent while the request is pending |
| `research-ingest-text` | Sanitize external research text | Header auth | Returns cleaned text and a suspicion flag |
| `research-finalize` | Persist evidence claims and finalize research | Header auth | Requires lead in `VERIFIED` |
| `generate-and-qa-draft` | Draft generation, claim verification, QA | Header auth | Reads the lead's `SUPPORTED` claims; unknown lead returns `UNKNOWN_LEAD` |
| `human-review-decision` | Approve or reject a reviewed draft | Header auth | Requires lead in `REVIEW` |
| `enqueue-lead` | Queue an approved lead | Header auth | Queue gate runs in `QUEUE` mode |
| `send-lead` | Attempt a send | Header auth | Lock, final gate, reservation, provider recheck |
| `unsubscribe` | Unsubscribe event | Header auth | Requires stable `event_id` |
| `bounce` | Bounce event | Header auth | `hard` flag selects hard or soft handling |
| `reply` | Reply event | Header auth | Classification selects handling |
| `circuit-breaker-reset` | Administrative breaker flag reset | Header auth | Does not restore global send permission |
| `discovery-target-prompt` | Start an autonomous discovery run | Header auth | Free-text prompt or structured fields |
| `discovery-status` | Read run counters and recent events | Header auth | Requires `discovery_run_id` |
| `feedback-report` | On-demand read-only counts | Header auth | Request body is ignored |
| `roadmap-audit` | On-demand read-only audit snapshot | Header auth | Observational only |

**Response behavior.** Webhook responses are produced by one `Shared Webhook Response` node that returns JSON from `response_body` (or the whole item) with a status code from `response_code`, defaulting to 200. Status codes visible in the controllers include 200, 202, 400, 404, 409, 422, 502, and 503. Transition errors `ILLEGAL_TRANSITION`, `EXPECTED_STATE_MISMATCH`, `TRANSITION_GUARD_FAILED`, and `CAS_FAILED` map to 409; other transition error codes map to 503. Scheduled triggers have no caller to respond to.

<details>
<summary><b>Example request payloads (structure derived from controller code)</b></summary>

<br>

These are examples with obvious placeholders. They are not recorded requests.

**Lead intake (`ingest-lead` or `discovery-candidate`)**

```json
{
  "lead_id": "<LEAD_ID>",
  "campaign_id": "<CAMPAIGN_ID>",
  "email": "<contact email>",
  "company_name": "<company name>",
  "contact_name": "<contact name>",
  "job_title": "<job title>",
  "domain": "<company domain>",
  "location": "<location>",
  "industry": "<industry>",
  "discovery_source": "<source identifier>",
  "discovery_query": "<query text>",
  "contact_source": "<contact source>",
  "email_discovery_source": "<email source>",
  "enrichment_provider": "<provider label>",
  "target_parameters": {},
  "company_validation_status": "<status>",
  "domain_validation_status": "<status>"
}
```

**Verification result (`verification-result`)**

```json
{
  "lead_id": "<LEAD_ID>",
  "status": "VALID",
  "provider": "<verifier label>",
  "signals": {}
}
```

**Research finalization (`research-finalize`)**

```json
{
  "lead_id": "<LEAD_ID>",
  "correlation_id": "<CORRELATION_ID>",
  "claims": [
    {
      "claim": "<claim text>",
      "source_url": "<source URL>",
      "source_type": "PUBLIC_WEB",
      "evidence": "<supporting evidence text>",
      "retrieved_content_ref": null,
      "provider_status": "<provider status>",
      "verification_status": "SUPPORTED"
    }
  ],
  "limitations": []
}
```

**Human review decision (`human-review-decision`)**

```json
{
  "lead_id": "<LEAD_ID>",
  "reviewer": "<reviewer identifier>",
  "approve": true,
  "review_id": "<optional REVIEW_ID>"
}
```

**Autonomous target prompt (`discovery-target-prompt`), free-text form**

```json
{
  "prompt": "<natural language description of the target companies and the requested counts>",
  "campaign_id": "<CAMPAIGN_ID>"
}
```

**Event payloads (`unsubscribe`, `bounce`, `reply`)**

```json
{ "event_id": "<STABLE_EVENT_ID>", "lead_id": "<LEAD_ID>" }
```

```json
{ "event_id": "<STABLE_EVENT_ID>", "lead_id": "<LEAD_ID>", "hard": true }
```

```json
{ "event_id": "<STABLE_EVENT_ID>", "lead_id": "<LEAD_ID>", "classification": "POSITIVE" }
```

</details>

---

## 🔎 Discovery & Intake

Discovery orchestration is provider-driven. The workflow does not search the internet itself. External discovery services supply candidate records either through the authenticated `discovery-candidate` and `ingest-lead` endpoints, or through the response of the configured `Discovery Provider` HTTP node during an autonomous run.

### Lead Intake Safety

| Control | Source-supported behavior |
|:--------|:--------------------------|
| **Campaign state** | Campaign must exist (404 otherwise) and be `ACTIVE` (409 otherwise) |
| **Suppression** | Normalized email is checked against the global `suppression` table |
| **Duplicate prevention** | Rejects a matching `lead_id`, a matching campaign and email pair, or the same email already present in a lifecycle state other than `SUPPRESSED` or `REJECTED` |
| **Advisory locking** | Transaction-level advisory lock keyed on campaign and email serializes concurrent intake |
| **Email normalization** | Trimmed and lowercased; format pattern and 254-character limit enforced |
| **Critical fields** | Missing `lead_id`, `campaign_id`, or a valid email yields `CRITICAL_FIELD_INVALID` (400) |
| **Content screening** | Company, contact, and title fields are screened for suspicious instruction-like patterns; a hit is rejected (400) and audited as `PROMPT_INJECTION_SUSPECTED` |
| **Exact field mapping** | Named request fields map to named `leads` columns; the row is inserted as `DISCOVERED` with verification `PENDING` |
| **Audit** | Every intake outcome writes an audit event (accepted, campaign missing, campaign inactive, suppressed, duplicate) |

### Pattern-Based Content Screening

Pattern-based defensive screening is implemented for selected inbound content: intake fields, research text, and research claims. The patterns target instruction-like phrases such as requests to ignore prior instructions, references to system or developer prompts, credential-related words, requests to forward messages, role-override phrasing, and phrases that ask to disable suppression, safety, approval, or gates.

| Surface | Behavior on a match |
|:--------|:--------------------|
| **Lead intake** | Request rejected with 400 and an audit event |
| **Research text** | Matches are replaced with a space; response carries `prompt_injection_suspected` and an audit event is written |
| **Research claims** | Matched text is cleaned; a claim marked `SUPPORTED` is downgraded to `UNSUPPORTED` and a limitation is recorded |

> [!IMPORTANT]
> This is a pattern filter on selected fields. It is not a claim of comprehensive protection against prompt injection.

---

## ✅ Email Verification

Verification is external. The `verification-result` endpoint records an outcome reported by an external verifier, and the autonomous path calls the `Email Verification Provider` node directly.

| Provider status | Lead transition | Additional effect |
|:----------------|:----------------|:------------------|
| `VALID` | `VERIFIED` | Response action `CONTINUE` |
| `INVALID` | `REJECTED` | Response action `REJECT` |
| `UNKNOWN`, `CATCH_ALL_RISK`, `CATCH_ALL`, `ERROR`, `TIMEOUT`, or any other value | `HOLD` | Review queue row of type `VERIFICATION_RISK` |

A verification row is written to `email_verifications` and the lead's verification fields are updated in the same transition statement. In the autonomous path, any result other than `VALID` records a candidate failure and no lead is created. The workflow records what the verifier reports; it does not assert deliverability, mailbox existence, or verifier accuracy. A provider failure remains a dependency or failure state.

---

## 🔬 Research & Evidence

Research evidence is treated as structured evidence, not as free-form truth.

| Item | Behavior |
|:-----|:---------|
| **Research request** | `research-request` sets the lead's research status to `EXTERNAL_DEPENDENCY_REQUIRED`, records an audit event with a correlation identifier, and returns the request; a repeated call reports the pending request instead of creating another |
| **Text ingestion** | `research-ingest-text` screens and cleans the text and records an audit event |
| **Finalization** | `research-finalize` requires `lead_id` and a `claims` array; the lead must be in `VERIFIED` and moves to `RESEARCHED` |
| **Claim identity** | The workflow generates a new `claim_id` for every stored claim |
| **Claim fields** | `claim`, `source_url`, `source_type`, `evidence`, `retrieved_content_ref`, `provider_status`, `verification_status`, `correlation_id` |
| **Claim states** | `SUPPORTED`, `UNSUPPORTED`, `CONTRADICTED`, `UNKNOWN`; unrecognized values become `UNKNOWN` |
| **Support rule** | `SUPPORTED` requires claim text, source URL, source type, evidence text, provider status, and either a retrieved content reference or source type `PUBLIC_WEB`; otherwise it is downgraded to `UNKNOWN` |
| **Quality tiers** | 3 or more supported claims: `HIGH`; 2: `MODERATE`; 1: `LOW`; 0: `NONE` |
| **Status** | `NONE` with no supported claims, `LIMITED` when limitations exist, otherwise `COMPLETE` |
| **Limitations** | Unsupported, contradicted, or unknown claims add `UNSUPPORTED_OR_UNCERTAIN_CLAIMS_SKIPPED`; screening matches add `PROMPT_INJECTION_CONTENT_UNSUPPORTED` |
| **Data level** | Level 1 when the company is validated (or at least one supported claim exists) and a contact name and source are present; level 2 for company-level data; otherwise level 3 |

In the autonomous path, a failed or unusable research provider response places the lead on `HOLD` as an external dependency, and a result with zero supported claims places it on `HOLD` for insufficient evidence. Manual finalization records a `NONE` quality result instead.

---

## 🧠 AI Drafting & Personalization

The AI provider is an external dependency reached through the configured `AI_PROVIDER_API_URL`. Its output is treated as untrusted input and processed by deterministic control logic.

| Input to the AI provider | Source |
|:-------------------------|:-------|
| Lead and company context | `leads` row (company name, website, verification status) |
| Contact context | Included only when personalization level is 1 and a contact name exists |
| Verified research | Only claims stored as `SUPPORTED`, each with `claim_id`, source, evidence |
| Personalization level | 1, 2, or 3 from the lead record |
| Campaign context | `service_product`, `campaign_objective`, `cta`, `tone` when present in the request body |
| Compliance requirements | `include_optout`, `no_false_claims`, `truthful_sender_identity` |

The system instructions require using only supplied verified information, using only supported claim IDs, respecting the personalization level exactly, returning a fixed JSON shape (`to`, `subject`, `body`, `personalization_level`, `claim_ids`, `needs_review`, `has_optout`), setting `has_optout` to true, and leaving permissions, approval, queue, and safety controls untouched.

**Personalization is bounded by evidence.** Contact-level personalization is available only at level 1, which requires validated company data and a sourced contact. Level 2 and 3 drafts receive no contact information. Depth is limited by the evidence and data quality present; it is not described as guaranteed or fully verified.

| Draft validation (deterministic) | Result |
|:---------------------------------|:-------|
| AI call failure or error status | Lead moves to `HOLD` with `EXTERNAL_DEPENDENCY_REQUIRED` (503) |
| Missing `to`, `subject`, or `body` | `AI_OUTPUT_INVALID` (502) |
| Claim ID not in the supplied supported set | `HALLUCINATED_CLAIM_ID` (422) |
| Placeholder tokens, wrong recipient, or missing opt-out content | Recorded as QA reasons |

---

## 🧾 Claim Verification & QA

The workflow keeps five concerns separate:

| Concern | What it is | Where it happens |
|:--------|:-----------|:-----------------|
| **1. Research evidence** | Stored claims with source and evidence | Research finalization |
| **2. AI-generated copy** | Draft text citing claim IDs | AI provider, then deterministic validation |
| **3. Claim verification** | External classification of every factual statement in the draft | `AI Claim Verification Provider` |
| **4. QA** | Rule-based decision over draft checks and claim verification | `Research & Draft Controller` |
| **5. Send authorization** | Fresh re-check of approval, QA binding, claims, and safety state | `Send Safety Controller` |

Claim verification asks the external provider to extract every factual statement and classify it as `SUPPORTED`, `UNSUPPORTED`, `CONTRADICTED`, or `UNKNOWN`, where support requires exact supplied claim IDs. The controller then requires an overall `PASS`, a well-formed response, every classified claim to reference known claims that have a source and evidence, coverage of every claim ID the draft cites, and all claims supported. A provider error or malformed response is a failed verification.

### QA Decisions

| Decision | Condition | Lifecycle effect |
|:---------|:----------|:-----------------|
| `PASS` | No QA reasons and the draft is not flagged for review | `QA` to `APPROVED` with approval status `AUTO_APPROVED` |
| `REVIEW` | No QA reasons and the draft sets `needs_review` | `QA` to `REVIEW`, review queue row `QA_UNCERTAIN`, approval status `PENDING_REVIEW` |
| `REMOVE_REGENERATE` | Only the opt-out content check failed and fewer than 3 drafts exist | Back to `DRAFTED` and a new draft is generated |
| `STOP` | Any other reason | `HOLD` with review queue row `QA_STOP` |

QA reason codes include `PLACEHOLDER`, `WRONG_RECIPIENT`, `OPTOUT_CONTENT`, `DRAFT_PRE_QA_INVALID`, and `AUTOMATIC_CLAIM_VERIFICATION_NOT_PASSED`. The draft and QA record (including a SHA-256 content hash) are persisted in the same statement as the move to `DRAFTED`; a separate transition then records `QA`, and a further transition applies the decision.

---

## 👤 Human Review

Human approval is a distinct lifecycle state and decision, not a side effect of AI output.

| Aspect | Behavior |
|:-------|:---------|
| **Entry** | A lead reaches `REVIEW` when QA returns `REVIEW`, or from `HOLD` where the graph allows it |
| **Endpoint** | `human-review-decision` with `lead_id`, `reviewer`, `approve`, optional `review_id` |
| **Transition** | Expected state `REVIEW`; approve moves to `APPROVED`, reject moves to `REJECTED` |
| **Approval status** | `HUMAN_APPROVED` or `HUMAN_REJECTED` |
| **Review queue** | Guard requires exactly one pending review row for the lead; the row is marked `RESOLVED` with the resolution |
| **Locking** | Transaction-level advisory lock keyed on the lead serializes concurrent decisions |
| **Audit** | `Human Review Decision` event with reviewer and approval status |
| **Send eligibility** | The send gate accepts a human-approved path only when the latest QA decision was `REVIEW` and approval status is `HUMAN_APPROVED` |

Not every lead requires human review: drafts that pass QA are approved automatically, and drafts that fail are held.

---

## 🛡️ Queue & Final Send Safety

Outbound sending is protected by a separate controller that treats missing or unknown state as a reason to block.

### Queueing

`enqueue-lead` (and the autonomous queue stage) reads a snapshot, evaluates the gate in `QUEUE` mode, and on success moves the lead from `APPROVED` to `QUEUED` while inserting a `send_queue` row.

| Queue element | Behavior |
|:--------------|:---------|
| **Idempotency key** | SHA-256 of campaign ID and lowercased normalized email |
| **Queue row** | `queue_id`, `lead_id`, `campaign_id`, `draft_id`, `idempotency_key`, optional `autonomous_run_id`, state `QUEUED`, `attempt_count` 0 |
| **Duplicate guard** | Insert is refused if the idempotency key already exists in the queue |
| **Autonomous cap** | For run-owned queue rows, a run-level lock and a count of active send slots keep the queue within the run's requested send count |
| **Run accounting** | Queued and sendable counters on the run row are incremented on first application |

### Final Send Gate

The gate runs before reservation and again at provider time. It collects every failing reason and blocks on any.

| Check family | Examples of reasons |
|:-------------|:--------------------|
| **Identity and fields** | `CRITICAL_FIELD_INVALID`, `SENDER_NOT_CONFIGURED` |
| **Lead state** | `LEAD_STATE_NOT_QUEUED` (or `LEAD_STATE_NOT_APPROVED` in queue mode) |
| **Campaign** | `CAMPAIGN_INACTIVE` including start and end date checks |
| **Verification** | `VERIFICATION_NOT_VALID` |
| **Draft binding** | `DRAFT_INVALID`, `QUEUE_DRAFT_BINDING_MISMATCH`, `DRAFT_RECIPIENT_MISMATCH`, `DRAFT_PLACEHOLDER` |
| **Approval and QA** | `QA_APPROVAL_NOT_VALID`, `QA_DRAFT_BINDING_MISMATCH`, `NOT_APPROVED`, `HUMAN_REJECTED` |
| **Claims** | `AUTOMATIC_CLAIM_VERIFICATION_NOT_PASSED`, `UNSUPPORTED_CLAIM_ID` |
| **Duplicates and suppression** | `DUPLICATE_OR_ALREADY_SENT`, `SUPPRESSED` |
| **Daily limit** | `DAILY_LIMIT`, `DAILY_LIMIT_STATE_INVALID` |
| **Global flags** | `GLOBAL_SEND_PERMISSION_FALSE`, `CIRCUIT_BREAKER_OPEN`, `SYSTEM_ANOMALY_BLOCKED` |
| **Queue and idempotency** | `QUEUE_NOT_SENDABLE`, `RETRY_NOT_DUE`, `ATTEMPT_LIMIT_INVALID`, `IDEMPOTENCY_KEY_INVALID`, `IDEMPOTENCY_CONFLICT`, `PROVIDER_MESSAGE_ID_ALREADY_PRESENT` |
| **Ambiguity** | `PENDING_AMBIGUOUS_SEND_REVIEW` |
| **Lock and reservation** | `SEND_LOCK_INVALID`, `SEND_LOCK_EXPIRED`, `SEND_LOCK_TOO_CLOSE_TO_EXPIRY`, `DAILY_RESERVATION_MISSING` |
| **Autonomous run** | `AUTONOMOUS_RUN_UNKNOWN`, `AUTONOMOUS_SEND_TARGET_EXCEEDED` |
| **Dependency freshness** | `DEPENDENCY_NOT_HEALTHY` for `database`, `queue`, `email_provider`, `workflow_engine` when a status is not `HEALTHY` or older than 15 minutes |
| **Provider configuration** | `EMAIL_PROVIDER_NOT_CONFIGURED` |

This is a layered, fail-closed control. It reduces the chance of an unintended send; it is not described as perfect sending safety.

### Atomic Send Lock and Reservation

| Step | Mechanism |
|:----:|:----------|
| **1. Lock acquisition** | Advisory transaction lock per lead, expired lock rows removed, new `send_locks` row with a 5-minute expiry only if no active lock exists; otherwise `LOCK_BUSY` (409) |
| **2. Snapshot** | Exactly one sendable queue row (`QUEUED` or `RETRY_WAIT`) must exist for the lead |
| **3. Gate in `FINAL_SEND` mode** | Any failing reason releases the lock and returns 409 with the reason list |
| **4. Reserve and mark** | One statement re-reads state, increments the campaign's daily counter only while below the limit, and sets the queue row to `SENDING` with an incremented attempt count; a rollback fragment decrements the counter if the mark did not apply |
| **5. Provider-time recheck** | Queue must still be `SENDING` without a provider message ID; gate runs in `PROVIDER_SEND` mode including lock holder, remaining lock time of at least 20 seconds, and reservation presence; failure returns the row to `QUEUED`, releases the reservation, and deletes the lock |
| **6. Provider call** | `Email Provider` node with a 15-second timeout, one try, and an error output; request carries the idempotency key, recipient, subject, body, and sender account |
| **7. Durable recording** | Confirmed success sets the queue row to `SENT` and moves the lead to `SENT` through the central controller, then records the message ID, resets the failure counter, releases the lock, and writes an audit event |

The daily counter is keyed by campaign and UTC date. A confirmed send requires a message identifier, no error, and a 2xx status when a status is present.

### Safety Control Matrix

| Control | Purpose | What It Prevents |
|:--------|:--------|:-----------------|
| **Suppression** | Global do-not-contact list by normalized email | Intake of, or sending to, suppressed addresses |
| **Duplicate protection** | Intake rules, cross-lead duplicate check, unique idempotency key | Double intake and repeat sends to the same address |
| **Expected-state checking** | Caller states the status it believes the lead has | Acting on stale assumptions about lifecycle state |
| **CAS-style updates** | Update succeeds only if the row still has the status that was read | Lost updates under concurrent writers |
| **Send lock** | Per-lead lock with expiry and holder identity | Concurrent send attempts for one lead |
| **Daily counters** | Campaign and UTC-date reservation below the daily limit | Exceeding the configured daily limit |
| **Circuit breaker** | Global flags consulted by every gate | Continued sending after repeated ambiguous outcomes |
| **Claim verification** | External classification of every draft claim against supplied evidence | Unsupported factual claims reaching a recipient |
| **QA gate** | Rule-based decision bound to the specific draft | Placeholders, wrong recipient, missing opt-out content |
| **Human review** | Distinct `REVIEW` state and decision endpoint | Uncertain drafts proceeding without a person's decision |
| **Ambiguous outcome hold** | `AMBIGUOUS` queue state with a blocking review row | Blind resend after an uncertain provider result |
| **Reconciliation** | Optional external lookup of the real outcome | Treating an unknown outcome as sent or not sent without evidence |
| **Dependency freshness** | Four components must report `HEALTHY` within 15 minutes | Sending while core dependencies are unverified |
| **Audit logging** | Event rows for transitions, sends, holds, and recovery | Untraceable state changes |

---

## 🚨 Circuit Breaker

| Element | Behavior |
|:--------|:---------|
| **Tracked signal** | `CONSECUTIVE_SEND_FAILURES` in `system_flags`, incremented on each ambiguous provider response held at send time and reset to 0 on a confirmed send |
| **Opening** | At 3 consecutive counted failures: `CIRCUIT_BREAKER` set to `OPEN` and `GLOBAL_SEND_PERMISSION` set to `FALSE` |
| **Alerting** | Route `ALERT` posts a message to the alert webhook after the flags are changed; alerting does not authorize anything, and reconciliation continues even if the alert call fails |
| **Gate effect** | The send gate requires `GLOBAL_SEND_PERMISSION` equal to `TRUE`, `CIRCUIT_BREAKER` equal to `CLOSED`, and `SYSTEM_ANOMALY` equal to `FALSE`; absent or unknown values block |
| **Administrative reset** | `circuit-breaker-reset` sets `CIRCUIT_BREAKER` to `CLOSED`, sets the failure counter to 0, and writes an audit event; the response states that global send permission is not restored and manual action is required |
| **Automatic recovery** | The five-minute maintenance run moves an open breaker through `RECOVERY_TEST` and back to `CLOSED` only with healthy dependency checks (see below) |

<details>
<summary><b>Automatic recovery actions (five-minute maintenance run)</b></summary>

<br>

| Action | Condition |
|:-------|:----------|
| `ENTER_RECOVERY_TEST` | Breaker `OPEN`, permission `FALSE`, anomaly `FALSE`, four fresh `HEALTHY` components, and the permission change was made by the circuit breaker |
| `HEALTHY_STREAK_PROGRESS` | Breaker `RECOVERY_TEST`, permission `FALSE`, anomaly `FALSE`, four healthy components, streak below 1 |
| `CLOSE_AND_RESTORE` | Breaker `RECOVERY_TEST`, permission `FALSE`, anomaly `FALSE`, four healthy components, streak of at least 1; sets breaker `CLOSED` and permission `TRUE` |
| `RESET_RECOVERY_STREAK` | Breaker `RECOVERY_TEST` with fewer than four healthy components |
| `NOOP` | Any other combination |

Health rows are read from a `health_checks` table. The exported workflow reads that table but contains no node that writes to it, so a separate health monitor is an external dependency.

</details>

The breaker counts the signals this workflow records (ambiguous send outcomes). It does not claim protection against every failure class.

---

## ⚠️ Ambiguous Send Outcomes

Ambiguity is a first-class outcome. It is not collapsed into "failed send".

| Situation | Handling |
|:----------|:---------|
| **Provider response without a confirmed message ID, with an error, or with a non-2xx status** | Queue row set to `AMBIGUOUS`, `AMBIGUOUS_SEND` review row created, lock released, failure counter incremented, `AMBIGUOUS_OUTCOME_HELD` audit event |
| **Database failure after the provider call** | Exact `SENDING` row durably set to `AMBIGUOUS` with a review row and lock release in one statement, response 202, reconciliation invoked only after that commit |
| **Row left in `SENDING` for over 10 minutes with no message ID and no active lock** | Found by the five-minute maintenance run, set to `AMBIGUOUS`, review row created, audited as crash recovery |

No automatic resend occurs from `AMBIGUOUS`. While a pending `AMBIGUOUS_SEND` review exists for a lead, the send gate blocks further attempts.

> [!IMPORTANT]
> Ambiguous email-provider outcomes are not automatically resent.

---

## ⚖️ Provider Reconciliation

Reconciliation is external and optional, configured through `EMAIL_PROVIDER_RECONCILE_URL`. When it is unset, ambiguous items remain held.

| Reconciliation response | Workflow action |
|:------------------------|:----------------|
| `SENT` with a message identifier | Queue row `AMBIGUOUS` to `SENT`, lead transitions to `SENT` through the central controller, lock cleared, `Email Sent Reconciled` audit event |
| `NOT_SENT` | Queue row becomes `RETRY_WAIT` with a 5-minute delay (or `FAILED` after 3 attempts, or `CANCELLED` if the lead reached a terminal state); the daily reservation is released; `Ambiguous Reconciled Not Sent` audit event |
| Any other response | Not treated as confirmation; no blind resend is authorized |

The reconciliation request carries `lead_id`, `queue_id`, `campaign_id`, `idempotency_key`, `provider_message_id`, `attempt_count`, and a mode of `AMBIGUOUS_SEND` or `CRASH_RECOVERY`. The workflow defines no reconciliation semantics beyond the two outcomes above. A `RETRY_WAIT` row due for retry is picked up by the five-minute maintenance run and re-enters the full send path, including the gate.

---

## 📡 Event Handling & Suppression

Lifecycle events require a stable `event_id`. Missing IDs return `EVENT_ID_REQUIRED` (400). Events are recorded in `reply_events`; a repeated `event_id` returns a 200 response with `duplicate_event: true` and `processed: false`.

| Event | Handling |
|:------|:---------|
| **Unsubscribe** | Records the event, upserts a suppression row, cancels `QUEUED` and `RETRY_WAIT` queue rows, marks the lead suppressed, audits, and transitions to `SUPPRESSED` (or `UNSUBSCRIBED` when the lead is `SENT` or `REPLIED`) |
| **Hard bounce** | Records the event, suppresses the email with reason `HARD_BOUNCE`, cancels pending queue rows, sets bounce reply status, transitions to `BOUNCED` |
| **Soft bounce** | Recorded and audited only; no lifecycle change |
| **Reply, positive** (`POSITIVE`, `INTERESTED`, `REPLIED_POSITIVE`) | Records the event, cancels pending queue rows, transitions to `REPLIED`, audits with `human_handling_required` set to true |
| **Reply, negative** (`NEGATIVE`, `NO_FURTHER_CONTACT`, `UNSUBSCRIBE`, `NOT_INTERESTED`) | Suppresses with reason `NO_FURTHER_CONTACT`, cancels pending queue rows, transitions to `SUPPRESSED` |
| **Reply, other** | Recorded as `REPLIED_OTHER` with human handling flagged; no lifecycle change |

Positive replies do not trigger an automatic reply. The workflow flags them for human handling.

### Suppression

Suppression is a global table keyed by normalized email. It is consulted at lead intake, autonomous pre-intake checks, queue gating, final send gating, the reservation statement, and the provider-time recheck. A suppressed address cannot be inserted as a new lead and cannot pass the send gate. This is a technical control; it is not legal compliance certification and does not replace legal review or a complete compliance program.

---

## 🤖 Autonomous Campaign Orchestration

Autonomy here is bounded and state-governed. A run is a durable row in `autonomous_campaign_runs`, every candidate passes through the normal intake and lifecycle pipeline, and every send still passes the full send gate.

### Target Definition

The `discovery-target-prompt` endpoint defines the discovery objective and orchestration context. It does not discover leads itself; an external discovery service performs discovery.

| Input mode | Fields read by the exported code |
|:-----------|:---------------------------------|
| **Free-text** | `prompt` (at least 8 characters), interpreted by the external AI provider into a structured target; `discovery_only` flag; `campaign_id` |
| **Structured** | `campaign_id`, `daily_limit` (greater than 0), at least one of `target_industry`, `target_region`, `target_persona`, `target_job_role`, and a count from `quantity`, `requested_company_count`, or `requested_send_count` (1 to 1000) |

The AI extraction instruction returns strict JSON with fields for `requested_company_count`, `requested_send_count`, `discovery_only`, `industry`, `company_type`, `geography`, `company_size`, `product_service_category`, `technology_criteria`, `website_requirements`, `youtube_social_requirements`, `target_contact_role`, `target_department`, `email_requirements`, `research_requirements`, `personalization_requirements`, `outreach_objective`, `desired_cta`, `email_tone`, `campaign_id`, `approved_discovery_sources`, `contact_requirement`, and `allow_company_level_outreach`, and is told not to invent quantities or critical parameters. Discovery-only mode is honored only when the prompt states it explicitly or the request sets `discovery_only` to true. The target must resolve to exactly one `ACTIVE` campaign; otherwise the run is held.

### Run Lifecycle

| Stage | Behavior |
|:-----:|:---------|
| **1. Run creation** | Inserts a run row with limits derived from configuration; status `DISCOVERY_IN_PROGRESS`; the start result carries a 202 `ACCEPTED` body |
| **2. Discovery batch** | `Discovery Provider` receives the current query, page, cursor, target count, and batch size; a provider error pauses the run as `PROVIDER_UNAVAILABLE` and no candidates are fabricated |
| **3. Candidate validation** | Requires a valid domain, company name, an HTTPS source URL, and five verified flags (identity, domain, company-domain match, business relevance, active business signal) |
| **4. Pre-intake checks** | Suppression, duplicate email in the campaign, and duplicate domain among sent, replied, bounced, or unsubscribed leads |
| **5. Contact and email discovery** | Contact discovery is called when contact-level outreach is required; email discovery is called with verification required |
| **6. Verification** | Only `VALID` continues; otherwise a candidate failure is recorded |
| **7. Intake and verified transition** | Same intake rules as manual intake, then the central transition `DISCOVERED` to `VERIFIED`; verified candidates are not inserted directly as `VERIFIED` |
| **8. Research, drafting, QA** | Uses the research, drafting, claim verification, and QA logic described above |
| **9. Queue and send accounting** | Approved leads enter the queue through the same gate with a per-run send-slot cap; sends are accounted to the run when they complete |
| **10. Counters and stop rules** | After each candidate, counters are read from durable run and queue rows and compared with stop rules |

### Bounds and Stop States

| Bound | Default | Configuration |
|:------|:--------|:--------------|
| Candidate over-fetch factor | 3 (clamped 1 to 10) | `AUTONOMOUS_CANDIDATE_OVERFETCH_FACTOR` |
| Discovery batch size | 25 (clamped 1 to 250) | `AUTONOMOUS_DISCOVERY_BATCH_SIZE` |
| Global candidate ceiling | 10000 (minimum 1000) | `AUTONOMOUS_MAX_TOTAL_CANDIDATES_GLOBAL` |
| Iterations | 500 | `AUTONOMOUS_MAX_ITERATIONS_GLOBAL` |
| Provider batches | 1000 | `AUTONOMOUS_MAX_PROVIDER_BATCHES_GLOBAL` |
| Provider pages | 1000 | `AUTONOMOUS_MAX_PROVIDER_PAGES_GLOBAL` |
| Runtime | 120 minutes (clamped 5 to 1440) | `AUTONOMOUS_MAX_RUNTIME_MINUTES` |
| Queries | 20 | Fixed in code |

The defaults are configured ceilings; the code derives lower effective limits for small targets. Run statuses visible in the code: `DISCOVERY_IN_PROGRESS`, `REPLACEMENT_DISCOVERY_IN_PROGRESS`, `TARGET_QUALIFIED_REACHED`, `TARGET_SENDABLE_COUNT_REACHED`, `TARGET_SEND_COMPLETED`, `GLOBAL_SEND_PAUSED`, `EXTERNAL_DEPENDENCY_REQUIRED`, `DISCOVERY_BUDGET_EXHAUSTED`, `DISCOVERY_RUNTIME_EXHAUSTED`, `SAFE_SHORTFALL`, `PROVIDER_UNAVAILABLE`, and `SAFETY_HOLD`. If the counter read is invalid or incomplete, the run moves to `SAFETY_HOLD` instead of guessing. Sending runs pause when global send permission is not `TRUE`, the breaker is not `CLOSED`, or dependency health is stale.

### Recovery

| Schedule | Scope | Behavior |
|:---------|:------|:---------|
| **Every 5 minutes** (`Every 5 Minutes`) | Send and breaker maintenance | Automatic breaker recovery progression, marking stuck `SENDING` rows `AMBIGUOUS`, dispatching due `RETRY_WAIT` rows through the send path, and invoking reconciliation for stuck rows when configured |
| **Every 10 minutes** (`Autonomous: Discovery Recovery Sweep`) | Autonomous runs | Claims at most one stale run with `FOR UPDATE SKIP LOCKED` and a recovery claim identifier (claims expire after 30 minutes) |

The discovery sweep targets runs that have not updated for 30 minutes in an in-progress state, and runs marked `TARGET_SENDABLE_COUNT_REACHED` whose active send slots have fallen below the requested count. It never auto-resends `AMBIGUOUS` outcomes, and it does not guarantee that every failure can be repaired automatically.

---

## 📊 Analytics, Audit & Reporting

| Endpoint or trigger | Mode | What it returns or writes |
|:--------------------|:-----|:--------------------------|
| `discovery-status` | On demand, read-only | Run row and recent `autonomous` audit events for a `discovery_run_id` |
| `feedback-report` | On demand, read-only | Counts of QA outcomes, QA failure reasons, and send outcome audit events (`Email Sent`, `AMBIGUOUS_OUTCOME_HELD`); flagged `read_only` |
| `roadmap-audit` | On demand, read-only | Static audit-style snapshot: reports `production_ready` as `false`, `live_runtime_verified` as `false`, lists external dependencies, and includes current safety flags; flagged `observational_only` |
| `Daily Analytics Snapshot` | Scheduled | Reads the `v_funnel` view and writes an `Analytics Snapshot` audit event with the row count; no response is returned |

The feedback report is a historical count report. The workflow contains no model retraining, optimization, or predictive logic. The roadmap audit is observational and is not evidence of production readiness or an independent guarantee of correctness.

Audit events (`audit_events`) capture event type, lead, campaign, source system, previous and new state, and metadata for lifecycle transitions, intake outcomes, research, QA, review, queueing, sends, ambiguous holds, reconciliation, circuit-breaker actions, recovery, and autonomous run events.

---

## 🗄️ Data Layer

The workflow expects an externally provisioned PostgreSQL schema. The exported JSON contains no `CREATE TABLE` statements and no migrations.

| Domain | Relations referenced in SQL |
|:-------|:----------------------------|
| **Lifecycle** | `leads` |
| **Campaign configuration** | `campaigns` |
| **Compliance** | `suppression` |
| **Verification** | `email_verifications` |
| **Evidence and research** | `evidence_claims`, `research_records` |
| **Drafting and QA** | `email_drafts`, `qa_results` |
| **Review** | `review_queue` |
| **Sending** | `send_queue`, `send_locks`, `daily_send_counters` |
| **Control plane** | `system_flags`, `health_checks` |
| **Events** | `reply_events` |
| **Audit** | `audit_events` |
| **Autonomous runs** | `autonomous_campaign_runs` |
| **Reporting** | `v_funnel` (view) |

### DB Gateway

`DB Gateway` is the single SQL boundary: a PostgreSQL node in `executeQuery` mode that runs the controller-supplied `sql` with the controller-supplied `params` as query replacements. Controllers build parameterized statements; the node itself contains no business logic. It is configured to continue with regular output on error, and the result controller treats missing or error results as failures (`DB_OPERATION_FAILED`, 503) rather than as success. Lifecycle updates, audit writes, queue persistence, and run accounting all pass through it. It is not an ORM, and the SQL uses PostgreSQL-specific features such as advisory locks, `FOR UPDATE`, and CTE-based writes, so portability to other databases is not claimed.

---

## 🔌 External Dependencies

| Dependency | Role | Required / Optional | Boundary |
|:-----------|:-----|:-------------------:|:---------|
| n8n | Runs the workflow | Required | Workflow engine |
| PostgreSQL | State, queue, audit | Required | `DB Gateway`; schema provisioned externally |
| External AI provider | Target understanding, drafting, claim verification | Required for AI paths | `AI_PROVIDER_API_URL` |
| External discovery provider | Candidate discovery | Required for autonomous runs | `DISCOVERY_PROVIDER_API_URL` |
| External contact discovery provider | Contact lookup | Used when contact-level outreach is required | `CONTACT_DISCOVERY_PROVIDER_API_URL` |
| External email discovery provider | Email lookup | Used in autonomous runs | `EMAIL_DISCOVERY_PROVIDER_API_URL` |
| External verification provider | Email verification | Used in autonomous runs; manual flow uses the result endpoint | `EMAIL_VERIFICATION_PROVIDER_API_URL` |
| External research provider | Company research | Used in autonomous runs; manual flow uses ingestion endpoints | `RESEARCH_PROVIDER_API_URL` |
| External email provider | Outbound delivery | Required for sending | `EMAIL_PROVIDER_SEND_URL` |
| External reconciliation provider | Ambiguous outcome resolution | Optional | `EMAIL_PROVIDER_RECONCILE_URL` |
| Alert destination | Circuit-breaker alert | Used on breaker opening | `ALERT_WEBHOOK_URL` |
| Health monitor | Writes `health_checks` | Required for sending | Not part of this workflow |
| n8n credentials | Webhook and provider header authentication | Required | Configured in n8n, not in this document |

### Technology

| Technology | Role |
|:-----------|:-----|
| n8n | Orchestration |
| PostgreSQL | State and persistence |
| JavaScript (n8n Code nodes) | Controllers and orchestration logic |
| HTTP APIs | Provider integrations |
| AI provider | Drafting and claim classification |
| Discovery and research providers | Candidate and evidence supply |
| Email provider | Outbound delivery |
| Reconciliation provider | Ambiguous outcome resolution |

---

## ⚙️ Environment Configuration

<details>
<summary><b>Environment variables referenced by the workflow</b></summary>

<br>

| Variable | Purpose | Required |
|:---------|:--------|:--------:|
| `AI_PROVIDER_API_URL` | External AI provider endpoint, used for target understanding, drafting, and claim verification | For AI paths |
| `DISCOVERY_PROVIDER_API_URL` | External discovery endpoint; if empty, autonomous runs pause as an external dependency | For autonomous runs |
| `CONTACT_DISCOVERY_PROVIDER_API_URL` | Contact discovery endpoint | When contact-level outreach is required |
| `EMAIL_DISCOVERY_PROVIDER_API_URL` | Email discovery endpoint | For autonomous runs |
| `EMAIL_VERIFICATION_PROVIDER_API_URL` | Email verification endpoint | For autonomous runs |
| `RESEARCH_PROVIDER_API_URL` | Research endpoint | For autonomous runs |
| `EMAIL_PROVIDER_SEND_URL` | Outbound send endpoint; blank value blocks the provider-time gate | For sending |
| `EMAIL_PROVIDER_RECONCILE_URL` | Optional reconciliation endpoint | Optional |
| `ALERT_WEBHOOK_URL` | Alert destination on circuit-breaker opening | For alerting |
| `N8N_INTERNAL_WEBHOOK_BASE_URL` | Read by the Autonomous Runtime Controller as an internal webhook base; the reviewed code reads the value but shows no request built from it | Optional |
| `ALLOW_TEST_CLOCK_OVERRIDE` | When equal to `true`, a request's `now_dt` is carried through the send-lock step | Test use only |
| `AUTONOMOUS_CANDIDATE_OVERFETCH_FACTOR` | Over-fetch factor | Optional, default 3 |
| `AUTONOMOUS_DISCOVERY_BATCH_SIZE` | Batch size | Optional, default 25 |
| `AUTONOMOUS_MAX_TOTAL_CANDIDATES_GLOBAL` | Candidate ceiling | Optional, default 10000 |
| `AUTONOMOUS_MAX_ITERATIONS_GLOBAL` | Iteration ceiling | Optional, default 500 |
| `AUTONOMOUS_MAX_PROVIDER_BATCHES_GLOBAL` | Provider batch ceiling | Optional, default 1000 |
| `AUTONOMOUS_MAX_PROVIDER_PAGES_GLOBAL` | Provider page ceiling | Optional, default 1000 |
| `AUTONOMOUS_MAX_RUNTIME_MINUTES` | Runtime ceiling | Optional, default 120 |

</details>

```bash
# EXAMPLE ONLY: placeholder values, not real endpoints
AI_PROVIDER_API_URL=<AI provider endpoint URL>
DISCOVERY_PROVIDER_API_URL=<discovery provider endpoint URL>
CONTACT_DISCOVERY_PROVIDER_API_URL=<contact discovery endpoint URL>
EMAIL_DISCOVERY_PROVIDER_API_URL=<email discovery endpoint URL>
EMAIL_VERIFICATION_PROVIDER_API_URL=<verification endpoint URL>
RESEARCH_PROVIDER_API_URL=<research endpoint URL>
EMAIL_PROVIDER_SEND_URL=<email send endpoint URL>
EMAIL_PROVIDER_RECONCILE_URL=<optional reconciliation endpoint URL>
ALERT_WEBHOOK_URL=<alert destination URL>
```

---

## 🚀 Setup & Import

> [!WARNING]
> Provider credentials, webhook authentication, and the database schema must be configured separately. Importing the JSON alone does not produce a working system.

1. **Import** the workflow JSON into n8n.
2. **Configure PostgreSQL** and attach a PostgreSQL credential to `DB Gateway`.
3. **Provision the database schema** for the relations listed under Data Layer; the JSON does not create them.
4. **Configure the provider endpoints** through the environment variables above.
5. **Configure webhook authentication** by attaching header-auth credentials to the webhook nodes.
6. **Configure the required credentials** for provider HTTP nodes that use header authentication. The `Reconcile Provider` and `Alert Webhook` nodes have no credential bound in the export.
7. **Seed control-plane data**: campaigns (status, dates, sender account and identity, daily limit), `system_flags` values for `GLOBAL_SEND_PERMISSION`, `CIRCUIT_BREAKER`, and `SYSTEM_ANOMALY`, and a health monitor writing `health_checks`.
8. **Configure external provider behavior** so response shapes match what the controllers parse.
9. **Validate webhook routing** with controlled test requests against each endpoint.
10. **Execute controlled tests** (see the validation scenarios) before any operational use. The workflow is exported inactive and must be activated deliberately.

---

## 🧪 Validation Scenarios

These are recommended test cases with the expected behavior supported by the JSON. They are not recorded results, and this repository does not claim they have been executed.

<details>
<summary><b>Expected behaviors by scenario</b></summary>

<br>

| Scenario | Expected Behavior |
|:---------|:------------------|
| Unknown lead ID | Central transition returns `UNKNOWN_LEAD_ID` without mutation; research request returns 404 |
| Invalid intake payload | 400 `CRITICAL_FIELD_INVALID` |
| Duplicate lead | Not inserted; 409 with duplicate flag; `Lead Duplicate Prevented` audit |
| Suppressed lead | Not inserted; 409 with suppressed flag; `Lead Intake Suppressed` audit |
| Invalid verification | `INVALID` moves the lead to `REJECTED` |
| Verification timeout | `TIMEOUT` or `ERROR` moves the lead to `HOLD` with a `VERIFICATION_RISK` review row |
| Research dependency failure | Autonomous path places the lead on `HOLD` as `EXTERNAL_DEPENDENCY_REQUIRED` |
| Prompt-injection content | Intake rejected (400) and audited; research text flagged; supported claims downgraded |
| Missing evidence | Incomplete `SUPPORTED` claims become `UNKNOWN`; zero supported claims gives quality `NONE` |
| Unsupported claim | Unknown claim ID gives 422; verifier not `PASS` leads to QA `STOP` and `HOLD` |
| AI draft failure | Lead moves to `HOLD` with 503 `EXTERNAL_DEPENDENCY_REQUIRED` |
| QA uncertainty | Draft flagged `needs_review` moves the lead to `REVIEW` |
| Human review required | Review row pending; decision endpoint sets `HUMAN_APPROVED` or `HUMAN_REJECTED` |
| Illegal transition | 409 `ILLEGAL_TRANSITION`; no mutation |
| Duplicate send attempt | Gate returns 409 `DUPLICATE_OR_ALREADY_SENT`; queue insert refuses a reused idempotency key |
| Concurrent send attempts | A second caller while a lock is active receives 409 `LOCK_BUSY` |
| Provider success | Queue `SENT`, lead `SENT`, message ID stored, failure counter reset, lock released |
| Provider ambiguous outcome | Queue `AMBIGUOUS`, review row, no resend, failure counter incremented |
| Reconciliation confirms `SENT` | Queue and lead move to `SENT` with `Email Sent Reconciled` audit |
| Reconciliation confirms `NOT_SENT` | Queue moves to `RETRY_WAIT` (or `FAILED` or `CANCELLED`), reservation released |
| Unsubscribe event | Suppression row, pending queue rows cancelled, lead `SUPPRESSED` or `UNSUBSCRIBED` |
| Bounce event | Hard: suppression and `BOUNCED`; soft: record only |
| Reply event | Positive: `REPLIED`, no auto-reply; negative: suppression; other: record only |
| Duplicate event ID | 200 with `duplicate_event` true and `processed` false |
| Circuit breaker open | Gate rejects with `CIRCUIT_BREAKER_OPEN` |
| Admin breaker reset | Breaker `CLOSED`, counter 0, permission not restored |
| Autonomous recovery | Stale run claimed once; no `AMBIGUOUS` row is resent |

</details>

---

## 🧯 Failure Modes

<details>
<summary><b>Failure handling by category</b></summary>

<br>

| Category | Workflow behavior |
|:---------|:------------------|
| **Validation failure** | Request rejected with 400 or 422 and an error code |
| **Dependency failure** | AI, research, or discovery failures become `EXTERNAL_DEPENDENCY_REQUIRED` holds or paused runs; no data is fabricated |
| **Provider timeout** | Node timeouts are set per provider (15 to 30 seconds); errors flow to result handling and fail closed |
| **Malformed provider response** | `AI_OUTPUT_INVALID`, invalid claim verifier output counted as a failed verification |
| **State mismatch** | `EXPECTED_STATE_MISMATCH` (409) |
| **CAS failure** | `CAS_FAILED` (409) |
| **Illegal transition** | `ILLEGAL_TRANSITION` (409) |
| **Precondition failure** | `PRECONDITION_NOT_APPLIED`; duplicate events soft-fail with a 200 duplicate response |
| **Guard failure** | `TRANSITION_GUARD_FAILED` (409) |
| **Postcondition failure** | `POSTCONDITION_NOT_APPLIED`; the statement aborts |
| **Ambiguous provider result** | Hold, review row, no resend |
| **Reconciliation failure** | Not treated as confirmation; item stays held |
| **Review hold** | `REVIEW` or `HOLD` with a pending review row |
| **Circuit breaker** | Sending blocked until recovery or reset plus restored permission |
| **Autonomous recovery** | Bounded, claim-based sweeps; counter read failure leads to `SAFETY_HOLD` |

</details>

This table describes configured behavior and does not claim that every failure can be repaired automatically.

---

## 🔐 Security & Secret Handling

| Mechanism | Description |
|:----------|:------------|
| **Authenticated webhooks** | All 18 webhook nodes use header authentication bound to n8n credentials |
| **Content screening** | Pattern-based screening of selected inbound fields |
| **Suppression controls** | Global suppression checked at intake and at every send gate stage |
| **State guards** | Legal-transition, expected-state, and compare-and-set checks |
| **Send locks** | Per-lead lock rows with expiry, plus advisory locks |
| **Fail-closed behavior** | Missing, unknown, or stale state blocks sending; unmatched routes leave the main path |
| **Ambiguous outcome protection** | No automatic resend and a blocking review row |
| **Credential separation** | Secrets live in n8n credentials and environment variables, not in the workflow logic or this document |

No credential IDs, credential names, tokens, or private URLs are reproduced in this README. The security posture depends on deployment configuration.

---

## ⚠️ Limitations

- Several external providers are required (AI, discovery, research, verification, email, and optionally reconciliation); provider availability and response shapes determine workflow behavior.
- Claim verification is an external semantic dependency.
- The PostgreSQL schema, including the `v_funnel` view and the `health_checks` writer, must be provisioned externally.
- n8n credentials and webhook authentication must be configured.
- Runtime behavior depends on deployment configuration and provider behavior.
- The JSON alone does not prove production operation, throughput, deliverability, or business results.
- The workflow is exported inactive.

<details>
<summary><b>Static review notes: items to confirm during execution testing</b></summary>

<br>

These observations come from reading the exported code; they have not been confirmed or refuted by running the workflow.

- In the `Send Safety Controller`, the send-preparation branch passes a variable named `providerUrl` that is declared only inside the later provider-recheck branch, so its scope should be verified in a controlled send test.
- The structured-fields form of `discovery-target-prompt` emits a `STRUCTURED_TARGET` stage, and no handler for that stage was located in the `Autonomous Runtime Controller`.
- After autonomous research finalization (`AUTONOMOUS_RESEARCHED` context) and after autonomous queueing (`AFTER_QUEUE` context), the transition result is routed to `AUTO`. A handler for the research context was located in the Lead & State Controller (reached through the `STATE` route), and no handler for either stage was located in the Autonomous Runtime Controller.
- The `Send Lead (Attempt)` entry point and the retry worker initiate sends; the autonomous path queues leads and accounts for sends when they complete.
- No node in the export resolves `AMBIGUOUS_SEND` review rows; the send gate treats a pending ambiguous review as blocking, so resolution is expected to occur outside this workflow.
- The feedback report reads a `reasons` column on `qa_results`, while the workflow's own QA inserts populate `checks`.
- A node note describes `discovery-candidate` as accepting a nested `candidate` object, while the code reads flat fields from the request body.
- The `Every 5 Minutes` and `Daily Analytics Snapshot` schedule rules rely on node defaults for their interval values.

</details>

---

