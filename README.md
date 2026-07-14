# code-performance-agent

A drop-in [Claude Code](https://docs.claude.com/en/docs/claude-code) sub-agent team that produces a **tiered, actionable code audit** across performance, security, quality, and correctness — with benchmarking guidance so perf claims are testable, not speculation.

Findings land in a single `findings.md` at the repo root, grouped as:

1. **Quick wins** (< 30 min each)
2. **Medium complexity** (1–4 hours)
3. **Hard bottlenecks** — the structural issues that actually cap the system

Every finding carries a `Why it matters` line (what actually breaks if you ignore it), every performance finding carries a `How to verify` line (how to prove the fix worked), and the report ends with a per-language **Suggested Benchmark Setup** so you can baseline before you fix.

---

## Requirements

- [Claude Code](https://docs.claude.com/en/docs/claude-code) installed and working on your machine.
- A target repo (or just a folder / a few files) you want to audit. The team supports Python, JavaScript / TypeScript, Go, Java, and falls back to a generic expert for anything else.

## Install

Two ways to install: **per-repo** (recommended for first-time use) or **global** (available in every project).

### Per-repo install

From this project's directory:

```bash
cp -R .claude /path/to/target-repo/
```

That drops seven agent files under `<target-repo>/.claude/agents/` and one slash command under `<target-repo>/.claude/commands/`. If the target already has a `.claude/` directory, the `cp -R` merges cleanly and leaves existing files (like `settings.local.json`) untouched.

### Global install

Copy just the team into your Claude Code user directory:

```bash
mkdir -p ~/.claude/agents ~/.claude/commands
cp .claude/agents/*.md   ~/.claude/agents/
cp .claude/commands/*.md ~/.claude/commands/
```

After this, `/scan-code` is available in every Claude Code session on your machine.

### Verify the install

Inside the target repo, open Claude Code:

```bash
cd /path/to/target-repo
claude
```

Then type `/` — you should see `scan-code` in the slash-command menu. If not, check that the `.claude/commands/scan-code.md` file is in place.

---

## Running a scan

All scans are triggered with the `/scan-code` slash command inside a Claude Code session. Output always goes to **`findings.md` at the repo root** regardless of what you scoped the scan to.

### Whole repo

```
/scan-code
```

Walks the whole repo respecting `.gitignore`. Skips vendored / generated directories: `node_modules`, `vendor`, `dist`, `build`, `.venv`, `venv`, `target`, `__pycache__`, `.next`, `.nuxt`, `.git`.

Best for: first-time audits, monthly health checks, or when you don't know where the biggest issues live.

### A specific folder

```
/scan-code src/api
```

```
/scan-code internal/pipeline
```

```
/scan-code app/controllers app/services
```

Space-separate multiple paths to scan several folders at once. Each path is walked independently (respecting `.gitignore`).

Best for: focused audits during a refactor, a service-specific review, or when the whole-repo scan is too noisy.

### Specific files

```
/scan-code agent.js
```

```
/scan-code server.js public/app.js
```

Individual files are scanned exactly — no sibling walk. Framework fingerprinting still reads the enclosing repo's manifest files (`package.json`, `pyproject.toml`, `go.mod`, `pom.xml`), so a single-file scan still gets language-aware findings, not generic ones.

Best for: PR review, "this file has been rewritten three times, what am I missing", or before landing a big change.

### Mix of folders and files

```
/scan-code src/api one_off_migration.py scripts/reindex.py
```

You can freely combine directories and files. Missing paths cause the scan to stop with a clear error rather than partially completing.

### Interpreting the output

The coordinator prints a one-line summary when done, e.g.:

```
Scanned Python (312 files), TypeScript (48 files). 14 quick wins · 9 medium · 4 hard bottlenecks. Report: ./findings.md
```

Open `findings.md`. The report is structured for reading top-down:

1. **Summary block** — languages, counts, top 3 recommendations.
2. **Tier 1 — Quick Wins.** Start here. Each item is under 30 min; you can usually knock the whole tier out in a session.
3. **Tier 2 — Medium.** These need a small design decision but pay off within a working day.
4. **Tier 3 — Hard Bottlenecks.** These are the real ceilings — architecture, data model, or trust boundaries. Read for context; plan for later.
5. **Suggested Benchmark Setup.** A per-language runnable kit so you can baseline before touching perf.

Every finding follows the same shape:

```
### [perf] N+1 query in user list endpoint — src/api/users.py:47
- Evidence: loop over users calls user.profile inside the serializer
- Why it matters: each iteration fires a separate SELECT for the related profile.
  100 users on the list page means 101 queries; latency scales linearly with page
  size and the DB becomes the bottleneck long before the app tier does.
- Fix: use select_related('profile') on the queryset
- Effort: ~15 min
- Confidence: high
- How to verify: wrk -t4 -c100 -d30s http://localhost:8000/api/users before/after;
  also enable Django's connection.queries to confirm query count drops from N+1 to 1.
```

The dimension tag in brackets (`perf`, `security`, `quality`, `correctness`) at the start of each finding lets you `grep` a category out quickly:

```bash
grep -A6 '^### \[security\]' findings.md
```

---

## The team

| Agent | Role |
|---|---|
| `code-scan-coordinator` | Entry point. Detects languages, dispatches experts in parallel, assembles the report, writes `findings.md`, and verifies the file landed on disk before claiming success. |
| `python-expert` | Python — CPython perf traps, Django / Flask / FastAPI, ORM N+1, `pickle` / `eval`, async. Emits a `pytest-benchmark` + `py-spy` + `wrk` kit. |
| `typescript-expert` | JS / TS + deep **React** (hooks, RSC, Suspense, data fetching) + deep **Node.js** (event loop, streams, worker threads, HTTP internals) + **Next.js, Express, Fastify, NestJS** + **cross-browser** (CSS feature support, Safari / iOS quirks, `browserslist`, polyfill strategy — also scans `.html/.css/.scss/.vue/.svelte`). Prototype pollution, promise fan-out, `any`-hides-bugs. Emits `vitest bench` + `clinic.js` + `autocannon` kit. |
| `go-expert` | Go — goroutine leaks, channel deadlocks, `defer` in hot loops, races. Emits `testing.B` + `benchstat` + `pprof` + `vegeta` kit. |
| `java-expert` | Java (+ best-effort Kotlin) — GC pressure, Spring N+1, pool exhaustion, deserialization RCE. Emits JMH + `async-profiler` + Gatling kit. |
| `generic-expert` | Fallback for unrecognized languages (Rust, Ruby, PHP, C#, Elixir, Swift, …). Stays at universal engineering principles. |

Language detection routes automatically. You do not need to tell `/scan-code` what language your repo is in.

---

## Troubleshooting

**`findings.md` didn't get written.** Check that `/scan-code` completed (didn't get interrupted). The report writer only writes at the end; a mid-run stop produces no file. Re-run.

**"No such file: `.claude/commands/scan-code.md`" or `/scan-code` doesn't appear in the menu.** The install didn't land. Verify with `ls -la .claude/commands/` in the target repo. If installed globally, verify with `ls -la ~/.claude/commands/`.

**Scan flagged nothing / feels too shallow.** Two common causes: (1) the scope was too narrow (a single file rarely has architectural findings — try a folder), or (2) the language isn't in the specialist set and fell through to `generic-expert`, which stays at universal principles by design. Both are correct behavior, not a bug.

**Scan is too noisy — hundreds of Tier-1 findings.** Narrow the scope: `/scan-code src/core` on the part of the codebase you actually care about. The coordinator does not currently support `--exclude` patterns; the fastest workaround is a scoped scan.

**One language's findings look thinner than another's.** The specialists have different depths (Python and Go are the most thorough; Kotlin is best-effort under `java-expert`). This is inherent, not a bug — the generic fallback is explicit about staying at universal principles.

**The report says a file line number that doesn't exist.** The model occasionally misreads line numbers on files it re-read partially. The `Evidence` field always quotes the code fragment — trust that over the line number when they disagree, and grep for the fragment.

---

## What v1 does not do

- **It does not run benchmarks.** Every perf finding tells you how to verify with a benchmark, and the report ends with a per-language kit, but running those benchmarks is your step. The target repo may not be runnable in-session without secrets, seed data, or external dependencies. A future `bench-runner` sub-agent may add opt-in execution.
- **It does not open PRs or auto-fix.** Output is a report. Applying fixes is your step.
- **It does not do cross-file dataflow / taint tracking.** Findings stay at file or function granularity. A deeper analysis pass is future work.
- **It does not integrate with CI.** First version is human-invoked in an interactive Claude Code session.
