---
status: pending
priority: p2
issue_id: "006"
tags: [code-review, testing, agent-native]
dependencies: []
---

# P2: Add kimi-for-coding coverage to catalog and alias tests

## Problem Statement

`test_chinese_providers_in_catalog` in `model_catalog.rs` asserts six Chinese providers but not `kimi-for-coding`. `test_chinese_model_aliases` checks that `find_model("kimi")` resolves, but that resolves to the `moonshot` provider, not `kimi-for-coding`. If the new provider entry is accidentally removed or renamed, no test catches it — silent regression risk.

Additionally, the `kimi` short alias points to `moonshot-v1-128k` (`.cn` domain) rather than to the new `kimi-for-coding` provider. This creates confusing behavior for agents that naturally try `model = "kimi"`.

## Findings

- **File:** `crates/openfang-runtime/src/model_catalog.rs` — `test_chinese_providers_in_catalog` (~line 2860)
- **File:** `crates/openfang-runtime/src/model_catalog.rs` — `test_chinese_model_aliases` (~line 2871)
- `catalog.get_provider("kimi-for-coding")` is never asserted
- All four kimi-k2 model variants are never individually asserted to resolve
- `find_model("kimi")` → `moonshot-v1-128k` is unintuitive now that kimi-for-coding exists

## Proposed Solutions

### Option A: Add assertions to existing tests + new dedicated test (Recommended)
**Pros:** Minimal diff; no new test file; immediate coverage
**Cons:** None
**Effort:** Trivial
**Risk:** None

```rust
// In test_chinese_providers_in_catalog:
assert!(catalog.get_provider("kimi-for-coding").is_some(), "kimi-for-coding provider missing");

// New test:
#[test]
fn test_kimi_for_coding_models() {
    let catalog = ModelCatalog::new();
    let provider = catalog.get_provider("kimi-for-coding").expect("kimi-for-coding missing");
    let model_ids: Vec<_> = provider.models.iter().map(|m| m.id.as_str()).collect();
    assert!(model_ids.contains(&"kimi-k2"), "kimi-k2 missing");
    assert!(model_ids.contains(&"kimi-k2-0905"), "kimi-k2-0905 missing");
    assert!(model_ids.contains(&"kimi-k2-thinking"), "kimi-k2-thinking missing");
    assert!(model_ids.contains(&"kimi-k2.5"), "kimi-k2.5 missing");

    // Alias resolution
    assert_eq!(catalog.find_model("kimi-coding"), Some("kimi-k2".to_string()));
    // kimi-for-coding base URL is moonshot.ai (not .cn)
    assert_eq!(provider.base_url, "https://api.moonshot.ai/v1");
}
```

## Recommended Action

Option A.

## Technical Details

- **Affected files:** `crates/openfang-runtime/src/model_catalog.rs`
- **Blocks:** Nothing
- **Database changes:** None

## Acceptance Criteria

- [ ] `test_chinese_providers_in_catalog` asserts `get_provider("kimi-for-coding").is_some()`
- [ ] New `test_kimi_for_coding_models` test verifies all 4 model variants and the `kimi-coding` alias
- [ ] `cargo test -p openfang-runtime --lib` passes

## Work Log

- 2026-02-27: Created from agent-native-reviewer findings on kimi-for-coding PR
