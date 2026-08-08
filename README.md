# CostAgent — LLM Observability & Evaluation

Design for observing and evaluating the LLM components of CostAgent. **No code yet — the
decisions are still open for debate.**

| Document | What it is |
|---|---|
| **[PROPOSAL.html](PROPOSAL.html)** | The whole proposal on one page — start here |
| **[SPEC.md](SPEC.md)** | The full spec: 15 decisions, component-by-component design, open questions |
| **[CORE-CHANGES.md](CORE-CHANGES.md)** | Working checklist for the CostAgent codebase — what to check, what to change, in order |

## Scope

Four bulk single LLM calls (validation, recommendation in English, recommendation in code,
negative impact) and one agentic loop (the cost RCA assistant). Covers where each signal is
stored, how calls correlate back to findings, offline and online evaluation, hallucination
control, and CostAgent accounting for its own cost.

Contains no CostLens rule logic, prompts or credentials.
