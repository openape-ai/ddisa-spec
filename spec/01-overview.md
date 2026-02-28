# 01 — Overview

## What is DDISA?

DDISA (DNS-Discoverable Identity & Service Authentication) is a protocol for discovering a domain's identity provider (IdP) through DNS. It enables service providers (SPs) to authenticate users without prior knowledge of their IdP, using only the user's email address and a standard DNS lookup.

## Problem

When a user presents an email address to a service provider, the SP needs to know *where* that user authenticates. Current approaches have significant drawbacks:

- **Centralized registries** (e.g., federation metadata aggregators) create single points of failure and require enrollment.
- **Hardcoded provider lists** ("Sign in with X") limit user choice and concentrate power in a few large identity providers.
- **Manual configuration** (e.g., entering an IdP URL) is error-prone and impractical at scale.

None of these approaches scale the way email does.

## Solution

DDISA introduces a DNS TXT record at `_ddisa.<domain>` that points to the domain's identity provider. The resolution flow mirrors what is already established for email:

1. User provides their email address (`user@example.com`).
2. SP extracts the domain (`example.com`).
3. SP queries DNS for `_ddisa.example.com TXT`.
4. DNS returns the IdP endpoint URL.
5. SP initiates an OAuth 2.0 Authorization Code + PKCE flow with the IdP.

This is analogous to MX record resolution for email delivery — simple, decentralized, and proven at internet scale.

## Design Principles

| Principle | Description |
|-----------|-------------|
| **DNS-native** | Uses existing DNS infrastructure. No new protocols, no new registries. |
| **No central registry** | Each domain controls its own identity configuration. Discovery is fully decentralized. |
| **Domain-based identity** | Identity is rooted in domain ownership, just like email. If you control the domain, you control the identity. |
| **Minimal surface** | The protocol specifies discovery and authentication only. No profile data, no authorization, no attribute exchange. |
| **Standards-based** | Builds on OAuth 2.0, WebAuthn, and DNS — battle-tested technologies with broad ecosystem support. |
