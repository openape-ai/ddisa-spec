# 02 — DNS Records

## Record Format

A domain that supports DDISA MUST publish a TXT record at the `_ddisa` subdomain:

```
_ddisa.<domain> TXT "v=ddisa1; idp=<url>; mode=<mode>"
```

## Fields

| Field | Required | Description |
|-------|----------|-------------|
| `v` | MUST | Protocol version. Currently `ddisa1`. |
| `idp` | MUST | The absolute URL of the domain's identity provider. MUST use `https`. |
| `mode` | MUST | SP admission policy. See [Policy Modes](#policy-modes). |
| `priority` | OPTIONAL | Numeric priority (lower = higher priority, like MX records). Default: `10`. Allows failover when multiple `_ddisa` records exist. |
| `policy_endpoint` | OPTIONAL | URL for dynamic SP admission decisions (used with `allowlist-user` mode). |

Fields are separated by semicolons (`;`). Whitespace around separators is tolerated. Field order is not significant, but `v` SHOULD appear first.

## Policy Modes

| Mode | Description |
|------|-------------|
| `open` | Any SP MAY initiate authentication. No prior registration required. |
| `allowlist-admin` | Only admin-approved SPs MAY authenticate users. The IdP MUST reject unknown SP IDs. |
| `allowlist-user` | Unknown SPs trigger a user consent prompt. The user decides whether to allow the SP. See [SP Manifest (Optional)](#sp-manifest-optional). |
| `deny` | No SPs are permitted. Authentication requests are always rejected. Useful for domains that publish a DDISA record to explicitly signal "no external authentication". |

### Choosing a Mode

- **`open`** — Public services, open federation, community platforms.
- **`allowlist-admin`** — Enterprise environments where IT controls which apps employees can use.
- **`allowlist-user`** — Privacy-conscious domains where users maintain control over their identity usage.
- **`deny`** — Domains that want to explicitly opt out of DDISA federation.

## SP Manifest (Optional)

In `allowlist-user` mode, the IdP MAY fetch a manifest from the SP to display context to the user during the consent prompt. The manifest is served at:

```
GET <sp_origin>/.well-known/sp-manifest.json
```

```json
{
  "sp_id": "app.example.com",
  "name": "Example App",
  "description": "A project management tool",
  "redirect_uris": ["https://app.example.com/callback"],
  "logo_uri": "https://app.example.com/logo.png",
  "contact": "admin@example.com"
}
```

This is purely informational — it helps the user make an informed decision. The security of the flow does not depend on the manifest (PKCE protects against code interception regardless). The manifest is NOT required and NOT relevant for `open`, `allowlist-admin`, or `deny` modes.

## Examples

**Open federation:**
```
_ddisa.example.com. 3600 IN TXT "v=ddisa1; idp=https://id.example.com; mode=open"
```

**Enterprise (admin-controlled):**
```
_ddisa.corp.example.org. 3600 IN TXT "v=ddisa1; idp=https://auth.corp.example.org; mode=allowlist-admin"
```

**User-controlled consent:**
```
_ddisa.privacy.example.com. 3600 IN TXT "v=ddisa1; idp=https://id.privacy.example.com; mode=allowlist-user"
```

**Explicit opt-out:**
```
_ddisa.internal.example.com. 3600 IN TXT "v=ddisa1; idp=https://id.internal.example.com; mode=deny"
```

**Failover with priority:**
```
_ddisa.bigcorp.com. 3600 IN TXT "v=ddisa1; idp=https://id-primary.bigcorp.com; mode=open; priority=10"
_ddisa.bigcorp.com. 3600 IN TXT "v=ddisa1; idp=https://id-backup.bigcorp.com; mode=open; priority=20"
```

## Resolution Algorithm

Given a user identifier in the form `user@domain`:

1. Extract the domain part from the email address.
2. Query DNS for `_ddisa.<domain> TXT`.
3. Parse all TXT records beginning with `v=ddisa1`.
4. If multiple records exist, select the one with the lowest `priority` value (default: `10`).
5. Validate that `v` equals `ddisa1`. If not, abort.
6. Extract the `idp` URL. Verify it uses `https`.
7. Check `mode`:
   - `open` → proceed with authentication.
   - `allowlist-admin` → SP MUST be pre-registered with the IdP.
   - `allowlist-user` → IdP prompts user for consent.
   - `deny` → abort.
8. Proceed with the OAuth 2.0 Authorization Code + PKCE flow (see [03 — Authentication Flow](03-authentication-flow.md)).

If the DNS query returns `NXDOMAIN` or no matching TXT record is found, the domain does not support DDISA. The SP MUST NOT fall back to guessing.

## DNS-over-HTTPS (DoH)

For environments where traditional DNS resolution is unavailable (browsers, edge runtimes, serverless), implementations MAY use DNS-over-HTTPS (DoH) providers:

- `https://cloudflare-dns.com/dns-query`
- `https://dns.google/resolve`
- `https://dns.quad9.net:5053/dns-query`

DoH requests MUST use the `application/dns-json` content type. Implementations SHOULD allow the DoH provider to be configurable.

## Caching and TTL

- SPs SHOULD respect the DNS TTL of the TXT record.
- SPs SHOULD NOT cache resolved IdP URLs for longer than the DNS TTL.
- A reasonable default TTL for DDISA records is 3600 seconds (1 hour).
- When a cached IdP endpoint becomes unreachable, SPs SHOULD re-resolve DNS before failing.
