# DDISA — DNS-Discoverable Identity & Service Authentication

**Status:** Draft  

## Motivation

Every service provider faces the same question: *where does this user authenticate?* Today, this requires either a centralized identity registry, hardcoded provider lists, or manual user configuration. DDISA solves this by leveraging DNS — the same infrastructure that already routes email — to publish identity provider endpoints. Just as MX records tell the world where to deliver mail for a domain, DDISA's `_ddisa` TXT records tell service providers where to authenticate users from that domain. No central registry. No vendor lock-in. Just DNS.

## Specification Documents

| Document | Description |
|----------|-------------|
| [01 — Overview](spec/01-overview.md) | Problem statement, solution, and design principles |
| [02 — DNS Records](spec/02-dns-records.md) | Record format, fields, resolution algorithm |
| [03 — Authentication Flow](spec/03-authentication-flow.md) | OAuth 2.0 + PKCE flow, human auth (Passkeys), agent auth (Ed25519) |
| [04 — JWT](spec/04-jwt.md) | Minimal AuthN token claims |
| [05 — Security](spec/05-security.md) | Threat model and security considerations |
| [06 — Compliance](spec/06-compliance.md) | NIS2, NIST CSF 2.0, and regulatory mapping |

## License

This specification is published under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
