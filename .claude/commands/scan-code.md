---
description: Run the code-scan agent team and produce a tiered findings.md
---

You are running the `/scan-code` command. Invoke the `code-scan-coordinator` sub-agent to scan this repository.

Scope hint (optional, may be empty, may be multiple space-separated paths): $ARGUMENTS

Instructions to the coordinator:
- If the scope hint is empty, scan the whole repo (respecting `.gitignore`).
- If the scope hint has one or more paths, treat them as the exact scan set. Each path may be a **file** or a **directory**, relative to the repo root or absolute. Mix is allowed (e.g., `src/api one_off_file.py`). Do not silently expand beyond the paths provided.
- Validate each path exists. If any path is missing, print the missing paths and stop — do not partially scan.
- Produce the report at `findings.md` in the repo root, regardless of scope.
- When finished, print a one-line summary to the user: languages scanned, tier counts, and the location of the report.
