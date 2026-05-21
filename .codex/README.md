# Codex Multi-Agent Scaffold

This directory contains the Codex-native scaffold for the Dr. RTL optimization
loop.

Unlike the original Claude Code scaffold, Codex should use runtime multi-agent
tools and registered agent roles:

- `spawn_agent`
- `wait_agent`
- `followup_task`
- `send_message`
- `list_agents`
- `close_agent`

## Layout

```text
.codex/
├── config.toml
├── agents/
│   ├── rtl_analyzer.toml
│   ├── rtl_optimizer.toml
│   ├── rtl_evaluator.toml
│   ├── rtl_recorder.toml
│   └── rtl_selector.toml
└── skills/
    └── rtl-opt/
        └── SKILL.md
```

## Role Registration

`config.toml` enables `multi_agent_v2` and registers role names:

- `rtl_analyzer`
- `rtl_optimizer`
- `rtl_evaluator`
- `rtl_recorder`
- `rtl_selector`
- `rtl_memory_archiver`
- `rtl_skill_extractor`

The root `AGENTS.md` acts as the coordinator. It should spawn these roles rather
than performing all phase logic in the root thread.

## Canonical Memory

All roles must treat filesystem artifacts as the only authoritative workflow
memory:

- `syn_flow/log/<design>.<version>.iter.json`
- `syn_flow/history/<design>.history.json`
- `syn_flow/output/<design>.<version>/PPA_report.json`
- `syn_flow/output/<design>.<version>/timing_word.json`
- `syn_flow/output/<design>.<version>/SEC_result.txt`
- `syn_flow/rtl/<design>.<version>.<ext>`

Do not rely on what an agent remembers from previous messages. At each role
invocation, reconstruct state from these files.

## Phase Boundary Rule

For every minor attempt, run these roles in order:

1. `rtl_analyzer`
2. `rtl_optimizer`
3. `rtl_evaluator`
4. `rtl_recorder`

After all five minor attempts for a major version complete, run:

5. `rtl_selector`
6. `rtl_memory_archiver`
7. `rtl_skill_extractor`

Analyze and Optimize may run in parallel when their dependencies are ready.
Evaluate should run serially unless the remote EDA flow is confirmed to support
safe concurrent runs. Record, promotion/history updates, archive, and skill
updates must be serial.

## Self-Evolving Loop

After every completed major round, the coordinator must archive the round into
`raw_memory/attempt_*` and then extract skills into:

- `.codex/skills/rtl-opt/SKILL.md`
- `skill/attempt_*/<design>.skills.json`
- `skill/rtl-opt-skills.md`

Future `rtl_optimizer` roles must read the updated Codex skill before proposing
new RTL transformations.
