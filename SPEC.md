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
traces.

**On cost — sized rather than assumed.** A trace with full payload is roughly 3–5 KB. At
10,000 findings × 4 calls = 40k traces per run, run daily, that is about **6 GB/month** —
pennies in object storage. The tracing overhead itself is a logging write and is negligible.
**Storage is not a reason to sample.**

*To verify, not assume:* whether your Databricks tier meters **MLflow trace ingestion**
(this has changed with MLflow 3 / managed MLflow, and differs between Free Edition and
paid), and whether inference tables bill as ordinary Delta storage in your catalog.

*The real reason to sample is signal-to-noise.* Forty thousand near-identical bulk traces
are unusable — nobody scrolls that. Sample so the trace UI stays browsable.

*Proposal:* **autolog full for the assistant, sampled (say 1 in 20) for the bulk calls.**
`llm_calls` carries every row regardless; traces are for debugging, not accounting.

---

### D4 · Blind-then-reveal review — and we build the flow, because none exists

**Confirmed: there is no finding review flow today.** The original plan — instrument the
review that already happens — has nothing to instrument. The flow has to be built, and it
becomes part of this work rather than a dependency of it.

That is a setback and an opportunity. Retrofitting unbiased labelling onto an existing
review is painful; designing it in from the start costs nothing.

**Proposal: blind-then-reveal, for every finding — not a 10% sample.**

```
reviewer opens a finding
   → sees: rule, resource, evidence, predicted saving          (the LLM verdict is HIDDEN)
   → records their own call: valid | not valid | unsure
   → THEN the LLM's verdict and recommendation are revealed
   → optional second field: did the LLM change your mind?
```

*Why this beats a blind sample:* every label is clean, not one in ten. And the reviewer
still gets the LLM's value — a moment later, framed as a comparison rather than a
suggestion. "Did the AI agree with you?" is engaging; "here is what the AI thinks, do you
concur?" is anchoring.

*Why it beats blind-always:* the reviewer keeps the assistance the product exists to
provide. Only the *order* changes.

*Counter:* one extra click, and some reviewers will resent feeling tested. Mitigation is
framing — this is calibration, and it is genuinely useful to them. If adoption suffers,
fall back to a blind sample and accept contaminated labels on the rest.

*Consequence for the build order:* the review flow moves from "instrument existing" to
"design and build", and it becomes **step 2** — before any eval code, because labels are
the input everything else needs.

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

### D7 · Prompt versioning by content hash, not by hand

**Confirmed: prompts are not versioned today.** This is step zero — without it no quality
change can be attributed to a prompt change, and every metric in this document becomes
uninterpretable the first time someone edits a string.

**Proposal:** prompts move out of inline strings into files, and the version is a
**content hash computed at load**, not a number someone remembers to bump.

```python
PROMPTS = load_prompts("prompts/")          # prompts/validate.md, recommend_en.md, ...
prompt_version = f"{name}@{sha256(template)[:12]}"     # validate@a3f91c2b7e04
```

*Why a hash rather than semver:* nobody forgets to bump a hash. It changes exactly when the
prompt changes, and never when it does not. A human-readable name is carried alongside for
legibility.

*Counter:* a hash is opaque in a dashboard, and it changes on a whitespace edit. Mitigation:
carry both — `validate@a3f91c2b` for correctness, plus an optional label for humans.

*This is the one item with no argument against doing it first.*

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

## 3b · Who looks at what

Confirmed: the UI is **Next.js hosted as a Databricks App**, and the persona works there.

| Audience | Surface | Fed by | Cares about |
|---|---|---|---|
| **Persona** — reviews findings, judges quality | **Next.js Databricks App** | `llm_calls` via Lakebase Postgres | Is this finding real? Did the recommendation help? |
| **Engineer** — debugging a bad output | MLflow trace UI | MLflow traces | Why did this specific call go wrong? |
| **Finance / exec** | A view in the same app | `llm_calls` ⋈ billing | Realised saving, and what the intelligence costs |

**The persona will never open MLflow.** That settles two things: traces are an engineering
tool rather than a product surface, and the blind-then-reveal flow in D4 is **a screen we
design in Next.js**, not a Databricks feature we switch on.

It also reinforces D1 — `llm_calls` has to reach Postgres for the app *and* Delta for the
billing join, which is exactly why it is written to Delta and synced.

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

## 5b · Hallucination and false positives

Two different problems with different owners, routinely conflated.

**False-positive findings come from the rules engine**, which is deterministic. That is
rule tuning, not hallucination. **Hallucination is the LLM inventing something** the
evidence does not support. The validation step sits between them — it exists to catch rule
false positives, and it can hallucinate while doing so, in both directions.

### D8 · The model never produces a number

Every figure in the output — predicted saving, current spend, node count, hours idle — is
**computed by the pipeline and templated in**. The model writes prose around numbers it
cannot change.

```
✗  "You could save roughly $400/month by downsizing this cluster"
       the model invented $400

✓  "You could save {{predicted_saving}} by downsizing {{resource}} from
    {{current_size}} to {{recommended_size}}"
       every figure supplied by the pipeline
```

*Why this is the single strongest control:* it makes a hallucinated saving **structurally
impossible** rather than unlikely. Everything below is second-order.

*Counter:* templated prose reads stiffer than generated prose, and the model cannot express
a nuance the template did not anticipate. Mitigation: let it choose *which* template and
write the surrounding explanation freely — it just cannot mint a figure.

### Controls, by how much they can be trusted

| Control | Mechanism | Deterministic? |
|---|---|---|
| **Numbers templated** | Model emits no figures | ✅ |
| **Grounded input only** | It sees the finding's evidence and nothing else — no retrieval, no memory | ✅ |
| **Constrained output** | JSON schema or tool call; it can only emit fields we defined | ✅ |
| **Entity validation** | Every resource id in the output must exist in the catalog — a set difference | ✅ |
| **Code must parse** | Syntax check · resources exist · dry-run applies | ✅ |
| **A real refusal path** | "Insufficient evidence" is a valid, non-penalised answer | ✅ |
| Prompt instruction not to invent | Words | ❌ weakest — never rely on it alone |

### Measuring hallucination

| Metric | How | Target |
|---|---|---|
| **Entity grounding rate** | % of resource ids in the output that exist | 100% |
| **Numeric fidelity** | % of figures matching a computed value | 100% by construction — anything less means the template broke |
| **Refusal appropriateness** | **Traps planted in the eval set** — findings with missing, thin or contradictory evidence | It must say "insufficient" |
| Claim support | LLM judge, per sentence, against the evidence | Last resort, for what the above cannot see |

**Plant the traps deliberately.** An eval set containing only well-evidenced findings cannot
detect confabulation. Ten deliberately under-evidenced findings will, and
**a model that never refuses is a model that always invents.**

### Reducing rule false positives

Ordinary engineering, and none of it involves a model:

- **Emit evidence and confidence, not just a verdict** — the LLM needs something to validate
  against, and so does the reviewer
- **Sample-size guards** — do not flag a cluster on two days of data
- **Exclusion lists** — a dev cluster meant to sit idle is not waste
- **Seasonality** — a monthly spike is not a leak
- **Cross-rule dedupe** — two rules firing on one cause is one finding

### D9 · The threshold moves with adoption, deliberately

| | LLM says **valid** | LLM says **invalid** |
|---|---|---|
| **Actually valid** | ✓ correct pass | ✗ **lost saving** — silent |
| **Actually invalid** | ✗ **wasted trust** — loud | ✓ correct filter |

The two errors are not symmetric, and which one hurts more **changes over time**.

*Early:* wasted trust dominates. One bad finding and people stop opening the app. Filter
aggressively; accept losing real findings.

*Once trusted:* lost saving dominates, because it is invisible. Relax the filter as
agreement rate proves out.

*Proposal:* make this an explicit, versioned setting rather than an emergent property of
prompt wording — and report both error rates separately in the UI so the trade-off is
visible to whoever owns it.

---

## 6 · Open questions

| # | Question | Blocks |
|---|---|---|
| Q1 | Findings table schema. **A unique finding id exists** — but is it *stable across runs*, or regenerated each time? | The realised-saving join. If ids churn, a finding cannot be tracked to an outcome 30 days later |
| Q2 | Does Model Serving pass custom request fields through to the inference table? | D2 — five-minute test, can run against Free Edition |
| Q3 | Are inference tables already enabled on the endpoints? | Whether payload capture exists today |
| Q4 | Is the Lakebase sync scheduled or continuous? | D1 — UI freshness |
| Q7 | Does your Databricks tier meter MLflow trace ingestion? | D3 — sampling rate |
| ~~Q5~~ | ~~Are prompts versioned?~~ **No.** Content-hash versioning is step zero — see D7 | closed |
| ~~Q6~~ | ~~Who reviews findings, and where?~~ **No flow exists.** We build it in the Next.js app — see D4 | closed |

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
