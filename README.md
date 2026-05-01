# AEP — Agentic Exchange Protocol

**Version:** 0.1.1-draft  
**Author:** Peter Myers  
**Status:** Public Draft — RFC  
**License:** Creative Commons Attribution 4.0 (CC BY 4.0)

---

## What Is This?

Email was designed for humans. Calendars were designed for humans. Messaging was designed for humans.

AI agents are now operating through all of these surfaces — and they are doing it badly, because the interfaces give them nothing to work with. No machine-readable intent. No policy layer. No trust model. No way for one agent to signal to another that a message is safe to process automatically.

**AEP (Agentic Exchange Protocol)** is a proposed open standard that upgrades existing communication interfaces — email, calendar, and messaging — so AI agents can operate effectively within them.

It does not replace email. It does not replace calendars. It adds a thin, structured layer on top of existing protocols that agents can read, act on, and respect — while humans remain in control.

---

## The Problem

Current AI agent deployments face a fundamental interface mismatch:

- **Email** has no way to signal "this was sent by an agent" or "this is safe to auto-process"
- **Calendar** has availability, not policy — agents cannot know that no meetings are allowed before 11am
- **Messaging** has no agent identity layer, no inline approval flow, no policy-aware routing
- **Cross-company communication** between agents has no trust model, no capability declaration, no non-repudiation

The result: agents either require constant human approval (defeating the purpose) or operate with no guardrails (dangerous).

AEP proposes to fix this at the protocol layer.

---

## Core Concepts

### 1. The Agentic Envelope

Every communication between agents carries a metadata wrapper — the **Agentic Envelope** — that declares:

- Who sent it (agent identity + company)
- What it is allowed to share (capability scope)
- What it intends to do (intent declaration)
- What response it expects (expected response type)
- What policy it is operating under (policy reference)

For email, this lives in extended headers. For calendar, in iCal extensions. For messaging, in a structured metadata block.

### 2. The Capability Manifest

Each organisation publishes a **Capability Manifest** — a machine-readable declaration of:

- What their agents are allowed to share with external parties
- Which external agents or organisations they trust
- Under what conditions automatic processing is permitted
- What escalates to a human

Think OAuth scopes, but for inter-organisational agent communication.

### 3. Trust Tiers

AEP defines four trust tiers:

| Tier | Name | Behaviour |
|---|---|---|
| T0 | Isolated | No cross-agent communication permitted |
| T1 | Verified | Agent-to-agent allowed after capability check; human notified |
| T2 | Trusted | Auto-process within policy; human receives summary |
| T3 | Delegated | Full agent authority within declared scope; audit log only |

### 4. The Escalation Protocol

When an agent encounters a request outside its policy scope, it does not silently fail or return an error. It packages the request — with full context — and routes it to the appropriate human with a structured decision request.

The human approves or denies. The result is returned to the originating agent. The exchange is logged.

---

## Surface Bindings

### Email (SMTP/MIME)

AEP extends email via new headers:

```
X-AEP-Version: 0.1
X-AEP-Agent-ID: agent@company.com
X-AEP-Scope: booking:read,booking:modify
X-AEP-Intent: BOOKING_INQUIRY
X-AEP-Trust-Tier: T2
X-AEP-Policy-Ref: https://company.com/.well-known/aep-manifest.json
X-AEP-Signature: <ed25519 signature of envelope>
```

An AEP-aware receiving server:
1. Detects `X-AEP-Version` header
2. Fetches the sender's capability manifest
3. Checks the declared scope against its own policy
4. Routes to the receiving agent (not the human inbox) if within policy
5. Escalates to human if outside policy

Non-AEP servers ignore these headers. Standard email delivery is unaffected.

### Calendar (iCalendar)

AEP extends iCal via `X-AEP-*` properties on `VCALENDAR` and `VEVENT` components:

```
X-AEP-REQUESTOR-AGENT: agent-id@company.com
X-AEP-INTENT: MEETING_REQUEST
X-AEP-POLICY-REF: https://company.com/.well-known/aep-manifest.json
X-AEP-AUTO-APPROVE: TRUE
```

The receiving calendar checks its **Scheduling Policy** before acting:

- Is the requestor a trusted agent?
- Does the proposed time fall within allowed windows?
- Does it conflict with existing blocks?
- Does it exceed the meeting budget for this period?

If all checks pass and auto-approve is declared: accepted automatically, human notified by summary.

### Messaging (Generic)

AEP defines a structured metadata block deliverable as a message attachment or system metadata field:

```json
{
  "aep_version": "0.1",
  "agent_id": "agent@company.com",
  "scope": ["booking:read"],
  "intent": "STATUS_UPDATE",
  "trust_tier": "T2",
  "policy_ref": "https://company.com/.well-known/aep-manifest.json",
  "requires_human_action": false,
  "escalation_deadline": null
}
```

---

## The Capability Manifest

Each organisation hosts a manifest at a well-known URL:

`https://yourdomain.com/.well-known/aep-manifest.json`

```json
{
  "aep_version": "0.1",
  "org": "ACME Corp",
  "agents": [
    {
      "agent_id": "bookings-agent@acme.com",
      "display_name": "ACME Bookings Agent",
      "scopes": ["booking:read", "booking:create", "calendar:read"],
      "trust_offered": "T2",
      "contact_human": "bookings@acme.com"
    }
  ],
  "trusted_orgs": [
    {
      "domain": "partnerco.com",
      "trust_tier": "T2",
      "allowed_scopes": ["booking:read", "availability:read"]
    }
  ],
  "policy": {
    "auto_approve_below_value": 50000,
    "escalation_contact": "ops@acme.com",
    "audit_log_required": true
  }
}
```

---

## Scheduling Policy (Calendar Extension)

Organisations declare scheduling rules that agents must respect:

```json
{
  "scheduling_policy": {
    "no_meetings_before": "11:00",
    "no_meetings_after": "14:00",
    "max_external_meetings_per_week": 3,
    "buffer_before_minutes": 15,
    "buffer_after_minutes": 30,
    "auto_approve_from_trusted_agents": true,
    "prep_time_by_type": {
      "REVIEW": 30,
      "EXTERNAL": 15,
      "INTERNAL": 0
    }
  }
}
```

---

## Security Model

### Signing
All AEP envelopes are signed using Ed25519. The sender's public key is published in their capability manifest. Recipients verify the signature before processing.

### Non-Repudiation
Every agent action is logged with:
- Envelope hash
- Timestamp
- Action taken
- Policy rule invoked
- Human approval reference (if required)

### Scope Containment
An agent cannot grant scopes it does not itself hold. Scope delegation chains are validated end-to-end.

### Human Override
Any human can revoke a T2/T3 agent's trust tier at any time. Revocation propagates to all parties within the manifest refresh window (default: 1 hour).

---

## Relationship to Existing Standards

| Standard | Relationship |
|---|---|
| MCP (Model Context Protocol) | AEP operates at the communication layer above MCP. MCP handles tool calls; AEP handles inter-organisational message routing and policy. |
| A2A (Google Agent-to-Agent) | A2A focuses on API-level agent calls. AEP focuses on upgrading existing human communication surfaces. Complementary. |
| OAuth 2.0 | AEP's capability manifest borrows scope concepts from OAuth. AEP is not an auth protocol — it is a communication policy protocol. |
| DKIM/SPF | AEP signing is analogous to DKIM for email authentication. AEP adds semantic layer on top. |
| OpenID Connect | AEP agent identity is compatible with OIDC agent identity proposals. |

---

## Repository Structure

```
aep/
├── README.md               ← This file
├── SPEC.md                 ← Full specification
├── CHANGELOG.md
├── schema/
│   ├── aep-envelope.json           ← JSON Schema: Agentic Envelope
│   ├── aep-manifest.json           ← JSON Schema: Capability Manifest
│   ├── aep-scheduling-policy.json  ← JSON Schema: Calendar Scheduling Policy
│   └── aep-escalation.json         ← JSON Schema: Escalation Record
└── examples/
    ├── email-headers.txt           ← Example SMTP headers
    ├── manifest-example.json       ← Example capability manifest
    ├── calendar-event.ics          ← Example AEP-extended iCal event
    └── escalation-record.json      ← Example escalation log entry
```

---

## Goal

AEP is designed to become an implemented standard.

That means email infrastructure providers (Postfix, Sendgrid, Fastmail, Google, Microsoft) shipping native AEP header handling. Calendar platforms evaluating scheduling policy before routing to a human inbox. Agent frameworks (LangChain, CrewAI, AutoGen) generating compliant envelopes by default.

The spec succeeds when a company can publish a capability manifest, configure their scheduling policy once, and let their agents operate across organisational boundaries — without bespoke integrations, without constant human approval, and without sacrificing auditability or control.

That is not a distant goal. The agent infrastructure to support it exists today. What is missing is the protocol layer that makes it interoperable.

This is that layer.

---

## Status

This is a **public draft**. AEP is published to invite discussion, adoption, and co-authorship.

**We are actively seeking input from:**
- Email infrastructure teams (Postfix, Sendgrid, Fastmail, etc.)
- Calendar platform engineers (Google Calendar, Microsoft Exchange, Fastmail)
- AI agent framework developers (LangChain, CrewAI, AutoGen, etc.)
- Enterprise IT architects deploying agentic AI
- Security researchers working on agent identity

---

## Contributing

Open an issue. Start a discussion. Submit a PR.

This spec will be shaped by the people who build agent infrastructure. If you are building in this space, your input belongs here.

---

## Author

**Peter Myers**  
GitHub: [@pmyers-abundance](https://github.com/pmyers-abundance)  
Related project: [GTFS-RTP](https://github.com/pmyers-abundance/gtfs-rtp)

---

## License

Creative Commons Attribution 4.0 International (CC BY 4.0)  
You are free to use, share, and adapt this specification with attribution.
