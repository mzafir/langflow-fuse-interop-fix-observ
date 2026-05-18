# Specification

## Goal

Provide a reusable diagnostic, runbook, and coding-agent skill for the Langflow/Langfuse observation evaluator interoperability problem.

## Problem

Langflow's built-in Langfuse tracer uses the Langfuse Python SDK v3 span API:

```text
start_span(...)
root_span.start_span(...)
```

Therefore Langfuse receives observations with:

```text
Type = SPAN
```

Langfuse observation-level LLM-as-judge evaluators can be configured to filter only:

```text
Type = GENERATION
```

When that filter is used against Langflow-originated observations, the evaluator sample table returns:

```text
No results.
```

## Supported fix

Configure the Langfuse evaluator to match Langflow's current observation type:

```text
Run on: Observations
Filter: Type any of SPAN
```

During initial validation, remove other filters such as Environment until matching samples appear.

## Automation requirements

The script must:

1. Detect the local Langflow venv.
2. Report relevant package versions.
3. Detect whether the installed Langflow tracer emits spans or generations.
4. Detect whether the Langflow tracer contains `flush()` behavior.
5. Print the confirmed Langfuse UI fix steps.
6. Avoid printing API keys or secrets.
7. Print launchctl setup/reset steps for `LANGFUSE_BASE_URL`, `LANGFUSE_HOST`, and tracing env variables.
8. Optionally install Langflow's `release-1.10.0` branch when explicitly requested.

## Non-goals

- Do not mutate Langfuse evaluator configuration through undocumented APIs.
- Do not print or store Langfuse secret keys.
- Do not force Langflow to emit `GENERATION` observations by monkey-patching production code.

## Commands

| Command | Purpose |
|---|---|
| `doctor` | Inspect Langflow/Langfuse local state. |
| `evaluator-steps` | Print the confirmed UI steps. |
| `launchctl-steps` | Print macOS launchctl env setup, reset, and region-safety steps. |
| `upgrade-langflow-110` | Explicitly install Langflow `release-1.10.0` branch into the local venv. |
| `all` | Run `doctor` and `evaluator-steps`. |

## launchctl requirements

The documentation must explain that Langflow Desktop inherits environment variables from macOS launchd, not from the user's current shell. Therefore `launchctl setenv` is required for GUI-launched Langflow Desktop.

The docs must include:

1. How to set `LANGFUSE_BASE_URL`.
2. How to set `LANGFUSE_HOST` to the same regional URL for backward compatibility.
3. How to unset stale values with `launchctl unsetenv`.
4. How to restart Langflow Desktop after env changes.
5. How to avoid printing secret key values.
6. How to avoid the wrong-region problem by keeping both host variables pointed at the confirmed working Langfuse Cloud region.

## Success criteria

After applying the evaluator UI change and running a Langflow flow:

```text
Langfuse evaluator sample table shows matching SPAN observations.
```
