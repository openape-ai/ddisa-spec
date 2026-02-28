# 02 — DNS Records

## Record Format

A domain that supports DDISA MUST publish a TXT record at the `_ddisa` subdomain:

```
_ddisa.<domain> TXT "v=ddisa1; idp=<url>; mode=<open|closed>"
```

## Fields

| Field | Required | Description |
|-------|----------|-------------|
| `v` | MUST | Protocol version. Currently `ddisa1`. |
| `idp` | MUST | The absolute URL of the domain's identity provider. MUST use `https`. |
| `mode` | MUST | `open` — any service provider MAY initiate authentication for users of this domain. `closed` — only pre-registered service providers MAY authenticate users. The IdP MUST reject unknown client IDs. |

Fields are separated by semicolons (`;`). Whitespace around separators is tolerated. Field order is not significant, but `v` SHOULD appear first.

## Examples

**A company using its own IdP (open federation):**
```
_ddisa.example.com. 3600 IN TXT "v=ddisa1; idp=https://id.example.com; mode=open"
```

**An organization using a managed IdP (restricted access):**
```
_ddisa.corp.example.org. 3600 IN TXT "v=ddisa1; idp=https://auth.provider.com/realms/corp; mode=closed"
```

**A small domain using a shared identity service:**
```
_ddisa.smallco.io. 3600 IN TXT "v=ddisa1; idp=https://login.identityprovider.com; mode=open"
```

## Resolution Algorithm

Given a user identifier in the form `user@domain`:

1. Extract the domain part from the email address.
2. Query DNS for `_ddisa.<domain> TXT`.
3. Parse the TXT record. If multiple TXT records exist at the name, use only the one beginning with `v=ddisa1`.
4. Validate that `v` equals `ddisa1`. If not, abort.
5. Extract the `idp` URL. Verify it uses `https`.
6. Proceed with the OAuth 2.0 Authorization Code + PKCE flow (see [03 — Authentication Flow](03-authentication-flow.md)).

If the DNS query returns `NXDOMAIN` or no matching TXT record is found, the domain does not support DDISA. The SP MUST NOT fall back to guessing.

## Caching and TTL

- SPs SHOULD respect the DNS TTL of the TXT record.
- SPs SHOULD NOT cache resolved IdP URLs for longer than the DNS TTL.
- A reasonable default TTL for DDISA records is 3600 seconds (1 hour).
- When a cached IdP endpoint becomes unreachable, SPs SHOULD re-resolve DNS before failing.
