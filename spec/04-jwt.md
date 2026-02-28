# 04 — Assertion JWT

## Design Principle

DDISA assertions carry the minimum claims necessary for authentication. They assert *who* the actor is and *how* they authenticated — nothing more. Authorization, roles, profiles, and attributes are explicitly out of scope.

## Required Claims

| Claim | Type | Description |
|-------|------|-------------|
| `sub` | string | The actor's email address (e.g., `user@example.com`). This is the canonical identifier. |
| `act` | string | Actor type. MUST be either `"human"` or `"agent"`. |
| `iss` | string | The IdP's issuer URL. MUST match the IdP URL discovered via DNS. |
| `aud` | string | The intended audience (SP's `sp_id`). |
| `exp` | number | Expiration time (Unix timestamp). |
| `iat` | number | Issued-at time (Unix timestamp). |
| `nonce` | string | The nonce value from the authorization request. MUST be present. |
| `jti` | string | Unique token identifier. MUST be a UUID or equivalent unique string. |

No additional claims are defined by this specification. IdPs SHOULD NOT add custom claims to the assertion. Service providers MUST NOT require claims beyond those listed above.

## Algorithm

The signing algorithm MUST be `ES256` (ECDSA using P-256 and SHA-256). This is fixed — there is no algorithm negotiation.

The IdP's public keys are published at `<idp_url>/.well-known/jwks.json` as a standard JSON Web Key Set (JWKS).

## The `act` Claim

The `act` claim distinguishes human users from automated agents. This distinction is critical for:

- **Audit trails.** Knowing whether an action was performed by a human or an agent is a regulatory requirement under NIS2 and NIST CSF.
- **Rate limiting.** SPs may apply different rate limits to agents vs. humans.
- **Risk assessment.** Agent-initiated actions may carry different risk profiles.
- **Accountability.** Agents act on behalf of a human; the `sub` claim identifies the responsible party, while `act` identifies the actor type.

The IdP determines `act` based on the authentication method used:

- Passkey/WebAuthn → `"human"`
- Ed25519 challenge-response → `"agent"`

## Example Assertion

### Header

```json
{
  "alg": "ES256",
  "typ": "JWT",
  "kid": "idp-signing-key-2025"
}
```

### Payload

```json
{
  "sub": "alice@example.com",
  "act": "human",
  "iss": "https://id.example.com",
  "aud": "https://app.serviceprovider.com",
  "iat": 1740700500,
  "exp": 1740700800,
  "nonce": "n-0S6_WzA2Mj",
  "jti": "550e8400-e29b-41d4-a716-446655440000"
}
```

## Assertion Lifetime

- Assertions MUST have a maximum lifetime of 300 seconds (5 minutes).
- DDISA does not define refresh tokens. Session management is the SP's responsibility.
- The `exp` claim MUST be validated by the SP. Expired assertions MUST be rejected.

## Validation

SPs MUST perform the following validation steps:

1. Verify the JWT signature against the IdP's JWKS (`<idp_url>/.well-known/jwks.json`).
2. Verify `alg` is `ES256`.
3. Verify `iss` matches the IdP URL discovered via DNS.
4. Verify `aud` matches the SP's own `sp_id`.
5. Verify `exp` has not passed.
6. Verify `nonce` matches the value sent in the authorization request.
7. Verify `act` is either `"human"` or `"agent"`.
