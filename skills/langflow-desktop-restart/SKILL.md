---
name: langflow-desktop-restart
description: Restart Langflow Desktop on macOS using deterministic repo automation that reloads launchctl Langfuse/Langflow env, verifies backend env, and avoids printing secrets.
---

# langflow-desktop-restart

Use this skill when Langflow Desktop needs a clean restart after Langfuse key, host, region, or tracing environment changes, especially after `401 Unauthorized` trace-export errors.

## Deterministic automation

Invoke the repo script:

```bash
cd /Users/abdwahab/code/langflow-fuse-interop-fix-observ
scripts/macos-restart-langflow-desktop
```

For a generic clone:

```bash
cd langflow-fuse-interop-fix-observ
scripts/macos-restart-langflow-desktop
```

## What the script does

1. Reads `LANGFUSE_HOST`, `LANGFUSE_BASE_URL`, `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`, and `LANGFLOW_DEACTIVATE_TRACING` from `launchctl`.
2. Stops only explicit Langflow Desktop/backend PIDs.
3. Relaunches `/Applications/Langflow .app` with those launchd values injected into the launcher environment.
4. Verifies the backend process environment and `http://127.0.0.1:7860/api/v1/version`.
5. Prints key presence/length only; it never prints Langfuse key values.

## Core rule

`launchctl` is the macOS GUI environment source, not the restart hook. The restart hook is the deterministic script above.

`.zshrc` is not a global macOS environment file. Langflow Desktop reads the environment of the process that launches the app, then the backend inherits from Desktop.

## Safety rules

- Do not print `LANGFUSE_PUBLIC_KEY` or `LANGFUSE_SECRET_KEY`.
- Do not use `pkill` or `killall`; use explicit PIDs only.
- Do not edit shell, launchd, Langflow, or credential config files without explicit approval for that exact file/change.
