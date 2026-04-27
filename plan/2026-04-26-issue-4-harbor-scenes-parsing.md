# Plan: Harbor `Job.from_yaml` scenes parsing (issue #4)

**Date:** 2026-04-26
**Issue:** [EYH0602/benchflow#4](https://github.com/EYH0602/benchflow/issues/4)
**Status:** revised post `/plan-eng-review` 2026-04-26 — scope expanded with 9 review decisions baked in

## Problem

`Job.from_yaml(...)` (Harbor-shape branch, `_from_harbor_yaml`) silently drops a top-level
`scenes:` block. Verified empirically:

```text
JobConfig dataclass fields: [..., 'context_root', 'exclude_tasks']  # no 'scenes'
Has 'scenes' attr on JobConfig: False
```

Multi-turn intent vanishes; the run falls through to `TrialConfig.from_legacy(...)` and
executes single-turn for every task in `datasets[].path`. No warning emitted.

The scene-aware machinery already exists everywhere else:
- `src/benchflow/_scene.py` — runtime
- `src/benchflow/trial.py` — `Role`, `Turn`, `Scene` dataclasses + `TrialConfig.scenes`
- `src/benchflow/trial_yaml.py:80` — per-trial loader handles `if "scenes" in raw:`

The Harbor `Job` loader is the only layer not taught about scenes.

## Goal

Teach `Job._from_harbor_yaml` to parse `scenes:`, plumb it through `JobConfig`, and execute
it via the scene-aware `Trial` path. Strictly additive — legacy single-turn YAMLs unchanged.

## Scope

**In (post-review, all decisions accepted):**
- Harbor YAML parsing in `_from_harbor_yaml` — top-level `scenes:` block.
- Native YAML parsing in `_from_native_yaml` — top-level `scenes:` block (decision 1A).
- `JobConfig` shape — add `scenes: list[Scene] | None = None` field.
- Unified `Job._run_single_task` `TrialConfig` construction (decision 2A — single call site, branch only on the `scenes` arg).
- Promote `_parse_scene` / `_parse_role` / `_parse_turn` → `parse_scene` / `parse_role` / `parse_turn` in `trial_yaml.py` (decision 2B).
- `parse_role` `model_name` fallback (decision 2C — `model = raw.get("model") or raw.get("model_name")`).
- Harbor `_from_harbor_yaml` reads top-level singular `agent:` and top-level `model_name:` as fallbacks when `agents:` array is absent (TODO 1, in-PR).
- Empty `scenes: []` emits a `logger.warning` and falls through to legacy (TODO 2, in-PR).
- When `cfg.scenes` is non-empty, derive `JobConfig.agent` / `JobConfig.model` from the first role of the first scene so `summary.json` reflects what actually ran (TODO 3, in-PR).
- Tests in `tests/test_yaml_config.py` and the `parse_role` test file — config-shape, behavior, literal issue #4 YAML, native parity, summary fidelity.

**Out:**
- No changes to `Trial`, `_scene.py`, `TrialConfig`, or `SDK.run`.
- No real-`Trial`-execution integration test — covered by existing `tests/test_scene_outbox_trial.py`.

## Files

- `src/benchflow/job.py` — `JobConfig.scenes`, scenes parsing in both loaders, top-level `agent:`/`model_name:` fallback, empty-scenes warning, summary-field derivation, unified `_run_single_task`.
- `src/benchflow/trial_yaml.py` — rename `_parse_*` → `parse_*`; `model_name` fallback in `parse_role`.
- `tests/test_yaml_config.py` — Harbor + native scenes parsing, behavior test for `_run_single_task`, issue #4 literal regression, additive guarantees, top-level singular-`agent:`/`model_name:` test, empty-scenes warning test, summary-field fidelity test.
- `tests/test_trial_yaml.py` (or wherever `parse_role` is tested) — `model_name` fallback test.

## Steps

### 1. Promote `_parse_*` to public + `model_name` fallback (`src/benchflow/trial_yaml.py`) — decisions 2B, 2C

- Rename `_parse_scene` → `parse_scene`, `_parse_role` → `parse_role`, `_parse_turn` → `parse_turn`. Update internal callers (one site at `trial_yaml.py:81`).
- In `parse_role`, change `model=raw.get("model")` → `model = raw.get("model") or raw.get("model_name")`.
- These functions are not in any `__all__` and not referenced by tests/external code (verified by grep) — rename is safe.

### 2. Add `scenes` to `JobConfig` (`src/benchflow/job.py:153`)

- Import `Scene` from `benchflow.trial` at module top (lazy import inside method if circulars surface).
- Add field: `scenes: list[Scene] | None = None`. Default `None`, **not** `[]`, so we can distinguish "not specified" from "explicitly empty" — empty fires a warning per Step 4.

### 3. Parse `scenes:` in both loaders — decision 1A

In **`_from_harbor_yaml` (`src/benchflow/job.py:312-368`)** and **`_from_native_yaml` (`src/benchflow/job.py:281-310`)**:

```python
from benchflow.trial_yaml import parse_scene
scenes_raw = raw.get("scenes")
if scenes_raw is None:
    scenes = None
elif not scenes_raw:  # explicit empty list
    logger.warning(
        "scenes: [] is empty — falling through to legacy single-turn execution. "
        "Remove the key entirely if that is the intent."
    )
    scenes = None
else:
    scenes = [parse_scene(s) for s in scenes_raw]
```

Pass `scenes=scenes` into the `JobConfig(...)` call in each loader.

### 4. Harbor top-level singular `agent:` / `model_name:` fallback (TODO 1)

In `_from_harbor_yaml`, before reading `agents[]`:

```python
agents = raw.get("agents") or []
agent_cfg = agents[0] if agents else {}
agent_name = agent_cfg.get("name") or raw.get("agent", DEFAULT_AGENT)
model_raw = agent_cfg.get("model_name") or raw.get("model_name") or raw.get("model")
model = effective_model(agent_name, model_raw)
```

This makes the issue #4 repro YAML (singular `agent: pi-acp` + top-level `model_name: vllm/...`) carry the user's intent into `JobConfig.agent` / `.model` instead of silently defaulting.

### 5. Derive summary fields from scenes when present (TODO 3)

After the `scenes` parse in both loaders, if `scenes` is non-empty, override the agent/model assigned to `JobConfig` so `summary.json` reflects what actually runs:

```python
if scenes and scenes[0].roles:
    primary_role = scenes[0].roles[0]
    agent_name = primary_role.agent
    model = primary_role.model or model
```

This matches what `TrialConfig.primary_agent` / `primary_model` already do at execution time.

### 6. Unified `Job._run_single_task` (`src/benchflow/job.py:418-438`) — decision 2A

Replace the two-branch construction with a single `TrialConfig(...)` call that decides scenes inline:

```python
from benchflow.trial import Scene, Trial, TrialConfig

resolved_skills = self._resolve_skills_dir(task_dir, cfg.skills_dir)
scenes = cfg.scenes or [
    Scene.single(
        agent=cfg.agent,
        model=cfg.model,
        prompts=cfg.prompts,
        skills_dir=resolved_skills,
    )
]
trial_config = TrialConfig(
    task_path=task_dir,
    scenes=scenes,
    job_name=self._job_name,
    jobs_dir=str(self._jobs_dir),
    environment=cfg.environment,
    sandbox_user=cfg.sandbox_user,
    sandbox_locked_paths=cfg.sandbox_locked_paths,
    sandbox_setup_timeout=cfg.sandbox_setup_timeout,
    context_root=cfg.context_root,
    agent_env=cfg.agent_env,
    skills_dir=resolved_skills,
    # Legacy mirror fields kept for any consumers reading them off TrialConfig:
    agent=cfg.agent,
    model=cfg.model,
    prompts=cfg.prompts,
)
trial = await Trial.create(trial_config)
return await trial.run()
```

Leave `_run_single_task_legacy` untouched — it's the test-mock SDK path, not relevant for scene-aware flows.

### 7. Tests

#### `tests/test_yaml_config.py`

**Test A — `test_harbor_yaml_parses_issue_4_repro` (decision 3B, regression):**
- Fixture YAML = the literal repro from issue #4 (2 scenes: `build-skill` + `answer`, role-level `model:`, second turn lacks `prompt:`).
- Assert `job._config.scenes` is a 2-element list.
- Assert `scenes[0].name == "build-skill"`, `scenes[0].roles[0].agent == "pi-acp"`, `scenes[0].roles[0].model == "vllm/Qwen/Qwen3.6-27B"`, `scenes[0].turns[0].prompt == "build the skill ..."`.
- Assert `scenes[1].name == "answer"`, `scenes[1].turns[0].prompt is None` (default-prompt path).
- Docstring: `"""Guards the fix from PR #<n> against the silent-scenes-drop regression in issue #4 — mirrors the literal repro YAML."""`

**Test B — `test_harbor_yaml_branch_passes_scenes_to_trial` (decision 3A, behavior):**
- `monkeypatch.setattr(benchflow.trial.Trial, "create", capturing_mock)` where the mock records its `TrialConfig` argument and returns a fake Trial whose `.run()` is a no-op.
- Build a Job from a Harbor scenes YAML, call `_run_single_task(task_dir, cfg)`.
- Assert the captured `TrialConfig.scenes == cfg.scenes` (same scene names, role count, turn count).
- This is the test that would have failed pre-fix.

**Test C — `test_harbor_yaml_without_scenes_unchanged`:**
- Existing harbor YAML (no `scenes:`). Assert `_config.scenes is None`.

**Test D — `test_native_yaml_parses_top_level_scenes` (decision 1A):**
- Native YAML with `scenes:` block. Assert `_config.scenes` populated.

**Test E — `test_native_yaml_without_scenes_unchanged` (decision 1A):**
- Native YAML, no `scenes:`. Assert `_config.scenes is None`.

**Test F — `test_harbor_yaml_top_level_singular_agent_and_model_name` (TODO 1):**
- YAML with `agent: pi-acp` (singular), `model_name: vllm/Qwen/Qwen3.6-27B` (top-level), no `agents:` array.
- Assert `_config.agent == "pi-acp"`, `_config.model == "vllm/Qwen/Qwen3.6-27B"`.

**Test G — `test_harbor_yaml_empty_scenes_warns` (TODO 2):**
- YAML with `scenes: []`. Assert `_config.scenes is None` AND a warning was logged (use `caplog`).

**Test H — `test_harbor_yaml_summary_fields_match_first_scene_role` (TODO 3):**
- YAML with `agents: [{name: x, model_name: y}]` AND `scenes: [{roles: [{name: r, agent: real-agent, model: real-model}], ...}]`.
- Assert `_config.agent == "real-agent"`, `_config.model == "real-model"` (scene roles win for summary).

#### `tests/test_trial_yaml.py` (or co-located with parse helpers)

**Test I — `test_parse_role_model_name_fallback` (decision 2C):**
- `parse_role({"name": "r", "agent": "a", "model_name": "m"})` → `Role(model="m")`.
- `parse_role({"name": "r", "agent": "a", "model": "m"})` → `Role(model="m")` (existing path, smoke).
- `parse_role({"name": "r", "agent": "a"})` → `Role(model=None)`.

## Risks / open questions

- **`Scene` symbol collision.** Two classes named `Scene`: `trial.py` (dataclass, the right one) and `_scene.py` (runtime). All imports must be `from benchflow.trial import Scene`. Test assertions should reference `benchflow.trial.Scene` to avoid future drift.

- **`skills_dir` precedence.** `JobConfig.skills_dir` (job-level) and `Scene.skills_dir` (per-scene) coexist. Today `from_legacy` injects job-level into the synthesized single Scene. With explicit `scenes:`, per-scene `skills_dir` (parsed by `parse_scene`) wins; job-level still flows through `TrialConfig.skills_dir` for trial-wide injection. Matches existing `trial_yaml` semantics — no special handling needed.

- **`agent_env` plumbing.** Harbor's top-level `environment.env` populates `JobConfig.agent_env`. Scene roles may also carry `env:`. Both flow through `TrialConfig` already — no change.

- **Empty `scenes: []`.** Per TODO 2 (in-PR), this now logs a warning and falls through to legacy execution. Test G covers this.

- **Public-API surface for `parse_*`.** Renaming drops the underscore; downstream code that imported `_parse_scene` would break. `grep -r "_parse_scene\|_parse_role\|_parse_turn"` showed only the internal `trial_yaml.py:81` reference. Safe.

## Verification

```bash
.venv/bin/python -m pytest tests/test_yaml_config.py tests/test_trial_yaml.py -v
.venv/bin/ty check src/benchflow/job.py src/benchflow/trial_yaml.py
ruff check src/benchflow/job.py src/benchflow/trial_yaml.py tests/test_yaml_config.py
```

Plus the issue's original repro: `j._config.scenes` is a 2-element list, `j._config.agent == "pi-acp"`, `j._config.model == "vllm/Qwen/Qwen3.6-27B"`.

## Estimated diff (revised)

- `src/benchflow/job.py`: ~50 lines (scenes parsing in 2 loaders, summary derivation, top-level singular fallback, unified `_run_single_task`).
- `src/benchflow/trial_yaml.py`: ~5 lines (rename + `model_name` fallback).
- `tests/test_yaml_config.py`: ~120 lines (Tests A–H).
- `tests/test_trial_yaml.py`: ~20 lines (Test I).

Total: ~195 lines added, ~15 removed (the dual-branch `_run_single_task` collapses).

## Out of scope (deliberate)

- Real `Trial.run` integration test for scenes — covered by `tests/test_scene_outbox_trial.py`.
- Conflict warning when both top-level `agents:` and scene roles define agents with different identities — TODO 3 already prefers scene roles for summary; explicit conflict detection is a future tightening.
- Refactoring `_run_single_task_legacy` to also support scenes — the legacy path exists only for test-mock SDK compat; real execution uses the unified path.

## GSTACK REVIEW REPORT

| Review | Trigger | Why | Runs | Status | Findings |
|--------|---------|-----|------|--------|----------|
| CEO Review | `/plan-ceo-review` | Scope & strategy | 0 | — | bug fix; CEO review optional |
| Codex Review | `/codex review` | Independent 2nd opinion | 0 | — | skipped — 25-LOC bug fix, not strategy |
| Eng Review | `/plan-eng-review` | Architecture & tests (required) | 1 | CLEAR (PLAN) | 6 issues found, all 9 decisions captured; 1 critical gap (empty-scenes silent fall-through) closed by in-PR warning |
| Design Review | `/plan-design-review` | UI/UX gaps | 0 | — | not applicable — library bug fix |
| DX Review | `/plan-devex-review` | Developer experience gaps | 0 | — | not applicable |

- **UNRESOLVED:** 0
- **VERDICT:** ENG CLEARED — ready to implement. Plan body has been revised to absorb all 9 review decisions (1A, 2A, 2B, 2C, 3A, 3B, TODOs 1/2/3 — all in-PR).
