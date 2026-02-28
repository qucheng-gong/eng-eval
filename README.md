# eng-eval Sample Output

**Command:** `/eng-eval /path/to/repo --since 30d`

---

## Engineer Productivity Report

**Repos analyzed:** repo-alpha (7d) · repo-beta (30d) · repo-gamma (30d)
**Generated:** 2026-02-28

---

### Summary

| Engineer | Level | Sig. LOC | Human-only | Actual | Leverage | Irreplaceability |
|----------|-------|----------|-----------|--------|----------|-----------------|
| Engineer A | **L5** | ~700 | ~45 hrs | ~28 hrs | ~1.6× | High |
| Engineer B | **L4** | ~8,000 | ~160 hrs | ~35 hrs | ~4.5× | Medium-High |
| Engineer C | **L3** | ~4,000 | ~70 hrs | ~40 hrs | ~1.8× | Medium |

---

### Engineer A — `repo-alpha` — last 7 days

| | |
|---|---|
| **Level** | L5 — Domain Expert |
| **Commits** | 7 |
| **Raw LOC / Sig. LOC** | 5,200 / ~700 |
| **Raw inflation sources** | 1 file-reorganization commit (content unchanged); docs + log files |
| **Churn** | 7% |
| **Fix ratio** | 29% |
| **Wall-clock span** | 2 days |
| **Human-only estimate** | ~45 hrs |
| **Actual work estimate** | ~28 hrs |
| **Leverage ratio** | ~1.6× |

**Skill evidence**
- Identified a domain-specific bias bug by reading output data — not discoverable from code inspection alone. Fix required understanding a semantic difference between two execution paths that look identical in code
- New module implementation mirrors existing baseline structure exactly — intentional symmetry that reduces future cognitive load
- Proactively extracted shared utilities when module count grew — DRY applied at the right threshold, not prematurely

**Gaps**
- Two commits fixed the same class of bug at different abstraction levels, suggesting the first fix lacked a full audit pass

---

### Engineer B — `repo-gamma` — last 30 days

| | |
|---|---|
| **Level** | L4 — Architect / Orchestrator |
| **Commits** | 15 |
| **Raw LOC / Sig. LOC** | 41,800 / ~8,000 |
| **Raw inflation sources** | 1 commit importing an external package verbatim (26.8k) |
| **Churn** | 10% |
| **Fix ratio** | 7% |
| **Wall-clock span** | 9 days |
| **Human-only estimate** | ~160 hrs |
| **Actual work estimate** | ~35 hrs |
| **Leverage ratio** | ~4.5× |

**Skill evidence**
- Used `Protocol`-based port interfaces with clean domain / application / infrastructure layer separation — hexagonal architecture signals deliberate design background, not accidental structure
- 7% fix ratio across a 9-day full-stack build (CLI + API + storage + DB + evaluation pipeline) indicates solid upfront design
- One commit with a trivial message was actually a cross-cutting infrastructure refactor unifying configuration across 5 subsystems — message completely misrepresents the work

**Gaps**
- Commit messages are the biggest weakness: short/vague messages bury high-value architectural contributions in an unreadable log

---

### Engineer C — `repo-beta` — last 30 days

| | |
|---|---|
| **Level** | L3 — Contributor |
| **Commits (excl. merge)** | 59 |
| **Raw LOC / Sig. LOC** | 10,786 / ~4,000 |
| **Raw inflation sources** | Cache files, config tweaks, data output files |
| **Churn** | 42% |
| **Fix ratio** | 10% |
| **Wall-clock span** | 29 days |
| **Human-only estimate** | ~70 hrs |
| **Actual work estimate** | ~40 hrs |
| **Leverage ratio** | ~1.8× |

**Skill evidence**
- Fixed an async startup bottleneck by switching from blocking `await` to `create_task` — understands event loop sequencing, not a trial-and-error change
- 10% fix ratio is the lowest on the team; first-commit quality consistently higher than peers
- Also served as de facto integration owner, handling cross-branch coordination beyond personal coding output

**Gaps**
- Messages describe operational state rather than engineering intent — unreadable after a few weeks

---

## Team-Level Observations

**1. Commit messages hide the most valuable work.**
In an AI-assisted workflow this matters more, not less — when AI writes the code, the commit message is often the only record of what human judgment was applied.

**2. High leverage ≠ high value.**
Engineer A's near-1× leverage is correct and expected: the bottleneck is domain observation and reasoning, not code volume. Evaluating by LOC or leverage alone inverts the ranking.

**3. Raw LOC is unreliable.** Inflated 3–7× across all repos by file moves, imported external code, and repetitive generated variants. Always compute significant LOC first.

**4. Fix ratio requires diff inspection.** Teams using conventional commits often label refactoring and cleanup as `fix:`. Read the diffs — the prefix alone is not reliable.
