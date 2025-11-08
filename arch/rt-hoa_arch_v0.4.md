# HOA Chat System — v0.4 Additive Augmentation
> **Version tag:** `rt-hoa_arch_v0.4` — **strictly additive** to `rt-hoa_arch_v0.3`. Nothing removed; all behaviors remain backward‑compatible. This version introduces a deterministic, **content‑agnostic interactive disambiguation** flow and **evidence‑anchored** answering that (1) always asks users for missing question context, (2) always reports when sources are missing, and (3) always reports conflicts found in sources. Loops are **not** restricted—conversation proceeds until the user stops.

---

## 0) Non‑Breaking Principles
- **No content assumptions.** The system never presumes what kinds of documents exist or their structure.
- **Additive only.** All v0.3 routes, modules, and formats continue to work unchanged.
- **Unlimited clarify loops.** The user can provide missing information over any number of turns.
- **No hallucinations.** User‑visible claims must be supported by verbatim evidence anchors (or explicitly reported as unavailable in `gaps`).

---

## 1) New Preflight Module — Answerability Gate (content‑agnostic)
**File:** `qa_preflight.py` (called by `qa.py` before any LLM/tool call)

**Purpose:** Determine if the question is safely answerable **without** predicting required context fields in advance and **without** assuming document content.

**Signals (generic):**
- `UnknownIntent`: intent confidence below `τ_intent` (cheap regex/classifier).
- `AmbiguousSubject`: unresolved coreference/deixis (e.g., “my unit”, “that invoice”, “today”).
- `InsufficientParameters`: a candidate tool can be identified but one or more parameters cannot be filled with confidence ≥ `τ_param` from: user text, session profile, or prior turns.
- `InsufficientGrounding`: optional fast retrieval indicates low coverage or inconsistent hits.

**Outputs (mutually exclusive):**
- `status:"clarify"` + **Clarify Envelope** (below), when any signal holds.
- `status:"proceed"` with `resolved_context` when none hold.

**Clarify Envelope (new, minimal, deterministic):**
```json
{
  "status": "clarify",
  "questions": [
    {
      "field": "subject",
      "prompt": "What exactly is the subject?",
      "options": [],
      "allow_free_text": true
    }
  ],
  "notes": { "reason": ["AmbiguousSubject"] }
}
```
*Frontend behavior:* render `questions[]` as inputs; post user replies back to `/ask` using the **existing** `context` field. No endpoint change required.

**Proceed Envelope (for telemetry):**
```json
{ "status": "proceed", "resolved_context": { "from": ["session","prior_turns","text"] } }
```

**Clarify exhaustion:** `qa_preflight` maintains a per-session, per-question fingerprint counter (stored in `session_context` or in-memory). If the same clarify envelope has been returned `>= clarify_policy.max_same_prompt_rounds` times without the user supplying new values, the fourth request upgrades the envelope to `status:"proceed_after_clarify_timeout"`. `qa.py` then runs the LLM but injects the missing-field list and instructs it to output `mode:"report_insufficient_evidence"`, explicitly stating that the response is contextual only. The validator enforces that such answers cite whatever partial evidence exists and that `gaps[]` include `reason:"clarify_timeout"` so the user understands why precision is unavailable.

**Config (defaults, adjustable):**
```yaml
gate_thresholds:
  tau_intent: 0.60
  tau_param:  0.70
  retrieval_min_coverage: 0.60
clarify_policy:
  max_same_prompt_rounds: 3   # after N identical clarify loops, escalate to best-effort answer
  cooldown_minutes: 10        # reset counter after inactivity window
```

---

## 2) Evidence‑Anchored Answering (content‑agnostic)
v0.4 adds an **internal** Answer Object while preserving v0.3’s visible 3‑level answer. The model may plan freely, but any user‑visible token that matters (numbers, dates, names, sections) must be **copied** from cited quotes.

### 2.1 Internal Answer Object (AO)
```json
{
  "mode": "answer | report_insufficient_evidence",
  "answer": "string",                 // candidate narrative; tokens must appear in evidence quotes
  "facts": [
    {
      "text": "string",
      "support": [
        {"source_id":"string","locator":"p.X Lm–Ln","quote":"verbatim excerpt"}
      ],
      "coverage": 0.0
    }
  ],
  "gaps": [
    {"need":"string","why":"no_quote_found | low_coverage | access_denied"}
  ],
  "conflicts": [
    {
      "key":"string",                 // normalized subject•predicate
      "values":[
        {"value":"string|number|date","source_id":"...","locator":"...","quote":"..."},
        {"value":"string|number|date","source_id":"...","locator":"...","quote":"..."}
      ],
      "delta":"string",
      "notes":"string"
    }
  ],
  "analysis":"string",                // synthesis only; no novel numbers/dates not in quotes
  "citations_index":[{"source_id":"...","ingest_hash":"..."}],
  "extractions": {
    "entities":["..."], "numbers":["..."], "dates":["..."]
  }
}
```
> **No enums, no ontology, no predicted document types.** Validation is by alignment to evidence, not by category guesses.

### 2.2 v0.3 Visible Format — Unchanged
The **response to `/ask`** continues to include the existing 3‑Level answer:
```json
{
  "answer": {
    "level1": "Directive sentence",
    "level2": "Rationale sentences",
    "level3": "Citations: ..."
  },
  "...": "..."
}
```
**Additive field (optional):**
```json
{ "evidence": <AnswerObject> }
```
Existing clients ignore `evidence`; upgraded clients can render richer panels (Facts, Gaps, Conflicts).

**Prompt addendum (system):**
```
Plan internally as needed. In user-visible output, include only facts that have at least one evidence anchor
(source_id + locator + verbatim quote). Place anything else in `gaps`. Do not invent numbers, dates, or names not
present in cited quotes. If evidence is missing or conflicting, say so and include it in `gaps` or `conflicts`.
```

---

## 3) Validator Augmentations (additive)
**File:** `validator.py` — keep all v0.3 checks; add the following **after** schema/format checks.

### 3.1 Span‑Alignment (anti‑hallucination)
Every number/date/§‑token in `answer.level1/2` must appear verbatim in the union of `evidence.facts[*].support[*].quote`. If not, either:
- Move to `evidence.gaps` and switch mode to `report_insufficient_evidence`, **or**
- Fail validation and trigger repair (see §4).

### 3.2 Locator Authenticity
Each `support[*].locator` must exist in the ingest **locator index** for that `source_id`. Non‑existent locators fail validation.

### 3.3 Conflict Reporting (mandatory)
From support quotes **only**, normalize values by key (subject•predicate). If ≥2 distinct values (outside tolerance) exist, `evidence.conflicts[]` must enumerate them with quotes/locators. Otherwise validation fails.

**Normalization & tolerances:**
- Numbers: strip commas, parse float; default δ% = 1.0 when same units are explicit. If units unverifiable, compare as strings and still report distinct values.
- Dates: ISO normalize; any difference ⇒ conflict.
- Strings: casefold/trim; Levenshtein > ε ⇒ distinct.

### 3.4 Clarify Enforcement
If preflight emitted `status:"clarify"` but the API returned a final answer, block. The system must ask the user first.

If preflight emitted `status:"proceed_after_clarify_timeout"`, the validator requires the final AO to set `mode:"report_insufficient_evidence"`, include `gaps[]` entries tagged `reason:"clarify_timeout"`, and have Level 1/2 explicitly acknowledge that the precise answer could not be given because the user did not supply the missing detail.

### 3.5 Primary vs Secondary (non‑blocking)
If `provenance.kind` exists and only secondary sources support a fact, annotate lowered confidence. Do **not** block.

---

## 4) Critique → Repair → Validate Loop (bounded attempts)
Add a small loop **inside** `qa.py` after the first draft:
1. **Draft** AO + v0.3 3‑levels.
2. **Validate** with §3 rules.
3. **If fail**, allow up to `N_REPAIRS` (default 2) where the model may call only deterministic tools (below) to fetch missing quotes or narrow patterns.
4. **If still failing**, return `mode:"report_insufficient_evidence"`, with populated `gaps/conflicts`, and keep 3‑Level answer consistent (Level 1 may read “Action required” or “Insufficient evidence” per existing policy).

---

## 5) Minimal Deterministic Tools (no summarization)
Add tool contracts (I/O strict, verbatim text only):

- `quote_fetch(source_id, locator|pattern) -> { "quote": "string", "locator": "string" }`
- `search_source(source_id, pattern, top_n) -> [{ "locator":"string", "snippet":"string" }]`
- `calc(expr) -> { "value": "string|number" }` *(validator recomputes to verify)*

These tools **do not** infer or summarize; they return raw text and real locators only.

---

## 6) Ingest Augmentation — Locator Index (additive)
**File:** `ingestion_service.py`

During ingest of any source (PDF/doc), produce a simple **locator index** and persist it.

**New table:** `source_locator_index`
```sql
create table if not exists source_locator_index (
  source_id text primary key,
  locators  text[] not null,
  provenance_kind text default 'unknown',  -- 'primary' | 'secondary' | 'unknown'
  md5       text not null,
  ingest_ts timestamptz not null default now()
);
```
- `locators` are opaque page/line (or byte span) identifiers like `p.12 L140-L162`.
- Validators **only** check membership; no ontology or content is assumed.

---

## 7) Session Context Cache (optional, additive)
**New table (optional):** `session_context`
```sql
create table if not exists session_context (
  session_id uuid primary key,
  data jsonb not null,         -- e.g., {"subject":"unit 5A","time_scope":"2025 YTD"}
  updated_at timestamptz not null default now(),
  expires_at timestamptz not null
);
```
- Used by preflight to avoid re-asking; UI displays current resolved fields with a “Change” link.
- If not implemented as a table, an in-memory cache keyed by session is sufficient.
- `data` also stores `clarify_counters` such as `{ "sq_ft_question": { "count": 3, "last_prompt_hash": "..." } }` so the gate knows when to trigger `proceed_after_clarify_timeout` per the configured threshold.

---

## 8) API Envelope (additions only)
No endpoint changes; responses gain optional fields.

### 8.1 `/ask` Response — Clarify
```json
{
  "status": "clarify",
  "questions": [
    {"field":"subject","prompt":"What exactly is the subject?","allow_free_text":true}
  ],
  "notes": {"reason":["AmbiguousSubject"]}
}
```

### 8.2 `/ask` Response — Answer (unchanged + additive)
```json
{
  "answer": {
    "level1": "Directive sentence",
    "level2": "Rationale sentences",
    "level3": "Citations: ..."
  },
  "evidence": { "...": "AnswerObject from §2" },   // optional, additive
  "latency_ms": 1930,
  "status": "proceed"                               // optional, additive
}
```

---

## 9) Prompt Scaffolding (additive excerpts)
**System additions:**
```
Ask the user for clarification whenever the preflight signals UnknownIntent, AmbiguousSubject, InsufficientParameters,
or InsufficientGrounding. In the absence of sufficient evidence to support a claim, do not invent text; instead,
populate `gaps` and switch to `mode: report_insufficient_evidence`. If multiple sources conflict, list them in `conflicts`
with verbatim quotes and locators; do not silently pick a winner unless the legal hierarchy rule applies and is itself quoted.
```

**User‑echo block (unchanged):** keep echoing validated input payload.

---

## 10) Wiring in `qa.py` (concise pseudo)
```python
gate = qa_preflight.check(question, session_ctx, prior_turns)
if gate["status"] == "clarify":
    return gate  # pass through to UI
elif gate["status"] == "proceed_after_clarify_timeout":
    force_report = True
else:
    force_report = False

attempts = 0
while True:
    draft = openai_client.run_qa(build_request(...))  # produces v0.3 3-levels + AO
    ok, errs = validator.evidence_check(draft)
    if force_report:
        draft["evidence"]["mode"] = "report_insufficient_evidence"
        draft["evidence"].setdefault("gaps", []).append({"need": gate["missing"], "why": "clarify_timeout"})
        ok = True
        break
    if ok or attempts >= N_REPAIRS:
        break
    attempts += 1
    draft = repair_with_tools(draft, errs)  # quote_fetch/search_source/calc only

if ok:
    return draft
else:
    return force_report_insufficient(draft)  # populate gaps/conflicts and harmonize Level 1-3
```

---

## 11) Frontend Additions (compatible)
- If response has `status:"clarify"`, render inputs; post answers back in `context`.
- In result view, show three panels under the 3‑Level answer: **Facts**, **Missing Information (gaps)**, **Conflicts**; each entry links `source_id + locator` (“Open source”).

---

## 12) Test Matrix (must‑pass cases)
1. **Ambiguity:** “What’s the square footage of my unit?” → preflight asks for subject/unit; unlimited clarify loops allowed.
2. **Missing evidence:** Ask for a fact not present in any source → `mode:"report_insufficient_evidence"` + `gaps[]`; `level3` reads `Citations: None — explanation …`.
3. **Conflicts:** Two sources provide different numbers → `conflicts[]` lists both with quotes/locators; answer explains conflict per hierarchy or defers.
4. **Fake locator:** Citation uses non‑indexed locator → validator blocks; repair or report.
5. **No novel tokens:** Any number/date in `level1/2` must appear in some quote; invented token → blocked.
6. **Loop tolerance:** User provides partial info repeatedly; system continues to ask until sufficient.

---

## 13) Operational Parameters
```yaml
attempt_limits:
  n_repairs: 2            # max Critique→Repair cycles
  clarify_rounds_cap: 0   # 0 = unlimited; UI controls loop, not backend

evidence_policy:
  require_span_alignment: true
  allow_secondary_sources: true
  lower_confidence_on_secondary: true
  conflict_delta_percent_default: 1.0
```

---

## 14) Migration Notes
- Deploy `qa_preflight.py`, validator extensions, and the ingest locator index job.
- No DB migrations required for core flows; `source_locator_index` and `session_context` are optional **additions**.
- Existing clients continue to function; upgraded clients can use `status`, `evidence`, `gaps`, and `conflicts` for richer UX.

---

**End of v0.4 Additive Augmentation**
