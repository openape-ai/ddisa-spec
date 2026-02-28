# 05 — Security Considerations

## Mitigation of Attack Vectors

| Attack | Description | Mitigation | Protocol Mechanism |
|--------|-------------|------------|--------------------|
| **Redirect URI Injection** | Attacker starts auth flow with malicious `redirect_uri` to steal the authorization code | Code is useless without `code_verifier` | PKCE (REQUIRED) |
| **Authorization Code Interception** | Attacker intercepts the code during redirect | Code alone cannot be exchanged for an assertion | PKCE (REQUIRED) |
| **DNS Spoofing** | Attacker poisons DNS to redirect IdP discovery | Assertion signature can't be forged; issuer mismatch detected | JWT signature + `iss` validation |
| **IdP Impersonation** | Rogue IdP at attacker-controlled URL | Requires domain compromise (DNS control); IdP pinning as defense-in-depth | HTTPS + DNSSEC (RECOMMENDED) |
| **Token Replay** | Captured assertion reused by attacker | Short-lived (max 300s), bound to specific session | `nonce` + `exp` + `jti` |
| **CSRF** | Attacker triggers auth flow in victim's browser | State parameter links request to response | `state` parameter (REQUIRED) |
| **Phishing** | Attacker tricks user into entering credentials on fake site | Passkeys are origin-bound — won't activate on wrong domain | WebAuthn (REQUIRED for humans) |
| **Credential Stuffing** | Reuse of leaked passwords from other services | No passwords in DDISA | Passkeys only |
| **Agent Key Theft** | Attacker steals agent's private key | Key revocation; scoped keys per agent instance | Ed25519 + IdP key management |
| **Man-in-the-Middle** | Attacker intercepts SP ↔ IdP communication | All endpoints MUST use HTTPS | TLS (REQUIRED) |

### Why PKCE is Critical

PKCE (Proof Key for Code Exchange) is the primary defense against authorization code attacks. The flow:

1. SP generates a random `code_verifier` and derives `code_challenge = sha256(code_verifier)`.
2. SP sends `code_challenge` in the authorization request.
3. IdP binds the authorization code to the `code_challenge`.
4. SP sends `code_verifier` in the token exchange.
5. IdP verifies: `sha256(code_verifier) == stored code_challenge`.

An attacker who intercepts or steals the authorization code cannot exchange it — they don't have the `code_verifier`, which never leaves the SP.

## Threat Model (Detailed)

### DNS Spoofing and Hijacking

**Threat:** An attacker poisons DNS responses to redirect IdP discovery to a malicious identity provider.

**Mitigations:**

- **DNSSEC.** Domain operators SHOULD sign their zones with DNSSEC. SPs SHOULD validate DNSSEC signatures when available.
- **HTTPS-only IdP URLs.** The `idp` field MUST use `https`. SPs MUST reject `http` URLs.
- **Issuer validation.** SPs MUST verify that the `iss` claim in the assertion matches the `idp` URL discovered via DNS.
- **JWT signature verification.** Even if DNS is compromised, the SP validates the assertion signature against the IdP's published keys. A spoofed DNS record pointing to a malicious IdP would require the attacker to also possess valid signing keys.

**Recommendation:** DNSSEC is RECOMMENDED for all domains publishing DDISA records. While DDISA is functional without DNSSEC, it significantly raises the bar for DNS-based attacks.

### IdP Impersonation

**Threat:** An attacker sets up a rogue IdP and modifies DNS records to point to it.

**Mitigations:**

- Domain control is the trust anchor. If an attacker can modify DNS records, they control the domain — this is equivalent to domain compromise, which is outside the scope of any application-layer protocol.
- SPs MAY implement IdP pinning (caching a known-good IdP for a domain and alerting on changes).

### Token Replay

**Threat:** A captured assertion is reused by an attacker.

**Mitigations:**

- Maximum assertion lifetime of 300 seconds.
- The `nonce` claim binds the assertion to a specific authentication session.
- The `jti` claim provides a unique identifier; SPs SHOULD track used `jti` values within the assertion's validity window.

## Passkeys over Passwords

DDISA mandates Passkeys (WebAuthn/FIDO2) for human authentication. This is a deliberate design choice:

| Property | Passwords | Passkeys |
|----------|-----------|----------|
| Phishing resistance | None | Built-in (origin-bound) |
| Credential reuse | Common | Impossible (unique per relying party) |
| Brute force | Vulnerable | Not applicable |
| MFA requirement | Separate factor needed | Built-in (device + biometric/PIN) |
| Regulatory alignment | Insufficient for NIS2 AAL2 | Meets NIS2 strong authentication |

Passwords are the root cause of the majority of account compromises. DDISA eliminates them by design.

## Agent Key Management

Agent authentication uses Ed25519 key pairs. Secure key management is critical:

- **Private keys MUST be stored securely.** Hardware security modules (HSMs), secure enclaves, or encrypted key stores are RECOMMENDED.
- **Key rotation.** IdPs MUST support multiple active keys per identity to enable rotation without downtime.
- **Key revocation.** IdPs MUST provide a mechanism to revoke individual agent keys immediately.
- **Scoping.** Organizations SHOULD issue distinct key pairs per agent instance. Sharing private keys across agents is NOT RECOMMENDED.
- **No key in token.** Agent public keys are NOT included in the assertion. The assertion only attests the authentication result.

## Email as Public Information

In DDISA, the user's email address serves as the subject identifier (`sub` claim) and is used for IdP discovery. This means:

- The email address is visible to the SP. This is by design — it is the identifier.
- The email domain is used for DNS lookup. This reveals the user's organizational affiliation, which is inherent to domain-based identity.
- DDISA does not expose any additional personal data. No display names, no profile pictures, no phone numbers.

The email address in DDISA plays the same role it does in email itself: a public, routable identifier. Systems that require pseudonymous authentication should evaluate whether domain-based identity is appropriate for their use case.
