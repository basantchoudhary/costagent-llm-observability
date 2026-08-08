# CostAgent — LLM observability & evaluation

**Status: draft for debate. No code until the decisions below are agreed.**

---

## 1 · What exists today

```
system tables (billing usage · clusters · node timeline)
   → PySpark → Lakebase bronze
   → PySpark → Lakebase core
   → rules engine  (deterministic)
   → findings
   → LLM bulk      ── validation
                    ── recommendation (English)
                    ── recommendation (code)
                    ── negative impact
   → Lakebase serving
   → Next.js Databricks App (UI)

plus: AI assistant for cost RCA — agentic loop with tool calls
```

Confirmed: **Lakebase is Postgres.** LLM calls go through **Databricks Model Serving via the
Python OpenAI client, chat completions**, behind **Databricks AI Gateway**.

### LLM components, by mode

| # | Component | Mode | Volume |
|---|---|---|---|
| 1 | Validation — does this finding hold up? | Single call | Bulk, one per finding |
| 2 | Recommendation (English) | Single call | Bulk |
| 3 | Recommendation (code) | Single call | Bulk |
| 4 | Negative impact of the recommendation | Single call | Bulk |
| 5 | Cost RCA assistant | **Agentic loop** | Interactive, low volume |

Four bulk single calls and one interactive agent. They need different treatment, and most
designs get this wrong by applying one approach to both.

---

## 2 · Decisions to agree

Each has the counter-argument stated, because the counter-arguments are real.

### D1 · `llm_calls` is written to Delta in UC, then served to Postgres

**Not** written directly to Lakebase Postgres.

*Why:* the metric that justifies the system — realised saving — needs a join to
`system.billing.usage`, which lives in Unity Catalog. A Postgres-resident fact table cannot
join to it without an export. Writing to Delta and syncing to Postgres for the app also
matches the pipeline's existing shape: everything already lands in Delta and is served
through Lakebase.

*Counter:* two hops instead of one, and the UI sees the data only after the sync. If the
app needs sub-second freshness on LLM telemetry, this is wrong.

*Open:* does the existing Lakebase serving sync run on a schedule, or continuously?

---

### D2 · One correlation id, `run_id`, propagated everywhere

Generated once per pipeline run. Stamped on: every `llm_calls` row, every MLflow trace tag,
and — if serving supports it — the request payload so it lands in the inference table.

*Why:* without it, an inference-table row cannot be linked to a finding, and the whole
observability story collapses to "we have logs".

*Counter:* putting an id in the request payload pollutes the prompt and may affect the
model's output. Better in a metadata field if one exists.

*Open:* does Databricks Model Serving pass `extra_body` / custom fields through to the
inference table, or are they stripped? **This needs a five-minute test.**

---

### D3 · MLflow autolog for tracing — one line, because they use the OpenAI client

This changes materially now that the client is known. `mlflow.openai.autolog()` captures
every chat completion as a trace automatically — inputs, outputs, tokens, latency — with no
call-site changes.

*Why:* the cheapest instrumentation available. It was going to be a per-component decision;
it is now a single line at the top of the job.

*Counter:* autolog on a bulk job that makes thousands of calls will create thousands of
traces. That is noise, and possibly a cost. May need sampling for the bulk path and full
tracing only for the assistant.

*Proposal:* **autolog on for the assistant, sampled (say 1 in 20) for the bulk calls.**
`llm_calls` carries every row regardless; traces are for debugging, not accounting.

---

### D4 · Labels come from the review that already happens

Every finding is reviewed by a person. That verdict is the label. No separate labelling
project.

*Counter — and this one is serious:* **anchoring.** A reviewer who sees "the LLM says this
finding is valid" is biased toward agreeing. The resulting dataset measures compliance, not
accuracy.

*Proposal:* for a **10% blind sample**, hide the LLM's verdict until after the human
records theirs. Measure agreement on the blind slice separately. If blind and non-blind
agreement diverge, the non-blind labels are contaminated and we know by how much.

---

### D5 · Realised saving is the headline metric — with attribution stated honestly

Recommendation → workload → `system.billing.usage`, comparing a 30-day window before and
after the change.

*Counter:* attribution is genuinely hard. The workload may have changed for unrelated
reasons; seasonality; another team's action. A naive before/after will overclaim.

*Proposal:* report it as a **range, not a number**, and only for workloads where no other
change was recorded in the window. Label the rest "unattributable" and report that count
too. An honest "£12k–£18k attributable, £30k unattributable" beats a confident £48k that a
finance partner can dismantle.

---

### D6 · Offline eval gates the build — but only on validation

*Counter:* on a small labelled set the metric is noisy, and a flaky gate gets disabled
within a month.

*Proposal:* gate only where the signal is strong enough. **Validation** is binary
classification with real labels — gate it. **Code recommendations** get deterministic
assertions (does it parse, do the resources exist) — gate those too, they cannot be noisy.
English recommendations and impact analysis get **reported, not gated**, until the dataset
is large enough to be stable.

---

## 3 · The `llm_calls` fact table

One row per LLM call. Written by the same PySpark step that makes the call.

```sql
CREATE TABLE costagent.observability.llm_calls (
  -- identity
  call_id           STRING,        -- uuid
  run_id            STRING,        -- pipeline run — the correlation key
  request_id        STRING,        -- from the serving response, joins to the inference table

  -- domain  ── TO CONFIRM against the findings table
  finding_id        STRING,
  rule_id           STRING,
  workspace_id      STRING,
  resource_key      STRING,        -- cluster / warehouse / job the finding is about

  -- what was asked
  call_type         STRING,        -- validate | recommend_en | recommend_code
                                   -- | impact | assistant
  prompt_version    STRING,        -- MUST be present. No attribution without it
  model             STRING,
  endpoint          STRING,

  -- what happened
  status            STRING,        -- ok | error | filtered | timeout
  input_tokens      BIGINT,
  output_tokens     BIGINT,
  usd               DECIMAL(12,6),
  latency_ms        BIGINT,
  retry_count       INT,
  called_at         TIMESTAMP,

  -- outcome  ── filled in later, by humans and by billing
  human_verdict     STRING,        -- agree | disagree | unsure | null
  blind             BOOLEAN,       -- was the LLM answer hidden from the reviewer?
  accepted          BOOLEAN,       -- did anyone act on it?
  applied_at        TIMESTAMP,
  predicted_saving_usd  DECIMAL(12,2),
  realised_saving_usd   DECIMAL(12,2),   -- from billing, 30 days later
  attribution       STRING         -- attributable | confounded | unattributable
)
```

Three columns carry the weight: **`prompt_version`** (no attribution without it),
**`blind`** (no trustworthy labels without it), **`attribution`** (no honest saving number
without it).

---

## 4 · Where each signal lives

| Signal | Store | Why there |
|---|---|---|
| Raw request/response payload | **AI Gateway inference table** | Free, zero code, governed |
| Per-call facts + outcomes | **`llm_calls` (Delta → Postgres)** | The domain layer. Nothing else can supply it |
| Traces (assistant; sampled bulk) | **MLflow** | Span structure for multi-step runs |
| Endpoint spend | **`system.serving.*`** | Already there |
| Quality metrics | Computed views over `llm_calls` | Derived, not stored twice |
| UI | **Next.js Databricks App** | Do not send people to a second tool |

**No OpenTelemetry, for now.** Everything runs in Databricks and every consumer is
Databricks-native; an OTel collector plus an external backend would be infrastructure with
no customer. MLflow tracing is OTel-compatible, so this is reversible.

---

## 5 · Evaluation

### Offline — before deploy, in CI

| Component | Ground truth | Metrics | Gated? |
|---|---|---|---|
| **Validation** | Human verdict (blind slice) | Precision, recall, confusion matrix. **Weight false negatives higher** — rejecting a real finding costs money | **Yes** |
| **Recommendation (code)** | Deterministic | Parses · resources exist · dry-run applies · changes what it claims | **Yes** |
| Recommendation (English) | Assertions + judge | Names the resource · quantifies saving · within length · no claim outside the rule | Report only |
| Negative impact | Past incidents | Did it name the risk that materialised? | Report only |
| RCA assistant | Labelled incidents | Top-1 cause accuracy · tool-selection precision · turns and cost per resolution · groundedness | Report only |

### Online — after deploy, on real traffic

| Component | Signal | Metric |
|---|---|---|
| Validation | Reviewer decision | Agreement rate — **blind and non-blind, reported separately** |
| Recommendation (English) | Did anyone act | Acceptance rate |
| Recommendation (code) | Applied and ran | Applied rate, success rate |
| All | Billing, +30 days | **Realised ÷ predicted**, with attribution class |
| Assistant | Feedback, abandonment | *This helped* rate, sessions ending without an answer |
| Everything | `llm_calls` | Cost per finding, p95 latency, failure rate |

### The number that justifies the system

**LLM spend ÷ savings identified.** A two-table join, and no vendor tool can compute it.

### The flywheel

```
finding → LLM → human reviews it anyway → verdict lands in llm_calls
                                                ↓
                                    a labelled row, free
                                                ↓
                              offline eval set grows weekly
```

The eval dataset is a by-product of work already happening. That single design choice is
the difference between having a golden set and permanently intending to build one.

---

## 6 · Open questions

| # | Question | Blocks |
|---|---|---|
| Q1 | **Findings table schema** — column names and types | `llm_calls` foreign keys |
| Q2 | Does Model Serving pass custom request fields through to the inference table? | D2 — five-minute test |
| Q3 | Are inference tables already enabled on the endpoints? | Whether payload capture exists today |
| Q4 | Is the Lakebase sync scheduled or continuous? | D1 — UI freshness |
| Q5 | Are prompts versioned anywhere today? | `prompt_version` — if not, this is step zero |
| Q6 | Who reviews findings, and in which UI? | D4 — where the blind slice is implemented |

---

## 7 · Build order — *not yet approved*

| # | Deliver | Why first |
|---|---|---|
| 0 | `prompt_version` on every call | Nothing is attributable without it |
| 1 | `llm_calls` table + write path | Everything else reads from it |
| 2 | Capture `human_verdict`, with the 10% blind slice | Labels start accruing from day one |
| 3 | Deterministic graders for code recommendations | Free, and catches the worst failures |
| 4 | Validation offline eval + CI gate | First real quality signal |
| 5 | Realised-saving join | The headline metric |
| 6 | MLflow autolog — assistant full, bulk sampled | Debugging |
| 7 | UI: three views in the Next.js app | Once the questions are known |

Value arrives at step 2: labels start accumulating before any eval code exists.
