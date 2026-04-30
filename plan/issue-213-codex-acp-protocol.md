# Plan: fix codex-acp `api_protocol` mislabel (issue #213)

## Problem

`src/benchflow/agents/registry.py:279-312` registers `codex-acp` with
`api_protocol="openai-completions"` and an `env_mapping` that translates
`BENCHFLOW_PROVIDER_BASE_URL` to `OPENAI_BASE_URL`. Both are wrong:

1. **Wrong protocol label.** Codex CLI speaks only the OpenAI Responses API.
   Upstream removed chat-completions support entirely
   (`codex-rs/model-provider-info/src/lib.rs:48-78` — `WireApi` has a single
   variant `Responses`; `wire_api = "chat"` raises `CHAT_WIRE_API_REMOVED_ERROR`
   pointing at openai/codex#7782).
2. **Dead env mapping.** Codex does not read the `OPENAI_BASE_URL` env var. It
   reads `openai_base_url` from TOML config (`codex-rs/config/src/config_toml.rs:284-285`)
   or accepts `-c openai_base_url=...` overrides via codex-acp's pass-through
   (`zed-industries/codex-acp src/lib.rs:34-54`). The env var is never consulted
   for the `openai` provider.

In practice the misconfiguration is inert for the default OpenAI flow
(`find_provider` returns `None`, the broken branches never fire), but it
silently misroutes if anyone pairs codex-acp with a multi-endpoint provider
(zai), and it misleads readers about what codex-acp can do.

`codex-acp` is a thin shim that exists solely to wrap OpenAI's Codex CLI. It
has exactly one valid backend (OpenAI Responses API). There is no protocol
negotiation to express.

## Fix

Minimal, honest change in `src/benchflow/agents/registry.py:279-312`:

- `api_protocol="openai-completions"` → `api_protocol=""`.
  Empty matches reality: single-target agent, no multi-endpoint routing.
- Remove the `env_mapping` block entirely. `BENCHFLOW_PROVIDER_BASE_URL` and
  `BENCHFLOW_PROVIDER_API_KEY` translation does nothing useful for codex-acp
  today — Codex ignores `OPENAI_BASE_URL`, and `OPENAI_API_KEY` is already
  required directly via `requires_env`.
- Keep `requires_env=["OPENAI_API_KEY"]`, `credential_files`,
  `subscription_auth`, and `disallow_web_tools_launch_suffix` unchanged —
  those are correct.

Net diff is roughly:

```diff
         requires_env=["OPENAI_API_KEY"],
-        api_protocol="openai-completions",
-        env_mapping={
-            "BENCHFLOW_PROVIDER_BASE_URL": "OPENAI_BASE_URL",
-            "BENCHFLOW_PROVIDER_API_KEY": "OPENAI_API_KEY",
-        },
+        api_protocol="",
         credential_files=[
```

## What this is *not*

- Not adding a new `"openai-responses"` enum value. There is no second
  Responses-API-capable provider in the registry, and codex-acp would never
  benefit from per-protocol endpoint selection (it has only one backend).
  Re-evaluate if a third-party Responses-capable provider is added.
- Not wiring `BENCHFLOW_PROVIDER_BASE_URL` through to `-c openai_base_url=...`.
  That requires a launcher script (compare `pi_acp_launcher.py`) and is only
  worth building when a real Responses-capable third-party endpoint exists.
  File as a follow-up.
- Not the cause of skillsbench/750's `-32603 Internal error` — that's a
  separate diagnosis (likely `allow_internet=false` blocking
  `api.openai.com`, or auth). EYH0602's hypothesis tying it to the protocol
  label does not survive reading `_agent_env.py:115-167` for the default
  OpenAI path.

## Verification

1. `tests/test_registry_invariants.py` — should pass without modification.
   `test_agent_field_shapes` accepts empty `api_protocol`
   (line 31: `VALID_API_PROTOCOLS = {"", "anthropic-messages", "openai-completions"}`).
   `test_agent_api_protocol_has_provider_endpoint` (line 144) skips agents
   with empty `api_protocol`.
2. `tests/test_registry_invariants.py::test_agent_negative_config_invariants`
   does not list codex-acp under `no_env_mapping`. Removing env_mapping is
   still allowed; if we want to make it sticky, append `"codex-acp"` to that
   set in a follow-up.
3. Run the full suite: `.venv/bin/python -m pytest tests/`.
4. Type and lint: `.venv/bin/ty check src/` and `ruff check .`.
5. Spot check: any test that asserts on codex-acp's `env_mapping` will need
   updating. Likely candidates in `tests/test_agent_registry.py` or
   `tests/test_providers.py` — check before committing.

## Follow-ups (separate issues / PRs)

- Investigate skillsbench/750's actual `-32603` cause (network policy under
  `allow_internet=false` + baked image).
- If/when a Responses-API third-party provider lands, build a small
  `codex_acp_launcher.py` that reads `BENCHFLOW_PROVIDER_BASE_URL` /
  `BENCHFLOW_PROVIDER_MODEL` and injects them as
  `-c openai_base_url=... -c model=...` (or `-c model_providers.X=... -c model_provider=X`)
  before exec'ing `codex-acp`. Add `"openai-responses"` to the protocol
  enum at that point.
- Consider documenting in `registry.py:279` that codex-acp is intentionally
  single-backend and does not honor `BENCHFLOW_PROVIDER_*` env vars.
