# Raw Memory Archive

This directory is append-only evidence for the self-evolving RTL optimization
loop.

After each completed major round, `rtl_memory_archiver` creates:

```text
raw_memory/attempt_<timestamp>_<design>_<major_version>/
├── metadata.json
├── history/
├── log/
├── rtl/
└── output/
```

These archives are the source of truth for skill extraction. Do not edit or
delete archived attempts unless explicitly requested.

The skill extractor reads these attempts to learn:

- successful promoted optimizations,
- SEC-passing but non-promoted attempts,
- SEC-failing or timing-degrading attempts,
- repeated anti-patterns,
- diversity strategies that found useful paths.
