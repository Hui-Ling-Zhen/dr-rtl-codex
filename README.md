# Dr. RTL Codex: Agentic RTL Optimization Across Harnesses

This repository is a Codex-adapted version of `dr-rtl`(https://github.com/hkust-zhiyao/Dr_RTL). The original project
uses Claude Code sub-agents to drive iterative RTL timing optimization with
synthesis and sequential equivalence checking (SEC) in the loop. This fork keeps
that workflow, adds Codex project instructions, and explores how the same agentic
RTL optimization loop changes under different agent harnesses.

At a high level, the optimization loop is unchanged: analyze synthesized timing
reports, propose RTL-level transformations, run synthesis plus SEC, score each
attempt, and promote the best SEC-passing version. The main difference is how the
outer agent is orchestrated.

- Claude Code reads `CLAUDE.md` and can invoke `.claude/agents/*.md` as native
  sub-agents.
- Codex reads `AGENTS.md`, loads `.codex/config.toml`, and can spawn registered
  role agents from `.codex/agents/*.toml` with native multi-agent tools. The
  Claude sub-agent files remain useful as reference prompts.

For the detailed Codex command flow, see `README-code.md`.

## Directory layout

```
dr-rtl-codex/
├── README-code.md          # Detailed Codex build/run instructions
├── AGENTS.md               # Codex root coordinator rules
├── CLAUDE.md               # Claude Code orchestrator rules
├── .codex/
│   ├── config.toml         # Codex multi-agent role registration
│   ├── agents/             # Codex role TOML files
│   └── skills/rtl-opt/     # Codex-safe RTL optimization skill summary
├── .claude/
│   ├── agents/             # Sub-agent definitions (timing, optimizer, evaluator, skill extractor)
│   └── skill/rtl-opt/      # RTL optimization skill pack
├── rtl_dataset/            # Baseline RTL designs (v0) — aes, arm_cpu*, tv80, i2c, pcie, ...
├── syn_flow/               # Optimization loop working directory
│   ├── rtl/                # Versioned RTL: <design>.v0.v, <design>.v0.1.v, ...
│   ├── log/                # Per-iteration iter.json files
│   ├── history/            # Per-design history.json (major rounds + promotions)
│   ├── output/             # Synthesis outputs: PPA_report.json, timing_word.json, SEC_result.txt
│   ├── design_all.json     # Design config: [top, clk, rst, ext]
│   ├── run_remote.py       # SSH + SCP wrapper that runs synthesis on the EDA host
│   └── add_design_remote.py
└── syn_flow_eda/           # Runs on the EDA machine (DC / Formality / Jasper SEC)
    ├── run_design.py       # Orchestrates dc_shell, fm_shell, jaspergold
    ├── scr/                # TCL templates: syn.tcl, fm.tcl, sec.tcl
    ├── lib/nangate.db      # Nangate standard-cell library
    └── reports/ netlist/ output/ log/
```

## Harness Adaptation

The original `dr-rtl` workflow assumes Claude Code as the harness:

```text
CLAUDE.md
.claude/agents/*.md
.claude/skill/rtl-opt/skill.md
```

In this repository, Codex is used as another outer executor:

```text
AGENTS.md
.codex/config.toml
.codex/agents/*.toml
.codex/skills/rtl-opt/SKILL.md
Codex CLI / SDK
  -> reads AGENTS.md
  -> spawns .codex registered roles
  -> edits and reads syn_flow/
  -> runs syn_flow/run_remote.py
  -> remote EDA host runs syn_flow_eda/run_design.py
```

This is not a one-to-one replacement for Claude Code sub-agents. Codex should
use native multi-agent tools (`spawn_agent`, `wait_agent`, `followup_task`,
`send_message`, `list_agents`, `close_agent`) to invoke registered RTL roles.

## Workflow

For each major version `v0` → `v9`:
1. Run 5 minor attempts (`vX.1` … `vX.5`), each with a distinct diversity strategy chosen by the orchestrator.
2. For every minor: **analyze** critical paths → **optimize** RTL → **synthesize + SEC** → **record** score.
3. Pick the lowest-scoring SEC-passing minor, promote it to the next major version.

Stop when `v10` is created (`max_major_reached`) or a full round completes with no promotion (`no_improvement`).

See `CLAUDE.md` for the original Claude Code spec and `AGENTS.md` for the Codex
adaptation.

## Sub-agents

| Agent | Role |
|---|---|
| `rtl-timing-analyzer` | Select K=10–25 critical paths from `timing_word.json`, diagnose root causes, classify amenability. |
| `rtl-optimizer` | Apply RTL transformations; SEC validates functional equivalence. |
| `rtl-synthesis-evaluator` | Run remote synthesis + SEC via a single Python command. Execution only. |
| `rtl-opt-skill-extractor` / `-per-design` | Distill reusable optimization patterns from completed trajectories. |

## Scoring

Lower is better. Baseline is `v0` for each design.

```
score = 0.5·WNS_norm + 0.35·TNS_norm + 0.15·Area_norm + penalty
WNS_norm  = (WNS  − WNS_baseline)  / |WNS_baseline|
TNS_norm  = (TNS  − TNS_baseline)  / |TNS_baseline|
Area_norm = (Area − Area_baseline) / Area_baseline
penalty   = 0.5 if Area_norm > 0.10 else 0
```

## Running

Synthesis itself runs remotely. The local agent harness only edits RTL, records
iteration state, and calls the remote wrapper.

```bash
# From syn_flow/: upload RTL, run DC + SEC on the EDA host, download reports
python3 run_remote.py <design> <version>      # e.g. tv80 v0.1

# Locally on the EDA host (syn_flow_eda/):
python3 run_design.py <design> <version>      # dc_shell → jaspergold → parse reports
```

Requires Synopsys Design Compiler, Synopsys Formality, and Cadence JasperGold on the EDA host, plus `paramiko` and `scp` locally.

### Running With Claude Code

Use Claude Code from the repository root. Claude Code should follow `CLAUDE.md`,
delegate to the native `.claude/agents/*.md` sub-agents, and use
`syn_flow/run_remote.py` for evaluation.

### Running With Codex

Codex should be launched from this repository root so it can read `AGENTS.md`.
If the workspace is not recognized as a Git repository, pass
`--skip-git-repo-check`. Codex must be allowed to write versioned RTL and logs,
so use a writable sandbox.

Installed Codex example:

```bash
codex exec \
  --cd "/path/to/dr-rtl-codex" \
  --skip-git-repo-check \
  --sandbox workspace-write \
  --config 'approval_policy="on-request"' \
  "请读取 AGENTS.md，对 tv80 从 v0 开始执行一轮 RTL timing optimization。先确认 v0 的 output 是否存在；如果不存在先评估 v0。然后生成 v0.1 到 v0.5 五个 minor attempts，按 Analyze -> Optimize -> Evaluate -> Record 执行，最后选择 SEC pass 且 score 最低的版本 promote 到 v1，并更新 syn_flow/log 和 syn_flow/history。"
```

Expected Codex execution chain:

```text
Codex
  -> read AGENTS.md
  -> load .codex/config.toml role definitions
  -> read syn_flow/design_all.json
  -> check syn_flow/output/<design>.v0/
  -> run syn_flow/run_remote.py <design> v0 if baseline output is missing
  -> spawn rtl_analyzer / rtl_optimizer / rtl_evaluator / rtl_recorder
  -> repeat role sequence for v0.1 ... v0.5
  -> spawn rtl_selector after all five records complete
  -> spawn rtl_memory_archiver to snapshot raw_memory/attempt_*
  -> spawn rtl_skill_extractor to update .codex/skills/rtl-opt/SKILL.md
  -> update syn_flow/rtl/, syn_flow/log/, and syn_flow/history/
```

For local source-built Codex and WSL2-specific commands, see `README-code.md`.

## Artifacts per iteration

- `syn_flow/log/<design>.<version>.iter.json` — diversity strategy, critical paths, before/after metrics, score.
- `syn_flow/history/<design>.history.json` — baseline PPA, best version, promotion log across major rounds.
- `syn_flow/output/<design>.<version>/` — `PPA_report.json`, `timing_word.json`, `SEC_result.txt`.
- `raw_memory/attempt_*` — archived completed rounds for skill extraction.
- `skill/attempt_*/*.skills.json` and `skill/rtl-opt-skills.md` — extracted reusable optimization knowledge.

## Codex Notes

- Codex does not automatically invoke `.claude/agents/rtl-timing-analyzer.md`,
  `.claude/agents/rtl-optimizer.md`, or
  `.claude/agents/rtl-synthesis-evaluator.md`.
- `AGENTS.md` is the primary Codex coordinator file and tells Codex how to
  invoke the registered Codex roles.
- Do not fabricate EDA results. If `PPA_report.json`, `timing_word.json`, or
  `SEC_result.txt` is missing, run the evaluation wrapper or report the exact
  environment blocker.
- A minimal first test is one major round on `tv80`: `v0 -> v1`.
