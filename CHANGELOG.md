# Changelog

## [0.1.1-draft] — 2026-05-01 (updated)

### Added — Trust Tier Upgrade Protocol (§4)

- New intent code `TIER_UPGRADE_REQUEST` — distinct from `APPROVAL_REQUEST`, makes tier-crossing machine-readable at the routing layer
- New optional envelope field `tier_required` — minimum trust tier required to process a request; allows receiving agents to evaluate tier requirements without fetching the sender's manifest
- New optional envelope field `tier_upgrade_proof` — JWS-signed tier upgrade verdict, time-bounded and scope-limited
- New §4 Trust Tier Upgrade Protocol — covers upgrade trigger, approval flow, proof usage, expiry rules, and CTEF compatibility
- CTEF compatibility note: `tier_upgrade_proof` is designed to be embeddable within a CTEF `claim_type: "authority"` attestation slot

### Added — Relationship to Existing Standards (§10)

- Formalised §10 with relationship table covering MCP, A2A, CTEF, OAuth 2.0, DKIM/SPF, OpenID Connect
- CTEF relationship documented in detail: complementary and composable, not overlapping

### Added — Open Questions (§11)

- Added Q7: Should `tier_upgrade_proof` support a short-lived bearer token model to avoid large JWS objects in every envelope?

### Changed

- §10 Open Questions renumbered to §11 to accommodate new §10 (Relationship to Existing Standards)
- §4 Trust Tiers renumbered to §5 to accommodate new §4 (Trust Tier Upgrade Protocol)

### Updated — `tier_upgrade_proof` schema aligned with CTEF v0.4 + ArkForge enforcement shape

- `tier_upgrade_proof` schema updated to CTEF-aligned structure: `claim_type: "authority"`, `claim_subtype: "tier_upgrade"`
- `requested_action` block added: `facet/limit/actual` reuses ArkForge constraint_evaluation shape
- `approval_evidence` block formalised: `verdict_jws`, `approver_did`, `evaluated_at`, optional `policy_ref` (JCS-hash of evaluated policy version — audit trail for compliance use cases)
- `validity` block formalised: `valid_until`, `scope_boundary`, `use_count_max` (replay-prevention)
- §2.2 optional field description updated
- §4.3 verification steps updated to reference new field paths
- §4.5 CTEF compatibility note expanded with alignment details

_Triggered by: feedback from @arkforge and @kenneives in a2aproject/A2A Discussion #1734_
_Schema three-way alignment: AEP ↔ ArkForge Trust Layer ↔ CTEF v0.4_

---

## [0.1.0-draft] — 2026-04-26

### Initial public draft

- First public release of AEP specification
- Agentic Envelope schema defined with required and optional fields
- 13 intent codes across Commerce, Calendar, General, Escalation, and Protocol categories
- Capability Manifest schema with agent declarations, trusted org declarations, and organisational policy
- Four Trust Tiers defined: T0 Isolated, T1 Verified, T2 Trusted, T3 Delegated
- Email binding via X-AEP-* SMTP headers
- Calendar binding via X-AEP-* iCal properties with Scheduling Policy schema
- Escalation Protocol with structured escalation records and timeout handling
- Ed25519 signing and verification model
- Audit log schema
- CC BY 4.0 license
- Open RFC process initiated — community feedback welcome
