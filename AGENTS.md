# AGENTS.md

This repository is a Codex-driven, self-evolving RTL timing optimization
workspace. Treat this file as the root coordinator instruction for Codex.

The detailed phase behavior lives in Codex agent roles registered by
`.codex/config.toml`:

- `rtl_analyzer`
- `rtl_optimizer`
- `rtl_evaluator`
- `rtl_recorder`
- `rtl_selector`
- `rtl_memory_archiver`
- `rtl_skill_extractor`

Do not inline those role details in the root thread. The root thread should
coordinate roles with Codex multi-agent tools.

## Core Rule

Use Codex native multi-agent orchestration when available:

- `spawn_agent`
- `wait_agent`
- `followup_task`
- `send_message`
- `list_agents`
- `close_agent`

The root coordinator owns workflow sequencing. Role agents own their scoped
filesystem work.

## Repository Map

- `syn_flow/` is the local optimization working directory.
- `syn_flow/rtl/` contains versioned RTL files such as `<design>.v0.v`,
  `<design>.v0.1.v`, and `<design>.v1.v`.
- `syn_flow/output/<design>.<version>/` contains generated evaluation outputs:
  `PPA_report.json`, `timing_word.json`, and `SEC_result.txt`.
- `syn_flow/log/<design>.<version>.iter.json` stores per-minor analysis,
  optimization, evaluation, and scoring data.
- `syn_flow/history/<design>.history.json` stores baseline PPA, best version,
  major rounds, promotions, and end reason.
- `syn_flow/design_all.json` maps each design to `[top, clk, rst, ext]`.
- `syn_flow/run_remote.py` uploads RTL to the EDA host, runs the remote flow,
  and downloads results.
- `.codex/config.toml` registers Codex multi-agent roles.
- `.codex/agents/*.toml` contains role-specific developer instructions.
- `.codex/skills/rtl-opt/SKILL.md` is the Codex optimization skill summary.
- `raw_memory/attempt_*` stores immutable archives of completed optimization
  attempts for self-evolution.
- `skill/attempt_*/` and `skill/rtl-opt-skills.md` store extracted reusable
  knowledge generated from raw memory.
- `.claude/` remains the Claude Code reference scaffold.

## Canonical Memory

Do not use chat memory as workflow state. Durable state must be reconstructed
from filesystem artifacts:

- `syn_flow/log/<design>.<version>.iter.json`
- `syn_flow/output/<design>.<version>/PPA_report.json`
- `syn_flow/output/<design>.<version>/timing_word.json`
- `syn_flow/output/<design>.<version>/SEC_result.txt`
- `syn_flow/history/<design>.history.json`
- `syn_flow/rtl/<design>.<version>.<ext>`
- `raw_memory/attempt_*/metadata.json`
- `raw_memory/attempt_*/log/*.iter.json`
- `raw_memory/attempt_*/history/*.history.json`
- `skill/attempt_*/*.skills.json`
- `skill/rtl-opt-skills.md`

Subagents should communicate complex state by writing these files, not by
passing long summaries through chat.

## Main EDA Entry

Single-version evaluation must go through:

```bash
cd syn_flow && python3 run_remote.py <design> <version>
```

If the environment uses `python`, use the local equivalent.

Do not run synthesis, SEC, or report parsing directly unless the user explicitly
asks for debugging.

## Optimization Workflow

When the user requests RTL optimization for `<design>`:

1. Confirm baseline output exists for `<design>.v0`.
2. If baseline output is missing, spawn `rtl_evaluator` for `v0`, then spawn
   `rtl_recorder` only if recording baseline data is needed.
3. For each major version `vX`, create five independent minors:
   `vX.1`, `vX.2`, `vX.3`, `vX.4`, and `vX.5`.
4. For each minor, run:
   Analyze -> Optimize -> Evaluate -> Record.
5. After all five minors complete Record, spawn `rtl_selector`.
6. After `rtl_selector` completes, spawn `rtl_memory_archiver` to snapshot the
   completed round into `raw_memory/attempt_*`.
7. After archiving, spawn `rtl_skill_extractor` to mine successes/failures and
   update `.codex/skills/rtl-opt/SKILL.md`.
8. Stop after the requested scope:
   one minor, one major round, or the full loop through `v10`.

Self-evolution is not optional for completed major rounds. If a run reaches
Select/Promote, it must also Archive and Extract before the coordinator reports
the round complete.

## Role Invocation Pattern

Use stable, lowercase task names. Example for `tv80`, base `v0`, target `v0.1`:

```text
spawn_agent(
  task_name = "tv80_v0_1_analyze",
  agent_type = "rtl_analyzer",
  message = "Analyze design tv80 from base_version v0 to target_version v0.1. Use minor_index 1. Write only syn_flow/log/tv80.v0.1.iter.json.",
  fork_turns = "none"
)
wait_agent(...)
```

Then spawn `rtl_optimizer`, `rtl_evaluator`, and `rtl_recorder` with the same
design/base/target identifiers. After all five minors are recorded, spawn
`rtl_selector` once for the major version.

After selector completion:

```text
spawn_agent(
  task_name = "tv80_v0_archive",
  agent_type = "rtl_memory_archiver",
  message = "Archive completed design tv80 major_version v0 into raw_memory/attempt_*. Include logs, RTL, outputs, history, and metadata.",
  fork_turns = "none"
)
wait_agent(...)
```

Then:

```text
spawn_agent(
  task_name = "tv80_v0_extract_skills",
  agent_type = "rtl_skill_extractor",
  message = "Extract reusable RTL optimization skills from raw_memory for design tv80 and update .codex/skills/rtl-opt/SKILL.md plus skill/ summaries.",
  fork_turns = "none"
)
wait_agent(...)
```

Use `followup_task` when a spawned role needs a correction or narrow continuation
that should trigger a new turn. Use `send_message` only for non-triggering
coordination. Use `list_agents` to inspect live/completed agents. Use
`close_agent` after a role has completed and its filesystem artifacts are
verified.

## Parallelism Policy

Allowed parallelism:

- Analyze can run in parallel across the five minor attempts after baseline
  reports exist.
- Optimize can run in parallel after each minor's analyze output exists.

Restricted parallelism:

- Evaluate should run serially by default because `run_remote.py`, EDA licenses,
  remote work directories, and generated output paths may conflict.
- Evaluate may be limited-concurrency only when the user confirms the remote
  flow supports independent work directories and enough EDA licenses.

Must be serial:

- Record should run after its matching Evaluate completes.
- Promotion must run once after all five Records for the major version complete.
- History updates must be serialized.
- Memory archive must run after promotion/history update.
- Skill extraction must run after memory archive.
- Skill pack updates must be serialized.
- Any cleanup of generated output must be explicit and user-approved.

## Filesystem Handoff Contract

The coordinator should verify phase completion by checking files:

- Analyze complete: `syn_flow/log/<design>.<target>.iter.json` exists.
- Optimize complete: `syn_flow/rtl/<design>.<target>.<ext>` exists and
  `iter.json` has `critical_paths[*].optimization`.
- Evaluate complete: target output directory contains `PPA_report.json`,
  `timing_word.json`, and `SEC_result.txt`.
- Record complete: `iter.json` has `evaluation` and `scoring`.
- Promotion complete: next major RTL exists and `history.json` is updated.
- Archive complete: a new `raw_memory/attempt_*` directory exists with
  `metadata.json`, `log/`, `rtl/`, and available `output/` artifacts.
- Skill extraction complete: `.codex/skills/rtl-opt/SKILL.md` is updated or
  explicitly preserved with an evidence-gap note, and `skill/attempt_*/` summary
  files exist.

If an expected file is missing, do not invent results. Ask the responsible role
to continue with `followup_task` or report the blocker.

## Versioning

Use the current major version as the base for all five minors:

```text
v0 -> v0.1 ... v0.5 -> best SEC-pass minor promotes to v1
v1 -> v1.1 ... v1.5 -> best SEC-pass minor promotes to v2
...
```

Stop conditions:

- `v10` is created: `end_reason = "max_major_reached"`.
- A requested major round has no promotable SEC-passing minor:
  `end_reason = "no_improvement"` when ending the workflow.

## Scoring Ownership

The root coordinator must not calculate scores directly unless no
`rtl_recorder` role is available. The `rtl_recorder` role owns per-minor
evaluation/scoring. The `rtl_selector` role owns winner selection and promotion.

Lower score is better. SEC-failing attempts are never promotable.

## Self-Evolving Skill Loop

The optimizer must treat `.codex/skills/rtl-opt/SKILL.md` as the current local
skill memory. The skill extractor is responsible for keeping that file current.

After each completed major round:

1. `rtl_memory_archiver` copies completed artifacts from `syn_flow/` into a new
   immutable `raw_memory/attempt_*` directory.
2. `rtl_skill_extractor` reads all relevant raw memory, identifies successful
   and failed strategies, writes per-design summaries under `skill/attempt_*/`,
   updates `skill/rtl-opt-skills.md`, and updates
   `.codex/skills/rtl-opt/SKILL.md`.
3. Future `rtl_optimizer` invocations must read the updated Codex skill before
   proposing transformations.

If evidence is insufficient, preserve existing skill content and write an
`Evidence Gaps` note rather than inventing rules. Successful and failed attempts
must both be represented; anti-patterns are as important as winning strategies.

The Claude skill at `.claude/skill/rtl-opt/skill.md` is the fuller reference
library. The Codex skill should copy necessary strategy detail from it when the
Codex skill would otherwise be only a short summary.

## Prompt Templates

### One Minor

```text
Use Codex multi-agent roles from .codex/config.toml. For <design>, execute one
minor attempt from <base_version> to <target_version>. Spawn rtl_analyzer, wait;
spawn rtl_optimizer, wait; spawn rtl_evaluator, wait; spawn rtl_recorder, wait.
Stop after Record. Use filesystem artifacts as the only workflow memory.
```

### One Major Round

```text
Use Codex multi-agent roles from .codex/config.toml. For <design>, execute one
major round from <base_version>. Create five independent minors. Analyze and
Optimize may run in parallel where dependencies allow. Evaluate serially unless
explicitly approved otherwise. Record each minor after its evaluation. After all
five Records, spawn rtl_selector once. Then spawn rtl_memory_archiver and
rtl_skill_extractor. Stop after promotion/history/archive/skill update.
```

## Safety

- Do not fabricate EDA results.
- Do not delete existing RTL, output, log, or history files unless the user asks.
- Keep generated changes under `syn_flow/rtl/`, `syn_flow/log/`,
  `syn_flow/history/`, and `syn_flow/output/` unless the user requests scaffold
  edits.
- This workspace may not be a Git repository; Codex may need
  `--skip-git-repo-check`.
