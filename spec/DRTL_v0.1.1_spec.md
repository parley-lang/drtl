# DRTL: Declarative Reasoning Topology Language

**Version:** 0.1.1
**Status:** Draft Specification

## 1. Overview

DRTL (Declarative Reasoning Topology Language) is a specification for defining composable, verifiable AI reasoning pipelines. Unlike traditional orchestration languages that route prompts, DRTL executes **typed epistemic operations** with **enforceable constraints**.

The core principle: **A reasoning runtime can refuse to proceed when obligations are unmet.**

## 2. Core Concepts

### 2.1 Topology

A topology is a directed graph of reasoning steps. It defines:
- **What** operations execute (nodes)
- **How** they connect (edges)
- **When** execution proceeds or halts (gates and obligations)

### 2.2 The Atomic Unit

The atomic unit is a **reasoning step**, not a conversation. Each step is a state transition: `T(State) -> State'`

### 2.3 State (The Epistemic Packet)

Every topology maintains a `State` object that flows between nodes. Required fields:

```yaml
state:
  artifacts: []      # Outputs from nodes (text, JSON, tool results)
  claims: []         # Extracted assertions with type and status
  obligations: []    # Required checks (pending/satisfied/failed)
  trace: []          # Append-only execution log
  variables: {}      # User-defined state (loop_count, injected_feedback, etc.)
```

## 3. Node Types

### 3.1 `generate`

Calls an LLM to produce output.

```yaml
- id: draft
  type: generate
  model: anthropic/claude-3-sonnet
  prompt_ref: prompts/task.md    # External file reference
  prompt: "Inline prompt text"   # Or inline prompt
  input: "{{variable}}"          # Template substitution
  output_key: "draft_output"     # Key in state.artifacts
```

### 3.2 `fan_out`

Parallel generation across multiple models.

```yaml
- id: parallel_draft
  type: fan_out
  input: intake.output
  participants:
    - model: anthropic/claude-opus
      prompt_ref: prompts/visionary.md
    - model: openai/gpt-4
      prompt_ref: prompts/pragmatist.md
    - model: google/gemini-pro
      prompt_ref: prompts/skeptic.md
  output_key: "parallel_outputs"  # Array of results
```

### 3.3 `aggregate`

Combines multiple outputs into one.

```yaml
- id: merge
  type: aggregate
  input: parallel_draft.outputs
  strategy: synthesize | vote | rank | concat
  model: anthropic/claude-sonnet  # Optional: LLM-powered synthesis
  output_key: "merged_output"
```

### 3.4 `verify`

Runs epistemic checks against structured claims. **This is Lever 2.**

**Important:** Verify nodes require structured input (JSON). If your source is unstructured text, add an explicit extraction node first.

```yaml
- id: fact_check
  type: verify
  input: extract_claims.structured_claims  # Must be structured JSON
  rules:
    - id: std.check_existence
      target: "product_names"    # JSON key in input
      mode: block | warn | observe
    - id: std.check_compute
      target: "calculations"
      mode: warn
    - id: std.check_links
      target: "urls"
      mode: block
  output_key: "verification_report"
```

**Standard Verification Library:**

| Rule ID | Category | Verification Method |
|---------|----------|---------------------|
| `std.check_compute` | Computational | Deterministic execution (math, dates, hashes) |
| `std.check_existence` | Existence | Search/index lookup (products, APIs, features) |
| `std.check_links` | URLs | HTTP validation |
| `std.check_citation` | References | Academic/documentation lookup |
| `std.check_code` | Code Safety | Static analysis, sandboxed execution |
| `std.check_logic` | Consistency | Self-consistency sampling |
| `std.check_protocol` | Format | Schema/regex validation |
| `std.check_tool_usage` | Meta | Verify tools were invoked before claims |

**Enforcement Modes:**

- `observe`: Log only, continue execution
- `warn`: Inject warning into context, continue
- `block`: Halt execution, require resolution

### 3.5 `gate`

Conditional routing based on predicates.

```yaml
- id: safety_gate
  type: gate
  input: verify.report
  condition: "input.blocking_failures == 0"
  on_pass: next_node
  on_fail:
    next: fallback_node
    inject: verify.report  # Pass failure context
```

**Routing shorthand:**

```yaml
# These are equivalent:
on_pass: next_node
on_pass:
  next: next_node
```

Use the expanded form when injection is needed:

```yaml
on_fail:
  next: rejection
  inject: verify_claims.verification_report
```

### 3.6 `debate`

Multi-turn serial exchange between participants with protocol enforcement.

```yaml
- id: council
  type: debate
  input: aggregate.output
  protocol_ref: protocols/parley_v1.yaml
  max_rounds: 5
  consensus:
    type: unanimous | majority | threshold
    requires: "explicit AGREE from all"
  participants:
    - id: architect
      role: author | reviewer | investor | pitcher
      model: anthropic/claude-opus
      persona: "Systems Architect"
      prompt_ref: prompts/architect.md
    - id: security
      role: reviewer
      model: openai/gpt-4
      persona: "Security Reviewer"
      prompt_ref: prompts/security.md
  output_key: "debate_outcome"
```

### 3.7 `transform`

Deterministic state mutation (no LLM calls).

```yaml
- id: increment_loop
  type: transform
  operations:
    - set: state.variables.loop_count
      value: "{{state.variables.loop_count + 1}}"
    - set: state.variables.last_code
      value: "{{fix.output}}"
```

### 3.8 `review`

Pauses execution and requires an external decision before proceeding. The decision-maker can be a human, an automated policy engine, another agent, or any external system.

```yaml
- id: escalation
  type: review
  actor: human | policy | agent | webhook  # Optional hint about decision-maker
  input:
    context: safety_gate.input
    violations: "{{injected}}"
  actions: [retry_with_search, reject, override]
```

**Fields:**

| Field | Required | Description |
|-------|----------|-------------|
| `actor` | No | Hint about expected decision-maker (human, policy, agent, webhook). Metadata only - runtime doesn't enforce. |
| `input` | Yes | Data provided to the reviewer for context |
| `message` | No | Human-readable explanation of why review is needed |
| `actions` | Yes | List of valid decisions the reviewer can make |

**Actions can route to other nodes:**

```yaml
actions:
  - retry_with_search:
      next: generate_with_search
  - reject:
      next: terminate
  - override:
      next: publish
```

Or be simple labels if routing is handled elsewhere:

```yaml
actions: [approve, reject, edit]
```

**Execution behavior:**

1. Runtime reaches `review` node
2. Execution pauses
3. External system receives input + message + available actions
4. External system returns chosen action
5. Runtime resumes, routing based on the action

## 4. Edge Semantics

### 4.1 Sequential

Default flow: output of node A becomes input to node B.

```yaml
edges:
  - from: intake
    to: draft
```

### 4.2 Conditional

Edge activates based on predicate.

```yaml
edges:
  - from: gate
    to: success_node
    if: passed
  - from: gate
    to: failure_node
    if: failed
```

**Note:** Conditional routing from `gate` nodes can also be defined in the node itself via `on_pass` and `on_fail`. When defined in the node, explicit edges are not required.

### 4.3 Loop

Edge points backward, creating iteration.

```yaml
edges:
  - from: loop_guard
    to: write_code    # Loop back
    if: under_limit
  - from: loop_guard
    to: human_review  # Exit loop
    if: exceeded
```

### 4.4 Fan-out / Fan-in

One node spawns parallel branches that reconverge.

```yaml
edges:
  - from: intake
    to: fan_out_node
  - from: fan_out_node
    to: aggregate_node  # Implicit fan-in
```

## 5. Protocols

Protocols define interaction rules for `debate` nodes. They are separate artifacts that enforce **hard constraints** on multi-turn interactions.

```yaml
# protocols/parley_v1.yaml
name: parley_v1
roles: [participant]  # All equal
message_types:
  - DISCUSS
  - PROPOSE
  - AGREE
  - COUNTER
  - QUESTION
turn_order: round_robin
consensus:
  type: unanimous
  requires: "explicit AGREE from all participants to a PROPOSE"
enforcement:
  validate_prefix: true
  block_on_violation: true
termination:
  condition: "consensus_reached OR max_rounds"
```

## 6. Artifacts and References

Topologies reference external files for prompts and protocols:

```yaml
artifacts:
  prompts:
    architect: "prompts/personas/architect.md"
    security: "prompts/personas/security.md"
  protocols:
    parley_v1: "protocols/parley_v1.yaml"
```

Reference syntax: `prompt_ref: artifacts.prompts.architect`

## 7. Runtime Contract

A DRTL-compliant runtime MUST:

1. **Execute the graph as declared** - nodes run in topological order respecting edges
2. **Honor enforcement modes** - `block` prevents advancement until resolved
3. **Produce a canonical trace** - every execution generates an auditable log
4. **Treat verification as state-changing** - `verify` results update `state.obligations`
5. **Support the ability to refuse** - the runtime can halt on unmet obligations

## 8. Topology Structure

Complete topology file structure:

```yaml
name: "Topology Name"
description: "What this topology does"
version: "1.0"

artifacts:
  prompts: { ... }
  protocols: { ... }

state_defaults:
  loop_count: 0
  custom_var: null

nodes:
  - id: node_1
    type: generate
    ...

edges:
  - from: node_1
    to: node_2
```

## 9. Design Principles

### 9.1 Explicit Over Implicit

- Extraction must be explicit: if verifying unstructured text, add an extraction node first
- Recovery paths must be declared: automatic retry is valid if defined in topology, hidden retry is not
- State mutations must be visible: use `transform` nodes, not implicit updates

### 9.2 Verification Requires Structure

The `verify` node operates on structured JSON input, not raw text. The `target` field is a JSON key, not an extraction instruction.

**Pattern for verifying unstructured output:**

```yaml
# 1. Generate raw text
- id: generate_summary
  type: generate
  output_key: "raw_summary"

# 2. Extract claims (explicit node)
- id: extract_claims
  type: generate
  model: openai/gpt-4o-mini
  input: generate_summary.raw_summary
  prompt: |
    Extract from this text:
    - product_names: any software products or versions mentioned
    - urls: any URLs mentioned
    Return as JSON.
  output_format: json
  output_key: "structured_claims"

# 3. Verify structured claims
- id: verify_claims
  type: verify
  input: extract_claims.structured_claims
  rules:
    - id: std.check_existence
      target: "product_names"  # JSON key
      mode: block
```

### 9.3 The Ability to Refuse

A reasoning runtime is distinguished from a workflow orchestrator by its ability to **refuse to produce output** when epistemic obligations are unmet. This is implemented through:

- `mode: block` on verification rules
- `gate` nodes that route to `review` or termination
- The `review` node as an interrupt point requiring external decision
