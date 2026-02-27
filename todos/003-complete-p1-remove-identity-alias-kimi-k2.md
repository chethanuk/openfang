---
status: pending
priority: p1
issue_id: "003"
tags: [code-review, simplicity, bug]
dependencies: []
---

# P1: Remove no-op identity alias ("kimi-k2", "kimi-k2") from model_catalog

## Problem Statement

`crates/openfang-runtime/src/model_catalog.rs` contains the alias `("kimi-k2", "kimi-k2")` — a tuple where source and target are identical. This is a pure no-op: it aliases a model to itself, adding dead weight to the alias table, potentially masking the intent of the surrounding aliases, and suggesting incorrect understanding of the alias API.

## Findings

- **File:** `crates/openfang-runtime/src/model_catalog.rs` — kimi-for-coding alias block
- The useful alias immediately after is `("kimi-coding", "kimi-k2")` which correctly maps the short name to the model
- The identity alias `("kimi-k2", "kimi-k2")` does nothing and should be removed
- No test would fail if it were removed — it adds zero functionality

## Proposed Solutions

### Option A: Delete the identity alias line (Recommended)
**Pros:** Simplest possible fix; no behavior change; reduces noise
**Cons:** None
**Effort:** Trivial (1 line)
**Risk:** None

```rust
// REMOVE this line:
("kimi-k2", "kimi-k2"),
// Keep this:
("kimi-coding", "kimi-k2"),
```

### Option B: Replace with a useful alias (e.g. `("kimi-k2-latest", "kimi-k2")`)
**Pros:** Adds value while fixing the no-op
**Cons:** Requires knowing what alias would actually be useful
**Effort:** Trivial
**Risk:** None

## Recommended Action

Option A: Delete the one-line identity alias.

## Technical Details

- **Affected files:** `crates/openfang-runtime/src/model_catalog.rs`
- **Blocks:** Nothing
- **Database changes:** None

## Acceptance Criteria

- [ ] Identity alias `("kimi-k2", "kimi-k2")` no longer exists in model_catalog.rs
- [ ] `("kimi-coding", "kimi-k2")` alias still exists and resolves correctly
- [ ] `cargo test -p openfang-runtime --lib` passes

## Work Log

- 2026-02-27: Created from code-simplicity-reviewer findings on kimi-for-coding PR
