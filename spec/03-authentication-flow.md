# 03 — Authentication Flow

## Overview

DDISA authentication combines DNS-based IdP discovery with an OAuth 2.0-inspired Authorization Code flow using PKCE. The protocol supports two actor types with distinct authentication mechanisms:

- **Human actors** — MUST authenticate via Passkeys (WebAuthn/FIDO2).
- **Agent actors** — MUST authenticate via Ed25519 challenge-response.

## Terminology

- **Assertion** — A signed JWT issued by the IdP upon successful authentication. Not an OIDC ID Token.
- **SP ID** — The service provider's identifier (analogous to OAuth `client_id`).
- **SP Manifest** — A JSON document served at `/.well-known/sp-manifest.json` describing the SP's identity and redirect URIs.

## Flow: SP → DNS → IdP → Assertion

```mermaid
sequenceDiagram
    participant U as User / Agent
    participant SP as Service Provider
    participant DNS as DNS
    participant IdP as Identity Provider

    U->>SP: Present email (user@example.com)
    SP->>DNS: Query _ddisa.example.com TXT
    DNS-->>SP: "v=ddisa1; idp=https://id.example.com; mode=open"
    SP->>IdP: Authorization Request (code + PKCE)
    IdP->>U: Authentication challenge
    Note over IdP,U: Human: Passkey/WebAuthn<br/>Agent: Ed25519 challenge-response
    U-->>IdP: Authentication response
    IdP-->>SP: Authorization code (via redirect)
    SP->>IdP: POST /token (code + code_verifier)
    IdP-->>SP: Assertion (signed JWT)
    SP->>SP: Validate assertion
```

## Step-by-Step

### 1. Discovery

The SP resolves the IdP URL via DNS as described in [02 — DNS Records](02-dns-records.md).

### 2. Authorization Request

The SP redirects to the IdP's `/authorize` endpoint with:

| Parameter | Required | Description |
|-----------|----------|-------------|
| `response_type` | MUST | `code` |
| `sp_id` | MUST | The SP's identifier |
| `redirect_uri` | MUST | The SP's callback URL |
| `state` | MUST | Opaque CSRF-protection value |
| `code_challenge` | MUST | PKCE challenge (S256) |
| `code_challenge_method` | MUST | `S256` |
| `nonce` | MUST | Binds the assertion to this request |
| `login_hint` | RECOMMENDED | The user's email address |

### 3. Authentication

The IdP authenticates the actor (see sections below).

### 4. Authorization Response

The IdP redirects back to the SP's `redirect_uri` with `code` and `state` parameters.

### 5. Token Exchange

The SP sends a POST request to the IdP's `/token` endpoint:

```json
{
  "grant_type": "authorization_code",
  "code": "<authorization_code>",
  "code_verifier": "<pkce_verifier>",
  "redirect_uri": "<redirect_uri>",
  "sp_id": "<sp_id>"
}
```

The IdP validates PKCE, consumes the code (single-use), and returns:

```json
{
  "assertion": "<signed_jwt>"
}
```

### 6. Assertion Validation

The SP validates the assertion JWT. See [04 — JWT](04-jwt.md).

The SP retrieves the IdP's public keys from `<idp_url>/.well-known/jwks.json` for signature verification.

## Human Authentication: Passkeys (WebAuthn)

DDISA-compliant IdPs MUST support Passkeys (WebAuthn/FIDO2) as the primary authentication method for human actors.

- The IdP MUST support WebAuthn with resident credentials (discoverable credentials).
- Passwords MUST NOT be the sole authentication factor. IdPs SHOULD offer a migration path to Passkeys for existing password-based accounts.
- The resulting assertion MUST contain `"act": "human"`.

### Rationale

Passkeys provide phishing-resistant, hardware-bound authentication. This aligns with NIS2 requirements for strong authentication and NIST SP 800-63B AAL2+ guidelines. See [05 — Security](05-security.md) and [06 — Compliance](06-compliance.md).

## Agent Authentication: Ed25519 Challenge-Response

DDISA-compliant IdPs MUST support Ed25519 challenge-response for agent (non-human) actors.

### Flow

1. The agent presents its public key (or key ID) and email to the IdP.
2. The IdP generates a random challenge (minimum 32 bytes, base64url-encoded).
3. The agent signs the challenge with its Ed25519 private key.
4. The IdP verifies the signature against the registered public key.
5. On success, the IdP issues an assertion with `"act": "agent"`.

### Key Registration

Agent public keys MUST be pre-registered with the IdP. The mechanism for key registration is out of scope for this specification, but the IdP MUST support associating multiple Ed25519 public keys with a single identity.

### Requirements

- Challenge MUST be single-use and time-limited (maximum 60 seconds).
- The IdP MUST reject replayed challenges.
- Ed25519 is the only REQUIRED algorithm. IdPs MAY support additional algorithms.
- The resulting assertion MUST contain `"act": "agent"`.

## Well-Known Endpoints

| Endpoint | Served by | Description |
|----------|-----------|-------------|
| `/.well-known/jwks.json` | IdP | JSON Web Key Set for assertion signature verification |
| `/.well-known/sp-manifest.json` | SP | SP identity, redirect URIs, and metadata |
