---
name: rtl-opt
description: "Codex-safe RTL timing optimization skill summary. Use during the rtl_optimizer role."
license: "Derived from .claude/skill/rtl-opt/skill.md; see original LICENSE.txt."
---

# Codex RTL Optimization Skill

This is the Codex-oriented, self-evolving RTL optimization skill pack. Use it
during the `rtl_optimizer` role.

This file starts from the original Claude skill library and should be updated by
`rtl_skill_extractor` after each archived major round. If a path type is
ambiguous or this file lacks detail, read the full reference skill at:

```text
.claude/skill/rtl-opt/skill.md
```

## Default Strategy Order

Prefer transformations in this order:

1. Conservative combinational cleanup.
2. Shared condition and comparison extraction.
3. One-hot pre-decode wires for state, opcode, and control signals.
4. Common subexpression extraction.
5. Local boolean, mux, and arithmetic restructuring.
6. Sequential-safe changes only when latency and protocol are unchanged.

Do not use aggressive sequential transformations unless the equivalence
argument is clear from the local RTL.

## High-Confidence Strategies

### One-Hot Pre-Decode For Control Signals

Applies to FSM or opcode signals that fan out into many repeated comparisons.
Create named wires such as `state_is_idle`, `op_is_add`, or `count_is_max` and
reuse them. This reduces comparator fanout and redundant decode logic.

Risk: low when the original encoding and behavior are unchanged.

### Condition Pre-Computation

Applies to counter logic, FSM-controlled datapaths, and nested conditionals.
Extract repeated conditions into shared wires. Preserve the original control
flow and always-block behavior.

Risk: low. This is the preferred first attempt for many control-heavy paths.

### Register Duplication For Fanout Reduction

Applies to high-fanout control or enable signals when duplicated registers can
preserve identical update, reset, and enable semantics. Split fanout cones with
local copies.

Risk: low to medium. Do not add observable latency or alter reset behavior.

### FSM Output Registration Or Boundary Retiming

Applies when FSM-derived outputs drive long downstream combinational paths.
Only use when the register movement is sequentially safe and does not change
architectural latency.

Risk: medium. Skip unless cycle behavior remains equivalent.

### Common Subexpression Extraction

Applies to repeated arithmetic, comparison, or bitwise expressions. Create a
single named wire for the repeated expression and reuse it.

Risk: low to medium. Keep widths explicit.

## Medium-Confidence Strategies

### Late Mux Select With Unconditional Computation

Applies when control selection is on the critical path before arithmetic.
Compute candidate values in parallel and select late.

Risk: medium. Area may increase and all candidate expressions must be valid for
all input combinations.

### Mux-Before-Adder Restructuring

Applies to structures like `sel ? (A + B) : (C + D)`. Consider rewriting as
`(sel ? A : C) + (sel ? B : D)` when equivalence is obvious and widths match.

Risk: medium. Verify signedness and width behavior.

### Logic Tree Balancing

Applies to long XOR, AND, OR, add, or compare trees. Reassociate only when the
operator is safe to regroup under the exact RTL semantics.

Risk: medium. Be careful with signed arithmetic, overflow, and reduction order.

### Hierarchical Case Or Lookup Decomposition

Applies to large case statements, encoders, decoders, lookup tables, or
structured protocol mappings. Split monolithic decode into smaller structured
sub-decodes when the hierarchy is explicit and equivalence is local.

Risk: medium. Requires careful functional mapping and evidence.

### Compare-Before-Mux Or Mux-Before-Compare

Applies when a selected value is immediately compared to a threshold. Compare
candidate values in parallel and mux the boolean result when widths and
signedness are clear.

Risk: medium. Preserve signedness and corner-case semantics exactly.

### Explicit Width Boundary Extraction

Applies to mixed-width arithmetic, implicit extensions, truncation, and fixed
constant operations. Create named wires for extensions/truncations so synthesis
sees stable boundaries.

Risk: low to medium. Do not shrink datapaths unless equivalence is proven.

## Sequential-Safe Options

Use only when no externally visible latency or protocol changes:

- Register duplication for fanout reduction when the duplicated register has
  identical update, reset, and enable semantics.
- Local retiming inside a purely internal cone when SEC should prove identical
  cycle behavior.

Skip the path if the change requires adding a pipeline stage, delaying an
output, changing a handshake, or modifying reset behavior.

## Anti-Patterns

Avoid these by default:

- Adding or removing architectural pipeline stages.
- Changing module interfaces.
- Changing externally visible latency.
- Removing resets or clock enables.
- Changing counter direction or FSM encoding for timing without a strong local
  equivalence argument.
- Flattening priority logic unless mutual exclusion is proven.
- Registering mux selectors speculatively.
- Rewriting memory addressing, LFSR feedback, or protocol control casually.
- Relying on synthesis to infer a transformation that the RTL no longer clearly
  expresses.

## Path Type To Strategy Mapping

| Path Characteristic | Preferred Strategy | Confidence |
| --- | --- | --- |
| Repeated FSM/opcode decode | One-hot pre-decode, condition extraction | High |
| Counter terminal checks | Condition pre-computation, explicit compare wires | High |
| High-fanout control | Register duplication or replicated local decode | Medium |
| Arithmetic gated by control | Late mux select, mux-before-adder | Medium |
| Large case or lookup | Hierarchical decomposition | Medium |
| XOR/GF/crypto logic | Logic tree balancing | Medium |
| Mixed-width arithmetic | Explicit width boundary extraction | Medium |
| Priority if/else | Preserve priority; avoid flattening without proof | Anti-pattern |
| Memory addressing | Usually skip; architectural or physical issue | Out-of-scope |
| LFSR feedback | Do not alter polynomial or feedback structure | Anti-pattern |

## Required Recording

Every optimized critical path must record:

- `applied`
- `type`
- `strategy`
- `detail`
- `skill_source`
- `skill_confidence`

Every skipped critical path must record:

- `applied: false`
- `type: "skipped"`
- `reason`

Do not predict SEC or PPA results in the Optimize phase.

## Evidence From Local Extraction

`rtl_skill_extractor` owns this section. It should append or update extracted
rules from `raw_memory/attempt_*` after each completed major round.

Required evidence format:

```text
- Strategy: <name>
  Applies to: <path characteristics>
  Evidence: <design.version or raw_memory attempt id>
  SEC pass rate: <N/M>
  Promotions: <count>
  Avg score / WNS / TNS / area movement: <metrics when available>
  Risk: <low|medium|high>
```

## Evidence Gaps

No local `raw_memory/attempt_*` evidence has been extracted yet. Until evidence
exists, prefer conservative strategies and cite the original Claude skill
reference when using a strategy that is not yet locally validated.
