---
status: pending
priority: p2
issue_id: "004"
tags: [code-review, security, filesystem]
dependencies: []
---

# P2: upsert_provider_url temp file lacks 0o600 permissions and no cleanup on failure

## Problem Statement

`upsert_provider_url` in `routes.rs` writes a temp file (`config.toml.tmp`) and then atomically renames it into place. Two issues:
1. The temp file is created with `std::fs::write()`, which uses the process umask (typically 0o644). The config file may contain API keys — it should be `0o600` (owner-read-write only).
2. If the `rename` call fails (e.g. cross-device), the temp file is left on disk at `config.toml.tmp` with the API keys in plaintext.

## Findings

- **File:** `crates/openfang-api/src/routes.rs` — `upsert_provider_url` function
- `std::fs::write(&tmp_path, ...)` does not set permissions
- No `defer`/`Drop` guard to delete the temp file on error paths
- `config.toml` itself is also not created with `0o600` if it doesn't exist yet

## Proposed Solutions

### Option A: Use `tempfile` crate for safe temp-file handling (Recommended)
**Pros:** `tempfile::NamedTempFile` handles cleanup automatically; can set permissions; already in workspace deps
**Cons:** Slightly more code
**Effort:** Small
**Risk:** Low

```rust
use tempfile::NamedTempFile;
use std::os::unix::fs::PermissionsExt;

let mut tmp = NamedTempFile::new_in(config_path.parent().unwrap())?;
tmp.write_all(toml::to_string_pretty(&doc)?.as_bytes())?;
// Set 0o600 before persisting
std::fs::set_permissions(tmp.path(), std::fs::Permissions::from_mode(0o600))?;
tmp.persist(config_path)?;
```

### Option B: Manually set permissions and add cleanup on error
**Pros:** No extra dependency
**Cons:** Easy to forget the cleanup path; more error-prone
**Effort:** Small
**Risk:** Medium

## Recommended Action

Option A: Replace `std::fs::write + rename` with `tempfile::NamedTempFile` + `persist()`.

## Technical Details

- **Affected files:** `crates/openfang-api/src/routes.rs`
- **Dep:** `tempfile` is already in `[workspace.dependencies]` in root `Cargo.toml`
- **Blocks:** Nothing
- **Database changes:** None
- **Platform note:** `PermissionsExt` is Unix-only; wrap with `#[cfg(unix)]` guard for Windows builds

## Acceptance Criteria

- [ ] Written config file has permissions `0o600` on Unix
- [ ] If `persist()` fails, no temp file remains in the config directory
- [ ] Existing tests pass
- [ ] Works on Windows (graceful fallback — no permissions API)

## Work Log

- 2026-02-27: Created from security-sentinel findings on kimi-for-coding PR
