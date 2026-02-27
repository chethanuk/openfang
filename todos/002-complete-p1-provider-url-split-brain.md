---
status: pending
priority: p1
issue_id: "002"
tags: [code-review, architecture, agent-native]
dependencies: []
---

# P1: set_provider_url writes catalog but resolve_base_url reads config — split-brain

## Problem Statement

`PUT /api/providers/{name}/url` updates two stores: `model_catalog.set_provider_url()` (in-memory) and `upsert_provider_url()` (config.toml on disk). However `resolve_base_url` in `kernel.rs` reads from `self.config.provider_urls`, which is the deserialized-at-boot `KernelConfig` struct. That struct is never updated in memory after boot. The result: a successful `PUT` response and a passing probe convince the caller the change took effect, but all subsequent agent spawns silently use the pre-boot URL because `resolve_base_url` reads the stale field.

## Findings

- **File:** `crates/openfang-kernel/src/kernel.rs` — `pub config: KernelConfig` at line ~42 (plain field, no lock)
- **File:** `crates/openfang-kernel/src/kernel.rs` — `resolve_base_url` reads `self.config.provider_urls.get(provider)`
- **File:** `crates/openfang-api/src/routes.rs` — `set_provider_url` calls `catalog.set_provider_url` but never touches `kernel.config`
- **File:** `crates/openfang-kernel/src/kernel.rs` — `ReloadProviderUrls` hot-action also only applies catalog overrides
- An agent that calls `PUT /api/providers/kimi-for-coding/url` successfully and then spawns a peer agent gets the old URL for all inferences — feature is silently broken

## Proposed Solutions

### Option A: Add a `RwLock<HashMap<String, String>>` runtime override map on AppState (Recommended)
**Pros:** Thread-safe; doesn't require changing `KernelConfig`; `resolve_base_url` can check this map first
**Cons:** Two sources of truth until a restart; need to populate from config on boot
**Effort:** Medium
**Risk:** Low

```rust
// AppState gains:
pub provider_url_overrides: Arc<RwLock<HashMap<String, String>>>,

// resolve_base_url updated to check overrides first:
fn resolve_base_url(&self, explicit_url: Option<&str>, provider: &str) -> Option<String> {
    explicit_url.map(|u| u.to_string())
        .or_else(|| self.url_overrides.read().get(provider).cloned())
        .or_else(|| self.config.provider_urls.get(provider).cloned())
}
```

### Option B: Wrap `config.provider_urls` in `Arc<RwLock<HashMap>>`
**Pros:** Single source of truth; no extra field
**Cons:** Requires changing `KernelConfig` struct and all callsites
**Effort:** Medium
**Risk:** Medium — wider diff

### Option C: Drive the update through `ReloadProviderUrls` hot-action after writing config
**Pros:** Reuses existing machinery
**Cons:** Hot-reload is async and eventual; race window between write and reload
**Effort:** Small
**Risk:** Medium

## Recommended Action

Option A: Add a `provider_url_overrides` map to `AppState`, populate from `config.provider_urls` at boot, update on every `set_provider_url` call, and check it first in `resolve_base_url`.

## Technical Details

- **Affected files:** `crates/openfang-kernel/src/kernel.rs`, `crates/openfang-api/src/routes.rs`, `crates/openfang-api/src/server.rs`
- **Blocks:** todo 005 (dashboard UI gap — fix this first)
- **Database changes:** None

## Acceptance Criteria

- [ ] After `PUT /api/providers/kimi-for-coding/url` with a new URL, a freshly spawned agent uses that URL (verified by checking driver config in integration test or by observing the outbound request destination)
- [ ] Restart the daemon — persisted URL is re-applied from config.toml
- [ ] `GET /api/providers` reflects the same URL that `resolve_base_url` will use
- [ ] All existing tests pass

## Work Log

- 2026-02-27: Created from agent-native-reviewer findings on kimi-for-coding PR
