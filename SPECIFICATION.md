# VALET Protocol Specification v1.0

**Status:** Draft  
**Date:** February 15, 2026  
**Editors:** Subcent

---

## Abstract

VALET (Verifiable Agent Limited-Expiry Token) is a lightweight, decentralized protocol enabling AI agents to prove delegated authority when accessing services. VALET provides cryptographic verification of agent identity and principal authorization without requiring centralized identity providers or real-time registry lookups.

VALET is built on RFC 9421 (HTTP Message Signatures) and adds a delegation layer for agent authorization.

---

## 1. Introduction

### 1.1 Problem Statement

AI agents need to prove three things when accessing services:
- **Identity**: Who is this agent?
- **Authorization**: Who delegated authority to it?
- **Expiry**: When does this authorization expire?

Existing authentication systems (OAuth, API keys, service accounts) were designed for human users or static credentials and do not address the unique characteristics of autonomous agents operating at scale.

### 1.2 Design Goals

- **Decentralized**: No single point of failure or control
- **Lightweight**: Minimal overhead per request
- **Verifiable**: All delegations cryptographically signed and publicly recorded
- **Standards-based**: Built on RFC 9421 HTTP Message Signatures
- **Human oversight**: Regular renewal enforces principal review

### 1.3 Scope

This specification defines:
- Agent identity representation
- Delegation structure and public recordkeeping
- HTTP headers for agent requests (using RFC 9421)
- Signature construction and verification
- Expiry and renewal model
- Recommended activity tracking format

This specification does NOT define:
- Specific storage implementations (IPFS, blockchain, etc.)
- Permission scopes (future extension)
- Revocation mechanisms (future extension)
- Sub-delegation (future extension)

---

## 2. Dependencies

VALET builds on the following standards:

- **RFC 9421**: HTTP Message Signatures - used for signing HTTP requests
- **RFC 8941**: Structured Field Values for HTTP
- **RFC 3986**: Uniform Resource Identifier (URI): Generic Syntax

Implementations MUST support RFC 9421 for signature generation and verification.

---

## 3. Core Concepts

### 3.1 Agent Identity

Agents are identified by public keys. Agent IDs are derived from the public key using the format:

```
agent:{key_type}:{base58_encoded_public_key}
```

Example:
```
agent:ed25519:5FHneW46xGXgs5mUiveU4sbTyGBzmstUspZC92UhjJM694ty
```

**Supported key types:**
- `ed25519` (REQUIRED)
- `secp256k1` (OPTIONAL)

### 3.2 Principal Identity

Principals are identified by their public keys using the same format as agents. The principal is the human or organization delegating authority to the agent.

The principal_id field contains the principal's public key, enabling cryptographic verification without external lookups.

### 3.3 Delegation

A delegation is a cryptographic statement by a principal authorizing an agent to act on their behalf for a limited time period. Delegations are stored at publicly accessible URLs to enable verification.

### 3.4 Public Record

Delegations MUST be stored at a publicly accessible URL to enable non-repudiation and verification. This prevents agents from fabricating delegations and provides an immutable record of authorization.

---

## 4. Delegation Structure

Delegations MUST contain exactly five fields:

```json
{
  "agent_id": "agent:ed25519:5FHneW46xGXgs5mUiveU4sbTyGBzmstUspZC92UhjJM694ty",
  "principal_id": "ed25519:ABC123...",
  "issued_at": "2026-02-14T08:00:00Z",
  "expires_at": "2026-02-15T08:00:00Z",
  "delegation_signature": "base64..."
}
```

### 4.1 Field Definitions

**agent_id** (string, required)  
The agent's identity as derived from its public key.

**principal_id** (string, required)  
The principal's public key identifier. This field contains the principal's actual public key, enabling services to verify the delegation signature without external lookups.

**issued_at** (ISO 8601 timestamp, required)  
When this delegation was created.

**expires_at** (ISO 8601 timestamp, required)  
When this delegation expires. After this time, the delegation MUST NOT be considered valid.

**delegation_signature** (base64 string, required)  
Principal's signature of the concatenated string: `agent_id || issued_at || expires_at`

The signature MUST be computed over the UTF-8 encoded bytes of the concatenation, in that exact order, with `||` representing direct concatenation with no delimiters.

### 4.2 Delegation Duration

Delegations SHOULD NOT exceed 24 hours (`expires_at - issued_at <= 24 hours`).

Principals MAY create delegations longer than 24 hours at their own risk. Services MAY reject delegations exceeding their acceptable duration limits.

**Recommendation:** Principals should renew delegations every 24 hours to maintain continuous oversight of agent activity.

### 4.3 Storage

Delegations MUST be stored at a publicly accessible URL. The storage mechanism is implementation-defined and MAY include:

- IPFS (Content-addressed storage)
- Blockchain-based systems
- Arweave (Permanent storage)
- Traditional web servers with appropriate availability guarantees

The URL scheme determines the storage type:
- `ipfs://QmYwAPJzv5...` - IPFS storage
- `https://example.com/delegations/123` - HTTPS-accessible storage
- `ar://arweave-tx-id` - Arweave storage

Services MUST be able to fetch delegations via standard HTTP(S) or IPFS gateway URLs.

---

## 5. HTTP Headers

VALET uses RFC 9421 (HTTP Message Signatures) with specific headers and signature parameters.

### 5.1 Required Headers

**VALET-Authorization**  
Base64-encoded delegation JSON.
```
VALET-Authorization: eyJhZ2VudF9pZCI6ImFnZW50OmVkMjU1MTk6NUZIbmVXNDZ4R1hnczVt...
```

**VALET-Agent**  
URL to the publicly accessible delegation record.
```
VALET-Agent: record=https://ipfs.io/ipfs/QmYwAPJzv5CZsnA636s8Bv...
```

Or with IPFS protocol:
```
VALET-Agent: record=ipfs://QmYwAPJzv5CZsnA636s8Bv...
```

**Signature-Input** (from RFC 9421)  
Signature metadata using the label `valet`. MUST include the `valet-authorization` component and version parameter.
```
Signature-Input: valet=("@method" "@path" "valet-authorization");created=1618884479;keyid="agent:ed25519:5FHn...";alg="ed25519";v="1.0"
```

**Signature** (from RFC 9421)  
Base64-encoded signature using the label `valet`.
```
Signature: valet=:MEUCIQDxK7VQX8...:
```

### 5.2 Signature Construction

Agents MUST sign requests according to RFC 9421 with the following requirements:

1. **Signature label**: MUST be `valet`
2. **Required covered components**:
   - `@method` - HTTP method
   - `@path` - Request path
   - `valet-authorization` - The VALET-Authorization header value
3. **Required signature parameters**:
   - `created` - UNIX timestamp when signature was created
   - `keyid` - Agent's public key identifier (the agent_id)
   - `alg` - Signature algorithm (e.g., "ed25519")
   - `v` - Protocol version (e.g., "1.0")

**Example signature base construction** (per RFC 9421):
```
"@method": POST
"@path": /api/send-email
"valet-authorization": eyJhZ2VudF9pZCI6ImFnZW50...
"@signature-params": ("@method" "@path" "valet-authorization");created=1618884479;keyid="agent:ed25519:5FHn...";alg="ed25519";v="1.0"
```

The agent signs this signature base with its private key to produce the signature value.

### 5.3 Optional Components

Agents MAY include additional components in the signature per RFC 9421, such as:
- `@authority` - Request authority/host
- `@query` - Query parameters
- `content-digest` - Hash of request body (see RFC 9530)

Services MAY require specific additional components.

### 5.4 Multiple Signatures

Since RFC 9421 supports multiple signatures, a request MAY contain both VALET signatures and other RFC 9421 signatures:

```
VALET-Authorization: eyJhZ2VudF9pZCI6ImFnZW50...
VALET-Agent: record=ipfs://QmYwAPJzv5CZsnA636s8Bv...
Signature-Input: valet=("@method" "@path" "valet-authorization");created=1618884479;keyid="agent:ed25519:5FHn...";alg="ed25519";v="1.0", othersig=("@method" "@authority");created=1618884480;keyid="other-key"
Signature: valet=:base64_sig1...:, othersig=:base64_sig2...:
```

Services verifying VALET MUST look specifically for the `valet` signature label.

---

## 6. Verification Flow

Services verify VALET requests as follows:

### 6.1 Extract VALET Signature

1. Look for signature with label `valet` in `Signature-Input` and `Signature` headers
2. If not present, request is not VALET-authenticated
3. Extract signature components and parameters per RFC 9421
4. Extract version from `v` parameter

### 6.2 Decode and Fetch Delegation

1. Decode `VALET-Authorization` header to get delegation JSON
2. Extract URL from `VALET-Agent` header (`record=<url>`)
3. Fetch delegation from the public URL
4. Verify fetched delegation matches decoded delegation from header
5. REJECT if they don't match (agent is lying about delegation)

### 6.3 Verify Principal Signature

1. Extract `principal_id` from delegation (this is the principal's public key)
2. Verify `delegation_signature` was created by signing `agent_id || issued_at || expires_at` with the principal's private key
3. REJECT if signature is invalid

### 6.4 Verify Delegation Expiry

1. Get current timestamp
2. Verify `issued_at <= current_time < expires_at`
3. REJECT if delegation has expired

### 6.5 Verify Agent Signature (RFC 9421)

1. Extract agent's public key from `keyid` parameter (this is the agent_id)
2. Reconstruct signature base per RFC 9421 using covered components
3. Verify signature using agent's public key
4. REJECT if signature is invalid

### 6.6 Verify Principal Authorization

Services MUST verify that the principal is authorized to access the service:
1. Check if `principal_id` corresponds to a known user/account (service-specific)
2. Apply service-specific authorization policies
3. REJECT if principal is not authorized

Services define their own authorization logic based on principal identity.

### 6.7 Verification Result

If all checks pass, the request is authenticated as:
- Coming from the specified agent
- Authorized by the specified principal
- Within the delegation validity period
- With a publicly verifiable delegation record

---

## 7. Activity Tracking (Recommended)

### 7.1 Purpose

To enable principal oversight during delegation renewal, agents SHOULD maintain activity records of their interactions with services. This allows principals to make informed decisions about whether to renew agent delegations based on actual behavior.

### 7.2 Activity Record Format

When implemented, agents SHOULD log each request/response as a separate record:

```json
{
  "agent_id": "agent:ed25519:5FHneW46xGXgs5mUiveU4sbTyGBzmstUspZC92UhjJM694ty",
  "timestamp": "2026-02-14T14:23:00Z",
  "service": "gmail.com",
  "method": "POST",
  "path": "/api/send",
  "status": 200,
  "source": "agent"
}
```

**Field Definitions:**

- **agent_id**: The agent's identifier
- **timestamp**: When the request was made (ISO 8601)
- **service**: The service domain/hostname
- **method**: HTTP method (GET, POST, etc.)
- **path**: Request path
- **status**: HTTP status code returned by the service
- **source**: Who reported this record. MUST be `"agent"` for agent-reported records. Future extensions (e.g., service receipts) MAY define additional values such as `"service"` for independently verified records.

### 7.3 Storage

Activity record storage is implementation-defined. Agents MAY use:
- IPFS (content-addressed, distributed)
- Cloud storage (S3, GCS, Azure Blob Storage)
- Blockchain-based storage
- Local databases
- Any other mechanism that allows retrieval

The storage mechanism SHOULD provide:
- Integrity (records cannot be tampered with)
- Availability (records can be retrieved when needed)
- Auditability (third parties can verify if needed)

### 7.4 Aggregation and Presentation

During delegation renewal, agents SHOULD:

1. Retrieve all activity records for the delegation period
2. Aggregate by service and HTTP status code
3. Present summary to principal, distinguishing record sources when available:

```
Activity Summary (Feb 14 08:00 - Feb 15 08:00):

Total Requests: 1,523
Success Rate: 98%

By Source:
  - Agent-reported: 1,200
  - Service-verified: 323

By Service:
  - gmail.com: 847 requests (0 errors)
  - calendar.google.com: 676 requests (31 errors)

By Status:
  - 2xx (Success): 1,489
  - 4xx (Client Error): 3
    - 429 (Rate Limited): 2
    - 403 (Forbidden): 1
  - 5xx (Server Error): 31
    - 500 (Internal Server Error): 31
```

4. Provide access to detailed records if principal requests

### 7.5 Service Participation (Optional)

Services MAY participate in activity verification by:

1. Logging their own records of agent requests
2. Returning a `VALET-Receipt` response header containing a URL or CID pointing to a signed activity record (see future extension: Service Receipts)
3. Making these records available to principals for cross-verification with agent-reported records

This is entirely OPTIONAL in v1.0. Service participation enables cross-verification but is not required for basic VALET operation. The `source` field in activity records (Section 7.2) is designed to distinguish agent-reported from service-verified records when service participation is available.

### 7.6 Benefits

Activity tracking provides:
- **Transparency**: Principals see what their agents actually do
- **Accountability**: Agents cannot hide misbehavior
- **Informed Decisions**: Renewal decisions based on real data, not blind trust
- **Debugging**: Helps identify issues with agent behavior
- **Audit Trail**: Historical record of agent actions

### 7.7 Privacy Considerations

Activity records contain only metadata (service, status, timestamp), not request/response content. This provides oversight without exposing sensitive data.

Principals should consider:
- How long to retain activity records
- Who has access to activity records
- Whether to share activity records with third parties

---

## 8. Complete Request Example

### 8.1 Delegation Creation

Principal creates delegation:

```json
{
  "agent_id": "agent:ed25519:5FHneW46xGXgs5mUiveU4sbTyGBzmstUspZC92UhjJM694ty",
  "principal_id": "ed25519:ABC123...",
  "issued_at": "2026-02-14T08:00:00Z",
  "expires_at": "2026-02-15T08:00:00Z",
  "delegation_signature": "iKqLm8NoPq..."
}
```

Stores to public location → receives URL: `https://ipfs.io/ipfs/QmYwAPJzv5CZsnA636s8Bv...`

### 8.2 Agent Request

```http
POST /api/send-email HTTP/1.1
Host: mail.example.com
Content-Type: application/json
VALET-Authorization: eyJhZ2VudF9pZCI6ImFnZW50OmVkMjU1MTk6NUZIbmVXNDZ4R1hnczVtVWl2ZVVUU2JUeUdCem1zdFVzcFpDOTJVaGpKTTY5NHR5IiwicHJpbmNpcGFsX2lkIjoiZWQyNTUxOTpBQkMxMjMuLi4iLCJpc3N1ZWRfYXQiOiIyMDI2LTAyLTE0VDA4OjAwOjAwWiIsImV4cGlyZXNfYXQiOiIyMDI2LTAyLTE1VDA4OjAwOjAwWiIsImRlbGVnYXRpb25fc2lnbmF0dXJlIjoiaUtxTG04Tm9QcS4uLiJ9
VALET-Agent: record=https://ipfs.io/ipfs/QmYwAPJzv5CZsnA636s8Bv...
Signature-Input: valet=("@method" "@path" "valet-authorization");created=1708077600;keyid="agent:ed25519:5FHneW46xGXgs5mUiveU4sbTyGBzmstUspZC92UhjJM694ty";alg="ed25519";v="1.0"
Signature: valet=:MEUCIQDxK7VQX8ZqkYT9jXMZ0ST+CZEfN+rT8gdVSFqH3CkPJgIgLbHDh2ERCVfK7hZP0nX4azAFGKm8jRtJfHDqC3Ef4wA=:

{"to":"user@example.com","subject":"Hello","body":"..."}
```

### 8.3 Service Verification

1. Extract `valet` signature from headers ✓
2. Decode VALET-Authorization header ✓
3. Fetch delegation from public URL ✓
4. Verify fetched matches decoded ✓
5. Verify principal signature on delegation ✓
6. Check delegation not expired ✓
7. Verify agent signature per RFC 9421 ✓
8. Verify principal authorized ✓

Request accepted.

---

## 9. Security Considerations

### 9.1 Key Management

**Agents:**
- Private keys MUST be stored securely
- Keys SHOULD be rotatable
- Compromised keys require new agent identity

**Principals:**
- SHOULD use hardware wallets or secure key storage
- SHOULD regularly review active delegations

### 9.2 Timestamp Validation

Services verifying signatures SHOULD check that the `created` timestamp in the signature parameters is recent (within acceptable window, e.g., ±5 minutes) to prevent replay attacks.

### 9.3 Delegation Expiry

Services MUST check delegation expiry. Expired delegations MUST be rejected even if the signature is valid.

### 9.4 Public Record Availability

Services SHOULD handle public record unavailability gracefully:
- Use multiple retrieval methods (IPFS gateways, direct HTTPS)
- Cache recently fetched delegations (respecting expiry)
- Define fail-open vs fail-closed policies

### 9.5 Signature Algorithms

Services MUST validate that the signature algorithm (`alg` parameter) is acceptable for their security requirements. Recommended algorithms:
- `ed25519` (EdDSA using Curve edwards25519)
- `ecdsa-p256-sha256` (ECDSA using Curve P-256)

Refer to RFC 9421 Section 3.3 for algorithm specifications.

### 9.6 Public Record Integrity

The public record provides non-repudiation. Services MUST verify:
- The record at the URL matches the header delegation
- The URL uses a trusted storage mechanism
- The record has not been tampered with (via content addressing or signatures)

Without public record verification, agents could fabricate delegations.

---

## 10. Implementation Guidelines

### 10.1 Agent Implementation

Agents need:
- Ed25519 key generation and signing
- HTTP client with RFC 9421 signature support
- Public storage client (IPFS gateway, HTTPS, etc.)
- Delegation management
- (Optional) Activity logging system

**Estimated implementation:** ~300 lines of code using existing RFC 9421 libraries.

### 10.2 Service Implementation  

Services need:
- RFC 9421 signature verification library
- HTTP client for fetching public records
- Principal-to-account mapping (service-specific)

**Estimated implementation:** ~200 lines of code using existing RFC 9421 libraries.

### 10.3 Principal Implementation

Principals need:
- Public storage access (IPFS node, web server, or hosted service)
- Key management for signing delegations
- Renewal workflow (manual or automated)
- (Optional) Activity record review interface

---

## 11. IANA Considerations

### 11.1 HTTP Header Registration

The following HTTP headers should be registered:

**VALET-Authorization**
- Header field name: VALET-Authorization
- Applicable protocol: HTTP
- Status: Standard
- Author/Change controller: IETF
- Specification document: this document

**VALET-Agent**
- Header field name: VALET-Agent
- Applicable protocol: HTTP
- Status: Standard
- Author/Change controller: IETF
- Specification document: this document

### 11.2 RFC 9421 Component Registry

The following component identifier should be registered in the "HTTP Signature Derived Component Names" registry:

**valet-authorization**
- Component Name: valet-authorization
- Description: Base64-encoded agent delegation
- Reference: this document

---

## 12. References

### 12.1 Normative References

**[RFC2119]** Bradner, S., "Key words for use in RFCs to Indicate Requirement Levels", BCP 14, RFC 2119, March 1997.

**[RFC9421]** Backman, A., Richer, J., Sporny, M., "HTTP Message Signatures", RFC 9421, February 2024.

**[RFC8941]** Nottingham, M., Kamp, P-H., "Structured Field Values for HTTP", RFC 8941, February 2021.

**[RFC3986]** Berners-Lee, T., Fielding, R., Masinter, L., "Uniform Resource Identifier (URI): Generic Syntax", RFC 3986, January 2005.

### 12.2 Informative References

**[IPFS]** "InterPlanetary File System", https://docs.ipfs.tech

**[RFC9530]** Polli, R., Pardue, L., "Digest Fields", RFC 9530, February 2024.

---

## 13. Future Extensions

This specification may be extended in future versions to support:
- Permission scopes (hierarchical scope system)
- Revocation lists (real-time delegation revocation)
- Sub-delegation (agents delegating to other agents)
- Service receipts (signed, independently verifiable activity records from services)
- Zero-knowledge proofs (privacy-preserving verification)

Extensions MUST NOT break backward compatibility with v1.0.

---

**END OF SPECIFICATION**