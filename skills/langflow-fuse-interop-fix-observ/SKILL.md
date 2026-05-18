---
name: langflow-fuse-interop-fix-observ
description: Diagnose and fix Langflow to Langfuse observation evaluator mismatches where Langfuse shows observations but evaluator samples return no results, especially when Type is set to GENERATION instead of SPAN.
---

# langflow-fuse-interop-fix-observ

Use this skill when the user reports any of these symptoms:

- Langfuse evaluator sample table shows `No results`.
- Langfuse receives observations from Langflow, but evaluators do not run.
- Langfuse evaluator is configured for `Type = GENERATION`.
- Langflow/Langfuse traces or observations appear after upgrading Langflow but scoring still does not work.

## Core diagnosis

Langflow's built-in Langfuse tracer emits `SPAN` observations via `start_span()`. It does not emit `GENERATION` observations by default.

If the Langfuse evaluator filter is:

```text
Type any of GENERATION
```

then Langflow-originated observations will not match.

## Commands

From the repo:

```bash
cd /Users/abdwahab/code/langflow-fuse-interop-fix-observ
bin/langflow-fuse-interop-fix-observ doctor
bin/langflow-fuse-interop-fix-observ evaluator-steps
```

For a generic clone:

```bash
git clone https://github.com/mzafir/langflow-fuse-interop-fix-observ.git
cd langflow-fuse-interop-fix-observ
bin/langflow-fuse-interop-fix-observ all
```

## Langfuse UI fix

1. Open the evaluator in Langfuse.
2. Enable Edit Mode.
3. Set **Run on** to **Observations**.
4. Keep **Run on live incoming observations** enabled.
5. Change filter from **Type any of GENERATION** to **Type any of SPAN**.
6. Remove the **Environment** filter while testing.
7. Set sampling to `100%`.
8. Save the evaluator.
9. Run the Langflow flow again.
10. Confirm the evaluator sample table shows rows.

## Optional Langflow upgrade

If `doctor` reports that the tracer lacks `flush()`, run:

```bash
bin/langflow-fuse-interop-fix-observ upgrade-langflow-110
```

This installs Langflow `release-1.10.0` into the detected local Langflow venv.

## Safety rules

- Do not print Langfuse public or secret key values.
- Do not mutate evaluator config through undocumented APIs.
- Do not claim Langflow emits `GENERATION` observations unless the local tracer actually uses generation APIs.

