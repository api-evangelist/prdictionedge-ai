---
name: Run a public business counterparty check
description: Independently verify a business counterparty (live GLEIF legal-entity record + bounded OFAC exact-name screening) before an agent onboards, contracts with or pays it, using AUX's durable certification attempt and receipt verification. No account or API key.
api: openapi/prdictionedge-ai-openapi.yml
base_url: https://api.aux.prdictionedge.ai
operations: [getCertificationProfiles, getCertificationRequirements, evaluateCertificationRequirements, createCertificationAttempt, getCertificationAttempt, verifyAuxCertification]
auth: none
generated: '2026-09-19'
method: generated
source: https://aux.prdictionedge.ai/agents/quickstart
---

# Run a public business counterparty check

AUX is **non-executing**: it tells you whether a bounded evidence contract is satisfied and hands you a signed receipt. It never pays, onboards or approves anything — the owner's policy decides what to do with the outcome.

## Inputs you need from the owner
- Exact legal entity name **as registered in GLEIF** (not a trading name)
- Complete GLEIF **legal address** (lines, city, region, ISO country, postal code) — this is often a registered-agent address, not the headquarters
- LEI if known (optional, strongly preferred)
- Never collect or send passwords, private keys, personal data, raw bank-account numbers or confidential invoices.

## Steps
1. **Discover what AUX can certify** — `GET /v1/certification-profiles` (`getCertificationProfiles`). Choose the narrowest live profile: `counterparty_registry_pre_action` for public GLEIF + OFAC evidence only; `counterparty_identity_pre_action` when the counterparty's domain identity also matters (adds a `proposal.vendor.domain` requirement); `vendor_payment_pre_action` only when you also hold signed private-source attestations (invoice, bank destination, history, agent authority).
2. **Read the requirement contract** — `GET /v1/certification-requirements?profile_id=<profile>` (`getCertificationRequirements`). It lists each requirement's `responsibility` (CALLER / AUX / BOTH) and `required_fields`.
3. **Rehearse without side effects** — `POST /v1/certification-requirements` (`evaluateCertificationRequirements`) with the assembled `CertificationRequest`. This is the dry run: it returns exact gaps and next actions and creates nothing. Fix gaps before step 4.
4. **Submit once, durably** — `POST /v1/certification-attempts` (`createCertificationAttempt`) with:
   ```json
   {"profile_id":"counterparty_registry_pre_action",
    "proposal":{"vendor":{"name":"<exact legal name>","lei":"<optional>",
      "registered_address":{"lines":["..."],"city":"...","region":"...","country":"US","postal_code":"..."}}}}
   ```
   Keep the returned `attempt_id` (`^auxattempt_[a-f0-9]{32}$`) and `status_url`. The attempt is **deterministic** — resubmitting the same request reuses it — so never resubmit to "hurry it up". A `202` means a required source is temporarily down and AUX is retrying for you (backoff 60/300/900/1800/3600 s for up to 24 h). A `422` means REQUIREMENTS_OUTSTANDING before an attempt was created: go back to step 3.
5. **Poll, don't resubmit** — `GET /v1/certification-attempts/{attempt_id}` (`getCertificationAttempt`) until the state is terminal. Terminal results are deleted 24 h later, so capture the result.
6. **Verify the receipt yourself** — `POST /v1/certifications/verify` (`verifyAuxCertification`) with the returned certification. Confirm profile, version, status, time, proposal hash and evidence-set hash. The signature is ES256 against `https://api.aux.prdictionedge.ai/.well-known/jwks.json`, so you may also verify offline.

## Reading the outcome
| State | Meaning | What to do |
|---|---|---|
| `CERTIFIED` | every bounded requirement satisfied by independent evidence | proceed under the owner's policy |
| `REQUIREMENTS_OUTSTANDING` | caller data or verifiable evidence missing (HTTP 422) | supply what the requirement contract names |
| `NOT_CERTIFIABLE` | independently verified evidence is adverse (HTTP 409) | stop and escalate; do not retry |
| `SOURCE_TEMPORARILY_UNAVAILABLE` | a source is down; **no decision made** (HTTP 503) | wait for the durable attempt; never treat as approval |

## Limits to state back to the owner
The OFAC check is exact normalized primary/alias name matching against the official files — not fuzzy, ownership-based or comprehensive sanctions clearance. A no-match is not proof an entity does not exist. A certification is bounded to its named requirements and is not legal advice or permission to execute.

## Related
conventions/prdictionedge-ai-conventions.yml · errors/prdictionedge-ai-problem-types.yml · authentication/prdictionedge-ai-authentication.yml · cli/prdictionedge-ai-cli.yml (the provider's runnable client does exactly these steps)
