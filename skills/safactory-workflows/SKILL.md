---
name: safactory-workflows
description: Use this skill to onboard a benchmark or custom environment into SAfactory, run Docker or RJob evaluation, or prepare GRPO/RL training. For benchmark onboarding it defines the required adapter files, intake fields, single-case contract, and Docker/RJob-specific checks.
---

# SAfactory Workflows

Use this skill for three related workflows:

1. onboard a benchmark or custom environment;
2. run Docker or RJob evaluation;
3. prepare or start GRPO/RL training after evaluation works.

The repository documentation is the source of truth. Do not invent a second runtime contract. For a new benchmark, read [references/environment-integration.md](references/environment-integration.md) and `docs/guides/custom-environment_CN.md` (or the English guide for an English request). For RJob-specific details, also read `docs/internal/rjob-mode_CN.md` or `docs/internal/rjob-mode.md`.

## Benchmark onboarding: intake gate

Before editing, collect these fields from the user or infer them from the benchmark source/README:

- `mode`: exactly `docker` or `rjob`;
- environment name, normally lowercase with underscores;
- benchmark source/check-out path or repository;
- dataset path and the shape of one dataset row;
- one or two test-case IDs/rows;
- the benchmark's native single-case command (or the README section that defines it);
- Docker image name, if already available;
- where the native benchmark writes its result/output file;
- where the native score/reward is located and its scale/meaning.

The user must have (or provide enough information to run) **1–2 test cases**. The first milestone is a single-case pipeline whose output file and reward can be inspected. If the native case command, output path, score field, or selected mode is unknown, ask for that specific missing value before implementing.

### Scope boundary

The skill creates and validates the SAfactory adapter only. It does not rewrite the benchmark's case-solving/evaluation logic inside an existing Docker image, add a new benchmark harness, or repair a broken native single-case command unless the user explicitly expands the scope. Assume the image/harness can already execute one case; the adapter passes the current case and model calls in, then translates the native result out.

## Adapter files and responsibilities

For `mybench`, the onboarding output is under `env/mybench/`:

| File | Required | Responsibility |
|---|---:|---|
| `runner.py`, `runner.mjs`, or `runner.sh` | yes | Read `SimulationStartRequest` from stdin or `SAFACTORY_START_REQUEST_JSON`; read the current row from `env_params.dataset`; call the model through the current Gateway session URL; invoke the already-available native single-case command; collect the native output; print one `SimulationStartResult` JSON. |
| `mybench_config.yaml` | Docker mode | Define task rows and runtime metadata: `env_name`, `env_image`, `dataset`, `env_num`, and `env_params` (plus dataset loading options when needed). One dataset row must represent one episode/case. |
| `mybench_start.yaml` | Docker mode | Define how the runtime starts: runner entrypoint, working directory, environment variables, Docker settings, and mounts. `agent_name` must equal `env_name`. |
| `rule_evaluator.py` | recommended for scored benchmarks | Read runtime `metrics` and available trajectory data and convert the native score/pass result to a SAfactory reward in the `0–10` range. SAfactory auto-discovers `env/mybench/rule_evaluator.py`; do not register its path in YAML. |
| `Dockerfile` | optional | Build a dedicated image only when an existing image is unavailable or needs adapter dependencies. Do not move benchmark case logic into the adapter. |

The runner/result contract, config fields, and evaluator interface must follow `docs/guides/custom-environment_CN.md`. Keep stdout limited to the machine-readable result; write diagnostics to stderr. Put native case ID, score, pass/fail, reason, and output path in `metrics` so evaluation does not rerun the case.

## Runtime modes

The user must select one mode in the intake prompt. Implement that mode first; do not silently substitute the other mode.

### Docker mode

Create/use:

```text
env/mybench/runner.py      # or runner.mjs / runner.sh
env/mybench/mybench_config.yaml
env/mybench/mybench_start.yaml
env/mybench/rule_evaluator.py  # when scored evaluation is required
```

The image is local, `container.mounts` are Docker bind mounts, and the smoke test uses `--mode docker`. The runner should use `SAFACTORY_GATEWAY_SESSION_URL_CONTAINER` rather than hardcoding `localhost`.

### RJob mode

Create/use both mode-specific files in addition to the runner/evaluator:

```text
env/mybench/runner.py          # or runner.mjs / runner.sh
env/mybench/mybench_config.rjob.yaml
env/mybench/mybench_start.rjob.yaml
env/mybench/rule_evaluator.py
```

`mybench_config.rjob.yaml` is the RJob task config (normally derived from the Docker task config). `mybench_start.rjob.yaml` contains the RJob runtime settings under `rjob:`. Use cluster-accessible images and storage; use `rjob.mount_config`/`rjob.mount` rather than local Docker bind mounts; list every local runner dependency in `rjob.embedded_files`. The Gateway URL must be reachable from the cluster and must not be `127.0.0.1` or `localhost`. The smoke test must use `--mode rjob` and the appropriate `--rjob-config`.

The RJob files change deployment mechanics, not the single-case input/output contract. Keep the same dataset-row semantics, Gateway session handling, result JSON, and evaluator across both modes unless the runtime genuinely requires a documented difference.

## Onboarding workflow

1. Inspect the repository layout, the benchmark README/source, `env/geo3k/`, and the custom-environment guide.
2. Confirm the selected mode and validate the native benchmark command on 1–2 cases. Record the actual output file and native score/reward before writing the evaluator.
3. Implement only the adapter boundary: request parsing, dataset-row mapping, Gateway call, native command invocation, result-file discovery, and `SimulationStartResult` serialization.
4. Add the task config and selected-mode start config. Keep `env_name`/`agent_name` identical and make all paths/mounts explicit.
5. Add `rule_evaluator.py` when the benchmark has deterministic scoring. Convert the native result to `0–10`; do not rerun the benchmark case in the evaluator.
6. Run the smallest smoke test for the selected mode with one worker and 1–2 cases. Verify the runtime result JSON, native output file, Gateway trajectory, and final reward.
7. If the user requests both modes, repeat the config/start-config and smoke-test checks for the second mode; do not assume a Docker mount works in RJob.

Typical commands are:

```bash
# Docker
python launcher.py \
  --mode docker \
  --agent-config env/mybench/mybench_config.yaml \
  --agent-start-config env/mybench/mybench_start.yaml \
  --gateway-base-url http://127.0.0.1:8000/v1/sessions \
  --llm-model YOUR_ROUTE_KEY \
  --enable-evaluation \
  --db-path sqlite://mybench_smoke.db \
  --job-id mybench-docker-smoke \
  --pool-size 1 --max-workers 1 --max-steps 10

# RJob
python launcher.py \
  --mode rjob \
  --rjob-config config.yaml \
  --agent-config env/mybench/mybench_config.rjob.yaml \
  --agent-start-config env/mybench/mybench_start.rjob.yaml \
  --gateway-base-url http://GATEWAY_HOST:8000/v1/sessions \
  --llm-model YOUR_ROUTE_KEY \
  --enable-evaluation \
  --db-path sqlite://mybench_smoke.db \
  --job-id mybench-rjob-smoke \
  --pool-size 1 --max-workers 1 --max-steps 10
```

Replace placeholders with real values. `--llm-model` must match a Gateway `llm_routes` key; never commit private endpoints or credentials.

## Other workflows and repository rules

- Read only the reference matching the request: `references/environment-integration.md`, `references/docker-evaluation.md`, or `references/grpo-training.md`.
- Treat `env/geo3k/` as the complete reference implementation, not as a hardcoded target.
- Keep Launcher, Gateway, and Buffer Server on the same storage backend or SQLite URI.
- Do not overwrite `gateway/config.local.yaml` without preserving user edits.
- Do not read `docs/internal/` except for RJob/Sandbox/internal deployment questions; RJob onboarding is the explicit exception above.
- Before recommending RL, establish that the selected-mode single-case evaluation and reward path work.
- For edits, inspect changed paths and configs. If Docker, RJob SDK, image, data, or credentials are unavailable, report the exact blocker and leave the next command rather than claiming the smoke test passed.
