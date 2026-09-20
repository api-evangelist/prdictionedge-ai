---
name: Verify an AUX certification and apply an acceptance policy
description: For a downstream agent, auditor, insurer or workflow engine that receives an AUX certification from someone else — verify its signature and commitments, then decide ACCEPT / REJECT against your own policy or your organisation's domain-published policy, without executing any action.
api: openapi/prdictionedge-ai-openapi.yml
base_url: https://api.aux.prdictionedge.ai
operations: [verifyAuxCertification, getAuxCertificationPolicyContract, checkAuxCertificationPolicy, getAuxDomainCertificationPolicyContract, checkAuxCertificationAgainstDomainPolicy]
auth: none
generated: '2026-09-19'
method: generated
source: https://api.aux.prdictionedge.ai/v1/certifications/policy-check
---

# Verify an AUX certification and apply an acceptance policy

Trust the evidence decision, not the messenger. A certification pasted into a message or a screenshot proves nothing until its signature and commitments are checked.

## Steps
1. **Verify integrity** — `POST /v1/certifications/verify` (`verifyAuxCertification`) with `{"certification": <the object you received>}`. A `400` means the object is malformed; an INVALID result means do not proceed. Offline alternative: verify the ES256 signature against the JWKS at `/.well-known/jwks.json` and compare `kid`.
2. **Read the policy contract** — `GET /v1/certifications/policy-check` (`getAuxCertificationPolicyContract`) to see the policy shape and the outcome vocabulary (`ACCEPT`, `REJECT`, `INVALID_RECEIPT`).
3. **Apply YOUR policy** — `POST /v1/certifications/policy-check` (`checkAuxCertificationPolicy`):
   ```json
   {"certification": {...},
    "policy": {"policy_id": "ap-vendor-onboarding-v1",
               "allowed_profiles": [{"profile_id": "counterparty_identity_pre_action", "versions": ["0.1.0"]}],
               "max_age_seconds": 86400,
               "required_requirement_ids": ["sanctions_identity", "legal_entity_status", "domain_identity"],
               "expected_proposal_hash": "<64-hex>"}}
   ```
   Pin `allowed_profiles[].versions` explicitly — profiles are independently versioned. `max_age_seconds` is 1..604800. The response binds a SHA-256 hash of the policy you sent, so log it with the outcome.
4. **Or apply your ORGANISATION's published policy** — `GET /v1/certifications/domain-policy-check` (`getAuxDomainCertificationPolicyContract`) for the contract, then `POST /v1/certifications/domain-policy-check` (`checkAuxCertificationAgainstDomainPolicy`) with `{"certification": {...}, "consumer_domain": "yourcompany.example"}`. AUX resolves the policy from that HTTPS domain itself, so a caller cannot substitute a weaker policy. `422` means the domain has not published a valid AUX policy; `503` means the policy source was unreachable — no decision was made.
5. **Act only on ACCEPT**, under the owner's authority. `REJECT` and `INVALID_RECEIPT` are terminal for this certification; request a fresh one that satisfies the policy rather than loosening the policy.

## Rules
- Nothing here executes an action; AUX returns a decision and a hash-bound record only.
- A certification's status must be `CERTIFIED`; a policy check on any other state cannot ACCEPT.
- Do not infer from an ACCEPT that AUX performed or approved the downstream transaction.

## Related
errors/prdictionedge-ai-problem-types.yml (policy_check_outcomes) · data-model/prdictionedge-ai-data-model.yml (ConsumerPolicy) · lifecycle/prdictionedge-ai-lifecycle.yml (profile versions)
