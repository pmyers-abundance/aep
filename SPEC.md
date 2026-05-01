# AEP Specification

**Version:** 0.1.1-draft  
**Author:** Peter Myers  
**Status:** Public Draft

---

## 1. Definitions

| Term | Definition |
|---|---|
| **Agent** | An AI system acting autonomously on behalf of an organisation or individual |
| **Agentic Envelope** | Structured metadata attached to a communication that declares agent identity, scope, and intent |
| **Capability Manifest** | A machine-readable document declaring what an organisation's agents are authorised to do and share |
| **Trust Tier** | A classification of the level of autonomy permitted between two agents |
| **Scheduling Policy** | A machine-readable ruleset governing when and how calendar events may be auto-accepted |
| **Escalation** | The routing of an agent-initiated request to a human for approval when it falls outside policy |
| **Scope** | A declared permission unit, similar to OAuth scopes, defining what an agent may access or share |

---

## 2. The Agentic Envelope

### 2.1 Required Fields

| Field | Type | Description |
|---|---|---|
| `aep_version` | String | AEP version. Currently `"0.1"` |
| `agent_id` | String | Fully-qualified agent identifier (e.g. `bookings-agent@company.com`) |
| `scope` | Array[String] | Declared permission scopes for this communication |
| `intent` | Enum | Intent code (see §2.3) |
| `trust_tier` | Enum | `T0` `T1` `T2` `T3` |
| `policy_ref` | URL | URL of sender's capability manifest |
| `signature` | String | Ed25519 signature of canonical envelope (base64url-encoded) |

### 2.2 Optional Fields

| Field | Type | Description |
|---|---|---|
| `requires_human_action` | Boolean | Whether this communication requires a human decision |
| `escalation_deadline` | ISO 8601 | Deadline for human response before escalation times out |
| `response_expected` | Enum | `ACK` `DATA` `APPROVAL` `NONE` |
| `correlation_id` | String | ID linking this to a prior exchange |
| `expiry` | ISO 8601 | Time after which this envelope should be discarded |
| `tier_required` | Enum | `T0` `T1` `T2` `T3` — minimum trust tier required to process this request. Allows receiving agents to evaluate tier requirements without fetching the sender's manifest. |
| `tier_upgrade_proof` | Object | JWS-signed tier upgrade verdict. Present when an agent operates under an approved tier upgrade. Contains `claim_type`, `claim_subtype`, `approval_evidence` (with `verdict_jws`, `approver_did`, `evaluated_at`, optional `policy_ref`), and `validity` (with `valid_until`, `scope_boundary`, `use_count_max`). See §4. |

### 2.3 Intent Codes

| Code | Category | Description |
|---|---|---|
| `BOOKING_INQUIRY` | Commerce | Query about booking availability or status |
| `BOOKING_CREATE` | Commerce | Create a new booking |
| `BOOKING_MODIFY` | Commerce | Modify an existing booking |
| `BOOKING_CANCEL` | Commerce | Cancel a booking |
| `MEETING_REQUEST` | Calendar | Request a calendar event |
| `MEETING_CANCEL` | Calendar | Cancel a calendar event |
| `MEETING_RESCHEDULE` | Calendar | Reschedule a calendar event |
| `STATUS_UPDATE` | General | Provide a status update |
| `DATA_REQUEST` | General | Request data within declared scope |
| `APPROVAL_REQUEST` | Escalation | Request human approval for an action |
| `ESCALATION_RESPONSE` | Escalation | Human response to an escalation |
| `TIER_UPGRADE_REQUEST` | Escalation | Agent requests elevation from current trust tier to a higher tier for a specific scope and duration. Distinct from `APPROVAL_REQUEST` — makes the tier-crossing nature machine-readable at the routing layer. |
| `CAPABILITY_QUERY` | Protocol | Query another agent's capabilities |
| `HEARTBEAT` | Protocol | Liveness check |

---

## 3. Capability Manifest

### 3.1 Manifest Schema

Published at: `https://{domain}/.well-known/aep-manifest.json`

```json
{
  "aep_version": "0.1",
  "org": "string",
  "org_domain": "string",
  "manifest_updated": "ISO 8601 timestamp",
  "agents": [ /* see §3.2 */ ],
  "trusted_orgs": [ /* see §3.3 */ ],
  "policy": { /* see §3.4 */ }
}
```

### 3.2 Agent Declaration

```json
{
  "agent_id": "string (email-format)",
  "display_name": "string",
  "scopes": ["string"],
  "trust_offered": "T0 | T1 | T2 | T3",
  "contact_human": "email address for escalation",
  "description": "string (optional)"
}
```

### 3.3 Trusted Organisation Declaration

```json
{
  "domain": "string",
  "trust_tier": "T0 | T1 | T2 | T3",
  "allowed_scopes": ["string"],
  "expires": "ISO 8601 (optional)"
}
```

### 3.4 Organisational Policy

```json
{
  "auto_approve_below_value": "integer (cents, optional)",
  "escalation_contact": "email",
  "escalation_timeout_minutes": "integer",
  "audit_log_required": "boolean",
  "manifest_refresh_minutes": "integer (default: 60)"
}
```

---

## 4. Trust Tier Upgrade Protocol

### 4.1 Upgrade Trigger

When an agent operating at tier T_n attempts an action that requires tier T_n+1 or higher:

1. The action is **blocked immediately**
2. The agent generates a `TIER_UPGRADE_REQUEST` envelope addressed to the target org's `escalation_contact`
3. The envelope MUST include:
   - Current `trust_tier`
   - `tier_required` — the minimum tier needed for the action
   - `scope` — the specific scope entries needed under the elevated tier
   - `context_summary` — plain-English description of why the upgrade is needed

### 4.2 Upgrade Approval Flow

1. The escalation is delivered to the human via the configured channel
2. Human approves with a scope and duration constraint, or denies
3. On approval, the AEP-aware receiving system or enforcement layer generates a `tier_upgrade_proof` object embedded within a CTEF `claim_type: "authority"` wrapper:

```json
{
  "claim_type": "authority",
  "claim_subtype": "tier_upgrade",
  "tier_upgrade_proof": {
    "from_tier": "T1",
    "to_tier": "T2",
    "intent_code": "TIER_UPGRADE_REQUEST",
    "requested_action": {
      "facet": "scope.booking.create",
      "limit": 1,
      "actual": null
    },
    "approval_evidence": {
      "verdict_jws": "<JWS from enforcement gateway>",
      "approver_did": "did:web:enforcement.example",
      "evaluated_at": "ISO 8601",
      "policy_ref": "sha256:<jcs-hash-of-evaluated-policy> (optional)"
    },
    "validity": {
      "valid_until": "ISO 8601",
      "scope_boundary": "session:<session-id>",
      "use_count_max": 1
    }
  }
}
```

**Field notes:**
- `claim_subtype: "tier_upgrade"` — distinguishes upgrade events from steady-state authority claims; verifiers route on subtype without re-parsing the full envelope
- `intent_code` — carries the AEP §2.3 intent string verbatim; no translation table required between AEP and CTEF implementations
- `requested_action.facet/limit/actual` — reuses the CTEF constraint_evaluation shape; the enforcement gateway evaluates the facet+limit pair and the upgrade verdict carries it for downstream audit
- `approval_evidence.verdict_jws` — the enforcement gateway's JWS-signed verdict verbatim; downstream verifiers verify the inner signature without re-escalating
- `approval_evidence.policy_ref` — optional; carries the JCS-hash of the policy version evaluated against. Recommended for compliance use cases where policy may change between upgrade grant and downstream verification
- `validity.scope_boundary + use_count_max` — primary replay-prevention anchors; explicit use_count_max enables single-use upgrade proofs for high-sensitivity actions

4. The proof is returned to the requesting agent

### 4.3 Using an Upgrade Proof

The requesting agent embeds `tier_upgrade_proof` in all subsequent envelopes within the approved scope. Receiving systems MUST:

1. Verify the JWS signature on `tier_upgrade_proof.approval_evidence.verdict_jws`
2. Confirm `validity.valid_until` has not passed
3. Confirm `validity.scope_boundary` matches the current session
4. Confirm `validity.use_count_max` has not been exceeded
5. Confirm `to_tier` meets or exceeds the action's `tier_required`
6. Process or reject accordingly

### 4.4 Upgrade Expiry and Limits

- Upgrade proofs are **session-scoped by default** — they expire at `expires_at` or session boundary, whichever is first
- Upgrade proofs are **scope-limited** — they do not grant general tier elevation; only the declared `scope` entries are covered
- **Permanent tier upgrades** require modifying the `trusted_orgs` entry in the capability manifest (bilateral, human-authored, out-of-band)
- A revoked upgrade proof renders all envelopes carrying it invalid immediately upon revocation propagation

### 4.5 CTEF Compatibility

The `tier_upgrade_proof` schema is aligned with CTEF v0.4 (in-flight). Key alignment points:

- `claim_type: "authority"` / `claim_subtype: "tier_upgrade"` is the canonical CTEF slot for AEP tier-transition evidence
- `requested_action.facet/limit/actual` reuses the CTEF constraint_evaluation shape from ArkForge Trust Layer (enforcement gateway)
- `approval_evidence.approver_did` resolves against the enforcement gateway's `.well-known/jwks.json` endpoint
- `approval_evidence.policy_ref` (optional) pins the policy version evaluated, enabling audit continuity when org policy changes post-grant

AEP-aware implementations using CTEF-compliant enforcement gateways SHOULD structure `tier_upgrade_proof` within the CTEF `authority` claim slot to enable cross-system harness validation. AEP implementations not using CTEF MAY use the `tier_upgrade_proof` object standalone within the AEP envelope.

---

## 5. Trust Tiers

### T0 — Isolated

- No cross-agent communication permitted
- All external requests rejected or escalated immediately
- Suitable for: systems handling confidential or regulated data

### T1 — Verified

- Agent-to-agent communication permitted after capability manifest verification
- Every action notifies a human
- Human can veto before execution (configurable timeout)
- Suitable for: early-stage deployments, sensitive domains

### T2 — Trusted

- Auto-process within declared policy
- Human receives summary digest (not per-action notification)
- Exceptions escalate immediately
- Suitable for: established agent relationships with clear scope boundaries

### T3 — Delegated

- Full agent authority within declared scope
- Audit log only — no human notification unless exception occurs
- Requires explicit bilateral T3 trust declaration in both manifests
- Suitable for: high-volume, well-understood, low-risk workflows (e.g. automated reporting)

---

## 5. Email Binding

### 5.1 Headers

AEP email headers are prefixed `X-AEP-`:

```
X-AEP-Version: 0.1
X-AEP-Agent-ID: bookings-agent@company.com
X-AEP-Scope: booking:read,booking:modify
X-AEP-Intent: BOOKING_INQUIRY
X-AEP-Trust-Tier: T2
X-AEP-Policy-Ref: https://company.com/.well-known/aep-manifest.json
X-AEP-Requires-Human: false
X-AEP-Response-Expected: DATA
X-AEP-Correlation-ID: 550e8400-e29b-41d4-a716-446655440000
X-AEP-Expiry: 2026-04-27T00:00:00Z
X-AEP-Signature: <base64url-encoded Ed25519 signature>
```

### 5.2 Processing Flow

```
Inbound email arrives
        │
        ▼
X-AEP-Version header present?
    No  → Route to human inbox (standard behaviour)
    Yes ↓
        │
        ▼
Fetch Policy-Ref manifest
        │
        ▼
Verify X-AEP-Signature
    Fail → Reject / quarantine
    Pass ↓
        │
        ▼
Check scope against own policy
    Outside policy → Escalate to human with context
    Within policy  ↓
        │
        ▼
Check Trust-Tier
    T0/T1 → Notify human, await approval
    T2/T3 → Auto-process
        │
        ▼
Route to receiving agent
        │
        ▼
Log action + audit record
```

### 5.3 Compatibility

Non-AEP email servers ignore `X-AEP-*` headers entirely. Standard email delivery, rendering, and filtering are unaffected. AEP adoption is opt-in and non-breaking.

---

## 6. Calendar Binding

### 6.1 iCal Extensions

AEP extends iCalendar (RFC 5545) via `X-AEP-*` properties:

```
BEGIN:VCALENDAR
VERSION:2.0
X-AEP-VERSION:0.1
BEGIN:VEVENT
SUMMARY:Quarterly Review
DTSTART:20260501T110000Z
DTEND:20260501T120000Z
X-AEP-REQUESTOR-AGENT:scheduling-agent@partnerco.com
X-AEP-INTENT:MEETING_REQUEST
X-AEP-POLICY-REF:https://partnerco.com/.well-known/aep-manifest.json
X-AEP-TRUST-TIER:T2
X-AEP-MEETING-TYPE:REVIEW
X-AEP-AUTO-APPROVE:TRUE
END:VEVENT
END:VCALENDAR
```

### 6.2 Scheduling Policy Check

Before accepting or rejecting, the receiving system evaluates its Scheduling Policy:

1. Is `X-AEP-REQUESTOR-AGENT` listed as a trusted agent in the manifest?
2. Does the proposed time satisfy `no_meetings_before` and `no_meetings_after`?
3. Does it conflict with existing events (including buffer time)?
4. Does it exceed `max_external_meetings_per_week`?
5. Is `X-AEP-AUTO-APPROVE: TRUE` declared by the requestor?
6. Does the receiving policy permit auto-approval from this trust tier?

All checks pass → Auto-accept, inject prep time, notify human by summary.  
Any check fails → Escalate to human with pre-populated decision card.

### 6.3 Scheduling Policy Schema

```json
{
  "scheduling_policy": {
    "no_meetings_before": "HH:MM (local time)",
    "no_meetings_after": "HH:MM (local time)",
    "timezone": "IANA timezone string",
    "max_external_meetings_per_week": "integer",
    "max_meetings_per_day": "integer",
    "buffer_before_minutes": "integer",
    "buffer_after_minutes": "integer",
    "auto_approve_from_trusted_agents": "boolean",
    "auto_approve_min_trust_tier": "T0 | T1 | T2 | T3",
    "prep_time_by_type": {
      "REVIEW": "integer (minutes)",
      "EXTERNAL": "integer (minutes)",
      "INTERNAL": "integer (minutes)"
    },
    "blackout_dates": ["YYYY-MM-DD"]
  }
}
```

---

## 7. Escalation Protocol

### 7.1 Escalation Record

When an agent hits a policy boundary, it generates an escalation record:

```json
{
  "escalation_id": "uuid",
  "triggered_at": "ISO 8601",
  "triggering_envelope": { /* Agentic Envelope */ },
  "policy_rule_violated": "string",
  "escalation_contact": "email",
  "deadline": "ISO 8601",
  "context_summary": "string (agent-generated plain English summary)",
  "options": [
    {
      "option_id": "approve",
      "label": "Approve",
      "description": "Allow the agent to proceed with this action"
    },
    {
      "option_id": "deny",
      "label": "Deny",
      "description": "Reject the request and notify the sender"
    },
    {
      "option_id": "modify",
      "label": "Modify and Approve",
      "description": "Human modifies parameters before approving"
    }
  ]
}
```

### 7.2 Escalation Delivery

Escalations should be delivered via the human's preferred channel with:
- Plain-English summary of what the agent is requesting
- The policy rule that triggered escalation
- Clear approve/deny/modify options
- Deadline for response
- Full audit context on request

### 7.3 Timeout Handling

If the human does not respond within `escalation_timeout_minutes`:
- Default action: **deny** (conservative)
- Agents may declare `timeout_default: approve` for low-risk workflows (requires explicit policy declaration)
- Timeout event is logged in the audit trail

---

## 8. Signing & Verification

### 8.1 Key Publication

Each agent publishes its Ed25519 public key in the capability manifest:

```json
{
  "agents": [
    {
      "agent_id": "bookings-agent@company.com",
      "public_key": "base64url-encoded Ed25519 public key",
      "key_id": "key-2026-04"
    }
  ]
}
```

### 8.2 Canonical Envelope for Signing

The signed payload is the JSON-serialised envelope with all fields present, keys alphabetically sorted, no whitespace:

```
{"aep_version":"0.1","agent_id":"bookings-agent@company.com","expiry":"...","intent":"BOOKING_INQUIRY",...}
```

### 8.3 Verification Steps

1. Fetch capability manifest from `policy_ref` URL
2. Find public key matching `agent_id`
3. Reconstruct canonical envelope (exclude `signature` field)
4. Verify Ed25519 signature against canonical envelope
5. Check manifest `manifest_updated` timestamp — reject if manifest is older than `manifest_refresh_minutes`
6. Verify `agent_id` is listed in the manifest with the declared scopes

---

## 9. Audit Log

Every AEP exchange produces a log entry:

| Field | Description |
|---|---|
| `log_id` | Unique log entry ID |
| `timestamp` | ISO 8601 |
| `direction` | `INBOUND` / `OUTBOUND` |
| `agent_id` | Sender agent |
| `intent` | Envelope intent code |
| `scope` | Declared scope |
| `trust_tier` | Trust tier used |
| `action_taken` | `AUTO_PROCESSED` / `ESCALATED` / `REJECTED` |
| `human_approved_by` | Email if human approval was involved |
| `policy_rule` | Rule invoked |
| `correlation_id` | Correlation to source envelope |
| `outcome` | `SUCCESS` / `FAILURE` / `TIMEOUT` |

---

## 10. Relationship to Existing Standards

| Standard | Relationship |
|---|---|
| MCP (Model Context Protocol) | AEP operates at the communication layer above MCP. MCP handles tool calls; AEP handles inter-organisational message routing and policy. |
| A2A (Google Agent-to-Agent) | A2A focuses on API-level agent calls. AEP focuses on upgrading existing human communication surfaces. Complementary. |
| CTEF (Composable Trust Evidence Format) | AEP trust tiers map to the CTEF `authority` claim_type. The `tier_upgrade_proof` field (§4) is designed to be embeddable as a structural sub-field within a CTEF authority-layer attestation. AEP's `aep-manifest.json` serves an upstream-of-handshake function — declaring bilateral trust before any wire exchange — analogous to a pre-declared trust registry. CTEF provides the cryptographic evidence substrate; AEP provides the policy and routing layer. Complementary and composable. |
| OAuth 2.0 | AEP's capability manifest borrows scope concepts from OAuth. AEP is not an auth protocol — it is a communication policy protocol. |
| DKIM/SPF | AEP signing is analogous to DKIM for email authentication. AEP adds a semantic layer on top. |
| OpenID Connect | AEP agent identity is compatible with OIDC agent identity proposals. |

---

## 11. Open Questions (RFC)

1. Should agent identity use email-format IDs or a dedicated URN scheme?
2. Should capability manifests support WebFinger (RFC 7033) for discovery?
3. How should scope revocation propagate in real time (vs. manifest refresh cycle)?
4. Should T3 trust require a signed bilateral agreement rather than just manifest declarations?
5. How should AEP interact with existing email security standards (DMARC, BIMI)?
6. Should escalation records be standardised for UI rendering (card schema)?
7. Should `tier_upgrade_proof` support a short-lived bearer token model to avoid embedding large JWS objects in every envelope?

---

## 11. License

Creative Commons Attribution 4.0 International (CC BY 4.0)  
You are free to use, share, and adapt this specification with attribution.

**Author:** Peter Myers — https://github.com/pmyers-abundance
