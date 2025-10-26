# HOA Chat System — Comprehensive Technical Checkpoint

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
