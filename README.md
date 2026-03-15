# AGR: Artificial General Research

> Autonomous iterative optimization for any measurable problem.
> Powered by [Claude Code](https://claude.com/claude-code).

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-v2.1.72+-blue)](https://claude.com/claude-code)

**AGR** is a Claude Code skill that turns any measurable optimization problem into an autonomous research loop. Define a metric, a correctness check, and what code to modify — AGR handles the rest: experiment, measure, keep or discard, repeat forever.

You go to sleep. You wake up to a faster, smaller, better system.

---

## Inspired By

- **[Andrej Karpathy's autoresearch](https://github.com/karpathy/autoresearch)** — the original vision of autonomous AI research. 630 lines of Python, 100 experiments per night, compounding gains. Thank you Andrej for everything you do for open source and the AI community.
- **[Udit Goenka's autoresearch](https://github.com/uditgoenka/autoresearch)** — generalized autoresearch beyond ML, introduced Metric/Guard separation.
- **[Frank Bria's Ralph Loop](https://github.com/frankbria/ralph-claude-code)** — the stop-hook pattern for fresh context per iteration in Claude Code.

---

## What AGR Contributes: 9 New Ideas

AGR is not a fork or a copy. It builds on Karpathy's, Goenka's, and Bria's work and introduces **9 new technical contributions** discovered through real-world experimentation:

### 1. Fresh Context Per Iteration (Ralph Loop + Externalized State)

**Problem**: Existing implementations run in one long conversation. By experiment 50+, the LLM context window is heavily compressed and the agent makes worse optimization decisions.

**Our solution**: Each iteration is a **disposable Claude Code instance** (`claude -p`). The agent reads ALL state from files, does ONE experiment, logs everything, and exits. The loop script (`run_agr.sh`) restarts it with a clean context.

**Key insight**: All state must be externalized to files — `results.tsv` (history), `STRATEGY.md` (brain), `git log` (code evolution), `baseline_checksums.json` (correctness). Nothing lives in the context window. This means iteration 100 has **identical reasoning quality** to iteration 1.

```
Iteration 1:   [fresh context] → reads files → optimizes → logs → DIES
Iteration 100: [fresh context] → reads files → optimizes → logs → DIES
                  ↑ Same quality, same speed, no degradation
```

### 2. Per-Benchmark Variance Analysis

**Problem**: We discovered that our dominant benchmark (`adaptive_esi`, 82% of total time) had **±1s measurement variance**. This noise masked a real 120ms improvement in `esi_idw_3d`. We incorrectly discarded 4 experiments that had genuine improvements.

**Our solution**: Instead of only checking `total_time < previous_best`, AGR evaluates each sub-benchmark independently:

- A benchmark "improved" only if it exceeds its measured noise band (>5% or >2 sigma)
- A benchmark "regressed" only if it worsened beyond its noise band
- KEEP if ANY benchmark genuinely improved without others genuinely regressing

**Why this matters**: Without this, a noisy dominant benchmark acts as a random gate that discards real improvements ~50% of the time. With per-benchmark analysis, signal is separated from noise.

### 3. Measurement Artifact Detection

**Problem**: After discarding 4 experiments, we noticed ALL of them showed the same "improvement" in `esi_idw_3d` (1.56s → ~1.44s). Was this real?

**Our solution**: If ALL experiments (including discards) show the same improvement in a benchmark, it's not an optimization — it's a **measurement artifact** (the baseline was an outlier). AGR detects this pattern and flags the baseline for re-measurement instead of crediting non-existent improvements.

### 4. Metric + Guard Separation with Rework Protocol

**Problem**: Traditional autoresearch treats the optimization metric and correctness as one combined check. If a change is faster but breaks tests, it's discarded entirely — losing a potentially good optimization idea.

**Our solution** (inspired by Goenka's Guard concept, extended with rework):
- **Metric**: the number being optimized (e.g., execution time)
- **Guard**: a pass/fail correctness check (e.g., checksums, tests)
- If Metric improved but Guard failed: **REWORK** — fix the implementation (not the approach), max 2 attempts
- If still failing after 2 reworks: discard

This saves good optimization ideas that simply have implementation bugs.

### 5. STRATEGY.md as Persistent Agent Brain

**Problem**: In a fresh-context-per-iteration system, the agent has no memory of WHY previous experiments succeeded or failed. It might repeat the same failed approach.

**Our solution**: `STRATEGY.md` is a structured document the agent reads first and updates last. It contains:

- **Current State**: best metric value, iteration count
- **Bottleneck Analysis**: per-benchmark breakdown with priorities
- **Ideas to Try**: prioritized list with expected impact
- **Ideas Already Tried**: what was tried, result, and **WHY** it worked or failed
- **Exhausted Approaches**: entire categories marked as "don't retry"
- **Key Insights**: accumulated knowledge about the codebase

The WHY is critical. Not just "compiler flags failed" but "compiler flags failed because `exp2f`/`log2f` are scalar CRT functions that can't auto-vectorize, and AVX2 causes frequency throttling on mixed workloads."

### 6. Exhausted Approaches Registry

**Problem**: After 4 failed compiler flag experiments (`/fp:fast`, `/arch:AVX2`, etc.), the agent kept trying new compiler flags.

**Our solution**: When a CATEGORY of approaches is depleted, it's added to "Exhausted Approaches" in STRATEGY.md with an explicit instruction not to retry:

```markdown
## Exhausted Approaches (don't retry)
- **Compiler flags**: 4 experiments failed. MSVC optimization is maxed.
- **LOO2D::eval micro-optimizations**: 3 experiments failed. Per-eval cost is near-optimal.
- **Leaf-level parallelism**: load balancing already adequate with tree-level scheduling.
```

Future iterations read this and skip entire categories, focusing on unexplored approaches.

### 7. Stuck Detection Protocol

**Problem**: After multiple consecutive discards, the agent tends to make increasingly minor variations of the same failed approach.

**Our solution**: When >5 consecutive discards are detected in `results.tsv`:

1. Re-read ALL source files (not just the hot path)
2. Review the entire results log for patterns (what categories work? what don't?)
3. Try **combining** 2-3 previous successful optimizations in a new way
4. Try the **opposite** approach of recent failures
5. Try a **radical architectural change** (different algorithm, not micro-opt)

### 8. Complexity Budget (Divide Large Changes)

**Problem**: With more turns available (100-200), the agent sometimes attempts massive refactors that span multiple files, exceed the turn limit, and produce incomplete changes.

**Our solution**: A "complexity budget" rule in `program.md`:

> If a change requires more than ~30 tool calls to implement, it's TOO BIG for one iteration. Break it into smaller steps:
> - Step 1: refactor to expose the optimization opportunity (keep if code is simpler)
> - Step 2: apply the optimization on the clean refactored code
> - Each step is a separate iteration with its own keep/discard decision

This leverages the **simplicity criterion** — a refactoring-only step that produces simpler code is kept even without performance improvement.

### 9. Supervisor Pattern with Discard Auditing

**Problem**: The autonomous agent discards experiments based on total metric. But a supervisor reviewing the data can spot improvements the agent missed.

**Our solution**: A supervisor (human or parent Claude Code session) periodically:

- Reads `results.tsv` to see all experiments including discards
- Audits discarded experiments for **hidden per-benchmark improvements**
- Checks if multiple discards share a common improvement (suggesting the baseline is the outlier)
- Adjusts `STRATEGY.md` between batches based on findings
- Views `progress.png` for visual pattern recognition

In our case study, the supervisor audit revealed that 4 discarded experiments all improved `esi_idw_3d` by ~7% — flagging a baseline measurement outlier that the autonomous agent couldn't detect on its own.

---

## Real Results

Tested on a C++/Python spatial analysis library ([spatialize](https://github.com/alges/spatialize)):

```
Baseline:    53.54s
After AGR:   28.73s  (-46.3%)

14 experiments, 7 kept, 7 discarded, 0 crashes
```

![AGR Optimization Timeline](progress.png)

*Top: total benchmark time per experiment (green = kept, gray = discarded, dashed = best so far). Bottom: per-benchmark breakdown showing where improvements came from.*

Optimizations found autonomously:
| # | Optimization | Improvement | Type |
|---|---|---|---|
| 1 | `std::pow(x,2)` → `x*x` in distance() | -16.4% | Micro-optimization |
| 2 | Pre-computed coordinate diffs + reused storage | -12.7% | Memory layout |
| 3 | `std::pow` → `exp2f(log2f())` in LOO inner loop | -2.2% | Math intrinsics |
| 4 | Vectorized KDE fitting, bypassed sklearn+joblib | -14.4% | Algorithmic (Python) |
| 5 | Cached grid_search evaluations across restarts | -2.5% | Caching |
| 6 | Parallelized estimate() tree loop + const ref | -9.8% | Parallelism |

The agent naturally progressed from easy micro-optimizations to algorithmic changes. When micro-opts plateaued (5 consecutive discards), the strategy document guided it toward architectural improvements.

---

## How It Works

### The Loop

```
┌─────────────────────────────────────────────────────────┐
│  AGR LOOP (run_agr.sh)                                  │
│                                                          │
│  while iterations < max:                                 │
│    1. Launch fresh Claude Code instance (claude -p)      │
│    2. Agent reads: results.tsv + STRATEGY.md             │
│    3. Agent picks ONE optimization idea                  │
│    4. Agent implements change                            │
│    5. Git commit BEFORE running (enables clean rollback) │
│    6. Run benchmark.py --verify (Metric + Guard)         │
│    7. Decision:                                          │
│       ├─ Guard FAIL + Metric up → REWORK (2 attempts)   │
│       ├─ Guard PASS + Metric up → KEEP                  │
│       ├─ Guard PASS + bench >5% → KEEP (noise-masked)   │
│       ├─ Code simpler?         → KEEP (simplification)  │
│       └─ None of above         → DISCARD + git reset    │
│    8. Log to results.tsv (even if discarded)             │
│    9. Update STRATEGY.md (what worked, what didn't, WHY) │
│   10. Agent exits → context destroyed                    │
│   11. analysis.py regenerates progress.png               │
│   12. Loop restarts → Step 1                             │
│                                                          │
│  All state in files. Nothing in context.                 │
└─────────────────────────────────────────────────────────┘
```

### Claude Code Flags

```bash
claude -p "$(cat program.md)" \
    --dangerously-skip-permissions \
    --max-turns 200 \
    --effort high
```

| Flag | What It Does | Why AGR Uses It |
|---|---|---|
| `-p` | Headless mode — read prompt, execute, exit | Fresh context each iteration (Ralph Loop) |
| `--dangerously-skip-permissions` | Skip all permission prompts | Full autonomy: read, write, compile, benchmark without asking |
| `--max-turns 200` | Max tool calls per session | Safety limit. 50 for interpreted languages, 100-200 for compiled (C++ build takes many turns) |
| `--effort high` | Deeper reasoning | Optimization decisions need code analysis, not quick answers |
| `--max-budget-usd N` | Cost cap per iteration | Optional. Prevents runaway cost on complex iterations |
| `-w` / `--worktree` | Git worktree isolation | Advanced: run parallel experiments on separate branches |

---

## Installation

```bash
# Clone and install as Claude Code skill
git clone https://github.com/JoaquinMulet/Artificial-General-Research.git
cp -r Artificial-General-Research/skills/agr ~/.claude/skills/

# Or project-specific
cp -r Artificial-General-Research/skills/agr .claude/skills/
```

## Usage

```bash
# Setup wizard (in Claude Code)
/agr speed
/agr accuracy
/agr "bundle size"

# Launch optimization loop
bash run_agr.sh --max 10

# Monitor progress
cat results.tsv          # experiment history
cat STRATEGY.md          # agent's current thinking
open progress.png        # visual timeline
```

## Generated Files

| File | Purpose | Agent modifies? |
|---|---|---|
| `benchmark.py` | Metric measurement + Guard verification | **Never** |
| `baseline_checksums.json` | Guard ground truth (checksums) | **Never** |
| `program.md` | Agent instructions per iteration | **Never** |
| `STRATEGY.md` | Persistent brain (ideas, history, insights) | **Yes** (every iteration) |
| `results.tsv` | Experiment log (append-only, even failures) | **Yes** (append only) |
| `analysis.py` | Generates progress.png | **Never** |
| `run_agr.sh` | Loop launcher | **Never** |
| `progress.png` | Optimization timeline chart | Auto-generated |

## Works For Any Measurable Problem

| Use Case | Metric | Guard |
|---|---|---|
| **Library speed** | Wall-clock time | Checksums match |
| **Bundle size** | KB after build | Tests pass |
| **ML accuracy** | F1 score | Min threshold met |
| **API latency** | p95 response time | Integration tests pass |
| **Lighthouse score** | Performance score | No visual regression |
| **SQL optimization** | Query execution time | Same result set |
| **Prompt engineering** | Eval score | Golden set matches |
| **Cloud costs** | $/month | Functionality tests pass |
| **Docker image size** | MB after build | Container health check passes |
| **Code coverage** | % coverage | No test regressions |

---

## Comparison

| Feature | Karpathy | Goenka | Bria (Ralph) | **AGR** |
|---|---|---|---|---|
| Domain | ML only | Any task | Any task | **Any task** |
| Context management | Long session | Long session | Fresh per iter | **Fresh per iter** |
| Correctness check | None | Guard (pass/fail) | None | **Checksums + Guard + Rework** |
| Variance handling | None | None | None | **Per-benchmark analysis** |
| Artifact detection | None | None | None | **Cross-experiment pattern detection** |
| Failed idea tracking | Git only | Results log | None | **Exhausted Approaches registry** |
| Stuck detection | None | >5 discards | None | **>5 discards + combine/opposite/radical** |
| Complexity management | None | None | None | **Complexity budget (divide large changes)** |
| Progress visualization | Notebook | None | None | **progress.png with benchmark breakdown** |
| Supervisor/audit | None | None | None | **Discard auditing for hidden improvements** |
| Simplicity criterion | Mentioned | Implemented | None | **Implemented** |
| Strategy persistence | None | None | None | **STRATEGY.md with WHY tracking** |

---

## Contributing

PRs welcome! Areas of interest:
- Adapters for other AI coding agents (OpenCode, Cursor CLI, Aider)
- Additional benchmark templates for new domains
- Variance analysis improvements
- Parallel experiment support via git worktrees
- Multi-agent coordination (different agents optimizing different benchmarks)

## License

MIT

## Credits

Built by [Joaquin Mulet](https://github.com/JoaquinMulet) with [Claude Code](https://claude.com/claude-code).

Standing on the shoulders of:
- [Andrej Karpathy](https://github.com/karpathy/autoresearch) — the original autoresearch vision
- [Udit Goenka](https://github.com/uditgoenka/autoresearch) — generalized autoresearch to non-ML tasks, Metric/Guard separation
- [Frank Bria](https://github.com/frankbria/ralph-claude-code) — the Ralph Loop pattern for Claude Code (fresh context per iteration via stop-hook re-invocation)
