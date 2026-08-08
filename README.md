# CostAgent — LLM Observability & Evaluation

Design and harness for observing and evaluating the LLM components of CostAgent.

**[SPEC.md](SPEC.md) — draft for debate. No code until the decisions are agreed.**

Covers the four bulk single calls (validation, recommendation in English, recommendation in
code, negative impact) and the agentic cost-RCA assistant: where each signal is stored, how
calls correlate back to findings, and what offline and online evaluation look like.

Contains no CostLens rule logic, prompts or credentials.
