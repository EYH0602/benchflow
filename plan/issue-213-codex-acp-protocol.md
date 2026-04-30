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
(`find_provider` returns `None`, the broken branches never fire). It also
cannot work for the only multi-endpoint provider currently in the registry
(zai): zai exposes anthropic-messages and openai chat-completions endpoints,
neither of which Codex CLI can speak (chat-completions was removed upstream,
Responses-only). So the zai+codex-acp routing path that
`tests/test_resolve_env_helpers.py:145` exercises is plumbing for a
physically-impossible integration — env vars get set "correctly" but the
underlying codex process cannot connect. The label is also misleading about
what codex-acp can do.

`codex-acp` is a thin shim that exists solely to wrap OpenAI's Codex CLI. It
has exactly one valid backend (OpenAI Responses API). There is no protocol
negotiation to express.

## Fix

Minimal, honest change in `src/benchflow/agents/registry.py:279-312`:

- `api_protocol="openai-completions"` → `api_protocol=""`. Empty matches
  reality: single-target agent, no multi-endpoint routing. Add a short
  comment in the same style as the `gemini` agent at `registry.py:326-329`
  explaining why it's intentionally empty (folds in former follow-up #3).
- Remove the `env_mapping` block entirely. `BENCHFLOW_PROVIDER_BASE_URL` and
  `BENCHFLOW_PROVIDER_API_KEY` translation does nothing useful for codex-acp
  today — Codex ignores `OPENAI_BASE_URL`, and `OPENAI_API_KEY` is already
  required directly via `requires_env`.
- Keep `requires_env=["OPENAI_API_KEY"]`, `credential_files`,
  `subscription_auth`, and `disallow_web_tools_launch_suffix` unchanged —
  those are correct.
- Add `"codex-acp"` to the `no_env_mapping` set in
  `tests/test_registry_invariants.py:128` so the negative invariant is
  sticky (prevents accidental re-add).

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

1. `tests/test_registry_invariants.py` — passes without modification.
   `test_agent_field_shapes` accepts empty `api_protocol`
   (line 31: `VALID_API_PROTOCOLS = {"", "anthropic-messages", "openai-completions"}`).
   `test_agent_api_protocol_has_provider_endpoint` (line 144) skips agents
   with empty `api_protocol`.
2. **Tests that break and need updating** (load-bearing — both must be
   handled in the same PR or CI fails):
   - `tests/test_agent_registry.py::test_codex_acp_has_mapping`
     (line 27-30) — delete; env_mapping is now empty by design.
   - `tests/test_resolve_env_helpers.py::test_zai_picks_openai_endpoint_for_codex_agent`
     (line 145-151) — delete. The integration it asserts (codex-acp routing
     to zai's openai endpoint) cannot work end-to-end: zai's openai endpoint
     speaks chat-completions, which Codex CLI no longer supports
     (`CHAT_WIRE_API_REMOVED_ERROR`). The test was pinning env-var shape for
     a physically impossible integration.
3. **Add regression test naming this PR/issue** (per CLAUDE.md regression
   rule). Suggested home: `tests/test_agent_registry.py`. Asserts:
   - `AGENTS["codex-acp"].api_protocol == ""`
   - `AGENTS["codex-acp"].env_mapping == {}`
   Docstring must reference issue #213 and the upstream
   `CHAT_WIRE_API_REMOVED_ERROR` reason. The negative invariant added to
   `no_env_mapping` (see Fix above) covers env_mapping; this test pins the
   protocol label too.
4. Run the full suite: `.venv/bin/python -m pytest tests/`.
5. Type and lint: `.venv/bin/ty check src/` and `ruff check .`.

## Follow-ups (separate issues / PRs)

- Investigate skillsbench/750's actual `-32603` cause (network policy under
  `allow_internet=false` + baked image).
- If/when a Responses-API third-party provider lands, build a small
  `codex_acp_launcher.py` that reads `BENCHFLOW_PROVIDER_BASE_URL` /
  `BENCHFLOW_PROVIDER_MODEL` and injects them as
  `-c openai_base_url=... -c model=...` (or `-c model_providers.X=... -c model_provider=X`)
  before exec'ing `codex-acp`. Add `"openai-responses"` to the protocol
  enum at that point.

## GSTACK REVIEW REPORT

| Review | Trigger | Why | Runs | Status | Findings |
|--------|---------|-----|------|--------|----------|
| CEO Review | `/plan-ceo-review` | Scope & strategy | 0 | — | not run (small config fix, no scope question) |
| Codex Review | `/codex review` | Independent 2nd opinion | 0 | — | not run |
| Eng Review | `/plan-eng-review` | Architecture & tests (required) | 1 | CLEAR (PLAN) | 1 issue surfaced + resolved (option A): `test_resolve_env_helpers.py:145` was missed by original verification list; plan now lists both load-bearing tests to delete and adds a PR-#213-named regression test |
| Design Review | `/plan-design-review` | UI/UX gaps | 0 | — | N/A (no UI) |
| DX Review | `/plan-devex-review` | Developer experience gaps | 0 | — | not run |

**UNRESOLVED:** 0
**VERDICT:** ENG CLEARED — ready to implement. zai+codex-acp is a
physically impossible integration (chat-completions removed upstream;
zai has no Responses endpoint), so deleting
`test_zai_picks_openai_endpoint_for_codex_agent` is correct and not a
regression.
