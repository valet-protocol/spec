# Proposal: Service Receipts

**Status:** Draft
**Date:** February 15, 2026
**Extends:** VALET Specification v1.0, Section 7.5
**Target:** v1.1

---

## Motivation

VALET v1.0 activity tracking is agent-reported. Agents log their own HTTP interactions and present summaries to principals during delegation renewal. This establishes an accountability norm, but principals have no way to distinguish self-reported records from independently verified ones.

Service Receipts allow services to return signed, immutable proof of what happened during a request. This gives principals a trust gradient: agent-reported records as the baseline, service-verified receipts as the gold standard.

### Design Constraints

- **Opt-in for services.** Services MUST NOT be required to issue receipts. VALET adoption depends on agents, not services.
- **Zero new infrastructure.** Receipts use the same storage layer (IPFS, HTTPS) and record format as agent-reported activity.
- **Incremental upgrade.** Agents that don't collect receipts and principals that don't check for them are unaffected.

---

## Specification

### 1. Receipt Format

A service receipt is an activity record with `source` set to `"service"` and an additional `service_signature` field:

```json
{
  "agent_id": "agent:ed25519:5FHneW46xGXgs5mUiveU4sbTyGBzmstUspZC92UhjJM694ty",
  "timestamp": "2026-02-14T14:23:00Z",
  "service": "mail.example.com",
  "method": "POST",
  "path": "/api/send",
  "status": 200,
  "source": "service",
  "service_signature": "base64...",
  "service_key": "ed25519:SERVICE_PUBLIC_KEY"
}
```

**Field Definitions:**

Fields inherited from v1.0 Activity Record (Section 7.2):
- **agent_id**, **timestamp**, **service**, **method**, **path**, **status**: Same as v1.0.

New fields:
- **source** (string, required): MUST be `"service"` for service-issued receipts. Agent-reported records use `"agent"`.
- **service_signature** (base64 string, required for service receipts): Service's signature over the UTF-8 encoded concatenation: `agent_id || timestamp || service || method || path || status` (direct concatenation, no delimiters).
- **service_key** (string, required for service receipts): The service's public key identifier, using the same `{key_type}:{base58_encoded_public_key}` format as principal IDs.

### 2. Response Header

After processing a VALET-authenticated request, services MAY return a receipt via an HTTP response header:

```
VALET-Receipt: <cid_or_url>
```

The value is a URL or CID pointing to the stored receipt. The storage location follows the same conventions as delegation storage (Section 4.3):

```
VALET-Receipt: ipfs://QmReceipt123...
VALET-Receipt: https://receipts.example.com/abc123
```

Services MAY also store the receipt directly on the same IPFS network used by the agent's delegation, reducing the number of storage systems principals need to query.

### 3. Agent Behavior

When an agent receives a response containing a `VALET-Receipt` header:

1. Agent SHOULD fetch the receipt from the provided URL/CID.
2. Agent SHOULD store the receipt alongside its own activity records.
3. Agent MUST NOT modify or omit service receipts when presenting activity to the principal.
4. Agent MAY verify the `service_signature` before storing, but MUST present unverifiable receipts to the principal with a warning rather than silently discarding them.

### 4. Principal Presentation

During delegation renewal, activity summaries SHOULD distinguish record sources:

```
Activity Summary (Feb 14 08:00 - Feb 15 08:00):

Total Requests: 1,523
Success Rate: 98%

By Source:
  - Agent-reported: 1,200
  - Service-verified: 323

By Service:
  - gmail.com: 847 requests (0 errors) [323 service-verified]
  - calendar.google.com: 676 requests (31 errors) [0 service-verified]
```

Principals SHOULD weight service-verified records more heavily than agent-reported records when assessing agent behavior.

### 5. Verification

Principals or auditors verifying a service receipt:

1. Fetch the receipt from the URL/CID.
2. Extract `service_key` from the receipt.
3. Reconstruct the signed message: `agent_id || timestamp || service || method || path || status`.
4. Verify `service_signature` against `service_key`.
5. Optionally verify that `service_key` belongs to the claimed `service` domain (out of scope for this proposal; could use DNS-based key discovery, `.well-known` endpoints, etc.).

### 6. Service Identity

This proposal intentionally does NOT define how to bind a `service_key` to a domain name. Possible approaches (for future work):

- **`.well-known/valet-keys`**: Service publishes its public key at a well-known URL.
- **DNS TXT records**: Service key published as a DNS record.
- **Certificate transparency**: Service keys registered in a public log.

For v1.1, principals MUST manually trust service keys (similar to SSH `known_hosts`). Automated service key discovery is deferred to a future extension.

---

## IANA Considerations

### HTTP Header Registration

**VALET-Receipt**
- Header field name: VALET-Receipt
- Applicable protocol: HTTP
- Status: Standard
- Specification document: this document

---

## Security Considerations

### Receipt Forgery

Without service key verification (Section 6), an agent could fabricate receipts with fake service signatures. Principals SHOULD maintain a trusted set of service keys and flag receipts from unknown keys.

### Selective Omission

An agent could collect receipts but omit unfavorable ones. Services that want stronger guarantees MAY publish receipts independently (e.g., to their own IPFS node) and provide principals with a separate feed. This is out of scope for v1.1.

### Privacy

Service receipts contain the same metadata as agent-reported records (no request/response bodies). However, service-stored receipts are controlled by the service, not the agent. Services SHOULD document their receipt retention policies.

---

## Compatibility

- **Backward compatible with v1.0.** The `source` field and `VALET-Receipt` header are additive. Agents and services that don't implement receipts are unaffected.
- **ActivityRecord extensibility.** The `source` field (`"agent"` | `"service"`) is already implemented in valet-openclaw. Adding `service_signature` and `service_key` fields to service-sourced records is a non-breaking extension.
- **ActivityReporter compatibility.** The existing reporter already groups records by source and displays the breakdown to principals.

---

## Open Questions

1. **Should receipts be batched?** A service handling many requests from an agent could return one receipt per request (noisy) or batch them periodically (more efficient but delayed). Recommendation: per-request for v1.1, batching as a future optimization.

2. **Should the receipt include the request body hash?** Adding a `content_digest` field (per RFC 9530) would let principals verify what the agent actually sent, not just that it made a request. This significantly increases the evidentiary value but also the privacy implications.

3. **Should services charge for receipts?** Issuing and storing receipts has a cost. A future micropayment extension could let services charge agents for receipt issuance. Out of scope for v1.1.
