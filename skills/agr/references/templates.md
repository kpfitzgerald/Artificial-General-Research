# Autoresearch Framework Templates

> These templates are adapted by the `/autoresearch-init` skill to fit each project.
> Variables in `{{CURLY_BRACES}}` are replaced during setup.

## benchmark.py Template

```python
"""
{{PROJECT_NAME}} Benchmark Suite
===========================
Fixed, reproducible benchmark. DO NOT MODIFY.

Metrics:
  - {{METRIC_NAME}} ({{METRIC_UNIT}}, {{METRIC_DIRECTION}} is better)
  - correctness (results must match baseline)

Usage:
    python benchmark.py              # run benchmarks
    python benchmark.py --save       # save baseline checksums
    python benchmark.py --verify     # verify correctness against baseline
"""

import sys
import os
import json
import hashlib
import time
import warnings
import statistics

# ============================================================================
# CONSTANTS — DO NOT MODIFY
# ============================================================================
SEED = 42
N_RUNS = {{N_RUNS}}              # median of N_RUNS for stability (3-5 recommended)
BASELINE_FILE = os.path.join(os.path.dirname(__file__), "baseline_checksums.json")

def _checksum(data):
    """Compute MD5 checksum for correctness verification."""
    import numpy as np
    if isinstance(data, np.ndarray):
        return hashlib.md5(np.ascontiguousarray(data).tobytes()).hexdigest()
    return hashlib.md5(str(data).encode()).hexdigest()


# ============================================================================
# DATA PREPARATION (not timed)
# ============================================================================
def prepare_data():
    """Prepare benchmark inputs. Called once before all benchmarks."""
    # {{SETUP_CODE}}
    return data


# ============================================================================
# INDIVIDUAL BENCHMARKS — each returns a dict for correctness checking
# ============================================================================

def bench_{{BENCH1_NAME}}(data):
    """{{BENCH1_DESCRIPTION}}"""
    # {{BENCH1_CODE}}
    return {"checksum": _checksum(result), "summary_stat": float(summary)}

# ... more benchmarks ...


# ============================================================================
# RUNNER (generic — copy as-is)
# ============================================================================
ALL_BENCHMARKS = [
    ("bench_{{BENCH1_NAME}}", bench_{{BENCH1_NAME}}),
    # ... more benchmarks ...
]


def run_all(n_runs=N_RUNS, verify=False, save_baseline=False):
    """Run full benchmark suite."""
    print(f"Preparing data...", flush=True)
    data = prepare_data()
    print(f"Running {len(ALL_BENCHMARKS)} benchmarks x {n_runs} runs.\n", flush=True)

    timing_results = {}
    correctness_results = {}

    for name, func in ALL_BENCHMARKS:
        times = []
        check = None
        for run_i in range(n_runs):
            try:
                t0 = time.perf_counter()
                check = func(data)
                t = time.perf_counter() - t0
                times.append(t)
                print(f"  {name} run {run_i+1}/{n_runs}: {t:.4f}s", flush=True)
            except Exception as e:
                print(f"  {name} run {run_i+1}/{n_runs}: FAILED ({e})", flush=True)
                times.append(float("inf"))

        median_t = statistics.median(times)
        timing_results[name] = median_t
        if check is not None:
            correctness_results[name] = check
        print(f"  -> {name} median: {median_t:.6f}s\n", flush=True)

    total = sum(v for v in timing_results.values() if v != float("inf"))

    # Save baseline
    if save_baseline:
        with open(BASELINE_FILE, "w") as f:
            json.dump(correctness_results, f, indent=2)
        print(f"\nBaseline saved to {BASELINE_FILE}")

    # Verify correctness
    correctness_ok = True
    if verify:
        if not os.path.exists(BASELINE_FILE):
            print("\nWARNING: No baseline found. Run with --save first.")
            correctness_ok = False
        else:
            with open(BASELINE_FILE) as f:
                baseline = json.load(f)
            print("\n=== CORRECTNESS ===")
            for name, current in correctness_results.items():
                if name not in baseline:
                    continue
                expected = baseline[name]
                match = all(current.get(k) == expected.get(k) for k in expected)
                print(f"  {name}: {'PASS' if match else 'FAIL'}")
                if not match:
                    correctness_ok = False

    # Parseable output
    print("\n---")
    print(f"total_{{METRIC_NAME}}:          {total:.6f}")
    for name, t in timing_results.items():
        print(f"{name + ':':30s}{t:.6f}")
    if verify:
        print(f"correctness:              {'PASS' if correctness_ok else 'FAIL'}")

    return timing_results, total, correctness_ok


if __name__ == "__main__":
    save = "--save" in sys.argv
    verify = "--verify" in sys.argv or not save
    results, total, ok = run_all(verify=verify, save_baseline=save)
    sys.exit(0 if (total < float("inf") and ok) else 1)
```

---

## program.md Template

```markdown
# {{PROJECT_NAME}} Auto-Research Program

You are an autonomous optimization agent for **{{PROJECT_NAME}}**.
Your goal: make {{METRIC_DESCRIPTION}} without breaking correctness.

## Environment
- **Project**: {{PROJECT_PATH}}
- **Benchmark**: `benchmark.py` — fixed, DO NOT MODIFY
- **Baseline**: `baseline_checksums.json` — DO NOT MODIFY
- **Results**: `results.tsv` — experiment log
- **Strategy**: `STRATEGY.md` — what to try next
- **Build**: {{BUILD_COMMAND_OR_NONE}}
- **Runtime**: {{RUNTIME_PATH}}

## One Iteration = One Experiment

### Step 1: Read State
Read `results.tsv` and `STRATEGY.md` to know where you are.

### Step 2: Decide What to Try
- Pick ONE idea from STRATEGY.md
- Prefer high-impact ideas targeting the biggest bottleneck
- Start simple, escalate only if simple ideas are exhausted
- If all ideas exhausted, analyze code for new ones

### Step 3: Implement
- Keep changes small and focused — one idea per experiment
- **CAN modify**: {{MODIFIABLE_FILES}}
- **CANNOT modify**: benchmark.py, baseline_checksums.json, run_autoresearch.sh, run_autoresearch.ps1, analysis.py

### Step 4: Build (if needed)
{{BUILD_INSTRUCTIONS}}
If build fails: fix if trivial, log as "crash" if fundamental.

### Step 5: Benchmark
```bash
{{RUNTIME_PATH}} benchmark.py --verify
```

### Step 6: Decide Keep or Discard

**KEEP** if `correctness: PASS` AND any of:
- total metric improved, OR
- ANY individual benchmark improved >5% vs best-ever, with no other regressing >5%, OR
- Code is SIMPLER (fewer lines, less complexity) with equal results

**DISCARD** if:
- `correctness: FAIL`
- No benchmark improved beyond measurement noise
- Build crashed and can't be fixed
- Small improvement but significant added complexity

If DISCARD: `git checkout -- {{MODIFIABLE_PATHS}}`

### Step 7: Log Results
Append to `results.tsv` (TAB-separated):
```
{commit}	{total}	{bench1}	{bench2}	...	{correctness}	{status}	{description}
```

### Step 8: Update Strategy
- Move tried idea to "Ideas Already Tried" with result and WHY it worked/failed
- Update "Current State" with new best if improved
- Add to "Exhausted Approaches" if a category of attempts is depleted
- Add new ideas if you discovered insights during implementation

### Step 9: Commit (if keeping)
```bash
git add -A && git commit -m "perf: {description}"
```

## Rules
1. ONE experiment per invocation
2. NEVER modify benchmark.py or baseline_checksums.json
3. Correctness is non-negotiable
4. Log EVERYTHING (even failures — they prevent re-trying bad ideas)
5. Update STRATEGY.md — it's your brain between sessions
6. Simple first — try simplest optimization before complex ones
7. Simplicity criterion: added complexity needs proportional improvement
8. Don't obsess over crashes — fix if trivial, skip if fundamental
```

---

## STRATEGY.md Template

```markdown
# Optimization Strategy

## Current State
- **Best {{METRIC_NAME}}**: {{BASELINE_VALUE}} (baseline)
- **Iteration count**: 0

## Bottleneck Analysis
| Benchmark | Value | % of total | Priority |
|---|---|---|---|
| {{BENCH1}} | {{VALUE}} | {{PCT}}% | {{PRIORITY}} |

## Variance Profile
| Benchmark | Median | Std Dev | Noise Band (±2σ) |
|---|---|---|---|
| {{BENCH1}} | {{MEDIAN}} | {{STDDEV}} | ±{{BAND}} |

Improvements must exceed 2σ noise band to be considered real.

## Ideas to Try (priority order)
1. {{IDEA_1}}
2. {{IDEA_2}}

## Ideas Already Tried
(none yet)

## Exhausted Approaches
(none yet)

## Key Insights
- {{INITIAL_INSIGHT}}
```

---

## analysis.py Template

```python
"""
Autoresearch Progress Analyzer — generates progress.png
"""
import pandas as pd
import matplotlib
import matplotlib.pyplot as plt
import numpy as np

matplotlib.use("Agg")

def load_results(path="results.tsv"):
    df = pd.read_csv(path, sep="\t")
    df["experiment"] = range(1, len(df) + 1)
    return df

def plot_progress(df, metric_col="total_{{METRIC_NAME}}", output="progress.png"):
    fig, axes = plt.subplots(2, 1, figsize=(14, 10), dpi=150,
                              gridspec_kw={"height_ratios": [3, 1]})

    ax = axes[0]
    ax.set_title("{{PROJECT_NAME}} Optimization Timeline", fontsize=16, fontweight="bold")
    ax.set_xlabel("Experiment #")
    ax.set_ylabel("{{METRIC_NAME}} ({{METRIC_UNIT}})")

    # Discarded (gray)
    discarded = df[df["status"] == "discard"]
    if len(discarded) > 0:
        ax.scatter(discarded["experiment"], discarded[metric_col],
                   c="lightgray", s=40, zorder=2, label="Discarded", alpha=0.7)

    # Crashed (red X)
    crashed = df[df["status"] == "crash"]
    if len(crashed) > 0:
        ax.scatter(crashed["experiment"],
                   [df[metric_col].max() * 1.05] * len(crashed),
                   c="red", marker="x", s=60, zorder=2, label="Crash")

    # Kept (green, connected)
    kept = df[df["status"] == "keep"]
    if len(kept) > 0:
        ax.scatter(kept["experiment"], kept[metric_col],
                   c="#2ecc71", s=80, zorder=3, edgecolors="darkgreen",
                   linewidths=0.5, label="Kept")
        ax.plot(kept["experiment"], kept[metric_col],
                c="#2ecc71", linewidth=2, zorder=2, alpha=0.7)

        # Running best line
        running_best = kept[metric_col].cummin()  # cummin for lower-is-better, cummax for higher
        ax.step(kept["experiment"], running_best,
                c="darkgreen", linewidth=1.5, linestyle="--", alpha=0.5,
                label="Best so far", where="post")

        # Annotate
        for _, row in kept.iterrows():
            desc = str(row["description"])[:40]
            ax.annotate(desc, xy=(row["experiment"], row[metric_col]),
                        xytext=(5, 10), textcoords="offset points",
                        fontsize=6, rotation=30, alpha=0.8,
                        arrowprops=dict(arrowstyle="-", alpha=0.3))

    ax.legend(loc="upper right")
    ax.grid(True, alpha=0.3)

    # Baseline reference
    baseline = df.iloc[0][metric_col]
    ax.axhline(y=baseline, color="gray", linestyle=":", alpha=0.5)
    ax.text(0.02, baseline, f"  baseline: {baseline:.2f}",
            transform=ax.get_yaxis_transform(), fontsize=8, color="gray")

    # Bottom: per-benchmark breakdown
    ax2 = axes[1]
    ax2.set_title("Benchmark Breakdown (kept)")
    ax2.set_xlabel("Experiment #")
    ax2.set_ylabel("{{METRIC_UNIT}}")

    bench_cols = [c for c in df.columns if c not in
                  ["commit", metric_col, "correctness", "status", "description", "experiment"]]
    colors = plt.cm.Set1(np.linspace(0, 1, max(len(bench_cols), 1)))

    if len(kept) > 0:
        for bench, color in zip(bench_cols, colors):
            if bench in kept.columns:
                ax2.plot(kept["experiment"], kept[bench], marker="o",
                         markersize=4, label=bench, color=color, linewidth=1.5)
        ax2.legend(loc="upper right", fontsize=8, ncol=3)
        ax2.grid(True, alpha=0.3)

    plt.tight_layout()
    plt.savefig(output, bbox_inches="tight")
    print(f"Saved: {output}")

    # Summary
    total = len(df)
    n_keep = len(kept)
    n_discard = len(discarded)
    n_crash = len(crashed)
    print(f"\n=== SUMMARY ===")
    print(f"Experiments: {total} ({n_keep} kept, {n_discard} discarded, {n_crash} crashed)")
    if len(kept) > 1:
        improvement = (1 - kept[metric_col].min() / kept.iloc[0][metric_col]) * 100
        print(f"Baseline: {kept.iloc[0][metric_col]:.2f} → Best: {kept[metric_col].min():.2f} ({improvement:.1f}%)")

if __name__ == "__main__":
    plot_progress(load_results())
```

---

## run_autoresearch.sh Template

```bash
#!/bin/bash
# Autoresearch Loop — Ralph pattern
# Usage: bash run_autoresearch.sh --max 10

PROJECT_DIR="$(cd "$(dirname "$0")" && pwd)"
cd "$PROJECT_DIR"

MAX_ITERATIONS=100

while [[ $# -gt 0 ]]; do
    case $1 in
        --max) MAX_ITERATIONS=$2; shift 2 ;;
        *) shift ;;
    esac
done

echo "========================================"
echo " AUTORESEARCH: {{PROJECT_NAME}}"
echo " Max iterations: $MAX_ITERATIONS"
echo "========================================"

for ((i=1; i<=MAX_ITERATIONS; i++)); do
    echo ""
    echo "=== ITERATION $i / $MAX_ITERATIONS [$(date '+%Y-%m-%d %H:%M:%S')] ==="
    echo ""

    claude -p "$(cat program.md)" \
        --dangerously-skip-permissions \
        --max-turns 50 \
        --effort high \
        2>&1 || true

    echo "--- Iteration $i done ---"
    {{RUNTIME_PATH}} analysis.py 2>/dev/null || true
    echo ""; cat results.tsv; echo ""
    sleep 2
done

echo "=== COMPLETE: $MAX_ITERATIONS iterations ==="
```

---

## run_autoresearch.ps1 Template

```powershell
# Autoresearch Loop — Ralph pattern (PowerShell)
param([int]$MaxIterations = 100)

$ProjectDir = Split-Path -Parent $MyInvocation.MyCommand.Path
Set-Location $ProjectDir
$Prompt = Get-Content "$ProjectDir\program.md" -Raw

for ($i = 1; $i -le $MaxIterations; $i++) {
    Write-Host "=== Iteration $i / $MaxIterations ===" -ForegroundColor Yellow
    claude -p $Prompt --dangerously-skip-permissions --max-turns 50 --effort high
    & "{{RUNTIME_PATH}}" "$ProjectDir\analysis.py" 2>$null
    Get-Content "$ProjectDir\results.tsv"
    Start-Sleep -Seconds 2
}
```

---

## .gitignore Template

```
# Autoresearch artifacts (local to each machine)
results.tsv
progress.png
*.log
__pycache__/
```
