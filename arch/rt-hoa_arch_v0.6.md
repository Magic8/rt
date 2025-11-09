# HOA Chat System — Comprehensive Technical Checkpoint

> **Version tag:** `rt-hoa_arch_v0.6` — superset of `rt-hoa_arch_v0.1` through `rt-hoa_arch_v0.5` *plus* the multimodal search architecture (`search_multimodal_vs_v0.1`). All earlier behaviors remain valid; this version layers in the three-tier (Text/Image/Spatial) retrieval stack across every access tier.

## 1. Full System Stack Overview

> **Priority Tiers:** All access control, vector-store retrieval, and policy logic operate on three numeric tiers:
> - **Tier 1 (Public)** — renters/guests, maps to legacy “PUBLIC”.
> - **Tier 2 (Owner)** — verified owners, legacy “OWNER”.
> - **Tier 3 (Board)** — directors/PM, legacy “BOARD”.
> When referencing tier-specific behavior below, both the numeric tier and the historical label are shown for clarity.

| Component | Function / Role | Hosted On | Executed On | Service / Product |
|------------|-----------------|------------|--------------|-------------------|
| **Google Sites page** | Public entry point embedding chat iframe | Google Sites (free) | Google servers | Google Sites |
| **Chat iframe (frontend UI)** | HTML/JS chat box, login form, “Members log in for greater access” | Netlify Free Static Hosting | User’s browser | Netlify Free Plan |
| **Backend API (Python)** | Handles login, session cookies, AccessRoster lookup, validator, OpenAI Responses API calls | VPS (Docker container) | VPS VM runtime | DigitalOcean Droplet $5 / mo *(or Linode/Vultr)* |
| **Database (Postgres)** | Stores AccessRoster, Sessions, Audit Logs | Supabase Cloud (managed Postgres) | Supabase servers | Supabase Free Plan |
| **Auth service** | Magic-link or passwordless login; sets HttpOnly cookie | Backend + Supabase Auth (optional) | VPS backend | Supabase Auth API (optional) |
| **Cookies (Session IDs)** | Persist login session (`hoa_session`) | Browser storage | Read server-side by backend | Standard browser feature |
| **Vector stores (5)** | HOA documents: public, private, privileged | OpenAI Vector Store | Queried inside Responses API | OpenAI Managed Vector Store |
| **LLM processing** | Generates answers using docs + legal search | OpenAI Cloud | OpenAI GPU servers | OpenAI GPT‑5 Responses API |
| **Web search tools (4)** | Federal, State, County, City legal lookups | OpenAI tool ecosystem | OpenAI Cloud | OpenAI Web Search Tool |
| **Validator / Policy Engine** | Enforces legal hierarchy, access tier rules, 3‑line format | Backend API | VPS CPU | Python module |
| **Audit log** | Permanent record of Q&A traffic | Supabase DB | Backend API writes entries | Supabase Postgres |
| **Email sending (magic links)** | Sends secure login emails | Backend (SMTP) | VPS → mail relay | Mailgun Free Tier / Gmail SMTP |

**Cost Summary:** ≈ **$5/month base (VPS)** + **OpenAI API usage**

---

## 2. Backend Architecture — Modules & Responsibilities

### app.py
Entrypoint (FastAPI / Flask). Defines routes:
- `/login/start`, `/login/verify`, `/session`, `/ask`
- Delegates to `auth`, `qa`, and `validator` modules.

### auth.py
Handles user authentication and session persistence.
- `start_login(email)` → send magic link
- `verify_login(token)` → create session + cookie
- `get_tier_from_request(request)` → resolve Priority Tier (`1/2/3`, mapped to Public/Owner/Board labels)

### roster.py
Access control registry (AccessRoster + Sessions tables).
- `get_tier_for_email(email)` → lookup tier
- `is_valid_session(session_id)` → verify session

### context_resolver.py (new)
Bridges per-session state into each request.
- `load_session_context(session_id)` → fetches stored clarifications / resolved fields from `session_context`.
- `merge_clarifications(existing_ctx, new_answers)` → updates context data and clarify counters.
- `serialize_for_prompt(ctx)` → produces JSON injected into the system/user prompt blocks so the LLM sees known vs missing information.

### policy_engine.py
Builds OpenAI Responses API call per Priority Tier.
- Selects vector stores and web search tools.
- Embeds strict instruction block:
  - 3‑level answer format (Level 1/2/3)
  - Legal hierarchy enforcement
  - Evidence anchoring / conflict reporting requirements
- Tool selection logic:
  - **Tier 1 (Public)** → Tier 1 vector stores + law tools
  - **Tier 2 (Owner)** → Tier 1 set + private_static + private_dynamic
  - **Tier 3 (Board)** → Tier 2 set + privileged_dynamic

### qa_preflight.py (new)
Answerability gate that runs before OpenAI is invoked.
- `check(question, session_ctx, prior_turns)` → returns `clarify`, `proceed`, or `proceed_after_clarify_timeout` envelopes.
- Uses configurable thresholds (`GATE_TAU_INTENT`, etc.) and updates clarify counters stored in `session_context`.

### retrieval_orchestrator.py (new)
Coordinates Layer A/B/C lookups prior to invoking the LLM.
- `route_query(question, tier, session_ctx)` → decides which layers/tools (text, image, spatial) to prime.
- `gather_multimodal_context()` → fetches Markdown chunks, image captions, and spatial JSON blobs; attaches them as tool inputs or context for the Responses API request.

### vision_catalog.py (new)
Indexes Layer B assets (floor plans, exhibits).
- `register_image(file_id, metadata)` → stores caption + tags.
- `search_captions(query, tier)` → returns image references for retrieval_orchestrator.

### spatial_index.py (new)
Stores and queries Layer C geometry JSON.
- `nearest(feature_type, subject_id)` → deterministic calculator for egress/amenity lookups.
- `path_between(start_id, end_id)` → optional corridor graph search with verification steps.

### multimodal_ingest.py (new)
Pipelines OCR → Markdown, image extraction, and spatial JSON ingestion.
- `process_pdf(source_pdf)` → produces Layer A Markdown chunks, Layer B images/captions, Layer C geometry files; updates vector stores + locator index.
- Reuses `ingestion_service.py` scheduling but adds ABBYY/OCR CLI hooks per §13 & §21.

### openai_client.py
Low-level API client.
- `run_qa(oai_request)` → calls OpenAI Responses API
- Returns `draft_answer`, `tool_trace`

### validator.py
Post‑processing gatekeeper.
- Checks format (Level 1/2/3 structure)
- Verifies explicit hierarchy phrasing (“Oakland law controls…”)
- Ensures citations cover every tool used
- Confirms source order: federal → state → county → city → CC&Rs → HOA rules
- Enforces tier leak prevention, clarifications, evidence anchoring, conflict reporting
- Returns safe fallback on violation

### qa.py
Pipeline orchestrator for multimodal retrieval.
- `answer_question(question, tier)`
  1. Call `qa_preflight` (clarify / proceed / timeout) + `retrieval_orchestrator` to assemble Layer A/B/C context.
  2. Build OpenAI request (policy_engine + oai/models) with tiered tool list (vector stores, web search, spatial_query, deterministic tools).
  3. Execute (oai/client) including critique→repair loops using deterministic tools.
  4. Validate (validator) with evidence anchoring + clarify enforcement.
  5. Log (audit) and persist updated `session_context` + cache entry; return answer or clarify envelope.

### audit.py
- `log_interaction(...)` → timestamp, email, tier, question, answer, tool_trace, validator result

### emailer.py
- `send_magic_link(email, token)` → via SMTP provider

### config.py
Holds constants:
- Vector store IDs
- Domain whitelists
- Cookie TTL
- OpenAI key
- Hierarchy order
- Fallback text templates

### ingestion_service.py (new)
Bridges Google Drive folders and OpenAI vector stores.
- `sync_vector_store(store_key)` → Reads `env_cfg[SRC_*]`, uploads new/changed files, and rebuilds embeddings when needed.
- `ensure_daily_refresh()` → Cron-invoked guard that re-syncs any store whose `last_synced_at` exceeds 24 hours.
- `handle_drive_webhook(event)` → Optional endpoint to react to Google Drive push notifications, allowing refresh-on-change instead of pure polling.

### deterministic_tools.py (new)
Implements deterministic helper endpoints invoked via OpenAI tool calls.
- `quote_fetch(source_id, locator)` → returns verbatim text spans.
- `search_source(source_id, pattern, top_n)` → returns locator/snippet pairs.
- `calc(expr)` → safe arithmetic evaluator whose outputs can be verified by the validator.

---

## 3. Tools Call Definitions

### 3.1 Vector Store Set (5 total)
| ID | Purpose | Tier | Example Contents |
|----|----------|------|------------------|
| `vs_public_static` | CC&Rs, Bylaws, recorded docs | PUBLIC | Governing documents |
| `vs_public_dynamic` | Rules, policies, announcements | PUBLIC | Quiet hours, smoking, parking |
| `vs_private_static` | Budgets, election rules, insurance | OWNER | Stable, member‑only disclosures |
| `vs_private_dynamic` | Meeting minutes, project notes | OWNER | Changing, owner‑viewable |
| `vs_privileged_dynamic` | Executive session, legal, security | BOARD | Privileged board-only materials |

> **Multimodal note:** Each vector store now contains Layer A text chunks and Layer B image captions, each labeled with `layer` metadata. Layer C spatial JSON is referenced via `geom_id` metadata so retrieval orchestrator can hydrate geometry on demand. Access is enforced via Priority Tiers (1/2/3) mapped to Public/Owner/Board.

### 3.2 Web Search Groups (4 total)
| Group | Scope | Example Domains |
|--------|--------|-----------------|
| **Federal** | FHA / ADA / EPA / FCC | `hud.gov`, `ada.gov`, `epa.gov` |
| **State** | California Davis–Stirling & Civil Code | `leginfo.legislature.ca.gov`, `davis-stirling.com` |
| **County** | Alameda ordinances, health codes | `acgov.org`, `alamedacounty.ca.gov`, `publichealth.acgov.org` |
| **City** | Oakland municipal code & fire safety | `oaklandca.gov`, `library.municode.com/ca/oakland`, `fireprevention.oaklandca.gov` |

Hierarchy of authority enforced:
**Federal → State → County → City → CC&Rs → HOA Rules/Policies → Board/Privileged Docs**

---

## 4. Processing Flow

1. User → Google Sites iframe → `/ask` API.  
2. Backend reads cookie → resolves tier via AccessRoster.  
3. Policy engine builds allowed tool list + strict instructions.  
4. OpenAI Responses API runs (5 vector stores, 4 web‑search groups).  
5. Validator checks hierarchy, formatting, access, citations.  
6. Audit log stores trace and final answer.  
7. Answer returned to iframe.

---

## 5. Logical Diagram (Text)

```
[ Google Sites ]
    ↓ (iframe)
[ Netlify Chat UI ]
    ↓ HTTPS JSON
[ Python Backend (VPS) ]
 ├─ Auth (magic links, cookies)
 ├─ AccessRoster (Supabase)
 ├─ PolicyEngine → OpenAI
 ├─ Validator (hierarchy & leak guard)
 ├─ AuditLog → Supabase
    ↓
[ OpenAI Cloud ]
 ├─ VectorStores (5)
 ├─ WebSearch Tools (4)
 └─ GPT‑5 Responses → validated answer
```

---

## 6. Tier Matrix Summary

| Priority Tier | Legacy Label | Who | Visible Stores | Hidden Stores | Purpose |
|---------------|--------------|-----|----------------|----------------|----------|
| **Tier 1** | Public | Renters / guests | `public_static`, `public_dynamic` (Layers A/B/C metadata limited to common areas) | All private/privileged stores | General building & law info |
| **Tier 2** | Owner | Titled members | Tier 1 + `private_static`, `private_dynamic` | `privileged_dynamic` | Budgets, election packets, project notes |
| **Tier 3** | Board | Directors / PM | Tier 2 + `privileged_dynamic` | none | Executive/session materials, legal memos |

---

## 7. Cost Structure

| Component | Cost | Notes |
|------------|------|-------|
| VPS backend | ~$5/mo | DigitalOcean / Linode lowest tier |
| Supabase DB/Auth | Free | generous limits |
| Netlify hosting | Free | static site |
| Google Sites | Free | embed host |
| OpenAI API | Pay‑per‑request | usage-based |
| Email (Mailgun/SMTP) | Free | for magic links |

**Typical total:** ≈ **$5 + API usage per month**

---

## 8. Future / Optional Add‑Ons
- **Owner Portal UI:** lightweight dashboard for BOD to upload new policy PDFs (auto‑vectorized).  
- **Alert System:** when CC&R updates detected → retrain vector store.  
- **Periodic Validation:** re‑run test queries weekly; flag mismatched hierarchy outputs.  
- **Version Tags:** attach effective date metadata to dynamic docs.  

---

### End of Checkpoint


---
## 9. Tiered Tools Call Architecture

| Priority Tier | Access | Vector Stores (Layers A/B) | Spatial Tooling (Layer C) | Web Search Groups | Purpose |
|---------------|--------|---------------------------|-------------------------|-------------------|----------|
| **Tier 1 (Public)** | Anonymous / renter | `vs_public_static`, `vs_public_dynamic` | common-area geometry only | State + County + City | General HOA / law questions. |
| **Tier 2 (Owner)** | Verified owner | Tier 1 + `vs_private_static`, `vs_private_dynamic` | units/amenities for owned units | Federal + State + County + City | Adds member documents and owner-specific spatial lookups. |
| **Tier 3 (Board)** | Board / PM | Tier 2 + `vs_privileged_dynamic` | full geometry set (including restricted areas) | Federal + State + County + City | Full visibility, including privileged docs and spatial data. |

### Python Example – Building Tiered Tools Call
```python
PRIORITY_BY_LABEL = {"PUBLIC": 1, "OWNER": 2, "BOARD": 3}

def build_tools_for_tier(tier_label: str):
    priority = PRIORITY_BY_LABEL[tier_label]

    # Common web-search tools (all tiers)
    law_tools = [
        {"type": "web_search", "name": "federal_law_search", "restrictions": {"domains": ["hud.gov", "ada.gov", "epa.gov"]}},
        {"type": "web_search", "name": "state_law_search", "restrictions": {"domains": ["leginfo.legislature.ca.gov", "davis-stirling.com"]}},
        {"type": "web_search", "name": "county_law_search", "restrictions": {"domains": ["acgov.org", "alamedacounty.ca.gov", "publichealth.acgov.org"]}},
        {"type": "web_search", "name": "city_law_search", "restrictions": {"domains": ["oaklandca.gov", "library.municode.com/ca/oakland", "fireprevention.oaklandca.gov"]}},
    ]

    # Vector stores per Priority Tier (Layers A/B text+caption metadata)
    base_vectors = [
        {"type": "file_search", "layer": "A/B", "name": "tier1_public", "vector_store_ids": ["vs_public_static", "vs_public_dynamic"]},
    ]

    if priority >= 2:
        base_vectors += [
            {"type": "file_search", "layer": "A/B", "name": "tier2_owner", "vector_store_ids": ["vs_private_static", "vs_private_dynamic"]},
        ]
    if priority == 3:
        base_vectors += [
            {"type": "file_search", "layer": "A/B", "name": "tier3_board", "vector_store_ids": ["vs_privileged_dynamic"]},
        ]

    spatial_tool = {"type": "spatial_query", "name": "layer_c_geometry", "permissions": priority}

    return base_vectors + [spatial_tool] + law_tools
```

---
## 10. Backend Data-Flow and Execution Diagram

```
┌───────────────┐
│ Google Sites  │
└──────┬────────┘
       │ (iframe embed)
       ▼
┌───────────────────────────┐
│ Netlify Chat UI (frontend)│
│ – login form / question box│
└────────┬──────────────────┘
         │ HTTPS JSON
         ▼
┌─────────────────────────────┐
│ Python Backend (API on VPS) │
│ • auth.py – session cookie             │
│ • roster.py – AccessRoster             │
│ • context_resolver.py – session ctx    │
│ • qa_preflight.py – answerability gate │
│ ├─ Clarify → UI
│ └─ Proceed → qa.py pipeline
│        • retrieval_orchestrator.py – Layer A/B/C routing
│        • policy_engine.py – tier tools + prompts
│        • oai/models.py – model spec
│        • oai/client.py – OpenAI call
│        • deterministic_tools / spatial_index / vision_catalog
│        • validator.py – hierarchy + evidence
│        • audit.py – Supabase log
└────────┬──────────────────────────┘
         │ Responses API call (with multimodal context)
         ▼
┌──────────────────────────┐
│ OpenAI Cloud Services   │
│ • Vector Stores (text + captions) │
│ • File Store (PDF/Image/Geom)     │
│ • Web Search Groups               │
│ • Deterministic tools (quote_fetch/search_source/calc) │
│ → GPT‑5 Reasoning                 │
└────────┬─────────────────┘
         │ Clarify envelope or validated answer + media references
         ▼
┌──────────────────────────┐
│ Netlify Chat UI shows reply│
└──────────────────────────┘
```

---
## 11. Inter-Module Interactions

| From → To | Description |
|------------|-------------|
| **auth.py → roster.py** | Validates login and resolves tier. |
| **app.py → context_resolver.py** | Loads/updates session context + clarify counters. |
| **context_resolver → qa_preflight.py** | Provides sanitized question + context for answerability gate. |
| **qa_preflight → Netlify UI** | Sends Clarify Envelope when context missing. |
| **qa_preflight → qa.py** | Signals `proceed`/`proceed_after_clarify_timeout`. |
| **app.py → retrieval_orchestrator.py** | Routes question to Layers A/B/C, fetches multimodal context. |
| **retrieval_orchestrator → policy_engine.py** | Supplies multimodal attachments + tool constraints per tier. |
| **policy_engine → oai/client** | Submits Responses API request. |
| **oai/client ↔ deterministic_tools.py / spatial_index / vision_catalog** | Fetches quotes, captions, spatial answers. |
| **oai/client → validator** | Returns draft + tool trace for validation. |
| **validator → audit** | Logs final sanitized answer + evidence status. |
| **qa.py → session_context** | Persists new clarifications / counters. |
| **audit → Supabase** | Persists question, tier, sources, result. |

---
## 12. Authority Reconciliation Rules

1. Always apply **most restrictive** legal or governing rule.  
2. Authority precedence: **Federal > State > County > City > CC&Rs > HOA Policies > Board Docs.**  
3. Validator enforces citation ordering and prevents contradictions.  
4. Output must conform to the 3-level structured format (replaces the earlier fixed 3-line rule):  
   - **Level 1 – Directive:** Single sentence that resolves the user’s request (Yes/No/Action Required).  
   - **Level 2 – Rationale:** Up to two concise sentences explaining the controlling rule; longer detail allowed only when essential to answer the question.  
   - **Level 3 – Citations:** `Citations:` prefix followed by sources ordered by the legal hierarchy.  
   - No strict word-count policing, but brevity, directness, and hierarchy compliance are enforced by the validator.

---

## 13. Document Ingestion & Vector Store Maintenance

| env_cfg key | Google Drive folder | Vector store ID | Tier | Notes |
|--------------|----------------------|-----------------|------|-------|
| `SRC_PUB_STAT` | `Public_Static/` | `vs_public_static` | Tier 1 (Public) | Recorded CC&Rs, bylaws, deeds |
| `SRC_PUB_DYN` | `Public_Dynamic/` | `vs_public_dynamic` | Tier 1 (Public) | Rules, notices, announcements |
| `SRC_OWN_STAT` | `Private_Static/` | `vs_private_static` | Tier 2 (Owner) | Budgets, insurance, election packets |
| `SRC_OWN_DYN` | `Private_Dynamic/` | `vs_private_dynamic` | Tier 2 (Owner) | Minutes, project trackers |
| `SRC_BOD_DYN` | `Privileged_Dynamic/` | `vs_privileged_dynamic` | Tier 3 (Board) | Executive + legal memos |

> **Mandatory principle:** Vector stores must only be refreshed via the webhook-triggered ingestion flow described below. Scheduled cron runs exist purely as a safety net and may not be used as the primary synchronization method in production.

### 13.1 Sync Workflow (ingestion_service.py + multimodal_ingest.py)
1. `collect_drive_manifest(folder_env_key)` → uses Google Drive API to list PDFs/docs in the folder specified by `env_cfg`.
2. For each PDF, `multimodal_ingest.process_pdf()` performs:
   - **Layer A (Text):** OCR → Markdown (ABBYY `sandwich` PDF or `ocrmypdf` + `pandoc`), heading normalization, YAML metadata injection, chunking by Article/Section (≈0.5–1k tokens).
   - **Layer B (Images):** `pdfimages` extraction, deterministic captioning, metadata assembly (building, level, features) and upload to File Store + caption embeddings.
   - **Layer C (Spatial):** JSON geometry describing units/amenities/exits/vertical transport with scale info and centroid/polygon coordinates; stored alongside reference to source image.
3. `upload_batch_to_openai(files)` → pushes Markdown, captions, spatial JSON into the OpenAI file store (with metadata linking to tiers) and tags them with `store_key`.
4. `refresh_vector_store(store_key)` →
   - Creates/updates the tier-specific vector stores with embeddings for Layer A chunks and Layer B captions (Layer C referenced via metadata ids like `geom_id`).
   - Updates `vector_store_syncs.last_synced_at` and writes locator arrays to `source_locator_index` (see §14).
5. `sync_vector_store(store_key)` orchestrates the above and is invoked:
   - At deployment (bootstrap all empty stores).
   - By `ensure_daily_refresh()` and Drive webhooks (mandatory per §13.2).

```python
def sync_vector_store(store_key: str) -> None:
    drive_folder = env_cfg[f"SRC_{store_key.upper()}"]
    manifest = collect_drive_manifest(drive_folder)
    pending = diff_against_openai(manifest, store_key)
    if not pending:
        mark_sync(store_key, status="noop")
        return
    batch_file_ids = upload_batch_to_openai(pending)
    refresh_vector_store(store_key, batch_file_ids)
    mark_sync(store_key, status="success", file_count=len(batch_file_ids))
```

### 13.2 Google Drive Webhook Mechanism (Mandatory)
1. **Channel registration:** At boot the ingestion service calls `drive.files.watch` for each folder referenced by the `SRC_*` env vars. Request payload includes:
   - `id`: locally generated channel UUID stored in `vector_store_syncs.webhook_channel_id`.
   - `type`: `web_hook`.
   - `address`: public HTTPS endpoint `https://api.hoa.example.com/ingest/drive-webhook`.
   - `token`: random verifier string persisted per channel; echoed back on every notification.
   - `expiration`: set to now + 6 days (max 7) so a daily renewal job can refresh before expiry.
2. **Verification handshake:** Google immediately POSTs to the endpoint with `X-Goog-Channel-Token` and `X-Goog-Resource-State=sync`. The API must respond `200 OK` within 10 seconds to activate the channel; otherwise the watch request is retried.
3. **Change notifications:** For each add/update/remove event, Drive sends headers: `X-Goog-Channel-ID`, `X-Goog-Channel-Token`, `X-Goog-Resource-ID`, `X-Goog-Resource-State`, `X-Goog-Changed`. The handler:
   - Validates the token and ensures the channel ID maps to a known `store_key`.
   - Enqueues `{store_key, resource_id, changed_flags}` into `drive_change_queue` with status `pending`.
   - Triggers `sync_vector_store(store_key)` immediately (debounced at 2‑minute intervals to batch bursts).
4. **Renewal & teardown:** `ensure_channel_freshness()` runs daily to renew channels expiring within 24 h (issue new `watch`, update stored IDs, call `drive.channels.stop` on the old one). On service shutdown the same stop call cleans up active channels to avoid orphaned webhooks.
5. **Failure fallback:** If the webhook endpoint fails (non‑2xx or repeated timeouts), Drive stops the channel. The ingestion scheduler detects a missing/expired channel and falls back to the 12‑hour cron refresh until the channel is re-established, ensuring the “refresh at least daily” contract still holds.

**Why mandatory?** This mechanism gives near-real-time propagation of HOA documents, satisfies the “refresh at least daily” requirement even under channel failures, and drastically reduces redundant polling calls to the Drive API. Implementations that skip the webhook cannot be promoted out of dev.

### 13.3 Failure & Retry Guardrails
- Each sync writes a status row (success/failure, error message, retry count). The backend surfaces the last sync timestamp to the admin console, ensuring visibility when a store quietly stops updating.
- Retries follow exponential backoff (up to 3 attempts) before alerting via email/slack (optional future enhancement).

---

## 14. Database Schema & Entities

All tables live in Supabase Postgres. `env_cfg["DATABASE_URL"]` points to the instance.

| Table | Purpose | Key Columns |
|-------|---------|-------------|
| `access_roster` | Canonical user/tier registry | `email` (PK, citext), `tier` (`PUBLIC/OWNER/BOARD`), `status` (`active/suspended`), `display_name`, `unit`, `created_at`, `updated_at` |
| `sessions` | Browser session tracking | `session_id` (PK, uuid), `email` (FK → roster), `tier_snapshot`, `expires_at`, `user_agent_hash`, `created_at` |
| `magic_links` | Optional passwordless login tokens | `token` (PK), `email`, `expires_at`, `consumed_at`, `ip_issued`, `delivery_channel` |
| `audit_log` | Immutable Q&A ledger | `id` (PK), `timestamp`, `email`, `tier`, `question`, `answer`, `citations`, `tool_trace` (jsonb), `validator_status`, `latency_ms`, `source_vector_ids` (text[]), `violations` (jsonb) |
| `vector_store_syncs` | Tracks Drive→OpenAI sync state | `store_key` (PK), `last_synced_at`, `last_sync_status`, `file_count`, `error_message`, `trigger` (`cron/webhook/manual`) |
| `drive_change_queue` (optional) | Buffer for webhook events | `id`, `store_key`, `drive_file_id`, `event_ts`, `status` |
| `source_locator_index` (new) | Lists valid locator strings per source for validator span checks | `source_id` (PK), `locators` (text[]), `provenance_kind` (`primary/secondary/unknown`), `md5`, `ingest_ts` |
| `session_context` (new) | Stores rolling conversation state & clarify counters | `session_id` (PK), `data` (jsonb), `updated_at`, `expires_at` |

Indexes: email lookups (`access_roster`, `sessions`), `audit_log tier/timestamp`, and `vector_store_syncs store_key` are indexed. All timestamps stored in UTC.

---

## 15. Structured Output Template & Validator Logic

**Policy instruction block fragment:**

```
You must respond with exactly three labeled levels:
Level 1 – Directive: <single sentence answer>
Level 2 – Rationale: <<=2 sentences explaining controlling rule; add extra clauses only if legally necessary>
Level 3 – Citations: Citations: <ordered list of sources>
- Always cite every source you referenced (vector file IDs or approved domains).
- Apply the hierarchy Federal > State > County > City > CC&Rs > HOA Policies > Board Docs.
- If conflicting rules exist, cite the strictest controlling reference.
```

**Validator checks (additions):**
- Regex enforces `Level 1`, `Level 2`, `Level 3` labels and presence of `Citations:` prefix.
- Ensures `Level 2` remains ≤2 sentences unless the LLM flagged `[[EXTENDED]]` token, which is only allowed when the validator sees `needs_extended_explanation=true` in the tool trace.
- Confirms every citation maps to either a vector file ID returned in `tool_trace` or an approved domain from the request’s web search group.

---

## 16. API JSON Contracts

All endpoints speak JSON over HTTPS. Errors use `{ "error": { "code": "...", "message": "..." } }`.

### `POST /login/start`
Request:
```json
{ "email": "owner@example.com", "login_method": "password" | "magic_link" }
```
Response (magic link path):
```json
{ "status": "link_sent", "expires_in_minutes": 15 }
```
Response (password path):
```json
{ "status": "password_required" }
```

### `POST /login/verify`
Request (password):
```json
{ "email": "owner@example.com", "password": "..." }
```
Request (magic link):
```json
{ "token": "uuid-token" }
```
Response:
```json
{ "status": "authenticated", "tier": "OWNER", "session_ttl_days": 14 }
```
`hoa_session` HttpOnly cookie carries the session_id.

### `GET /session`
Response:
```json
{
  "authenticated": true,
  "email": "owner@example.com",
  "tier": "OWNER",
  "expires_at": "2024-10-31T18:22:00Z"
}
```

### `POST /ask`
Request:
```json
{
  "question": "Can I install a Level 2 charger?",
  "context": {
    "free_text": "optional",
    "clarifications": { "subject": "parking spot" }
  },
  "client_ts": "2024-10-24T21:10:00Z"
}
```
Response — Clarify envelope:
```json
{
  "status": "clarify",
  "questions": [
    { "field": "unit", "prompt": "Which unit is this about?", "allow_free_text": true }
  ],
  "notes": { "reason": ["AmbiguousSubject"] }
}
```
Response — Answer (success):
```json
{
  "status": "proceed",
  "answer": {
    "level1": "Directive sentence",
    "level2": "Rationale sentences",
    "level3": "Citations: ..."
  },
  "evidence": {
    "mode": "answer",
    "facts": [
      {
        "text": "Square footage is 910 sq ft",
        "support": [
          {
            "source_id": "vs_public_static:file_abc",
            "locator": "p.12 L140-L150",
            "quote": "Unit 5A measures 910 square feet."
          }
        ]
      }
    ],
    "gaps": [],
    "conflicts": []
  },
  "sources": [
    { "type": "vector", "id": "vs_public_static:file_abc", "title": "CC&R 4.2" },
    { "type": "web", "domain": "oaklandca.gov", "url": "https://oaklandca.gov/..." }
  ],
  "media": [
    { "type": "image", "id": "floorplan_p097", "caption": "Building 1 – Level 1" },
    { "type": "spatial", "id": "geom_p097", "details": { "unit": "101", "nearest_exit": "EXIT_A" } }
  ],
  "tool_trace": { ... },
  "latency_ms": 1850
}
```
`media[]` carries optional Layer B/Layer C assets for clients that can render plan thumbnails or spatial highlights; each entry references the same `image_id`/`geom_id` cited in `evidence`.
Response — Validator fallback / insufficient evidence:
```json
{
  "status": "proceed_after_clarify_timeout",
  "answer": {
    "level1": "Unable to provide the precise square footage without your unit number.",
    "level2": "Available documents discuss building-wide ranges but do not specify your unit; please confirm the unit to continue.",
    "level3": "Citations: vs_public_dynamic:file_xyz"
  },
  "evidence": {
    "mode": "report_insufficient_evidence",
    "facts": [],
    "gaps": [ { "need": "unit", "why": "clarify_timeout" } ],
    "conflicts": []
  }
}
```

### `POST /ingest/drive-webhook` (optional)
Request body mirrors Google Drive push notifications. Only fields used: `resourceId`, `resourceUri`, `state`, `changed`: true/false, `channelParams` (mapped to `store_key`). Response: `204 No Content`.

> **Note:** Clarification answers collected by the UI are sent back via `context.clarifications`. `context_resolver.py` merges these into `session_context` so that `qa_preflight.py` sees the latest data before deciding whether to proceed.

---

## 17. env_cfg Configuration Map

| Key | Description |
|-----|-------------|
| `OPENAI_API_KEY` | Secret for Responses + file/vector store operations. |
| `OPENAI_PROJECT` | Optional OpenAI workspace identifier. |
| `VECTOR_STORE_PUBLIC_STATIC_ID` etc. | Explicit IDs for the five stores (`_PUBLIC_STATIC`, `_PUBLIC_DYNAMIC`, `_PRIVATE_STATIC`, `_PRIVATE_DYNAMIC`, `_PRIVILEGED_DYNAMIC`). |
| `SRC_PUB_STAT` | Google Drive folder ID or path for Public_Static files. |
| `SRC_PUB_DYN` | Drive folder ID for Public_Dynamic. |
| `SRC_OWN_STAT` | Drive folder ID for Private_Static. |
| `SRC_OWN_DYN` | Drive folder ID for Private_Dynamic. |
| `SRC_BOD_DYN` | Drive folder ID for Privileged_Dynamic. |
| `DRIVE_SERVICE_ACCOUNT_JSON` | Credential blob used by ingestion_service. |
| `DATABASE_URL` | Supabase Postgres connection string. |
| `SUPABASE_SERVICE_ROLE_KEY` | Used for privileged inserts (audit log). |
| `COOKIE_NAME` (default `hoa_session`) | HttpOnly cookie label. |
| `COOKIE_TTL_DAYS` | Login persistence duration. |
| `MAGIC_LINK_TTL_MIN` | Validity window for login tokens. |
| `PASSWORD_PEPPER` | Optional constant used alongside bcrypt hashes. |
| `ALLOWED_ORIGINS` | CSV for CORS (Google Sites + Netlify domain). |
| `ALLOWED_WEB_DOMAINS_FED/STATE/COUNTY/CITY` | Whitelisted domains for web search groups, enabling dynamic updates without code changes. |
| `FALLBACK_TEXT_GENERIC` | Template used when validator blocks an answer. |
| `CRON_REFRESH_SPEC` | Cron expression consumed by the scheduler calling `ensure_daily_refresh`. |
| `OBSERVABILITY_ENDPOINT` | Where ingestion/validator send alerts (e.g., Slack webhook) when retries fail. |
| `GATE_TAU_INTENT` / `GATE_TAU_PARAM` / `GATE_MIN_COVERAGE` | Thresholds used by `qa_preflight` decisioning. |
| `CLARIFY_MAX_SAME_PROMPT` | Number of identical clarify envelopes allowed before forcing best-effort answer. |
| `CLARIFY_COOLDOWN_MINUTES` | Window after which clarify counters reset for a session. |
| `QA_MAX_REPAIRS` | Cap on critique→repair cycles when validator fails. |
| `EVIDENCE_CONFLICT_DELTA_PERCENT` | Default numeric delta required before reporting conflicts as distinct values. |
| `RESPONSE_CACHE_TTL_STATIC` / `RESPONSE_CACHE_TTL_DYNAMIC` | TTLs for cached answers based on doc class. |
| `LAYER_A_OCR_BIN` | Path/CLI for ABBYY or `ocrmypdf`. |
| `LAYER_B_IMAGE_DIR` / `LAYER_C_GEOM_DIR` | Working directories for extracted plan images and spatial JSON. |
| `LAYER_B_VECTOR_ID` | Vector store ID holding image caption embeddings. |
| `LAYER_C_BUCKET` | Storage bucket for geometry JSON + corridor graphs. |
| `SPATIAL_SCALE_DEFAULT` | Fallback pixels-per-foot when missing. |

All configuration keys live under an `env_cfg` module that loads `.env` values at startup and exposes `env_cfg["KEY"]`. Secrets never hard-coded; local development uses `.env.local`, production uses container secrets.

---

## 18. Responses API Practices & Client Library Structure

### 18.1 Package Layout
- `oai/models.py` → `OAIModelSpec` dataclass pulls defaults from `env_cfg` (model, verbosity, reasoning effort, temperature, seed, max_output_tokens, response_format). Provides `.override(**kwargs)` and `.to_payload()` helpers.
- `oai/client.py` → canonical OpenAI REST wrapper; adds auth headers, handles retries, drops `temperature`/`seed` when the model name matches `gpt-5*` (bug workaround) and injects determinism preambles automatically.
- `oai/prompts.py` → utilities to assemble system/user roles, Level 1‑3 policy instructions, determinism preamble text, and legal hierarchy reminders.
- `oai/inputs.py` → pydantic schemas for user payloads and tool inputs. Each schema exposes `validate_and_echo()` which returns sanitized JSON that can be injected into the prompt or sent as tool arguments.
- `oai/files.py` → resolves vector store IDs, file attachments, and image handles based on tier, ensuring wrappers decide what resources each call may touch.
- `oai/cache/` → `backend.py` (Redis/Postgres adapter) + `keys.py` (deterministic hash of model spec + prompt fingerprint + attachments + tier + user question).
- `oai/pipelines/qa.py` → orchestrates prompt assembly, cache lookup, OpenAI invocation, JSON schema validation, and audit logging.
- `oai/tools.py` → tier-aware tool definitions (vector stores, web search groups, ingestion helpers) referenced by `policy_engine`.

### 18.2 Strict JSON Output Enforcement
1. Define canonical schema:
```json
{
  "type": "object",
  "properties": {
    "level1": {"type": "string"},
    "level2": {"type": "string"},
    "level3": {"type": "string", "pattern": "^Citations:"},
    "sources": {"type": "array", "items": {"type": "object"}}
  },
  "required": ["level1", "level2", "level3"],
  "additionalProperties": false
}
```
2. `prompts.json_schema_instruction()` injects the schema description plus the Level 1‑3 policy, while `client.py` sets `response_format` to `{"type": "json_schema", ...}` so OpenAI enforces it server-side.
3. Post-call, `qa_pipeline` deserializes via the same schema. Failures trigger validator fallback before any response leaves `/ask`.

### 18.3 Input Handling & Tool Echoing
- FastAPI layer validates incoming JSON using `QuestionInput` and `ToolInput` schemas. Invalid questions never reach the model.
- `inputs.QuestionInput.echo()` returns `{ "question": ..., "context": ..., "tier": ... }`. This JSON is injected either as a hidden system block (“Validated input payload: …”) or as a tool argument so the model is forced to operate on sanitized data only.
- Structured tool calls (law lookups, escalation prompts) each define their own JSON schema enforced via Responses API `input_schema`. This prevents free-form tool replies.

### 18.4 Model Spec Defaults & Determinism
- `OAIModelSpec.from_env()` loads defaults such as `model=env_cfg["DEFAULT_OAI_MODEL"]`, `temperature=env_cfg.get("OAI_TEMPERATURE", 0.1)`, etc. Callers use `spec.override(temperature=0.0)` per request as needed.
- When `spec.model.startswith("gpt-5")`, `client.py` omits `temperature` and `seed` fields (workaround). In that branch, `prompts.determinism_preamble()` is prepended: “Treat temperature as 0.0 and reproduce identical outputs for identical inputs.”
- Non-gpt-5 models receive the explicit temperature/seed values so answers remain reproducible.

### 18.5 Response Caching
- `cache.keys.fingerprint(spec, prompt_hash, tier, question, attachments_hash)` computes a SHA-256 key.
- `cache.backend` stores `{answer_json, tool_trace, created_at}`, with TTL tied to ingestion freshness (e.g., 6 hours for dynamic vectors, 24 hours for static docs).
- `qa_pipeline` performs: `if cache.hit(): return cached_answer`. On miss it calls OpenAI, writes the cache entry, then passes data to `validator/audit`.

### 18.6 Unified Wrapper Invocation
```python
spec = OAIModelSpec.from_env().override(reasoning="medium")
question = QuestionInput(question="…", context="…")
messages = prompts.build(
    system_blocks=[prompts.tier_policy(tier), prompts.determinism_preamble(spec)],
    user_blocks=[question.to_user_block()]
)
payload = OAIPayload(
    spec=spec,
    messages=messages,
    tools=tools.for_tier(tier),
    attachments=files.for_tier(tier),
    input_echo=question.echo()
)
response = qa_pipeline.run(payload)
```
- The wrapper ensures every call path passes through the same validation, caching, and logging checks, keeping behavior uniform across the chat agent, admin queries, or background QA jobs.

---

## 19. Clarification & Evidence Safeguards (Additions from v0.4)

These additions are additive—older flows stay valid, but new modules ensure underspecified questions trigger clarifications and that every answer is anchored to verbatim evidence.

### 19.1 `qa_preflight.py` — Answerability Gate
- Runs before any LLM call; consumes sanitized user input + `session_context`.
- Detects generic signals: `UnknownIntent`, `AmbiguousSubject`, `InsufficientParameters`, `InsufficientGrounding` (cheap retrieval coverage check).
- Returns either `{"status":"clarify","questions":[...],"notes":{...}}`, `{"status":"proceed",...}` or `{"status":"proceed_after_clarify_timeout",...}` (see §19.3).
- Thresholds come from `env_cfg` (see §17).

### 19.2 Clarify Envelope & Exhaustion Flow
- Clarify envelopes contain one or more prompts (field, wording, allowed options). Frontend renders inputs and posts answers back via the existing `context` field in `/ask`.
- If the same clarify envelope fires `CLARIFY_MAX_SAME_PROMPT` times without new data, the next call upgrades status to `proceed_after_clarify_timeout`. Downstream logic must still flag the answer as contextual only and set `mode:"report_insufficient_evidence"` with `gaps[]` reason `clarify_timeout`.

### 19.3 Session Context Cache (`session_context` table)
- Stores arbitrary JSON (resolved fields, free-form clarifications, and `clarify_counters`).
- Updated on every `/ask`; expires after configurable TTL.
- Allows the gate to avoid re-asking when info is already present and to track identical prompt loops.

### 19.4 Evidence-Anchored Answering
- OpenAI draft now returns both the legacy 3-level answer and an internal **Answer Object (AO)** containing `mode`, `facts[]`, `support[]` quotes (source_id + locator + verbatim text), `gaps[]`, `conflicts[]`, `analysis`, and `citations_index`.
- Policy reminder: “Only surface facts that have at least one evidence anchor; otherwise place them in `gaps` and explain why.”

### 19.5 Validator Augmentations
- **Span alignment:** every number/date/token in Level 1/2 must appear verbatim within the union of AO quotes; otherwise fail validation or move to `gaps`.
- **Locator authenticity:** each `support.locator` must exist in `source_locator_index` for that source.
- **Conflict reporting:** if AO quotes reveal multiple distinct values for the same subject/predicate, the final response must include a `conflicts[]` entry citing all sources. Hierarchy rules may resolve the conflict, but the underlying discrepancy is still reported.
- **Clarify enforcement:** answers are blocked when gate said `clarify`, and forced to mention `clarify_timeout` when coming from `proceed_after_clarify_timeout`.

### 19.6 Critique→Repair Loop
- After the first draft, validator runs. If it fails and `attempts < QA_MAX_REPAIRS`, the system calls deterministic tools (see below) to fetch missing quotes, then retries once more.
- After the limit, the pipeline returns `mode:"report_insufficient_evidence"` with populated `gaps/conflicts` while keeping Level 1/2 truthful.

### 19.7 Deterministic Tools (New Contracts)
| Tool | Request Schema | Response Schema | Notes |
|------|----------------|-----------------|-------|
| `quote_fetch` | `{ "source_id": str, "locator": str }` | `{ "quote": str, "locator": str }` | Returns verbatim span only. |
| `search_source` | `{ "source_id": str, "pattern": str, "top_n": int }` | `[ {"locator": str, "snippet": str} ]` | Helps find candidate locators. |
| `calc` | `{ "expr": str }` | `{ "value": str }` | Deterministic calculator; validator re-runs expression locally. |
| `image_lookup` | `{ "image_id": str }` | `{ "caption": str, "file_url": str }` | Resolves Layer B asset for frontend display. |
| `spatial_query` | `{ "geom_id": str, "op": "nearest|path|area", "subject": {...} }` | `{ "result": {...}, "evidence": {"geom_id": str, "locator": str} }` | Deterministic Layer C operations (nearest exit, corridor path, etc.). |

### 19.8 Locator Index Integration
- `ingestion_service.py` now emits locator arrays per source and writes them to `source_locator_index` (see §14). Validators rely on this list instead of guessing structure, preserving the “no content assumptions” principle.

### 19.9 API Envelope Additions
- `/ask` may return `{"status":"clarify", ...}` when the gate halts execution or the legacy 3-level answer plus `"evidence": { ... }` AO payload when successful.
- Responses may include `"status":"proceed"` or `"status":"proceed_after_clarify_timeout"` to aid telemetry.

### 19.10 Prompt & Policy Updates
- System prompt now enforces: “Ask for clarification when preflight says so; if evidence is missing, say so; cite conflicts explicitly; never invent tokens outside your evidence quotes.”
- User echo block remains unchanged but now includes `SessionContext` JSON so the model knows what is known vs unknown.

### 19.11 `qa.py` Loop (Enhanced Pseudocode)
```python
gate = qa_preflight.check(question, session_ctx, prior_turns)
if gate["status"] == "clarify":
    return gate
force_report = gate["status"] == "proceed_after_clarify_timeout"

attempts = 0
while True:
    draft = openai_client.run_qa(build_request(..., force_report=force_report))
    ok, errs = validator.evidence_check(draft, force_report)
    if ok:
        break
    if attempts >= QA_MAX_REPAIRS:
        draft = force_report_insufficient(draft, reason="validator_failed")
        break
    attempts += 1
    repair_with_tools(draft, errs)
return draft
```

### 19.12 Frontend Expectations
- Recognize `status:"clarify"` and render dynamic forms; include answers in the next `/ask` under `context.clarifications`.
- Display AO sections (Facts, Gaps, Conflicts) under the Level 1/2/3 answer when `evidence` is present; each element links to `source_id + locator`.

### 19.13 Test Matrix (Required Cases)
1. Ambiguous request → clarifies indefinitely until user provides detail or timeout triggers best-effort answer.
2. Missing evidence → `mode:"report_insufficient_evidence"` with `gaps[]` and Level 3 noting absence of citations.
3. Conflicting docs → `conflicts[]` enumerates the discrepancies plus hierarchy reasoning.
4. Fake locator injected → validator rejects until corrected.
5. Clarify timeout → Level 1 states inability to answer due to missing detail; `gaps` reason `clarify_timeout` present.

### 19.14 Operational Parameters
```yaml
attempt_limits:
  qa_max_repairs: 2
  clarify_rounds_cap: 0   # 0 = unlimited; UI governs loop
clarify_policy:
  max_same_prompt_rounds: 3
  cooldown_minutes: 10
evidence_policy:
  require_span_alignment: true
  conflict_delta_percent: 1.0
  allow_secondary_sources: true
```

### 19.15 Prototype Checklist
| File / Module | Prototype Functions / Responsibilities |
|---------------|----------------------------------------|
| `qa_preflight.py` | `check(question, session_ctx, prior_turns)` returning gate envelopes; helper `fingerprint_question()` for counters. |
| `validator.py` | Existing format checks + new `evidence_check()` enforcing span alignment, locator authenticity, conflict reporting, clarify timeout handling. |
| `qa.py` | Orchestrates preflight, OpenAI call, repair loop, cache writes. |
| `ingestion_service.py` | Emits vector stores + `build_locator_index(source_id)` to populate `source_locator_index`. |
| `deterministic_tools.py` | Implements `quote_fetch`, `search_source`, `calc` wrappers. |
| `context_resolver.py` (conceptual) | Reads/writes `session_context`, merges clarifications, exposes `SessionContext` JSON to prompts. |
| Database Schemas (§14) | Tables for roster, sessions, magic links, audit log, vector store syncs, drive queue, source locator index, session context. |
| API Contracts (§16 + §19.9) | `/ask` clarify envelope, legacy Level 1/2/3 answer, optional AO payload. |

All prototypes are documented; implementation can proceed module-by-module following the Phase plan discussed earlier.

---

## 20. End-to-End Data/Control Flow Diagram

```
User Browser (Google Sites iframe)
  │ 1. Question JSON / clarification answers
  ▼
Netlify Chat UI
  │ 2. POST /ask
  ▼
app.py (FastAPI)
  │ 3. auth.py → roster.py (resolve tier/session)
  │ 4. context_resolver.py (load session_context, merge clarifications)
  │ 5. qa_preflight.py (answerability gate)
  ├─ if Clarify → return Clarify Envelope to UI
  └─ else → qa.py pipeline
        │ a. retrieval_orchestrator.py routes query to Layer A/B/C
        │ b. policy_engine.py builds tiered tool + prompt payload (with multimodal attachments)
        │ c. oai/models.py supplies OAIModelSpec from env_cfg
        │ d. oai/client.py calls OpenAI Responses API with:
        │       - attachments from files.py (vector stores, web search groups, media refs)
        │       - strict response_format JSON schema
        │ e. Deterministic tools (deterministic_tools.py, spatial_index.py, vision_catalog.py)
        │ f. validator.py enforces format + evidence rules
        │ g. qa.py logs to audit_log, updates session_context, writes cache
  ▼
Supabase Postgres
  • access_roster, sessions, session_context, magic_links
  • audit_log, vector_store_syncs, drive_change_queue, source_locator_index
  ▼
OpenAI Vector Stores / Web Search Tools
  • vs_public_static … vs_privileged_dynamic
  • federal/state/county/city domain groups
  ▼
Return payload:
  - status (clarify | proceed | proceed_after_clarify_timeout)
  - Level 1/2/3 answer (if any)
  - evidence Answer Object (facts/gaps/conflicts)
  - tool_trace, latency, cache indicator
```

This diagram highlights files (`app.py`, `qa_preflight.py`, `qa.py`, `validator.py`, `ingestion_service.py`), data stores (Supabase tables, vector stores), and control/data movement between them.

---

## 21. Multimodal Search Stack (Layers A/B/C)

### 21.1 Principles
- Archive PDFs remain authoritative (“sandwich” searchable versions with visible image + hidden OCR text).
- Search corpus is Markdown + metadata (Layer A), captioned image metadata (Layer B), and verified spatial JSON (Layer C).
- Safety: life-safety answers (egress, routing) must reference Layer C annotations; no on-the-fly vision reasoning.
- Determinism: all answers cite file_id, page, locator, image_id, or geom_id.

### 21.2 Layers Overview
| Layer | Function | Input | Stored As | Indexed For |
|-------|----------|-------|-----------|-------------|
| A – Text | Legal text, minutes, notices | OCR’d PDF → Markdown | `.md` chunks with YAML | Semantic search in tiered vector stores |
| B – Image | Floor plans, diagrams | Extracted PNG/TIFF + captions | File Store image + caption metadata | Caption embeddings + metadata filters |
| C – Spatial | Coordinates, exits, units, amenities | Verified JSON | `.json` per sheet | Geometry queries (nearest/path/area) |

### 21.3 Layer A Details
- OCR via ABBYY or `ocrmypdf` pipeline; convert to Markdown; clean headings using regex (Article/Section rules).
- Add YAML metadata (doc_id, article, section, pages, doc_type, `file_id_master_pdf`).
- Chunk by headings (≈500–1000 tokens) for vector store ingestion; embed with `text-embedding-3-large`.

### 21.4 Layer B Details
- Extract plan images with `pdfimages -png`.
- Create deterministic captions listing building, level, units, vertical transport, amenities.
- Store metadata: `doc_id`, `page`, `image_path`, `building`, `level`, `features`, `file_id`.
- Upload raw image to File Store; embed caption text and store within the same tier-specific vector store as Layer A (with metadata flag `layer:"B"`).

### 21.5 Layer C Details
- JSON schema records scale, units, amenity polygons, exits, stairs, elevators, corridor graph, etc.
- Example snippet:
```json
{
  "doc_id": "428Alice_CC&Rs_ExhibitD",
  "page": 97,
  "image_file_id": "file_xyz",
  "scale": {"pixels_per_foot": 3.2, "source": "title block 1/8\"=1'-0\""},
  "units": [{"unit": "101", "centroid": [12.4,38.1]}],
  "egress": [{"id": "EXIT_A", "centroid": [5.0,10.0]}]
}
```
- Stored in Supabase/File Store; referenced by metadata `geom_id`. `spatial_index.py` loads JSON and executes deterministic functions (nearest exit, corridor path, area estimates).

### 21.6 Query Routing & Retrieval
- `retrieval_orchestrator.route_query()` inspects the sanitized question + policy metadata to decide which layers to tap.
- Textual/legal questions → Layer A only.
- “Show plan / where is X?” → Layer B; orchestrator fetches caption hits and returns image IDs for the UI to render.
- Spatial questions (“nearest exit”, “distance to amenity”) → Layer C; orchestrator invokes `spatial_index` functions and returns numeric/context data with citations referencing the JSON source + image locator.
- All retrieved assets (Markdown chunks, captions, geometry outputs) are attached to the Responses API call as structured tool inputs to keep the LLM deterministic.

### 21.7 Response Assembly
- AO facts referencing spatial/image data must cite both the text snippet (if available) and the asset metadata (e.g., `image:floorplan_p097`, `geom:geom_p097`).
- Frontend displays optional `media` array:
```json
"media": [
  {"type":"image","id":"floorplan_p097","caption":"Building 1 Level 1"},
  {"type":"map","id":"geom_p097","highlight":{"unit":"101","path":[...]}}
]
```
- Level 1/2 must still explain whether the answer is exact or contextual; Level 3 cites both textual and spatial references.

### 21.8 Tooling & API Hooks
- Deterministic tool extensions:
  - `image_lookup(image_id)` → returns signed URL + caption.
  - `spatial_query(geom_id, op, params)` → e.g., `{op:"nearest", "subject":"unit:101", "target":"exit"}`.
- `env_cfg` gains paths/IDs for Layer B/C stores (`LAYER_B_VECTOR_ID`, `LAYER_C_BUCKET`, etc.).
- `/ask` response schema (Section 16) already allows `evidence` + `status`; v0.6 optionally adds `media` and `spatial` keys for clients that can render images/maps.

### 21.9 QA & Risk Checklist
- OCR QA for every document (spot-check every Article, especially election rules and “No Cumulative Voting”).
- Image fidelity: no downsampling; keep PNG/TIFF for plan linework.
- Captions reviewed for accuracy; geometry verified with scale references.
- Life-safety disclaimers included whenever spatial answers mention exits/occupancy.
- Traceability: each asset linked back to page/exhibit + file_id.

### 21.10 Automation Opportunities
- Optional computer-vision helpers (symbol detection, polygon tracing) produce drafts; humans verify before ingest.
- Corridor graphs stored alongside geometry to support path queries later.
### 13.4 Layer-Specific Storage Layout
```
/corpus/<store_key>/
  <doc_name>_Searchable.pdf     # “sandwich” PDF archived
  <doc_name>.md                 # Layer A Markdown chunked
  /images/                      # Layer B PNG/TIFF
    p097_001.png
  /spatial/
    geom_p097.json              # Layer C geometry
```
- Each chunk/image/geom file records `doc_id`, `pages`, `file_id_master_pdf`, `vector_store_id`, and permitted tiers.
- Vector stores reference these assets via metadata (`file_id`, `image_id`, `geom_id`). Retrieval orchestrator uses the metadata to fetch the appropriate layer without re-uploading binaries.
