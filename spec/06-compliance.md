# 06 — Compliance Mapping

## Overview

DDISA was designed with regulatory requirements in mind. This document maps DDISA features to specific provisions of NIS2 (EU) and NIST CSF 2.0 / EO 14028 (US).

## NIS2 (EU Directive 2022/2555)

NIS2 establishes cybersecurity obligations for essential and important entities across the EU. Several provisions are directly addressed by DDISA.

| NIS2 Requirement | Article | DDISA Feature |
|-----------------|---------|---------------|
| Risk-appropriate security measures | Art. 21(1) | Phishing-resistant authentication (Passkeys), no passwords |
| Multi-factor authentication | Art. 21(2)(j) | Passkeys provide inherent MFA (possession + biometric/PIN) |
| Supply chain security | Art. 21(2)(d) | Decentralized IdP discovery — no single vendor dependency |
| Incident detection and response | Art. 21(2)(b) | `act` claim enables distinguishing human vs. agent actions in audit logs |
| Access control and asset management | Art. 21(2)(i) | Domain-based identity ties access to organizational boundaries |
| Cryptography and encryption | Art. 21(2)(h) | Ed25519 for agent auth, HTTPS-only, JWT signatures |

### Key Points

- **Passkeys satisfy NIS2's strong authentication requirement.** They meet AAL2 or higher, eliminating the need for separate MFA solutions.
- **The `act` claim supports NIS2 incident response.** When investigating incidents, organizations can distinguish whether actions were performed by humans or automated agents.
- **Decentralized discovery avoids supply chain concentration.** Each domain controls its own IdP configuration, reducing dependency on centralized federation services.

## NIST Cybersecurity Framework 2.0 / EO 14028

Executive Order 14028 ("Improving the Nation's Cybersecurity") and NIST CSF 2.0 establish identity and authentication requirements for US federal systems and critical infrastructure.

| NIST CSF 2.0 Category | Subcategory | DDISA Feature |
|-----------------------|-------------|---------------|
| **Identify (ID)** | ID.AM — Asset Management | Domain-based identity aligns identities to organizational domains |
| **Protect (PR)** | PR.AA — Identity Management, Authentication, and Access Control | Passkeys (AAL2+), Ed25519 agent auth, PKCE |
| **Protect (PR)** | PR.AA-03 — Users, services, and hardware are authenticated | Distinct auth methods for humans (WebAuthn) and agents (Ed25519) |
| **Detect (DE)** | DE.AE — Adverse Event Analysis | `act` claim enables automated vs. human action differentiation in SIEM |
| **Respond (RS)** | RS.AN — Incident Analysis | JWT claims provide audit trail: who (`sub`), what type (`act`), when (`exp`), where (`aud`) |

### NIST SP 800-63B Alignment

| AAL Level | Requirement | DDISA |
|-----------|------------|-------|
| AAL1 | Single factor | Not supported (below DDISA minimum) |
| AAL2 | Two factors, phishing-resistant recommended | Passkeys (possession + biometric/PIN) ✓ |
| AAL3 | Hardware-bound, verifier impersonation resistant | Passkeys with hardware authenticators ✓ |

### EO 14028 Specifics

- **Zero Trust Architecture:** DDISA's per-request authentication model (short-lived tokens, no persistent sessions at the protocol level) aligns with Zero Trust principles.
- **Software Supply Chain:** Agent authentication with Ed25519 keys provides cryptographic identity for automated systems interacting across organizational boundaries.
- **Phishing-Resistant MFA:** EO 14028 requires phishing-resistant MFA for federal systems. Passkeys satisfy this requirement by design.

## Summary

| DDISA Feature | NIS2 | NIST CSF 2.0 | EO 14028 |
|--------------|------|---------------|----------|
| Passkey authentication | Art. 21(2)(j) | PR.AA-03, SP 800-63B AAL2+ | Phishing-resistant MFA ✓ |
| `act` claim (human/agent) | Art. 21(2)(b) | DE.AE, RS.AN | Audit requirements ✓ |
| Ed25519 agent auth | Art. 21(2)(h) | PR.AA-03 | Software supply chain ✓ |
| DNS-based discovery | Art. 21(2)(d) | ID.AM | Zero Trust ✓ |
| Short-lived JWTs | Art. 21(1) | PR.AA | Zero Trust ✓ |
| No passwords | Art. 21(2)(j) | SP 800-63B | Phishing-resistant MFA ✓ |
