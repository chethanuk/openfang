# Feature: Kimi (Coding) Provider — Moonshot AI Kimi K2

> **Commits:** `c083c0d` → `7da9e55` on `main`
> **Issue:** [#43 Add kimi-for-coding provider](https://github.com/RightNow-AI/openfang/issues/43)

---

## What was built

Added **`kimi-for-coding`** as a fully-integrated LLM provider backed by Moonshot AI's Kimi K2 model family — a code-optimized reasoning model with 128K–256K context windows at $0.60/$2.50 per million tokens.

---

## Live API Demo

### 1. Provider registered in catalog

```
GET /api/providers
```

```json
{
  "id": "kimi-for-coding",
  "display_name": "Kimi (Coding)",
  "base_url": "https://api.moonshot.ai/v1",
  "api_key_env": "MOONSHOT_API_KEY",
  "key_required": true,
  "model_count": 4
}
```

### 2. All 4 model variants available

```
GET /api/models?provider=kimi-for-coding
```

| Model | Context | Input $/M | Output $/M | Vision |
|-------|---------|-----------|-----------|--------|
| `kimi-k2` | 128K | $0.60 | $2.50 | ✗ |
| `kimi-k2-0905` | 256K | $0.60 | $2.50 | ✗ |
| `kimi-k2-thinking` | 256K | $0.60 | $2.50 | ✗ |
| `kimi-k2.5` | 256K | $0.60 | $2.50 | ✓ |

### 3. SSRF protection blocks private-IP URLs

```
PUT /api/providers/kimi-for-coding/url  {"base_url": "http://169.254.169.254/"}
→ 400 Bad Request

PUT /api/providers/kimi-for-coding/url  {"base_url": "http://192.168.1.1/"}
→ 400 Bad Request

PUT /api/providers/kimi-for-coding/url  {"base_url": "http://localhost/"}
→ 400 Bad Request
```

### 4. Valid URL override works (and persists in-memory immediately)

```
PUT /api/providers/kimi-for-coding/url  {"base_url": "https://api.moonshot.ai/v1"}
→ {"base_url":"https://api.moonshot.ai/v1","latency_ms":227,"provider":"kimi-for-coding","reachable":false,"status":"saved"}
```

The URL is immediately visible to `resolve_base_url` (no restart required — fixed split-brain bug).

### 5. Dashboard shows URL override for all providers

The Base URL editor in Settings → Providers is now visible for `kimi-for-coding` and all cloud providers, not just local (Ollama/vLLM) providers.

---

## Using the provider

### In config.toml

```toml
[default_model]
provider = "kimi-for-coding"
model = "kimi-k2"
api_key_env = "MOONSHOT_API_KEY"
```

### Via environment

```bash
export MOONSHOT_API_KEY="your-moonshot-api-key"
openfang start
```

### Via the init wizard

The `openfang init` wizard now includes **Kimi (Coding)** in the provider selection list.

### In an agent manifest

```json
{
  "provider": "kimi-for-coding",
  "model": "kimi-k2",
  "system": "You are an expert software engineer."
}
```

### Via alias

```bash
# Short alias
model = "kimi-coding"   # resolves to kimi-k2

# Full ID
model = "kimi-k2"
```

---

## Security fixes shipped with this feature

| Issue | Fix |
|-------|-----|
| SSRF via provider URL endpoint | Private-IP block-list in `is_safe_provider_url()` |
| SSRF via redirect chains in probe | `redirect::Policy::none()` on probe reqwest client |
| Config file world-readable | `tempfile::NamedTempFile` + `0o600` permissions |
| TOML comments destroyed on write | Replaced `toml::Value` with `toml_edit::DocumentMut` |
| Provider URL split-brain | `DashMap provider_url_overrides` on kernel |
| Rate-limiter underpriced mutation | PUT /url now costs 50 tokens (was 5) |

---

## Test coverage

```
openfang-api    36 tests  ✓
openfang-kernel 203 tests ✓
openfang-runtime 614 tests ✓ (includes new test_kimi_for_coding_models)
─────────────────────────────
Total           853 tests  0 failed
```

New test: `test_kimi_for_coding_models` verifies all 4 model variants, the `kimi-coding` alias, and the moonshot.ai base URL.
