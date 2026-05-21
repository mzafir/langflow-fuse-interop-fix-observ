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

Related US-region setup notes can be linked from the broader implementation document:

```text
https://docs.google.com/document/d/18Fkk57vFv8dI7nkWknJk4OzRCUR41Fca/edit
```

That document is for **US-region Langfuse Cloud projects only**. For other Langfuse regions, replace `https://us.cloud.langfuse.com` with the region host that matches your Langfuse project.

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

Langflow Desktop is launched by macOS, so shell-only exports like this are not enough. The host below is a **US-region example**:

```bash
export LANGFUSE_BASE_URL=https://us.cloud.langfuse.com
```

Use `launchctl setenv` so GUI apps launched from Finder/Dock inherit the values.

### Set or update Langfuse environment variables

Use the correct regional Langfuse Cloud host for your keys. For US-region projects:

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

If Langflow is stuck on the wrong region, unset both host variables before setting the correct one. The example below resets to the US-region host:

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
https://us.cloud.langfuse.com  # US-region host
```

Prefer `LANGFUSE_BASE_URL` because Langflow uses it first. Keep `LANGFUSE_HOST` set to the same value for backward compatibility. Do not hard-code `https://us.cloud.langfuse.com` unless the Langfuse project is actually in the US region.

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

macOS standalone script:

```bash
scripts/macos-upgrade-langflow-desktop-110 --dry-run
scripts/macos-upgrade-langflow-desktop-110 --yes --restart-app
```

If Langflow Desktop needs launchd environment repair at the same time, pass the confirmed regional Langfuse host explicitly:

```bash
scripts/macos-upgrade-langflow-desktop-110 \
  --yes \
  --langfuse-host "https://us.cloud.langfuse.com" \
  --enable-tracing \
  --restart-app
```

Use the US host only for US-region Langfuse projects. The script does not print secret key values and does not edit Langfuse evaluator configuration through APIs.

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

## Skills and deterministic automations

Every repo skill points to a deterministic script or CLI command. Keep skills thin; put operational logic in `scripts/` or `bin/` so behavior can be reviewed and changed without burying shell logic in a skill file.

| Skill | Automation | Purpose |
|---|---|---|
| `skills/langflow-fuse-interop-fix-observ/SKILL.md` | `bin/langflow-fuse-interop-fix-observ doctor`, `launchctl-steps`, `evaluator-steps` | Diagnose SPAN vs GENERATION evaluator mismatch and print the safe UI/runbook steps. |
| `skills/langflow-desktop-restart/SKILL.md` | `scripts/macos-restart-langflow-desktop` | Restart Langflow Desktop with launchd Langfuse/Langflow env injected, then verify backend env and health. |
| `skills/langflow-desktop-upgrade-110/SKILL.md` | `scripts/macos-upgrade-langflow-desktop-110` | Upgrade the local macOS Langflow Desktop venv to the 1.10 release branch after dry-run/approval. |

### Skill components

Each skill has two parts:

1. A thin `SKILL.md` file that tells Copilot CLI when the workflow applies and which command to run.
2. A deterministic script or CLI command that performs the actual work.

This separation is intentional. Update operational behavior in `scripts/` or `bin/`, not by embedding long shell programs in the skill text.

### Step-by-step invocation

Diagnose Langflow/Langfuse interop:

```bash
cd /Users/abdwahab/code/langflow-fuse-interop-fix-observ
bin/langflow-fuse-interop-fix-observ doctor
```

Print the launchctl environment runbook:

```bash
bin/langflow-fuse-interop-fix-observ launchctl-steps
```

Print the Langfuse evaluator UI fix:

```bash
bin/langflow-fuse-interop-fix-observ evaluator-steps
```

Restart Langflow Desktop after key, host, region, or tracing env changes:

```bash
scripts/macos-restart-langflow-desktop
```

Dry-run the Langflow Desktop 1.10 upgrade:

```bash
scripts/macos-upgrade-langflow-desktop-110 --dry-run
```

Apply the 1.10 upgrade after reviewing the dry-run:

```bash
scripts/macos-upgrade-langflow-desktop-110 --yes --restart-app
```

### Session-start activation

Install these repo skills into the local Copilot skill directory and add them to the local `all-skills` loader if they should be available at every Copilot CLI session start:

```text
skills/langflow-fuse-interop-fix-observ/SKILL.md
skills/langflow-desktop-restart/SKILL.md
skills/langflow-desktop-upgrade-110/SKILL.md
```

Expected `all-skills` entries:

```text
langflow-fuse-interop-fix-observ
langflow-desktop-restart
langflow-desktop-upgrade-110
```

After changing local skill registration, start a fresh Copilot CLI session if `/skill` does not immediately show the new skills.

## What this does not solve

This does not make Langflow emit `GENERATION` observations. It aligns the Langfuse evaluator with what Langflow actually emits today: `SPAN` observations.

For a deeper future fix, Langflow would need to emit generation observations directly or add a dedicated bridge component that calls Langfuse generation/observation APIs with explicit input/output.

## Operational footnote: Langflow Desktop 401 trace-export failure

Observed failure:

```text
Failed to export span batch code: 401, reason: Unauthorized
Cannot connect to Langfuse: status_code: 401, body: Invalid credentials. Confirm that you've configured the correct host.
```

Root cause:

1. The Langfuse key pair was valid for the US Langfuse Cloud region.
2. `launchctl` had the corrected US-region values.
3. The process that launched Langflow Desktop still had stale `LANGFUSE_HOST` and `LANGFUSE_BASE_URL` values for a different Langfuse Cloud region.
4. Langflow Desktop inherited that stale launcher environment, then the backend inherited it from Desktop.
5. The backend sent traces to the wrong regional host and Langfuse rejected the request with `401 Unauthorized`.

Important detail: `.zshrc` is not a global macOS environment file. It only affects interactive zsh shells. macOS GUI apps inherit from launchd or from the process that runs `open`, so an already-running tool or terminal can keep stale values even after `.zshrc` is updated.

Patch/fix applied:

1. Confirmed the key pair authenticated against the intended regional Langfuse API without printing secrets.
2. Loaded the intended `LANGFUSE_BASE_URL`, `LANGFUSE_HOST`, and tracing values into `launchctl`.
3. Fully stopped the old Langflow Desktop/backend PIDs.
4. Relaunched Langflow Desktop with the current launchd values explicitly injected into the `open` launcher environment.
5. Verified the running backend process had the intended regional host and that `http://127.0.0.1:7860/api/v1/version` was healthy.
6. Confirmed Langfuse returned recent traces and the 401 errors stopped.

Use this deterministic restart automation when the same stale-launcher-env failure appears again:

```bash
scripts/macos-restart-langflow-desktop
```
