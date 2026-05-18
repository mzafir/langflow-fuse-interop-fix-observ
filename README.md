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

Related setup notes can be linked from the broader implementation document:

```text
https://docs.google.com/document/d/18Fkk57vFv8dI7nkWknJk4OzRCUR41Fca/edit
```

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

## macOS launchctl environment setup

Langflow Desktop is launched by macOS, so shell-only exports like this are not enough:

```bash
export LANGFUSE_BASE_URL=https://us.cloud.langfuse.com
```

Use `launchctl setenv` so GUI apps launched from Finder/Dock inherit the values.

### Set or update Langfuse environment variables

Use the correct regional Langfuse Cloud host for your keys:

```bash
launchctl setenv LANGFUSE_BASE_URL "https://us.cloud.langfuse.com"
launchctl setenv LANGFUSE_HOST "https://us.cloud.langfuse.com"
launchctl setenv LANGFLOW_DEACTIVATE_TRACING "False"
```

Set keys the same way, but do not paste them into docs, commits, screenshots, or logs:

```bash
launchctl setenv LANGFUSE_PUBLIC_KEY "pk-lf-..."
launchctl setenv LANGFUSE_SECRET_KEY "sk-lf-..."
```

Restart Langflow Desktop after changing launchd env:

```bash
osascript -e 'tell application "Langflow " to quit' || true
open "/Applications/Langflow .app"
```

### Check what launchd will pass to Langflow

These commands print whether values are set. Be careful with key variables because `launchctl getenv` prints the actual value.

```bash
launchctl getenv LANGFUSE_BASE_URL
launchctl getenv LANGFUSE_HOST
launchctl getenv LANGFLOW_DEACTIVATE_TRACING
```

For secrets, prefer checking presence without printing the value:

```bash
[ -n "$(launchctl getenv LANGFUSE_PUBLIC_KEY)" ] && echo "LANGFUSE_PUBLIC_KEY is set"
[ -n "$(launchctl getenv LANGFUSE_SECRET_KEY)" ] && echo "LANGFUSE_SECRET_KEY is set"
```

### Reset stale or wrong values

If Langflow is stuck on the wrong region, unset both host variables before setting the correct one:

```bash
launchctl unsetenv LANGFUSE_BASE_URL
launchctl unsetenv LANGFUSE_HOST

launchctl setenv LANGFUSE_BASE_URL "https://us.cloud.langfuse.com"
launchctl setenv LANGFUSE_HOST "https://us.cloud.langfuse.com"
```

Then restart Langflow Desktop.

### Avoid getting stuck on the wrong Langfuse region

Langfuse Cloud hosts are region-specific. Keys that work on one regional host can fail on another. For example:

```text
https://cloud.langfuse.com     # may be wrong for US-region keys
https://us.cloud.langfuse.com  # confirmed working for US-region keys
```

Prefer `LANGFUSE_BASE_URL` because Langflow uses it first. Keep `LANGFUSE_HOST` set to the same value for backward compatibility.

If authentication fails or traces disappear after a restart:

1. Run `launchctl getenv LANGFUSE_BASE_URL`.
2. Run `launchctl getenv LANGFUSE_HOST`.
3. Confirm both point to the same working region.
4. Unset stale values with `launchctl unsetenv`.
5. Set both variables again with `launchctl setenv`.
6. Restart Langflow Desktop.

You can also print this runbook from the automation:

```bash
bin/langflow-fuse-interop-fix-observ launchctl-steps
```

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
