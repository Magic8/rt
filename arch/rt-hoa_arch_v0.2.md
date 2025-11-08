# HOA Chat System — Comprehensive Technical Checkpoint

> **Version tag:** `rt-hoa_arch_v0.2` — strict augmentation of `rt-hoa_arch_v0.1.md`.

## 1. Full System Stack Overview

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
- `get_tier_from_request(request)` → resolve user tier (PUBLIC / OWNER / BOARD)

### roster.py
Access control registry (AccessRoster + Sessions tables).
- `get_tier_for_email(email)` → lookup tier
- `is_valid_session(session_id)` → verify session

### policy_engine.py
Builds OpenAI Responses API call per tier.
- Selects vector stores and web search tools.
- Embeds strict instruction block:
  - 3‑line answer format
  - Legal hierarchy enforcement
  - “Most restrictive lawful rule controls.”
- Tool selection logic:
  - PUBLIC → public stores + law tools
  - OWNER  → + private_static + private_dynamic
  - BOARD  → + privileged_dynamic

### openai_client.py
Low-level API client.
- `run_qa(oai_request)` → calls OpenAI Responses API
- Returns `draft_answer`, `tool_trace`

### validator.py
Post‑processing gatekeeper.
- Checks format (3‑line structure)
- Verifies explicit hierarchy phrasing (“Oakland law controls…”)
- Ensures citations cover every tool used
- Confirms source order: federal → state → county → city → CC&Rs → HOA rules
- Enforces tier leak prevention
- Returns safe fallback on violation

### qa.py
Pipeline orchestrator.
- `answer_question(question, tier)`
  1. Build OpenAI request (policy_engine)
  2. Execute (openai_client)
  3. Validate (validator)
  4. Log (audit)
  5. Return answer

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

---

## 3. Tools Call Definitions

### 3.1 Vector Store Set (5 total)
| ID | Purpose | Tier | Example Contents |
|----|----------|------|------------------|
| `vs_public_static` | CC&Rs, Bylaws, recorded docs | PUBLIC | Governing documents |
| `vs_public_dynamic` | Rules, policies, announcements | PUBLIC | Quiet hours, smoking, parking |
| `vs_private_static` | Budgets, election rules, insurance | OWNER | Stable, member‑only disclosures |
| `vs_private_dynamic` | Meeting minutes, project notes | OWNER | Changing, owner‑viewable |
| `vs_privileged_dynamic` | Executive session, legal, security | BOARD | Privileged board‑only materials |

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

| Tier | Who | Visible Stores | Hidden Stores | Purpose |
|------|------|----------------|----------------|----------|
| **Public (A)** | Renters / guests | public_static, public_dynamic | all private/privileged | general building & law info |
| **Owner (B)** | Titled members | + private_static, private_dynamic | privileged_dynamic | budget, election, project info |
| **Board (C)** | Directors / PM | + privileged_dynamic | none | internal executive materials |

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

| Tier | Access | Vector Stores | Web Search Groups | Purpose |
|------|---------|----------------|-------------------|----------|
| **Public (A)** | Anonymous / renter | `vs_public_static`, `vs_public_dynamic` | State + County + City | General HOA / law questions. |
| **Owner (B)** | Verified owner | + `vs_private_static`, `vs_private_dynamic` | Federal + State + County + City | Adds member documents and state/federal law. |
| **Board (C)** | Board / PM | + `vs_privileged_dynamic` | Federal + State + County + City | Full visibility, including privileged docs. |

### Python Example – Building Tiered Tools Call
```python
def build_tools_for_tier(tier: str):
    # Common web-search tools
    law_tools = [
        {"type": "web_search", "name": "federal_law_search", "restrictions": {"domains": ["hud.gov", "ada.gov", "epa.gov"]}},
        {"type": "web_search", "name": "state_law_search", "restrictions": {"domains": ["leginfo.legislature.ca.gov", "davis-stirling.com"]}},
        {"type": "web_search", "name": "county_law_search", "restrictions": {"domains": ["acgov.org", "alamedacounty.ca.gov", "publichealth.acgov.org"]}},
        {"type": "web_search", "name": "city_law_search", "restrictions": {"domains": ["oaklandca.gov", "library.municode.com/ca/oakland", "fireprevention.oaklandca.gov"]}},
    ]

    # Vector stores per tier
    base_vectors = [
        {"type": "file_search", "name": "public_static", "vector_store_ids": ["vs_public_static"]},
        {"type": "file_search", "name": "public_dynamic", "vector_store_ids": ["vs_public_dynamic"]},
    ]

    if tier == "OWNER":
        base_vectors += [
            {"type": "file_search", "name": "private_static", "vector_store_ids": ["vs_private_static"]},
            {"type": "file_search", "name": "private_dynamic", "vector_store_ids": ["vs_private_dynamic"]},
        ]
    elif tier == "BOARD":
        base_vectors += [
            {"type": "file_search", "name": "private_static", "vector_store_ids": ["vs_private_static"]},
            {"type": "file_search", "name": "private_dynamic", "vector_store_ids": ["vs_private_dynamic"]},
            {"type": "file_search", "name": "privileged_dynamic", "vector_store_ids": ["vs_privileged_dynamic"]},
        ]

    return base_vectors + law_tools
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
│ • auth.py – session cookie   │
│ • roster.py – AccessRoster   │
│ • policy_engine.py – builds request│
│ • openai_client.py – calls OpenAI │
│ • validator.py – checks hierarchy │
│ • audit.py – logs to Supabase     │
└────────┬──────────────────────────┘
         │ Responses API call
         ▼
┌──────────────────────────┐
│ OpenAI Cloud Services   │
│ • 5 Vector Stores        │
│ • 4 Web Search Groups    │
│ → GPT‑5 Reasoning        │
└────────┬─────────────────┘
         │ Validated Answer
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
| **app.py → policy_engine.py** | Builds tier-specific tool set and prompt. |
| **policy_engine → openai_client** | Submits Responses API request. |
| **openai_client → validator** | Returns draft + trace for validation. |
| **validator → audit** | Logs final sanitized answer. |
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
| `SRC_PUB_STAT` | `Public_Static/` | `vs_public_static` | PUBLIC | Recorded CC&Rs, bylaws, deeds |
| `SRC_PUB_DYN` | `Public_Dynamic/` | `vs_public_dynamic` | PUBLIC | Rules, notices, announcements |
| `SRC_OWN_STAT` | `Private_Static/` | `vs_private_static` | OWNER | Budgets, insurance, election packets |
| `SRC_OWN_DYN` | `Private_Dynamic/` | `vs_private_dynamic` | OWNER | Minutes, project trackers |
| `SRC_BOD_DYN` | `Privileged_Dynamic/` | `vs_privileged_dynamic` | BOARD | Executive + legal memos |

> **Mandatory principle:** Vector stores must only be refreshed via the webhook-triggered ingestion flow described below. Scheduled cron runs exist purely as a safety net and may not be used as the primary synchronization method in production.

### 13.1 Sync Workflow (ingestion_service.py)
1. `collect_drive_manifest(folder_env_key)` → uses Google Drive API to list PDFs/docs in the folder specified by `env_cfg`.
2. `upload_batch_to_openai(files)` → pushes new/updated files into the OpenAI file store and tags them with `store_key` metadata.
3. `refresh_vector_store(store_key)` →
   - Creates the store if empty, otherwise calls OpenAI “file_batch add + recompute” to refresh embeddings.
   - Updates `vector_store_syncs.last_synced_at` (see §14).
4. `sync_vector_store(store_key)` orchestrates the above and is invoked:
   - At deployment (boot strap all empty stores).
   - By `ensure_daily_refresh()` (run via cron every 12 hours).

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
  "context": "optional free-text",
  "client_ts": "2024-10-24T21:10:00Z"
}
```
Response (success):
```json
{
  "answer": {
    "level1": "Directive sentence",
    "level2": "Rationale sentences",
    "level3": "Citations: ..."
  },
  "sources": [
    { "type": "vector", "id": "vs_public_static:file_abc", "title": "CC&R 4.2" },
    { "type": "web", "domain": "oaklandca.gov", "url": "https://oaklandca.gov/..." }
  ],
  "tool_trace": { ... },
  "latency_ms": 1850
}
```

Response (validator fallback):
```json
{
  "answer": {
    "level1": "Unable to answer right now.",
    "level2": "Validator rejected the draft because it violated format or hierarchy rules.",
    "level3": "Citations: None"
  },
  "error": { "code": "VALIDATION_FAILED" }
}
```

### `POST /ingest/drive-webhook` (optional)
Request body mirrors Google Drive push notifications. Only fields used: `resourceId`, `resourceUri`, `state`, `changed`: true/false, `channelParams` (mapped to `store_key`). Response: `204 No Content`.

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

All configuration keys live under an `env_cfg` module that loads `.env` values at startup and exposes `env_cfg["KEY"]`. Secrets never hard-coded; local development uses `.env.local`, production uses container secrets.
