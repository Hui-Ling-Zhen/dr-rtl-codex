# Running dr-rtl-codex With Codex

This document explains how to use the local `codex` codebase to drive
`dr-rtl-codex`.

## Conclusion

The current `codex` code can be used as the outer agent executor for
`dr-rtl-codex`, but it cannot directly and natively execute the original
Claude Code sub-agent setup.

`dr-rtl-codex` was originally designed around:

```text
CLAUDE.md
.claude/agents/*.md
.claude/skill/rtl-opt/skill.md
```

Codex uses `AGENTS.md` as its project instruction mechanism. This repository
also includes a Codex-native multi-agent scaffold under `.codex/`:

```text
AGENTS.md
.codex/config.toml
.codex/agents/*.toml
.codex/skills/rtl-opt/SKILL.md
```

Codex should read `AGENTS.md`, load `.codex/config.toml`, and use native
multi-agent tools to spawn the registered RTL roles.

## Required Components

There are three layers:

```text
Codex CLI / SDK
  -> reads dr-rtl-codex/AGENTS.md
  -> edits/reads dr-rtl-codex/syn_flow
  -> runs dr-rtl-codex/syn_flow/run_remote.py
  -> remote EDA host runs run_design.py
```

Important paths:

```text
dr-rtl-codex/AGENTS.md
dr-rtl-codex/.codex/config.toml
dr-rtl-codex/.codex/agents/
dr-rtl-codex/.codex/skills/rtl-opt/SKILL.md
dr-rtl-codex/CLAUDE.md
dr-rtl-codex/.claude/agents/
dr-rtl-codex/.claude/skill/rtl-opt/skill.md
dr-rtl-codex/syn_flow/run_remote.py
dr-rtl-codex/syn_flow_eda/run_design.py
dr-rtl-codex/syn_flow/design_all.json
dr-rtl-codex/syn_flow/rtl/
dr-rtl-codex/syn_flow/output/
dr-rtl-codex/syn_flow/log/
dr-rtl-codex/syn_flow/history/
```

## Step 1: Build Or Install Codex

The local `codex` repository is the OpenAI Codex CLI source tree. According to
its documentation, Windows usage is expected through WSL2.

From WSL2/Linux:

```bash
cd "/mnt/d/0000 key_code/simt-verification/codex/codex-rs"
cargo build
```

Check that Codex can run:

```bash
cargo run --bin codex -- --help
```

If a global `codex` command is already installed, this is enough:

```bash
codex --help
```

## Step 2: Verify The EDA Flow Alone

Before asking Codex to optimize RTL, first confirm that the normal EDA wrapper
works.

From WSL2/Linux:

```bash
cd "/mnt/d/0000 key_code/simt-verification/dr-rtl-codex/syn_flow"
python3 run_remote.py tv80 v0
```

This should execute:

```text
syn_flow/run_remote.py
  -> SSH/SCP upload RTL and design_all.json
  -> remote run_design.py
  -> DC synthesis
  -> Jasper SEC or simulation flow
  -> parse reports
  -> download output/<design>.<version>/
```

Expected local output after success:

```text
dr-rtl-codex/syn_flow/output/tv80.v0/PPA_report.json
dr-rtl-codex/syn_flow/output/tv80.v0/timing_word.json
dr-rtl-codex/syn_flow/output/tv80.v0/SEC_result.txt
```

If this step fails, fix the EDA environment first. Codex cannot complete the
optimization loop without these reports.

Common prerequisites:

- `syn_flow/run_remote.py` has valid SSH host, username, password or key, and
  remote path.
- Python packages `paramiko` and `scp` are installed locally.
- The remote host has the expected EDA work directory.
- The remote host has Synopsys Design Compiler and Cadence JasperGold available.
- The remote shell environment loads the required EDA licenses and tool paths.

## Step 3: Run Codex Against dr-rtl-codex

Using the local source-built Codex:

```bash
cd "/mnt/d/0000 key_code/simt-verification/codex/codex-rs"

cargo run --bin codex -- exec \
  --cd "/mnt/d/0000 key_code/simt-verification/dr-rtl-codex" \
  --skip-git-repo-check \
  --sandbox workspace-write \
  --config 'approval_policy="on-request"' \
  "请读取 AGENTS.md，并使用 .codex/config.toml 中注册的 Codex multi-agent roles。对 tv80 从 v0 开始执行一轮 RTL timing optimization：先确认 v0 output 是否存在；然后对 v0.1 到 v0.5 使用 spawn_agent/wait_agent 调度 rtl_analyzer、rtl_optimizer、rtl_evaluator、rtl_recorder；evaluate 默认串行；最后 spawn rtl_selector promote 最优 SEC-pass minor 到 v1，并更新 syn_flow/log 和 syn_flow/history。"
```

Using an installed `codex` command:

```bash
codex exec \
  --cd "/mnt/d/0000 key_code/simt-verification/dr-rtl-codex" \
  --skip-git-repo-check \
  --sandbox workspace-write \
  --config 'approval_policy="on-request"' \
  "请读取 AGENTS.md，并使用 .codex/config.toml 中注册的 Codex multi-agent roles。对 tv80 从 v0 开始执行一轮 RTL timing optimization：先确认 v0 output 是否存在；然后对 v0.1 到 v0.5 使用 spawn_agent/wait_agent 调度 rtl_analyzer、rtl_optimizer、rtl_evaluator、rtl_recorder；evaluate 默认串行；最后 spawn rtl_selector promote 最优 SEC-pass minor 到 v1，并更新 syn_flow/log 和 syn_flow/history。"
```

Use `--skip-git-repo-check` because this workspace may not be a Git repository.

Use `--sandbox workspace-write` because Codex must be able to create and update:

```text
syn_flow/rtl/<design>.<version>.<ext>
syn_flow/log/<design>.<version>.iter.json
syn_flow/history/<design>.history.json
```

Use approval mode according to your safety preference. `on-request` is safer
because Codex asks before important command execution.

## Expected Codex Execution Chain

When the prompt above is executed, Codex should follow this chain:

```text
Codex
  -> read AGENTS.md
  -> load .codex/config.toml role definitions
  -> read syn_flow/design_all.json
  -> determine RTL extension and design metadata
  -> check syn_flow/output/<design>.v0/
  -> run syn_flow/run_remote.py <design> v0 if baseline output is missing
  -> spawn rtl_analyzer for each minor attempt
  -> spawn rtl_optimizer after each matching analyze completes
  -> spawn rtl_evaluator serially for each target version
  -> spawn rtl_recorder after each evaluation completes
  -> spawn rtl_selector after all five records complete
  -> spawn rtl_memory_archiver to archive raw_memory/attempt_*
  -> spawn rtl_skill_extractor to update extracted skills
  -> promote best minor to syn_flow/rtl/<design>.v1.<ext>
  -> update syn_flow/history/<design>.history.json
```

## Key Generated Files

Baseline and per-version evaluation output:

```text
syn_flow/output/<design>.<version>/PPA_report.json
syn_flow/output/<design>.<version>/timing_word.json
syn_flow/output/<design>.<version>/SEC_result.txt
```

Per-minor optimization record:

```text
syn_flow/log/<design>.<version>.iter.json
```

Per-design promotion history:

```text
syn_flow/history/<design>.history.json
```

Self-evolving memory and skill outputs:

```text
raw_memory/attempt_<timestamp>_<design>_<major_version>/
skill/attempt_<timestamp>/<design>.skills.json
skill/rtl-opt-skills.md
.codex/skills/rtl-opt/SKILL.md
```

Versioned RTL:

```text
syn_flow/rtl/<design>.v0.<ext>
syn_flow/rtl/<design>.v0.1.<ext>
syn_flow/rtl/<design>.v0.2.<ext>
syn_flow/rtl/<design>.v1.<ext>
```

## Important Limitation

Codex does not automatically spawn the original Claude Code agents:

```text
.claude/agents/rtl-timing-analyzer.md
.claude/agents/rtl-optimizer.md
.claude/agents/rtl-synthesis-evaluator.md
```

Instead, `AGENTS.md` tells Codex to use native multi-agent roles registered in:

```text
.codex/config.toml
.codex/agents/rtl_analyzer.toml
.codex/agents/rtl_optimizer.toml
.codex/agents/rtl_evaluator.toml
.codex/agents/rtl_recorder.toml
.codex/agents/rtl_selector.toml
```

So this is not a one-to-one Claude Code replacement:

```text
Claude Code native sub-agents: automatic through .claude/agents
Codex: native multi-agent roles through .codex/config.toml and spawn_agent
```

## Troubleshooting

If Codex does not read the project rules, confirm that it is launched with:

```bash
--cd "/mnt/d/0000 key_code/simt-verification/dr-rtl-codex"
```

If Codex refuses to run because the folder is not a Git repository, add:

```bash
--skip-git-repo-check
```

If the EDA step fails, test manually:

```bash
cd "/mnt/d/0000 key_code/simt-verification/dr-rtl-codex/syn_flow"
python3 run_remote.py tv80 v0
```

If `timing_word.json` is missing, the synthesis/report parsing flow did not
finish successfully.

If `SEC_result.txt` is missing, the SEC or simulation/report parsing flow did
not finish successfully.

If `iter.json` is missing, the agent optimization phase did not run or did not
write the expected log file.

## Minimal First Test

The recommended first Codex test is only one major round:

```bash
codex exec \
  --cd "/mnt/d/0000 key_code/simt-verification/dr-rtl-codex" \
  --skip-git-repo-check \
  --sandbox workspace-write \
  --config 'approval_policy="on-request"' \
  "请按照 AGENTS.md，只对 tv80 执行 v0 -> v1 的一轮优化，不要继续 v1 之后的轮次。"
```

After that, check:

```text
syn_flow/log/
syn_flow/history/
syn_flow/rtl/
syn_flow/output/
```
