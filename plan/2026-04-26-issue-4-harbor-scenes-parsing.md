# Plan: Harbor `Job.from_yaml` scenes parsing (issue #4)

**Date:** 2026-04-26
**Issue:** [EYH0602/benchflow#4](https://github.com/EYH0602/benchflow/issues/4)
**Status:** drafted, not yet executed

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

**In:**
- Harbor YAML parsing in `_from_harbor_yaml`.
- `JobConfig` shape (add `scenes` field).
- `Job._run_single_task` branching.
- Regression test in `tests/test_yaml_config.py`.

**Out:**
- `_from_native_yaml` (already covered indirectly via the per-trial `trial_yaml` loader).
- No changes to `Trial`, `_scene.py`, `TrialConfig`, or `SDK.run`.
- No `model_name` fallback inside scene roles for now — defer unless requested. Existing
  `model:` key works as-is for the issue's repro YAML.

## Files

- `src/benchflow/job.py` — add field, parse, branch.
- `tests/test_yaml_config.py` — add regression test(s).

## Steps

### 1. Add `scenes` to `JobConfig` (`src/benchflow/job.py:153`)

- Import `Scene` from `benchflow.trial` at module top (lazy import inside method if circulars).
- Add field: `scenes: list[Scene] | None = None`
  - Default `None`, **not** `[]`, so we can distinguish "not specified" from "explicitly empty"
    in the branch in step 3.

### 2. Parse `scenes:` in `_from_harbor_yaml` (`src/benchflow/job.py:312-368`)

After existing parses (orchestrator, datasets, env), add:

```python
from benchflow.trial_yaml import _parse_scene  # or import at module top
scenes_raw = raw.get("scenes")
scenes = [_parse_scene(s) for s in scenes_raw] if scenes_raw else None
```

Pass `scenes=scenes` into `JobConfig(...)`.

### 3. Branch in `Job._run_single_task` (`src/benchflow/job.py:418-438`)

When `cfg.scenes` is truthy, build the trial config directly instead of via `from_legacy`:

```python
from benchflow.trial import Trial, TrialConfig

if cfg.scenes:
    trial_config = TrialConfig(
        task_path=task_dir,
        scenes=cfg.scenes,
        job_name=self._job_name,
        jobs_dir=str(self._jobs_dir),
        environment=cfg.environment,
        sandbox_user=cfg.sandbox_user,
        sandbox_locked_paths=cfg.sandbox_locked_paths,
        sandbox_setup_timeout=cfg.sandbox_setup_timeout,
        context_root=cfg.context_root,
        agent_env=cfg.agent_env,
        skills_dir=self._resolve_skills_dir(task_dir, cfg.skills_dir),
    )
else:
    trial_config = TrialConfig.from_legacy(
        task_path=task_dir,
        agent=cfg.agent,
        model=cfg.model,
        prompts=cfg.prompts,
        agent_env=cfg.agent_env,
        job_name=self._job_name,
        jobs_dir=str(self._jobs_dir),
        environment=cfg.environment,
        skills_dir=self._resolve_skills_dir(task_dir, cfg.skills_dir),
        sandbox_user=cfg.sandbox_user,
        sandbox_locked_paths=cfg.sandbox_locked_paths,
        sandbox_setup_timeout=cfg.sandbox_setup_timeout,
        context_root=cfg.context_root,
    )
```

Leave `_run_single_task_legacy` untouched — it's the test-mock SDK path, not relevant for
scene-aware flows.

### 4. Regression test (`tests/test_yaml_config.py`)

**Test A — `test_harbor_yaml_parses_top_level_scenes`:**
- tmp_path Harbor YAML with `agents:` + `datasets:` + `scenes:` (one scene, one role, one turn).
- `Job.from_yaml(...)`.
- Assert `job._config.scenes is not None`.
- Assert `len(job._config.scenes) == 1`.
- Assert `job._config.scenes[0].name`, `roles[0].agent`, `turns[0].prompt` match the YAML.
- Docstring: `"""Guards the fix from PR #<n> against the regression where Harbor YAML silently dropped scenes (issue #4)."""`

**Test B — `test_harbor_yaml_without_scenes_unchanged`:**
- Existing harbor YAML (no `scenes:`).
- Assert `job._config.scenes is None`.
- Locks in the additive guarantee — legacy YAMLs see no behavior change.

## Risks / open questions

- **`Scene` symbol collision.** Two classes named `Scene`: `trial.py` (dataclass, the right one)
  and `_scene.py` (runtime). All imports must be `from benchflow.trial import Scene`. Test
  assertions should reference `benchflow.trial.Scene` to avoid future drift.

- **`skills_dir` precedence.** `JobConfig.skills_dir` (job-level) and `Scene.skills_dir`
  (per-scene) coexist. Today `from_legacy` injects job-level into the synthesized single
  Scene. With explicit `scenes:`, per-scene `skills_dir` (parsed by `_parse_scene`) wins;
  job-level still flows through `TrialConfig.skills_dir` for trial-wide injection. Matches
  existing `trial_yaml` semantics — no special handling needed.

- **`agent_env` plumbing.** Harbor's top-level `environment.env` populates `JobConfig.agent_env`.
  Scene roles may also carry `env:`. Both flow through `TrialConfig` already — no change.

- **Empty `scenes: []`.** A truthy-but-empty list would currently fall into the `if cfg.scenes:`
  false branch (because `bool([]) is False`), so we'd silently degrade to `from_legacy`. That
  matches user intent ("you wrote nothing, you got default"). No special test required.

## Verification

```bash
.venv/bin/python -m pytest tests/test_yaml_config.py -v
.venv/bin/ty check src/benchflow/job.py
ruff check src/benchflow/job.py tests/test_yaml_config.py
```

Plus the issue's original repro: `j._config.scenes` should now be a 2-element list, not absent.

## Estimated diff

~25 lines `job.py`, ~30 lines test. No deletions. No public API removals.

## Out of scope (follow-ups, if desired)

- `model_name` fallback in `_parse_role` for Harbor convention consistency.
- A warning/error path when both top-level `agents:` and `scenes:` define conflicting
  agent identities (today: scene roles silently override).
- Native YAML (`_from_native_yaml`) parity — does it need a `scenes:` shortcut, or does the
  existing per-trial `trial_yaml` loader cover the use case?
