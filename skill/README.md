# Extracted Skill Memory

This directory stores generated skill summaries derived from
`raw_memory/attempt_*`.

Expected outputs from `rtl_skill_extractor`:

```text
skill/
├── attempt_<timestamp>/
│   └── <design>.skills.json
└── rtl-opt-skills.md
```

`skill/attempt_*/*.skills.json` contains per-design pattern statistics.
`skill/rtl-opt-skills.md` contains cross-design reusable patterns and
anti-patterns.

The active optimizer-facing skill pack is:

```text
.codex/skills/rtl-opt/SKILL.md
```

That file should be updated from this extracted evidence after every completed
major round.
