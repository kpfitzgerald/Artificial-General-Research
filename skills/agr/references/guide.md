# Autoresearch Framework — Complete Guide

> Inspired by [Andrej Karpathy's autoresearch](https://github.com/karpathy/autoresearch).
> Thank you Andrej for your tireless contributions to open source and AI education.
> Your vision of autonomous AI research agents inspired this framework.

## Table of Contents

1. [What Is Autoresearch?](#1-what-is-autoresearch)
2. [Architecture](#2-architecture)
3. [How It Works Step by Step](#3-how-it-works-step-by-step)
4. [Claude Code CLI Flags Explained](#4-claude-code-cli-flags-explained)
5. [The Ralph Loop Pattern](#5-the-ralph-loop-pattern)
6. [Variance Analysis System](#6-variance-analysis-system)
7. [Decision Logic](#7-decision-logic)
8. [File-by-File Reference](#8-file-by-file-reference)
9. [Setting Up From Scratch](#9-setting-up-from-scratch)
10. [The Supervisor Pattern](#10-the-supervisor-pattern)
11. [Lessons Learned](#11-lessons-learned)
12. [Troubleshooting](#12-troubleshooting)
13. [Adapting to Your Use Case](#13-adapting-to-your-use-case)

---

## 1. What Is Autoresearch?

Autoresearch is an autonomous optimization system where an AI agent iteratively
improves a codebase by:

1. Proposing a change
2. Measuring its impact
3. Keeping improvements, discarding regressions
4. Repeating forever

**Karpathy's original** focused on LLM training (optimizing val_bpb on H100 GPUs).
**This framework** generalizes the pattern to any measurable optimization problem.

### What You Need

Your problem must have these properties:

| Property | Example | Why Required |
|---|---|---|
| **Measurable metric** | execution time, accuracy, bundle size | Agent needs a number to optimize |
| **Reproducible benchmark** | fixed test suite with fixed inputs | Same code must produce same metric |
| **Correctness verification** | checksums, test assertions, golden outputs | Speed without correctness = bug |
| **Modifiable code** | source files the agent can edit | Agent needs to make changes |
| **Build step** (optional) | C++ compilation, webpack bundle | Some changes need compilation |

---

## 2. Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  AUTORESEARCH SYSTEM                                            │
│                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐      │
│  │ run_auto     │    │ Claude Code  │    │ benchmark.py │      │
│  │ research.sh  │───▶│ Instance     │───▶│ (FIXED)      │      │
│  │ (LOOP)       │    │ (DISPOSABLE) │    │              │      │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘      │
│         │                   │                    │              │
│         │            ┌──────▼───────┐    ┌──────▼───────┐      │
│         │            │ Source Code  │    │ Results      │      │
│         │            │ (MODIFIED)   │    │ (LOGGED)     │      │
│         │            └──────────────┘    └──────────────┘      │
│         │                                                       │
│         ▼                                                       │
│  ┌──────────────────────────────────────────────────────┐      │
│  │  PERSISTENT STATE (survives between iterations)       │      │
│  │                                                       │      │
│  │  results.tsv      → ALL experiments (keep/discard)    │      │
│  │  STRATEGY.md      → Brain (ideas, history, insights)  │      │
│  │  baseline.json    → Correctness reference             │      │
│  │  progress.png     → Visual timeline                   │      │
│  │  git history      → Kept changes as commits           │      │
│  └──────────────────────────────────────────────────────┘      │
│                                                                 │
│  ┌──────────────────────────────────────────────────────┐      │
│  │  IMMUTABLE FILES (never modified by agent)            │      │
│  │                                                       │      │
│  │  benchmark.py     → Measurement + verification        │      │
│  │  baseline.json    → Ground truth checksums            │      │
│  │  program.md       → Agent instructions                │      │
│  │  analysis.py      → Chart generator                   │      │
│  │  run_autoresearch → Loop script                       │      │
│  └──────────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────────┘
```

### Key Design Principle: All State Is Externalized

The AI agent's context window is **disposable**. Every piece of information it
needs lives in files:

- What's been tried → `results.tsv`
- What to try next → `STRATEGY.md`
- Current best code → `git HEAD`
- Whether results are correct → `baseline_checksums.json`

This means:
- Context window can be small and fast
- Agent quality doesn't degrade over time
- Any crash is recoverable (just restart)
- Multiple agents could theoretically run in parallel (with git worktrees)

---

## 3. How It Works Step by Step

### The Loop (run_autoresearch.sh)

```
while iteration < max_iterations:
    1. Launch fresh Claude Code instance
    2. Claude reads program.md (its instructions)
    3. Claude reads results.tsv + STRATEGY.md (state)
    4. Claude picks ONE optimization to try
    5. Claude implements the change
    6. Claude rebuilds (if needed)
    7. Claude runs benchmark.py --verify
    8. Claude decides: KEEP or DISCARD
    9. Claude logs to results.tsv
    10. Claude updates STRATEGY.md
    11. Claude exits (context destroyed)
    12. analysis.py regenerates progress.png
    13. Loop restarts → Step 1
```

### Inside One Iteration (program.md)

```
Step 1: READ STATE
  └─ results.tsv → what's been tried, current best
  └─ STRATEGY.md → what to try next, bottleneck analysis

Step 2: DECIDE
  └─ Pick ONE idea from STRATEGY.md ideas list
  └─ Prefer high-impact (biggest bottleneck first)
  └─ If ideas exhausted → analyze code for new ones

Step 3: IMPLEMENT
  └─ Edit source files (small, focused change)
  └─ One idea per experiment (isolate variables)

Step 4: BUILD (if C++ or compiled language)
  └─ Run build command
  └─ If fails: fix if trivial, log crash if fundamental

Step 5: BENCHMARK
  └─ Run: benchmark.py --verify
  └─ Parse: total metric + per-benchmark + correctness

Step 6: DECIDE KEEP/DISCARD
  └─ See Decision Logic section below

Step 7: LOG
  └─ Append to results.tsv (even if discarded!)

Step 8: UPDATE STRATEGY
  └─ Move idea to "Already Tried"
  └─ Record WHY it worked or failed
  └─ Add new ideas if insights discovered

Step 9: COMMIT (if keeping)
  └─ git add + commit with descriptive message
```

---

## 4. Claude Code CLI Flags Explained

The loop script launches Claude with specific flags. Here's what each does
and why it's needed:

### `claude -p "$(cat program.md)"`

**What**: Print mode (headless). Runs non-interactively — reads the prompt,
executes, prints output, exits. No TUI, no interactive input.

**Why**: Each iteration needs a FRESH context. `-p` ensures Claude starts
clean every time. The prompt (program.md content) is passed via stdin.

**Alternative**: Without `-p`, Claude opens its interactive TUI. Good if you
want to watch, but can't be automated in a loop.

### `--dangerously-skip-permissions`

**What**: Skips ALL permission prompts. Claude can read/write/execute
anything without asking.

**Why**: The agent needs full autonomy to:
- Read source files
- Edit code
- Run build commands
- Execute benchmarks
- Write to results.tsv
- Run git commands

Without this flag, Claude would pause and ask permission for every tool call,
making autonomous operation impossible.

**Safety**: Only use in isolated environments or projects where you trust the
agent. The agent can modify any file and run any command.

### `--max-turns 50`

**What**: Limits the number of tool-call turns per session. After 50 turns,
Claude stops even if not done.

**Why**: Safety limit to prevent infinite loops or runaway agents. One
iteration typically needs:
- ~5 turns: read state files
- ~5 turns: analyze code
- ~5 turns: implement change
- ~5 turns: build
- ~10 turns: run benchmark (long-running command)
- ~5 turns: log results and update strategy
- Total: ~35 turns for a smooth iteration

30 turns was too tight (caused incomplete iterations). 50 gives enough room
for complex changes with retries.

**Trade-off**: More turns = more tokens = more cost per iteration. But an
incomplete iteration is wasted entirely, so it's better to allow enough.

### `--effort high`

**What**: Sets Claude's reasoning depth. Options: `low`, `medium`, `high`, `max`.

**Why**: Optimization decisions need deep analysis:
- Reading complex C++ code
- Understanding performance implications
- Deciding which optimization to try
- Analyzing benchmark results

`high` gives thorough reasoning without the extreme cost of `max`.

**Trade-off**: Higher effort = more thinking tokens = higher cost and latency.
For simple tasks, `medium` suffices. For optimization requiring code analysis,
`high` is recommended.

### Other Useful Flags (optional)

| Flag | What it does | When to use |
|---|---|---|
| `--max-budget-usd 5.00` | Cap spending per iteration | Cost control for expensive models |
| `--model sonnet` | Use faster/cheaper model | When optimization decisions are simple |
| `--output-format json` | Structured JSON output | If you need programmatic parsing of results |
| `--no-session-persistence` | Don't save session to disk | Reduces disk usage for many iterations |
| `--fallback-model sonnet` | Auto-fallback if primary overloaded | Prevents stalls during high traffic |
| `--append-system-prompt "..."` | Add custom system instructions | Project-specific constraints |
| `-w` / `--worktree` | Run in git worktree | Parallel experiments on separate branches |
| `-n "iter-42"` / `--name` | Name the session | Easier to find in `/resume` picker |
| `--verbose` | Show more detail | Debugging agent behavior |

### Flags NOT used (and why)

| Flag | Why not used |
|---|---|
| `--allowed-tools "..."` | Replaced by `--dangerously-skip-permissions` (more permissive). Use `--allowed-tools` if you want finer control: `--allowed-tools "Bash,Read,Write,Edit,Glob,Grep"` |
| `--continue` / `--resume` | We WANT fresh context each iteration. These continue previous sessions. |
| `--input-format stream-json` | For real-time streaming input, not batch operations |
| `--permission-mode auto` | Alternative to `--dangerously-skip-permissions`. Options: `default` (ask), `acceptEdits` (auto-accept edits), `bypassPermissions` (skip all), `plan` (plan only), `auto` (intelligent auto-accept). `bypassPermissions` is equivalent to `--dangerously-skip-permissions` |

### Advanced: Using --worktree for Parallel Experiments

Claude Code can create isolated git worktrees per session. This enables
running MULTIPLE optimization agents in parallel on different branches:

```bash
# Agent 1: optimize adaptive_esi
claude -p "$(cat program_adaptive.md)" -w "opt-adaptive" --dangerously-skip-permissions

# Agent 2: optimize esi_ess (parallel, different worktree)
claude -p "$(cat program_ess.md)" -w "opt-ess" --dangerously-skip-permissions
```

Each agent works on an isolated copy of the repo. Merge results later.

---

## 5. The Ralph Loop Pattern

Named after the [Ralph Loop Claude plugin](https://claude.com/plugins/ralph-loop),
this pattern solves the context degradation problem.

### The Problem

In Karpathy's original autoresearch, the agent runs in ONE long session:

```
Iteration 1:  [fresh context] → great reasoning
Iteration 50: [compressed context] → degraded reasoning
Iteration 100: [heavily compressed] → poor decisions
```

The context window fills up with logs, diffs, and outputs. Claude's automatic
compression helps but inevitably loses nuance.

### The Solution: Die and Restart

```
Iteration 1:  [fresh] → reads files → optimizes → logs → DIES
Iteration 2:  [fresh] → reads files → optimizes → logs → DIES
Iteration 100: [fresh] → reads files → optimizes → logs → DIES
                  ↑ Same quality as iteration 1!
```

Each iteration gets the SAME quality of reasoning because:
1. Context starts clean
2. All state is in files (not in context)
3. The agent reconstructs its understanding from files each time

### Trade-offs

| Aspect | Long session | Ralph Loop |
|---|---|---|
| Reasoning quality | Degrades over time | Constant |
| Per-iteration startup | Zero | ~10s (reading files) |
| State continuity | Implicit (in context) | Explicit (in files) |
| Crash recovery | Manual | Automatic (next iter starts fresh) |
| Cost per iteration | Lower (cached context) | Higher (fresh context each time) |
| Parallelism | Impossible | Possible (git worktrees) |

---

## 6. Variance Analysis System

### The Problem We Discovered

During our spatialize optimization, we found that `adaptive_esi` (the dominant
benchmark at 81% of total time) had ±1s measurement variance. This meant:

- A real 0.12s improvement in `esi_idw_3d` was MASKED by noise
- The agent discarded 4 experiments that had real per-benchmark improvements
- Total time was dominated by random fluctuation in the largest benchmark

### The Solution: Per-Benchmark Acceptance

Instead of only checking `total_time < best_total_time`, we check each
benchmark independently:

```python
for benchmark in benchmarks:
    best_ever = min(all kept values for this benchmark)
    improvement = (best_ever - current) / best_ever
    if improvement > 0.05:  # >5% improvement
        # Check no other benchmark regressed >5%
        if no_regression:
            KEEP  # Real improvement, not noise
```

### Establishing Variance Profile

During initial setup, run the benchmark N times (N=5 recommended) WITHOUT
changing code. This gives you the natural variance of each benchmark:

```
Run 1: bench_a=25.3s  bench_b=2.1s  bench_c=1.4s
Run 2: bench_a=26.1s  bench_b=2.0s  bench_c=1.5s
Run 3: bench_a=25.8s  bench_b=2.1s  bench_c=1.4s
Run 4: bench_a=26.4s  bench_b=2.0s  bench_c=1.5s
Run 5: bench_a=25.5s  bench_b=2.1s  bench_c=1.4s

Variance: bench_a=±0.5s  bench_b=±0.05s  bench_c=±0.05s
```

Now you know: a 0.1s improvement in bench_b is REAL (>2σ), but a 0.5s
improvement in bench_a is NOISE (within 2σ).

### Artifact Detection

If ALL experiments (including discards) show the same "improvement" in a
benchmark, it's likely an artifact:

```
Kept baseline:  bench_c = 1.56s
Discard #1:     bench_c = 1.44s  ← "improved"?
Discard #2:     bench_c = 1.44s  ← same "improvement"?
Discard #3:     bench_c = 1.44s  ← ALL show 1.44s?
Discard #4:     bench_c = 1.45s  ← suspicious...
```

This pattern means the BASELINE measurement (1.56s) was the outlier, not that
all experiments improved bench_c. The true value is ~1.44s.

---

## 7. Decision Logic

Complete decision tree for keep/discard:

```
┌─ Benchmark finished
│
├─ correctness == FAIL?
│  └─ YES → DISCARD immediately (broken change)
│
├─ Build crashed?
│  ├─ Fixable (typo, missing include)? → Fix and retry
│  └─ Fundamental? → Log as CRASH, revert, move on
│
├─ Benchmark timed out (>2× expected)?
│  └─ → Log as CRASH, revert, move on
│
├─ total_metric improved?
│  └─ YES → KEEP (clear win)
│
├─ total_metric didn't improve, but individual benchmark improved >5%?
│  ├─ Any other benchmark regressed >5%?
│  │  ├─ YES → DISCARD (trade-off, not pure win)
│  │  └─ NO → KEEP (real improvement masked by noise)
│  └─ NO individual improvement >5%
│
├─ No metric improved, but code is SIMPLER?
│  ├─ Fewer lines, removed complexity, cleaner design?
│  │  └─ YES → KEEP (Karpathy's simplicity criterion)
│  └─ NO → DISCARD (no benefit)
│
└─ Small improvement (<2%) with significant added complexity?
   └─ → DISCARD (complexity cost > marginal improvement)
```

### Karpathy's Simplicity Criterion

From the original autoresearch:

> "A 0.001 val_bpb improvement that adds 20 lines of hacky code? Probably not
> worth it. Removing code with equal/better results? DEFINITELY keep."

This is critical. Without it, agents tend to accumulate complexity that makes
future optimizations harder.

---

## 8. File-by-File Reference

### benchmark.py (IMMUTABLE)

The ground truth. Defines what to measure and how to verify correctness.

**Requirements**:
- Reproducible: same code → same results (use fixed seeds)
- Fast enough: total benchmark time should be 30-120s (more = slow iterations)
- Comprehensive: cover all hot paths
- Parseable output: last lines must be grep-able

**Output format** (grep-friendly):
```
---
total_time_s:             31.847075
bench_adaptive_esi_2d:    25.882603
bench_esi_ess_2d:         2.124782
correctness:              PASS
```

**Flags**:
- `--save`: save baseline checksums (run once at setup)
- `--verify`: verify results match baseline (run every iteration)

### baseline_checksums.json (IMMUTABLE)

Generated by `benchmark.py --save`. Contains hashes of expected outputs:

```json
{
  "bench_name": {
    "estimation_checksum": "ad07f9d4965501c3c98c8dc2bc45268f",
    "estimation_mean": -0.006841679569333792
  }
}
```

Any code change that alters these values = FAIL = automatic discard.

### program.md (IMMUTABLE)

Instructions for the agent. Read at the START of every iteration. Contains:
- Environment description (paths, build commands)
- The 9-step iteration protocol
- Keep/discard decision rules
- File permissions (what can/cannot be modified)
- Architecture reference (where the hot paths are)

### STRATEGY.md (MUTABLE — agent updates each iteration)

The agent's persistent brain. Contains:
- **Current State**: best metric, iteration count
- **Bottleneck Analysis**: per-benchmark breakdown with priorities
- **Variance Profile**: measurement noise bands
- **Ideas to Try**: prioritized list of optimization ideas
- **Ideas Already Tried**: what was tried, result, and WHY it worked/failed
- **Exhausted Approaches**: categories that are depleted (don't retry)
- **Key Insights**: accumulated knowledge

### results.tsv (MUTABLE — append only)

Tab-separated log of ALL experiments:

```
commit	total_time_s	bench1	bench2	correctness	status	description
baseline	53.54	33.39	14.59	PASS	keep	initial measurement
abc1234	44.74	30.36	9.93	PASS	keep	pow→arithmetic in distance()
def5678	32.23	26.34	2.12	PASS	discard	fuse two-pass → no improvement
```

**Critical**: even DISCARDED and CRASHED experiments are logged. This prevents
the agent from re-trying failed ideas.

### analysis.py (IMMUTABLE)

Generates progress.png with:
- Top panel: total metric over time (green=kept, gray=discarded, red=crash)
- Bottom panel: per-benchmark breakdown
- Running best line
- Annotations with descriptions

### run_autoresearch.sh / .ps1 (IMMUTABLE)

The Ralph Loop launcher. Iterates, launching fresh Claude instances.

---

## 9. Setting Up From Scratch

### Step 1: Identify Your Optimization Target

Answer these questions:
1. What metric? (time, accuracy, size, cost, score)
2. Lower or higher is better?
3. How to measure it? (script, test suite, API call)
4. How to verify correctness? (tests, checksums, golden outputs)
5. What code can change? (which files/directories)
6. Build step needed? (compile, bundle, deploy)

### Step 2: Create the Benchmark

Write `benchmark.py` that:
1. Prepares fixed input data (deterministic, seeded)
2. Runs each sub-benchmark N times (N=3-5)
3. Takes the median (robust to outliers)
4. Computes checksums of outputs
5. Prints parseable results
6. Supports `--save` and `--verify` flags

### Step 3: Establish Baseline

```bash
python benchmark.py --save
```

This creates `baseline_checksums.json` and prints the baseline measurements.

### Step 4: Measure Variance

Run the benchmark 5 times WITHOUT changing code:

```bash
for i in {1..5}; do python benchmark.py; done
```

Calculate std dev per benchmark. Record in STRATEGY.md under "Variance Profile".

### Step 5: Create Supporting Files

Use the templates in [templates.md](templates.md) to create:
- `program.md` (fill in paths, commands, file permissions)
- `STRATEGY.md` (fill in baseline values, initial ideas)
- `results.tsv` (one baseline row)
- `analysis.py` (adapt column names)
- `run_autoresearch.sh` (adapt paths and runtime)

### Step 6: Initialize Git

```bash
git init  # or use existing repo
git checkout -b autoresearch/my-optimization
git add benchmark.py baseline_checksums.json program.md analysis.py run_autoresearch.sh
git commit -m "autoresearch: initial setup"
```

### Step 7: Test One Iteration

```bash
claude -p "$(cat program.md)" --dangerously-skip-permissions --max-turns 50 --effort high
```

Verify:
- Agent read the state files
- Made a change
- Ran the benchmark
- Logged to results.tsv
- Updated STRATEGY.md

### Step 8: Launch the Loop

```bash
bash run_autoresearch.sh --max 10  # start with 10 to validate
```

---

## 10. The Supervisor Pattern

While the agent works autonomously, a supervisor (human or parent agent)
monitors and intervenes when needed.

### What the Supervisor Does

| Action | How | When |
|---|---|---|
| Check progress | Read `results.tsv` | Periodically or on request |
| View timeline | View `progress.png` | After each batch |
| Audit discards | Analyze per-benchmark data in discarded rows | After streaks of discards |
| Adjust strategy | Edit `STRATEGY.md` directly | When agent is stuck |
| Add ideas | Add to "Ideas to Try" in STRATEGY.md | When human has insights |
| Stop/restart | Ctrl+C / re-run script | Any time |
| Change parameters | Edit run_autoresearch.sh | Between batches |

### Running as Supervisor from Claude Code

If you're in a Claude Code session and want to supervise:

```bash
# Launch in background
bash run_autoresearch.sh --max 3 &

# Check progress
cat results.tsv

# View chart
# (open progress.png)

# Audit discards
python -c "
import pandas as pd
df = pd.read_csv('results.tsv', sep='\t')
print(df[df['status']=='discard'])
"
```

---

## 11. Lessons Learned

### From Karpathy's Autoresearch
1. **One metric, one file, one loop** — simplicity is the architecture
2. **Log everything** — even failures are data that prevents re-trying
3. **Git as experiment tracker** — commits are checkpoints, resets are rollbacks
4. **Never stop** — overnight runs produce 100+ experiments
5. **Simplicity criterion** — don't accumulate complexity for marginal gains

### From Our Implementation
6. **Fresh context per iteration** — quality doesn't degrade at iteration 100
7. **Correctness is non-negotiable** — checksums catch silent regressions
8. **Variance masks real improvements** — per-benchmark analysis is essential
9. **Document failures with WHY** — prevents the agent from repeating mistakes
10. **Exhausted categories** — mark whole categories as "don't retry" (e.g., "compiler flags")
11. **Supervisor audits** — a human/agent reviewing discards finds hidden improvements
12. **Build scripts are fragile** — document platform-specific compilation issues in skills
13. **Protect immutable files** — explicitly list files the agent CANNOT modify
14. **50 turns > 30 turns** — complex changes (C++ build + benchmark) need room
15. **`--effort high`** — optimization reasoning needs deep analysis, not speed

### Anti-Patterns to Avoid
- **Multiple changes per iteration** — can't isolate what helped/hurt
- **Modifying the benchmark** — invalidates all previous measurements
- **Ignoring crashes** — they often indicate interesting edge cases
- **Only checking total metric** — per-benchmark analysis reveals more
- **Long sessions** — context degrades, use Ralph pattern instead
- **No variance baseline** — you can't distinguish signal from noise

---

## 12. Troubleshooting

### Agent runs out of turns (max-turns hit)

**Symptom**: "Error: Reached max turns (N)"
**Fix**: Increase `--max-turns` in run_autoresearch.sh. 50 is good for most
cases. Complex changes with C++ builds may need 60-70.

### Agent keeps retrying failed ideas

**Symptom**: Same category of experiments keeps appearing
**Fix**: Add to "Exhausted Approaches" in STRATEGY.md with explicit instructions
not to retry. Example: "Compiler flags: 4 experiments failed. Do not try more."

### Correctness check always fails

**Symptom**: Every experiment gets FAIL on correctness
**Possible causes**:
- Floating point non-determinism (different thread ordering with OpenMP)
- Fix: use `OMP_NUM_THREADS=1` in benchmark or relax checksum to mean comparison
- Baseline was saved with different code than current HEAD
- Fix: re-run `benchmark.py --save` on current code

### Build fails inside Claude

**Symptom**: Agent can't compile C++ code
**Fix**: Create a build script (like `build.ps1`) that encapsulates all
platform-specific setup (VS DevShell, environment variables, etc.)

### Measurement variance is too high

**Symptom**: Same code gives wildly different benchmark results
**Fix**:
- Close other applications during benchmarking
- Increase N_RUNS (median of more runs = more stable)
- Pin CPU frequency if possible
- Use `taskset` (Linux) or processor affinity to isolate cores
- Run benchmarks with process priority (nice -20 / high priority)

### Agent modifies files it shouldn't

**Symptom**: benchmark.py or run_autoresearch.sh gets edited
**Fix**: Add explicit list of protected files in program.md. We learned this
the hard way when the agent edited run_autoresearch.sh and broke the loop.

---

## 13. Adapting to Your Use Case

### Example: Optimize ML Model Accuracy

```
Metric:        F1 score (higher is better)
Verifier:      Minimum accuracy threshold on test set
Scope:         model config, feature engineering, hyperparameters
Benchmark:     train + evaluate on fixed dataset
Build:         None (Python only)
```

### Example: Reduce Docker Image Size

```
Metric:        Image size in MB (lower is better)
Verifier:      Container starts and passes health check
Scope:         Dockerfile, .dockerignore, dependencies
Benchmark:     docker build + docker images | grep size
Build:         docker build
```

### Example: Optimize API Latency

```
Metric:        p95 latency in ms (lower is better)
Verifier:      Integration tests pass, correct responses
Scope:         query optimization, caching, code paths
Benchmark:     Load test with fixed request set
Build:         None (or restart server)
```

### Example: Improve LLM Prompt Quality

```
Metric:        Eval score on golden dataset (higher is better)
Verifier:      No harmful outputs, format compliance
Scope:         system prompt, few-shot examples
Benchmark:     Run prompt against eval set, score with judge LLM
Build:         None
```

### Key Adaptation Points

1. **benchmark.py**: Change what you measure and how
2. **program.md**: Change file permissions and build commands
3. **STRATEGY.md**: Change initial ideas and bottleneck analysis
4. **analysis.py**: Change `cummin` to `cummax` if higher is better
5. **results.tsv**: Change column names to match your benchmarks
6. **run_autoresearch.sh**: Change runtime path and build commands
