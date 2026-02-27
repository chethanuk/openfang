---
status: pending
priority: p3
issue_id: "008"
tags: [code-review, ux, configuration]
dependencies: []
---

# P3: upsert_provider_url TOML round-trip destroys user comments

## Problem Statement

`upsert_provider_url` in `routes.rs` reads `config.toml`, deserializes it with `toml::from_str` into a `toml::Value`, modifies the value, then serializes it back with `toml::to_string_pretty`. The `toml` crate does not preserve comments. A user's hand-crafted config with comments explaining their setup (e.g. `# my private vLLM server`) is silently stripped on the first API call that writes to config.

## Findings

- **File:** `crates/openfang-api/src/routes.rs` — `upsert_provider_url` uses `toml::from_str` + `toml::to_string_pretty`
- The `toml_edit` crate preserves comments and whitespace during round-trips
- `toml_edit` is not currently in `Cargo.toml` but is a well-maintained drop-in for this use case

## Proposed Solutions

### Option A: Replace toml::Value with toml_edit::DocumentMut (Recommended)
**Pros:** Preserves comments, formatting, and whitespace; the correct solution
**Cons:** New dependency
**Effort:** Small
**Risk:** Low

```rust
// Add to Cargo.toml: toml_edit = "0.22"
use toml_edit::{DocumentMut, value};

let content = std::fs::read_to_string(&config_path)?;
let mut doc: DocumentMut = content.parse()?;
doc["provider_urls"][&name] = value(base_url.as_str());
// write doc.to_string() back
```

### Option B: Append/patch with regex instead of round-tripping
**Pros:** No new dep
**Cons:** Brittle; error-prone for nested TOML; not recommended
**Effort:** Medium
**Risk:** High

## Recommended Action

Option A: Add `toml_edit` to workspace deps and use `DocumentMut` for config mutations.

## Technical Details

- **Affected files:** `crates/openfang-api/src/routes.rs`, root `Cargo.toml`
- **New dep:** `toml_edit = "0.22"`
- **Blocks:** Nothing
- **Database changes:** None

## Acceptance Criteria

- [ ] After `PUT /api/providers/{name}/url`, user comments in `config.toml` are preserved
- [ ] Blank lines and section ordering are preserved
- [ ] Existing tests pass
- [ ] `cargo clippy` clean

## Work Log

- 2026-02-27: Created from architecture-strategist findings on kimi-for-coding PR
