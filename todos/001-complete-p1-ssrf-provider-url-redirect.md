---
status: pending
priority: p1
issue_id: "001"
tags: [code-review, security, ssrf]
dependencies: []
---

# P1: SSRF via unauthenticated PUT /api/providers/{name}/url

## Problem Statement

`PUT /api/providers/{name}/url` accepts any URL from any caller (no auth required by default) and immediately calls `probe_provider` on the submitted URL to verify reachability. An attacker can supply `http://169.254.169.254/latest/meta-data/` (AWS IMDS) or any internal service — the probe will exfiltrate its response via timing, and all subsequent LLM calls for that provider will be redirected to the attacker-controlled server, forwarding the API key in the `Authorization` header.

## Findings

- **File:** `crates/openfang-api/src/routes.rs` — `set_provider_url` handler (~line 5930–5970)
- **File:** `crates/openfang-api/src/routes.rs` — `upsert_provider_url` helper
- **File:** `crates/openfang-kernel/src/kernel.rs` — `resolve_base_url` reads the config that gets updated
- No auth guard on the `PUT /api/providers/{name}/url` route in `server.rs`
- `probe_provider` follows HTTP 3xx redirects with no `redirect::Policy::none()` set on the `reqwest::Client`
- Supplied URL is not validated against a private-IP block-list before the probe or before writing to config

## Proposed Solutions

### Option A: Validate URL against private-IP allowlist before accepting (Recommended)
**Pros:** Eliminates SSRF at the boundary; simple to implement; doesn't require auth changes
**Cons:** Requires parsing URLs and resolving hostnames; DNS TOCTOU gap
**Effort:** Small
**Risk:** Low — defense-in-depth even if bypassed

```rust
fn is_safe_url(url: &str) -> bool {
    let parsed = url::Url::parse(url).ok()?;
    let host = parsed.host_str()?;
    // reject loopback, link-local (169.254.*), private RFC1918, metadata IPs
    !is_private_or_loopback(host)
}
```

### Option B: Require API key authentication on this route
**Pros:** Prevents unauthenticated callers from triggering the side-effect at all
**Cons:** Existing `api_key = ""` (disabled) installs are immediately unprotected again; orthogonal to SSRF
**Effort:** Small
**Risk:** Medium — auth alone doesn't prevent SSRF once an authenticated attacker exists

### Option C: Disable redirect-following on the probe client + add IP validation
**Pros:** Belt-and-suspenders: even if a URL passes validation, redirect chains can't escape to internal IPs
**Cons:** Legitimate providers behind redirecting CDNs may fail the probe
**Effort:** Small
**Risk:** Low

## Recommended Action

Implement Option A + Option C together: validate the URL before the probe (reject private-IP destinations) and set `redirect::Policy::none()` on the reqwest client used by `probe_provider`.

## Technical Details

- **Affected files:** `crates/openfang-api/src/routes.rs`, `crates/openfang-api/src/server.rs`
- **Crate to add:** `ipnetwork` or inline CIDR check using `std::net::IpAddr`
- **Blocks:** None
- **Database changes:** None

## Acceptance Criteria

- [ ] `PUT /api/providers/{name}/url` with `url = "http://169.254.169.254/"` returns 400
- [ ] `PUT /api/providers/{name}/url` with `url = "http://10.0.0.1/"` returns 400
- [ ] `PUT /api/providers/{name}/url` with `url = "http://192.168.1.1/"` returns 400
- [ ] `PUT /api/providers/{name}/url` with a valid public HTTPS URL succeeds
- [ ] `probe_provider` does not follow redirects (no 3xx chaining)
- [ ] Existing tests still pass

## Work Log

- 2026-02-27: Created from security-sentinel + agent-native review of kimi-for-coding PR
