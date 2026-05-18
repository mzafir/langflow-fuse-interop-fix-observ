# langflow-fuse-interop-fix-observ

Fix the Langflow -> Langfuse observation evaluator mismatch where Langfuse receives observations, but an evaluator sample shows **No results** because it is filtering for `GENERATION` observations while Langflow emits `SPAN` observations.

## Problem statement

Langflow 1.9/1.10 sends Langfuse telemetry through the Langfuse Python SDK v3 tracer. That tracer creates Langfuse observations with:

```text
Type = SPAN
```

Langfuse's newer LLM-as-judge evaluator UI commonly defaults examples toward:

```text
Run on: Observations
Filter: Type any of GENERATION
```

That combination does not match Langflow's built-in traces:

```text
Evaluator expects: GENERATION observations
Langflow emits:    SPAN observations
Result:            No results
```

This repo documents and automates the safe parts of the fix.

## Confirmed working evaluator setup

In Langfuse, edit the evaluator configuration:

1. Set **Run on** to **Observations**.
2. Change the filter from **Type any of GENERATION** to **Type any of SPAN**.
3. Remove the **Environment** filter until samples appear.
4. Keep **Run on live incoming observations** enabled.
5. Set sampling to `100%` while testing.
6. Save the evaluator.
7. Run a Langflow flow again.
8. Re-open the evaluator sample table and confirm rows appear.

After matching rows appear, tighten filters one at a time.

## Install

```bash
git clone https://github.com/mzafir/langflow-fuse-interop-fix-observ.git
cd langflow-fuse-interop-fix-observ
chmod +x bin/langflow-fuse-interop-fix-observ
```

## Run the diagnostic

```bash
bin/langflow-fuse-interop-fix-observ doctor
```

The diagnostic checks:

- local Langflow backend version, when reachable
- installed `langflow`, `langflow-base`, `lfx`, and `langfuse` package versions
- whether Langflow's installed tracer emits `SPAN` observations
- whether the Langfuse tracer includes the 1.10 `flush()` behavior
- whether relevant Langfuse environment variables are present

It never prints API key values.

## Print the evaluator fix steps

```bash
bin/langflow-fuse-interop-fix-observ evaluator-steps
```

## Optional: upgrade local Langflow Desktop venv to the 1.10 release branch

Langflow `1.10.0` may not be published on PyPI yet. If you need the unreleased branch that includes the Langfuse client reuse and `flush()` fix, run:

```bash
bin/langflow-fuse-interop-fix-observ upgrade-langflow-110
```

This installs these packages from `langflow-ai/langflow@release-1.10.0` into the detected Langflow venv:

```text
lfx 0.5.0
langflow-base 0.10.0
langflow 1.10.0
```

By default, the script uses:

```text
~/.langflow/.langflow-venv
```

Override it with:

```bash
bin/langflow-fuse-interop-fix-observ doctor \
  --venv "$HOME/.langflow/.langflow-venv"
```

## One-command local fix flow

```bash
bin/langflow-fuse-interop-fix-observ all
```

This runs diagnostics and prints the exact Langfuse UI evaluator changes. It does not change your Langfuse evaluator through the API because evaluator editing is safest and most transparent in the Langfuse UI.

Use `upgrade-langflow-110` separately if the diagnostic says the `flush()` fix is missing.

## What this does not solve

This does not make Langflow emit `GENERATION` observations. It aligns the Langfuse evaluator with what Langflow actually emits today: `SPAN` observations.

For a deeper future fix, Langflow would need to emit generation observations directly or add a dedicated bridge component that calls Langfuse generation/observation APIs with explicit input/output.

