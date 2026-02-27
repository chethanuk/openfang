---
status: pending
priority: p3
issue_id: "007"
tags: [code-review, security, rate-limiting]
dependencies: []
---

# P3: GCRA rate-limiter underprices PUT /api/providers/{name}/url

## Problem Statement

`PUT /api/providers/{name}/url` costs only 5 GCRA tokens (the default for most routes). This endpoint is more expensive than typical reads — it triggers a reachability probe (external HTTP request), writes to disk, and potentially mutates all future LLM routing. An attacker with a valid API key could call it in a tight loop to perform port-scanning or resource exhaustion via repeated probe calls, at minimal token cost.

## Findings

- **File:** `crates/openfang-api/src/server.rs` — rate-limiter middleware configuration
- **File:** `crates/openfang-api/src/routes.rs` — `set_provider_url` handler
- Default per-route cost is 5 tokens; this route should cost ~100 to reflect its side-effect weight
- The probe makes an external HTTP request on every call — this should be limited aggressively

## Proposed Solutions

### Option A: Increase route cost to 100 tokens in the rate-limiter config
**Pros:** Simple; disproportionate cost deters abuse
**Cons:** Slightly breaks UX if a legitimate user clicks Save repeatedly
**Effort:** Trivial
**Risk:** None

### Option B: Deduplicate probe calls — skip the probe if URL hasn't changed
**Pros:** Reduces unnecessary external requests; better UX
**Cons:** More code
**Effort:** Small
**Risk:** Low

## Recommended Action

Option A as a quick fix; Option B as a follow-up.

## Technical Details

- **Affected files:** `crates/openfang-api/src/server.rs` (rate-limiter config)
- **Blocks:** Nothing
- **Database changes:** None

## Acceptance Criteria

- [ ] `PUT /api/providers/{name}/url` has a rate-limit cost of at least 100 tokens
- [ ] A tight loop of 20 calls within 1 second is rejected by the rate-limiter
- [ ] Existing tests pass

## Work Log

- 2026-02-27: Created from security-sentinel findings on kimi-for-coding PR
