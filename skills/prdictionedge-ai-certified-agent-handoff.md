---
name: Hand a certification to another agent and consume it once
description: Create a signed, recipient-policy-gated, non-executing handoff bound to a certification, a sender, a recipient, an intended action, a nonce and an expiry; let the recipient verify it, authenticate itself with its domain-published key, and consume it exactly once with a signed receipt.
api: openapi/prdictionedge-ai-openapi.yml
base_url: https://api.aux.prdictionedge.ai
operations: [createAuxCertificationHandoff, verifyAuxCertificationHandoff, getAuxHandoffConsumptionContract, consumeAuxCertificationHandoff, verifyAuxHandoffConsumptionReceipt]
auth: recipient-domain ES256 JWS on consume only
generated: '2026-09-19'
method: generated
source: https://api.aux.prdictionedge.ai/v1/certification-handoffs/consume
---

# Hand a certification to another agent and consume it once

Two parties: the **sender** agent that holds a `CERTIFIED` certification, and the **recipient** agent at a domain that has published an AUX acceptance policy and an AUX handoff-consumer key. AUX stays non-executing throughout; every receipt carries `action_executed: false`.

## Sender
1. **Create the handoff** — `POST /v1/certification-handoffs` (`createAuxCertificationHandoff`):
   ```json
   {"certification": {...},
    "recipient_domain": "recipient.example",
    "handoff": {"handoff_id": "<unique>", "sender_agent_id": "<you>", "recipient_agent_id": "<them>",
                "intended_action": "onboard_counterparty", "nonce": "<random>",
                "expires_at": "<ISO-8601, in the future, <= 24 h from now>"}}
   ```
   AUX first resolves the recipient domain's published policy and evaluates the certification against it. `409` = the recipient's policy rejected it (no receipt issued); `422` = invalid certification or the recipient has not published a valid policy; `503` = policy source or signing unavailable, retry later. All six `handoff` fields are required, max 256 characters each.
2. **Transmit the signed handoff** to the recipient over your own channel. There is no revoke operation — the handoff simply expires at `expires_at`, so keep windows short.

## Recipient
3. **Verify before trusting** — `POST /v1/certification-handoffs/verify` (`verifyAuxCertificationHandoff`) with `{"handoff": {...}, "expected": {"sender_agent_id": "...", "recipient_agent_id": "<you>", "intended_action": "...", "nonce": "..."}}`. Any mismatch on the expected bindings is a reason to stop.
4. **Read the consumption contract** — `GET /v1/certification-handoffs/consume` (`getAuxHandoffConsumptionContract`). Your domain must serve `https://<recipient_domain>/.well-known/aux-handoff-consumer.json` with the ES256 key you will sign with.
5. **Sign a consumer assertion** — a JWS Compact Serialization, `alg` ES256, `typ` `AUX-HANDOFF-CONSUMER+JSON`, lifetime <= 300 s, whose payload binds `schema, issuer, subject, audience, recipient_domain, recipient_agent_id, handoff_id, handoff_sha256, intended_action, jti, issued_at, expires_at`. Compute `handoff_sha256` over the exact handoff you verified.
6. **Consume exactly once** — `POST /v1/certification-handoffs/consume` (`consumeAuxCertificationHandoff`) with `{"handoff": {...}, "consumer_assertion": {"recipient_domain": "<you>", "jws": "<compact JWS>"}}`. Outcomes: `CONSUMED_ONCE` (success), `ALREADY_CONSUMED` (HTTP 409 — someone, possibly you, already consumed it; **do not retry**), `INVALID_HANDOFF` (422), `RECIPIENT_AUTHENTICATION_FAILED` (401 — re-check the key at your well-known path and the `handoff_sha256`). A `503` with `retryable: true` means AUX's own store/HMAC/signing was unavailable and nothing was consumed.
7. **Keep and verify the consumption receipt** — `POST /v1/certification-handoffs/consume/verify` (`verifyAuxHandoffConsumptionReceipt`) with `{"consumption_receipt": {...}}`. This is the audit record that a domain-authenticated principal presented this exact handoff once.

## Rules
- Consumption is irreversible by design; the replay protection is the point. Treat 409 as final.
- AUX stores only hashed handoff/nonce commitments, a pseudonymous principal commitment and the receipt until cleanup — do not expect to list or look up handoffs later.
- The intended action still has to be executed (or not) by the recipient under its owner's authority.

## Related
authentication/prdictionedge-ai-authentication.yml (recipient-domain-jws) · errors/prdictionedge-ai-problem-types.yml (consumption_outcomes) · conventions/prdictionedge-ai-conventions.yml (reversibility)
