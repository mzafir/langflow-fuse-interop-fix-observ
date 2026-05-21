---
name: langflow-desktop-upgrade-110
description: Upgrade local macOS Langflow Desktop venv to the Langflow 1.10 release branch using deterministic repo automation.
---

# langflow-desktop-upgrade-110

Use this skill when the local Langflow Desktop venv needs the 1.10 release branch for Langfuse client reuse and `flush()` delivery reliability.

## Deterministic automation

Dry-run first:

```bash
cd /Users/abdwahab/code/langflow-fuse-interop-fix-observ
scripts/macos-upgrade-langflow-desktop-110 --dry-run
```

Apply the upgrade only after the user confirms:

```bash
cd /Users/abdwahab/code/langflow-fuse-interop-fix-observ
scripts/macos-upgrade-langflow-desktop-110 --yes --restart-app
```

If launchd environment repair is also required, pass the confirmed regional Langfuse host explicitly:

```bash
scripts/macos-upgrade-langflow-desktop-110 \
  --yes \
  --langfuse-host "https://us.cloud.langfuse.com" \
  --enable-tracing \
  --restart-app
```

Use the US host only for US-region Langfuse projects.

## Safety rules

- Do not print Langfuse public or secret key values.
- Do not mutate the venv without `--yes`.
- Do not mutate Langfuse evaluator configuration through undocumented APIs.
- Do not claim Langflow emits `GENERATION` observations unless the installed tracer actually uses generation APIs.
