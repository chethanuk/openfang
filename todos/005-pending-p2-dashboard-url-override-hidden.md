---
status: pending
priority: p2
issue_id: "005"
tags: [code-review, agent-native, ui]
dependencies: ["002"]
---

# P2: Dashboard URL override UI hidden for key-required providers (kimi-for-coding invisible)

## Problem Statement

`GET /api/providers` sets `is_local: true` only for providers where `!key_required`. The dashboard template `index_body.html` wraps the "Base URL" editor in `<template x-if="p.is_local">`, so users cannot override the base URL for `kimi-for-coding` (or any other cloud provider) from the dashboard. The API endpoint `PUT /api/providers/{name}/url` works fine — so agents have access but users do not. This inverts the normal agent-native parity requirement.

## Findings

- **File:** `crates/openfang-api/src/routes.rs` — `get_providers` handler sets `is_local: !p.key_required`
- **File:** `crates/openfang-api/static/index_body.html` — `<template x-if="p.is_local">` around Base URL input (~line 2890)
- **File:** `crates/openfang-api/static/index_body.html` — `loadProviders` JS pre-populates `providerUrlInputs` only for `is_local` providers
- Intended use case for `is_local`: Ollama, vLLM, LM Studio — local inference servers
- Feature was extended to all providers via the API route without updating the UI guard

## Proposed Solutions

### Option A: Show URL override for all providers, not just is_local (Recommended)
**Pros:** Full parity; exposes existing API capability in UI; minimal diff
**Cons:** UI becomes slightly more complex (more inputs visible)
**Effort:** Small
**Risk:** Low

```html
<!-- Change from: -->
<template x-if="p.is_local">
<!-- To: -->
<template x-if="true">
```

And in the JS, pre-populate `providerUrlInputs[p.id] = p.base_url` for all providers.

### Option B: Add a separate `is_configurable_url` field to the provider API response
**Pros:** More granular control over which providers show the UI
**Cons:** Extra field, extra maintenance
**Effort:** Small
**Risk:** Low

### Option C: Keep is_local UI, add a separate "Advanced" panel for URL overrides
**Pros:** Cleaner UX separation
**Cons:** More work; delays fix
**Effort:** Medium
**Risk:** Low

## Recommended Action

Option A: Change the `x-if` condition and update the JS pre-population to apply to all providers. Keep the existing `is_local` logic for any other UI element that relies on it.

## Technical Details

- **Affected files:** `crates/openfang-api/static/index_body.html`
- **Depends on:** todo 002 (split-brain fix) — once split-brain is fixed, the UI change becomes useful
- **Database changes:** None

## Acceptance Criteria

- [ ] `kimi-for-coding` shows a "Base URL" input in the dashboard providers tab
- [ ] Typing a URL and clicking Save calls `PUT /api/providers/kimi-for-coding/url`
- [ ] The input is pre-populated with the current `base_url` from the API
- [ ] Existing local providers (ollama, vllm) still show the input as before
- [ ] After todo 002 is fixed, the saved URL is immediately used for new agent spawns

## Work Log

- 2026-02-27: Created from agent-native-reviewer findings on kimi-for-coding PR
