# Benchmark / Environment Integration Workflow

Use this reference when the user asks to add a benchmark, custom environment, or new task suite to SAfactory. The detailed runtime contract is defined by `docs/guides/custom-environment.md` and `docs/guides/custom-environment_CN.md`; this file turns it into an agent workflow.

## Intake gate

Do not start implementation until the request identifies:

- mode: `docker` or `rjob`;
- environment name;
- benchmark source/check-out path;
- dataset path and one-row shape;
- 1–2 smoke-test rows/case IDs;
- native single-case command or the benchmark README section that defines it;
- existing Docker image, if any;
- native result/output file location;
- native score/reward location and scale.

The user-facing copy/paste prompt is in the root `README_CN.md` / `README.md` under “Benchmark 接入 / Benchmark Onboarding Prompt”. If information is missing, ask only for the missing field. The first deliverable is one working single-case pipeline, not a full benchmark batch.

## Scope boundary

The adapter owns only the SAfactory boundary:

- parse `SimulationStartRequest`;
- map `env_params.dataset` to one native case;
- call the model through the session-aware Gateway;
- invoke the native command that already exists in the image/harness;
- find/read the native result;
- emit the SAfactory result JSON and `metrics`.

Do not reimplement benchmark case-solving logic, scoring logic already provided by the benchmark, or image internals as part of onboarding. If the native single-case command does not work independently, report that as a blocker or request explicit scope expansion.

## Files to create or adapt

For environment `mybench`, create the selected-mode files below under `env/mybench/`.

| File | Required | What it does |
|---|---:|---|
| `runner.py`, `runner.mjs`, or `runner.sh` | yes | Reads the request, gets `env_params.dataset`, calls the current Gateway session, runs one native case, reads the native result, and prints one `SimulationStartResult` JSON. |
| `mybench_config.yaml` | Docker mode | Defines `env_name`, `env_image`, `dataset`, `env_num`, `env_params`, and dataset loading options. Each dataset row is one episode. |
| `mybench_start.yaml` | Docker mode | Defines `container.runner_entrypoint`, workdir, env vars, Docker mounts/args, and `agent_name`; `agent_name` must equal `env_name`. |
| `mybench_config.rjob.yaml` | RJob mode | RJob variant of the task config. Keep dataset-row and `env_params` semantics aligned with the Docker config. |
| `mybench_start.rjob.yaml` | RJob mode | RJob variant of the start config, including `rjob:` resources, cleanup, embedded files, and cluster-accessible mounts. |
| `rule_evaluator.py` | scored benchmark | Converts runtime `metrics` and trajectory information into a `0–10` reward. Auto-discovered at `env/mybench/rule_evaluator.py`; do not add an evaluator path to YAML. |
| `Dockerfile` | optional | Builds a dedicated image when no suitable image exists. It is not a place to rewrite the benchmark's native case logic. |

RJob mode still needs the runner and evaluator. The two `.rjob.yaml` files are additional mode-specific deployment configs, not replacements for the runtime contract.

## Runner contract

The runner must:

1. read JSON from stdin or `SAFACTORY_START_REQUEST_JSON`;
2. use `request.session_id` and `SAFACTORY_GATEWAY_SESSION_URL_CONTAINER` (or the request's session URL) for model calls;
3. read the current row from `request.env_params.dataset`;
4. pass that row to the native single-case command without looping over the dataset;
5. capture native score/pass/failure/output path in `metrics`;
6. print exactly one result object like:

```json
{
  "session_id": "same-session-id",
  "status": "succeeded",
  "total_reward": 0.0,
  "step_count": 1,
  "terminated": true,
  "truncated": false,
  "error_text": null,
  "metrics": {
    "bench_case_id": "case-001",
    "bench_score": 0.73,
    "bench_passed": true,
    "bench_reason": "all required checks passed",
    "bench_output_path": "/workspace/Safactory/results/mybench/case-001.json"
  }
}
```

Keep diagnostics on stderr. A controlled task failure should be represented by a failed result; a non-zero process exit is reserved for runtime/infrastructure failure in JSON result mode.

## Config patterns

Docker task config:

```yaml
environments:
  - env_name: mybench
    env_image: mybench-image:latest
    env_num: 1
    dataset: ./datasets/cases.jsonl
    dataset_load_mode: eager
    env_params:
      task_family: mybench
      bench_root: /workspace/MyBench
      output_root: /workspace/Safactory/results/mybench
```

Docker start config:

```yaml
agent_name: mybench
container:
  workdir: /workspace/MyBench
  runner_entrypoint:
    source: ./runner.py
    target: /tmp/safactory-mybench-runner.py
    command: "python /tmp/safactory-mybench-runner.py"
  mounts:
    - source: ./results
      target: /workspace/Safactory/results
      mode: rw
  env:
    NO_PROXY: host.docker.internal,localhost,127.0.0.1,::1
    no_proxy: host.docker.internal,localhost,127.0.0.1,::1
  extra_args:
    - --add-host=host.docker.internal:host-gateway
  idle_command: "tail -f /dev/null"
```

RJob start config keeps the `container.runner_entrypoint` contract but adds, for example:

```yaml
rjob:
  name_prefix: mybench
  image_pull_policy: IfNotPresent
  no_packaging: true
  cleanup_on_finish: true
  resources:
    cpu: 1
    gpu: 0
    memory_in_mb: 1024
  embedded_files:
    - source: ./runner.py
      target: /tmp/safactory-mybench-runner.py
  mount_config:
    - "gpfs://CLUSTER_STORAGE/results:/workspace/Safactory/results"
```

Do not copy local Docker bind mounts into RJob. RJob images and mounted storage must be accessible from the cluster, and a local runner dependency must be listed in `rjob.embedded_files`.

## Validation

1. Run or otherwise verify the native command on the 1–2 supplied cases first.
2. Build/pull the selected-mode image and verify the Gateway is reachable from the runtime.
3. Run one-worker smoke evaluation with `--enable-evaluation`.
4. Verify all four artifacts: runner result JSON, native benchmark output file, Gateway trajectory/request log, and final `0–10` reward.
5. For RJob, also verify the global `--rjob-config`, cluster storage, image pull, and non-loopback Gateway URL.

Use the root README commands as the command source. Replace `YOUR_ROUTE_KEY` with an actual Gateway `llm_routes` key; never commit private routes or credentials.
