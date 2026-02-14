# VALET Protocol Specification v1.0

**Status:** Draft  
**Date:** February 14, 2026  
**Editors:** Subcent

---

## Abstract

VALET (Verified Agent Legitimacy and Endorsement Token) is a lightweight, decentralized protocol enabling AI agents to prove delegated authority when accessing services. VALET provides cryptographic verification of agent identity and principal authorization without requiring centralized identity providers or real-time registry lookups.

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
- **Verifiable**: All delegations cryptographically signed
- **Standards-based**: Built on RFC 9421 HTTP Message Signatures
- **Human oversight**: Regular renewal enforces principal review

### 1.3 Scope

This specification defines:
- Agent identity representation
- Delegation structure
- HTTP headers for agent requests (using RFC 9421)
- Signature construction and verification
- Expiry and renewal model

This specification does NOT define:
- Activity tracking (future extension)
- Violation reporting (future extension)
- Permission scopes (service-defined)
- Revocation mechanisms (future extension)

---

## 2. Dependencies

VALET builds on the following standards:

- **RFC 9421**: HTTP Message Signatures - used for signing HTTP requests
- **RFC 8941**: Structured Field Values for HTTP
- **RFC 3986**: Uniform Resource Identifier (URI): Generic Syntax
- **IPFS**: InterPlanetary File System - used for delegation storage

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

### 3.3 Delegation

A delegation is a cryptographic statement by a principal authorizing an agent to act on their behalf for a limited time period. Delegations are stored as JSON objects on IPFS.

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
The principal's public key identifier.

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

Delegations MUST be stored as JSON on IPFS. The IPFS hash (Content IDentifier, CID) of the delegation is used in HTTP headers.

---

## 5. HTTP Headers

VALET uses RFC 9421 (HTTP Message Signatures) with a specific signature label and required components.

### 5.1 Required Headers

**VALET-Delegation**  
IPFS CID of the delegation JSON.
```
VALET-Delegation: QmYwAPJzv5CZsnA636s8Bv...
```

**Signature-Input** (from RFC 9421)  
Signature metadata using the label `valet`. MUST include the `valet-delegation` component.
```
Signature-Input: valet=("@method" "@path" "valet-delegation");created=1618884479;keyid="agent:ed25519:5FHn...";alg="ed25519"
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
   - `valet-delegation` - The VALET-Delegation header value
3. **Required signature parameters**:
   - `created` - UNIX timestamp when signature was created
   - `keyid` - Agent's public key identifier (the agent_id)
   - `alg` - Signature algorithm (e.g., "ed25519")

**Example signature base construction** (per RFC 9421):
```
"@method": POST
"@path": /api/send-email
"valet-delegation": QmYwAPJzv5CZsnA636s8Bv...
"@signature-params": ("@method" "@path" "valet-delegation");created=1618884479;keyid="agent:ed25519:5FHn...";alg="ed25519"
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
VALET-Delegation: QmYwAPJzv5CZsnA636s8Bv...
Signature-Input: valet=("@method" "@path" "valet-delegation");created=1618884479;keyid="agent:ed25519:5FHn...";alg="ed25519", othersig=("@method" "@authority");created=1618884480;keyid="other-key"
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

### 6.2 Verify Agent Signature (RFC 9421)

1. Extract agent's public key from `keyid` parameter (this is the agent_id)
2. Reconstruct signature base per RFC 9421 using covered components
3. Verify signature using agent's public key
4. REJECT if signature is invalid

### 6.3 Verify Delegation

1. Extract IPFS CID from `VALET-Delegation` header
2. Fetch delegation JSON from IPFS
3. Verify delegation contains required fields
4. Extract `principal_id` from delegation
5. Verify `delegation_signature` using principal's public key
6. Verify delegation has not expired (`current_time < expires_at`)
7. REJECT if any verification fails

### 6.4 Verify Principal Authorization

Services MUST verify that the principal is authorized to access the service:
1. Check if `principal_id` corresponds to a known user/account
2. Apply service-specific authorization policies
3. REJECT if principal is not authorized

Services define their own authorization logic based on principal identity.

### 6.5 Verification Result

If all checks pass, the request is authenticated as:
- Coming from the specified agent
- Authorized by the specified principal
- Within the delegation validity period

---

## 7. Complete Request Example

### 7.1 Delegation Creation

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

Stores to IPFS → receives CID: `QmYwAPJzv5CZsnA636s8Bv...`

### 7.2 Agent Request

```http
POST /api/send-email HTTP/1.1
Host: mail.example.com
Content-Type: application/json
VALET-Delegation: QmYwAPJzv5CZsnA636s8Bv...
Signature-Input: valet=("@method" "@path" "valet-delegation");created=1708077600;keyid="agent:ed25519:5FHneW46xGXgs5mUiveU4sbTyGBzmstUspZC92UhjJM694ty";alg="ed25519"
Signature: valet=:MEUCIQDxK7VQX8ZqkYT9jXMZ0ST+CZEfN+rT8gdVSFqH3CkPJgIgLbHDh2ERCVfK7hZP0nX4azAFGKm8jRtJfHDqC3Ef4wA=:

{"to":"user@example.com","subject":"Hello","body":"..."}
```

### 7.3 Service Verification

1. Extract `valet` signature from headers ✓
2. Verify agent signature per RFC 9421 ✓
3. Fetch delegation from IPFS using CID ✓
4. Verify principal signature on delegation ✓
5. Check delegation not expired ✓
6. Verify principal authorized ✓

Request accepted.

---

## 8. Security Considerations

### 8.1 Key Management

**Agents:**
- Private keys MUST be stored securely
- Keys SHOULD be rotatable
- Compromised keys require new agent identity

**Principals:**
- SHOULD use hardware wallets or secure key storage
- SHOULD regularly review active delegations

### 8.2 Timestamp Validation

Services verifying signatures SHOULD check that the `created` timestamp in the signature parameters is recent (within acceptable window, e.g., ±5 minutes) to prevent replay attacks.

### 8.3 Delegation Expiry

Services MUST check delegation expiry. Expired delegations MUST be rejected even if the signature is valid.

### 8.4 IPFS Availability

Services SHOULD handle IPFS unavailability gracefully:
- Use IPFS gateway redundancy
- Cache recently fetched delegations (respecting expiry)
- Define fail-open vs fail-closed policies

### 8.5 Signature Algorithms

Services MUST validate that the signature algorithm (`alg` parameter) is acceptable for their security requirements. Recommended algorithms:
- `ed25519` (EdDSA using Curve edwards25519)
- `ecdsa-p256-sha256` (ECDSA using Curve P-256)

Refer to RFC 9421 Section 3.3 for algorithm specifications.

---

## 9. Implementation Guidelines

### 9.1 Agent Implementation

Agents need:
- Ed25519 key generation and signing
- IPFS client (lightweight gateway-based)
- RFC 9421 signature implementation
- HTTP header construction

**Estimated implementation:** ~300 lines of code using existing RFC 9421 libraries.

### 9.2 Service Implementation  

Services need:
- RFC 9421 signature verification library
- IPFS client or gateway access
- Principal-to-account mapping

**Estimated implementation:** ~200 lines of code using existing RFC 9421 libraries.

### 9.3 Principal Implementation

Principals need:
- IPFS node or hosted IPFS service
- Key management for signing delegations
- Renewal workflow (manual or automated)

---

## 10. IANA Considerations

### 10.1 HTTP Header Registration

The following HTTP header should be registered:

**VALET-Delegation**
- Header field name: VALET-Delegation
- Applicable protocol: HTTP
- Status: Standard
- Author/Change controller: IETF
- Specification document: this document

### 10.2 RFC 9421 Component Registry

The following component identifier should be registered in the "HTTP Signature Derived Component Names" registry:

**valet-delegation**
- Component Name: valet-delegation
- Description: IPFS CID of agent delegation
- Reference: this document

---

## 11. References

### 11.1 Normative References

**[RFC2119]** Bradner, S., "Key words for use in RFCs to Indicate Requirement Levels", BCP 14, RFC 2119, March 1997.

**[RFC9421]** Backman, A., Richer, J., Sporny, M., "HTTP Message Signatures", RFC 9421, February 2024.

**[RFC8941]** Nottingham, M., Kamp, P-H., "Structured Field Values for HTTP", RFC 8941, February 2021.

**[RFC3986]** Berners-Lee, T., Fielding, R., Masinter, L., "Uniform Resource Identifier (URI): Generic Syntax", RFC 3986, January 2005.

### 11.2 Informative References

**[IPFS]** "InterPlanetary File System", https://docs.ipfs.tech

**[RFC9530]** Polli, R., Pardue, L., "Digest Fields", RFC 9530, February 2024.

---

## 12. Future Extensions

This specification may be extended in future versions to support:
- Activity record tracking
- Violation reporting
- Revocation lists
- Permission scopes
- Sub-delegation

Extensions MUST NOT break backward compatibility with v1.0.

---

**END OF SPECIFICATION**
