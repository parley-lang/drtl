# DRTL - Declarative Reasoning Topology Language

> *What if AI reasoning was structured, auditable, and enforceable - instead of prompt-driven and implicit?*

DRTL is a declarative language for defining **deterministic AI reasoning topologies**.

It began as an experiment in multi-LLM debate. But building that reliably exposed a deeper need:

**Reasoning itself must be structured. Not improvised.**

DRTL is not a debate framework.<br />
It is not a prompt router.<br />
It is not a workflow wrapper.

**It is a contract language for AI reasoning execution.**

Debate is just one topology pattern.

---

## What Is DRTL?

DRTL describes reasoning as a directed graph of typed epistemic operations.

Instead of embedding structure in prompts, you declare:
- **What** operations execute (generate, verify, debate, review, transform, ...)
- **How** they connect (sequence, parallel, loop, conditional)
- **When** execution proceeds or halts (gates, enforcement modes, obligations)
- **Who** must approve before continuation (human, policy, agent)

The topology is the source of truth. The runtime executes it - deterministically.

## Core Principle

> A reasoning runtime can refuse to proceed when obligations are unmet.

If a model asserts something verifiable without verification, execution can halt.<br />
If a constraint fails, routing changes.<br />
If a gate condition is false, the topology controls what happens next.

Structure governs reasoning - not prompts.

The core principle: **a reasoning runtime can refuse to proceed when obligations are unmet.**


```yaml
# A reasoning pipeline that blocks on unverified claims
nodes:
  - id: generate_summary
    type: generate
    model: gemini/gemini-2.5-flash
    prompt: "Summarize the key features of Aurora Serverless v3"
    output_key: "raw_summary"

  - id: extract_claims
    type: generate
    model: openai/gpt-4o-mini
    prompt: "Extract product names, URLs, and performance claims as JSON from: {{generate_summary.raw_summary}}"
    output_format: json
    output_key: "structured_claims"

  - id: verify_claims
    type: verify
    input: extract_claims.structured_claims
    rules:
      - id: std.check_existence
        target: "product_names"
        mode: block        # [!] Hard stop on unrecognized entities
      - id: std.check_links
        target: "urls"
        mode: block
    output_key: "verification_report"

  - id: safety_gate
    type: gate
    input: verify_claims.verification_report
    condition: "input.blocking_failures == 0"
    on_pass: publish
    on_fail:
      next: rejection
      inject: verify_claims.verification_report

  - id: rejection
    type: review
    actor: human
    input:
      verification_report: "{{injected}}"
    actions: [retry_with_search, reject, override]
```

When this topology runs and the model hallucinates a product name, the runtime refuses to publish. See [`examples/aurora_demo/`](examples/aurora_demo/) for the full topology and execution trace.

---

## Why a topology language?

Most multi-agent frameworks route prompts. DRTL executes **typed epistemic operations with enforceable constraints**.

The difference matters when:

- You need **reproducibility** - the same topology run twice produces the same reasoning structure, not a different interpretation of a natural language prompt
- You need **auditability** - every execution produces a canonical trace log that maps claims to verification results to routing decisions
- You need **refusal** - `mode: block` is a hard runtime constraint, not a suggestion in a system prompt
- You need **human-in-the-loop as a first-class node** - the `review` node pauses execution and waits for an external decision before resuming

---

## Node Types

| Type | Description |
|------|-------------|
| `generate` | LLM call producing output |
| `fan_out` | Parallel generation across multiple models |
| `aggregate` | Combine multiple outputs (synthesize, vote, rank) |
| `verify` | Epistemic checks against structured claims |
| `gate` | Conditional routing based on predicates |
| `debate` | Multi-turn structured exchange with consensus protocol |
| `transform` | Deterministic state mutation (no LLM) |
| `review` | Execution pause requiring external decision (human, policy, agent) |

---

## Enforcement Modes

Verification rules run in one of three modes:

- `observe` - log only, continue
- `warn` - inject warning into context, continue
- `block` - halt execution, require resolution

---

## Repository Structure

```
drtl/
├── spec/
│   └── DRTL_v0.1.1_spec.md            # Language specification
├── examples/
│   └── aurora_demo/
│       ├── aurora_demo.yaml           # Example topology
│       └── aurora_demo_trace_log.txt  # Execution trace
└── README.md
```

**Coming:**
- `runtime/` - Parley reference runtime (Python, LangGraph + LiteLLM)
- `examples/debate/` - Multi-LLM consensus topology (the original Parley use case)
- Topology generator - describe your pipeline in natural language, get a DRTL topology file

---

## Status

| Component | Status |
|-----------|--------|
| DRTL spec v0.1.1 | Draft complete |
| Example topologies | Aurora fact-check |
| Parley reference runtime | In progress |
| Topology generator | Planned |
| Debate example topology | In progress |

---

## Spec

Read the full specification: [`spec/DRTL_v0.1.1_spec.md`](spec/DRTL_v0.1.1_spec.md)

---

## License

MIT
