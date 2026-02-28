---
name: eng-eval
description: Analyze a git repo to evaluate engineer skill level and productivity. Invoke with `/eng-eval <repo_path> [--since <N>d] [--author <name>]`.
disable-model-invocation: true
---

# Engineer Productivity Evaluator

Analyze the git repository at `$ARGUMENTS` and output one table per author.

## Argument Parsing

- `<repo_path>` — first non-flag token, or current directory if omitted
- `--since <Nd>` — lookback window, default `7d`
- `--author <name>` — filter to one author; default = all authors

## Step 1 — Data Collection

```bash
# Commit list with metadata
git -C <repo_path> log --since="<N> days ago" \
  --format="%H|%an|%ad|%s" --date=iso [--author="<name>"]

# Per-commit diff stats
git -C <repo_path> log --since="<N> days ago" \
  --stat --format="COMMIT:%H %ad %an" --date=iso [--author="<name>"]

# Full diffs for code quality sampling (first 15 commits)
git -C <repo_path> log --since="<N> days ago" -15 \
  --patch --format="COMMIT:%H" [--author="<name>"]
```

## Step 2 — Compute Metrics Per Author

### 2a — Strip non-significant LOC first

Raw `git log --stat` numbers are almost always inflated. Inspect each commit's diff and exclude:

| Category | Examples | Treatment |
|----------|---------|-----------|
| File moves / reorganization | "Organize scripts into subfolders", bulk rename | LOC = 0 (content unchanged) |
| Imported / vendored external code | "Import ppt skill", copy-pasted library | LOC = 0 (not the author's work) |
| Generated / repetitive variants | 10 strategy files that are 90%+ identical | Count only the unique delta |
| Data / cache / result files | `.json` caches, `.log`, `.csv` outputs | LOC = 0 |
| Config one-liners | timeout 600→1200, path string swap | LOC = 0 |
| Mechanical bulk changes | `logger.info`→`logger.debug` en masse | LOC = 0 |
| Docs / markdown | README, CHANGELOG, design docs | Count at 20% weight |

Report both `raw_loc` and `significant_loc` in the table. All downstream metrics use `significant_loc`.

### 2b — Metrics

| Metric | Definition |
|--------|-----------|
| `commits` | Excluding merge commits |
| `raw_loc` | Total additions from `git log --stat` |
| `significant_loc` | After stripping above categories |
| `churn%` | deletions / (significant_loc + 1) × 100 |
| `fix_ratio%` | fix/bug/revert/hotfix commits / commits × 100. Verify by reading the diff — teams using conventional commits often mislabel features as `fix:` |
| `avg_files_per_commit` | Excluding pure-move commits |
| `wall_clock_days` | First to last commit in window |

## Step 3 — Skill Level Assessment

**Core principle:** Engineer value = work that AI cannot do alone + making human+AI > AI alone.
Writing components and boilerplate is increasingly AI-substitutable. What matters is whether the
engineer is operating in the irreplaceable zone: domain judgment, hypothesis formation, debugging
via data observation, architectural decisions requiring context AI lacks.

**Do NOT conflate high LOC / high leverage ratio with high level.** An engineer generating 50k LOC
of strategy variants with AI in 4 days may be doing less valuable work than one spending 40 hours
on 600 lines of financial logic that requires trade-by-trade data inspection to get right.

Assign a level by reading the actual diffs through this lens:

**L5 — Domain Expert / Frontier**
Works where AI still genuinely struggles: complex domain debugging (requires interpreting
data/logs/outputs, not just reading code), novel algorithmic design grounded in domain knowledge,
architectural decisions that require business/system context AI lacks. Low AI leverage is expected
and correct here — the human is doing the irreplaceable 70%. The commits may be small but each
one represents significant human judgment. Can catch bugs AI would never find because they require
understanding what the output *should* mean.

**L4 — Architect / Orchestrator**
System-level design with clear intent (hexagonal architecture, clean interfaces, coherent
abstractions). Low churn. Debugging targets root causes. Decomposes work well so AI can execute
each piece effectively. The human+AI combination produces work qualitatively better than AI alone
because the human provides structure, constraints, and validation AI cannot self-generate.

**L3 — Productive Contributor**
Delivers working features with some rework. Uses AI for implementation, handles integration and
debugging. Commits are coherent. Design is borrowed from existing patterns rather than invented.
AI leverage is high but the human's contribution is real: integration judgment, debugging, testing.

**L2 — AI Executor**
Primarily generates and submits AI output. High LOC, high leverage, but limited evidence of human
judgment shaping the output. Fixes are symptomatic (same bug class recurs). The work AI could
produce without much human guidance — the human's role is mostly "run the prompt and commit."
High churn, inconsistent quality, fixes that patch rather than solve.

**L1 — Struggling**
High churn with no convergence. Same class of bug fixed multiple times. Cannot verify whether AI
output is correct. Large commits with no understanding of what was built.

Key questions to ask when reading diffs:
- Is this work AI-substitutable, or does it require human domain judgment?
- Does debugging show root-cause understanding (data-driven), or trial-and-error (code-driven)?
- Would the output be meaningfully worse without this human's involvement?
- Is the engineer operating at the frontier where AI still makes domain errors?

## Step 4 — Time Estimation

### Step 4a — Classify work type

First, determine the dominant work type from the diffs. This drives the entire time model.

**Type A — Pure implementation**: Writing new features, APIs, scripts. Time ≈ proportional to LOC.

**Type B — Research / experimental**: Data science, quant strategy, ML experiments. Each commit is the *output* of a cycle that includes: running experiments, observing results, forming hypotheses, debugging via data inspection, iterating. The code written is the *tip of the iceberg*; most time is spent watching outputs, not writing code.

**Type C — Debugging / integration**: Tracking down bugs through log analysis, reproducing issues, tracing execution. Time proportional to fix complexity, not LOC changed.

**Type D — Refactoring / architecture**: Restructuring existing code. Time proportional to system understanding required, not lines moved.

Most repos are a mix. Identify the dominant type and weight accordingly.

### Step 4b — Human-equivalent time

**Type A (implementation)** — LOC-based:
- Complex algorithm / systems code: 30 LOC/hr
- Backend / API / data pipeline: 50 LOC/hr
- Scripts / boilerplate-heavy: 80 LOC/hr
- Documentation / config / tests: 120 LOC/hr

**Type B (research/experimental)** — Iteration-based, NOT LOC-based:
Each meaningful experiment iteration = write variant + run + observe + conclude.
Estimate the number of distinct experiment iterations visible in the commit history
(version bumps, strategy variants, parameter sweeps, ablation commits each count as iterations).
```
human_only_hrs = iterations × hrs_per_iteration
```
Calibrate `hrs_per_iteration` from domain context:
- Quant backtest (run + analyze results): 2–4 hrs/iteration
- ML training run (setup + wait + analyze): 3–8 hrs/iteration
- Data pipeline debug (reproduce + trace + fix): 2–5 hrs/iteration

**Type C (debugging)** — Fix-complexity-based:
- Trivial fix (typo, wrong constant): 0.25 hrs
- Targeted bug (wrong logic in one place): 0.5–2 hrs
- Systemic bug (root cause spans multiple components): 2–6 hrs

**Type D (refactoring)** — Understanding-based:
- Mechanical rename/move: 0.25× LOC rate
- Logic-preserving restructure: 0.5× LOC rate
- Architectural rework requiring system understanding: estimate from scope of change

### Step 4c — Actual estimated work time

Always split into two components before computing leverage:

```
human_only_hrs = analysis_hrs + coding_hrs
actual_hrs     = analysis_hrs + (coding_hrs / ai_coding_leverage)
leverage       = human_only_hrs / actual_hrs
```

**analysis_hrs**: time observing outputs, forming hypotheses, debugging via data, making design
decisions. AI compresses this very little — treat as equal in both scenarios.

**coding_hrs**: time writing actual code. AI typically compresses 4–10×.

Consequences:
- Type B (research-heavy): analysis dominates → overall leverage is low (1.2–2×) even when AI
  helps significantly with coding
- Type A (implementation-heavy): coding dominates → overall leverage tracks AI coding leverage

**Rule:** `human_only_hrs` must always be strictly greater than `actual_hrs`. If AI helped with
coding at all, the numbers must satisfy `leverage > 1.0`. Never set them equal.

Be explicit about uncertainty. If LOC is dominated by file moves, generated fixtures, or copy-pasted
boilerplate, note that and reduce the human_only estimate accordingly.

## Step 5 — Output

For each author, output exactly this table (no prose before it, summary after):

```
### <Author Name> — <Repo> — last <N> days

| | |
|---|---|
| **Level** | L? — <label> |
| **Commits** | N |
| **+LOC / -LOC** | N / N |
| **Churn** | N% |
| **Fix ratio** | N% |
| **Avg LOC/commit** | N |
| **Avg files/commit** | N |
| **Wall-clock span** | N days |
| **Human-only estimate** | ~N hrs |
| **Actual work estimate** | ~N hrs |
| **Leverage ratio** | ~N× |

**Skill evidence** (2–4 bullets, cite commit hashes for strong claims)
- ...

**Gaps** (1–2 bullets only if clearly evidenced)
- ...
```

Do not output AI confidence scores, AI usage signals, or any commentary about whether AI was used. The goal is to evaluate the engineer's contribution and capability, not their tooling choices.

If data is insufficient (<4 commits), note limited sample and reduce confidence in level assignment.
