---
sidebar_position: 2
---

# Chapter 2: Application Architecture

## 2.1 Overall Architecture Pattern
- **This sub-system is a REST API Server (FastAPI)** — implementation: **LDS-Chatbot** (`main.py`).
- It does **not** provide an end-user frontend UI as part of the integration contract.
- The LDS main system (Laravel) is the caller and UI host; this Agent only exposes HTTP APIs.
- Optional admin components may exist for operations, but they are not part of the LDS-to-Agent runtime API contract.
- **Canonical API contract:** LDS-Chatbot `docs/openapi.json` (live at `/openapi.json` and `/docs` on the Agent host).

## 2.2 Main Application Architecture

### 2.2.1 REST API Server Architecture
Tech Stack
- FastAPI (Python Web Framework)
- Uvicorn (ASGI Server)
- Pydantic (Data Validation)
- Azure OpenAI (AI Service)
- SQLAlchemy (Optional, used for OAuth)

Directory Structure (LDS-Chatbot)
```
├── main.py              # FastAPI app: general_bot, ilo_bot, jobs, auth, health
├── api/
│   ├── utils.py         # OpenAI, LDS HTTP client, CHATBOT_SCHEMA
│   ├── auth.py          # JWT / OAuth helpers
│   ├── lds_form_state.py # LDS Options/Patterns fetch, form context
│   ├── ilo_prompt.py    # ILO prompt loader (docs/prompts/ILO_Prompt.txt)
│   └── rag_common.py    # Shared RAG retrieval
├── docs/
│   ├── openapi.json     # Canonical OpenAPI for Agent API
│   └── prompts/         # ILO_Prompt.txt
├── database.py          # Users, AccessLog, AgentJob (async general_bot jobs)
├── deploy/              # nginx.conf, backup.sh, systemd units, env examples
├── admin_portal/        # Optional operations UI/backend
└── requirements.txt
```

Endpoint Catalog
- Chapter 2 focuses on architecture only (no endpoint list).
- Runtime integration endpoints are documented in [Chapter 6: Main System Integration](./main_system_integration.md).
- LDS Options/lookup API background is documented in [Chapter 7: LDS REST API for Learning Design Facilitator (Appendix)](./lds_rest_api_for_chatbot.md).
- Testable endpoint examples are documented in [Chapter 5: API Testing with Postman](./postman.md).

Integration Ports and Links (for LDS caller)

| Item | Default Port | Example Link |
|------|--------------|--------------|
| API public entry (via reverse proxy) | 443 (production) / 80 (testing) | `https://ideals-ldf.cite.hku.hk/...` or `http://ronald-test.cite.hku.hk/...` |
| FastAPI app (internal process) | 5000 (default deploy) | `http://127.0.0.1:5000/docs` |
| Admin backend (optional operations only) | 5001 (default deploy) | `http://127.0.0.1:5001/...` |
| Docs site (optional static site) | 3000 (local preview) | `http://127.0.0.1:3000/` |

AI Integration Layer (api/utils.py)
- call_openai() - Calls Azure OpenAI API
- run_chat_with_optional_tools() - Chat supporting tool calls
- validate_response_format() - Validates JSON response format
- generate_suggested_questions() - Generates follow-up questions

LDS API Integration Layer (`api/lds_form_state.py`, `api/utils.py`)
- make_lds_request() - LDS API request (Bearer/Token/OAuth)
- OPTIONS_ENDPOINTS / PATTERNS_ENDPOINTS - grade levels, subjects, Bloom, ILO types, DP/PA/ILO patterns
- Resolves `form_state` IDs to human-readable names for prompts

Response Format Validation
- CHATBOT_SCHEMA - JSON Schema definition
- validate_response_format() - Validation function
- Double validation: forced format during API call + validation after response

### 2.2.2 Core Business Logic
See Chapter 4 for LDF logic details.

### 2.2.3 Data Flow (LDS ↔ Agent API)
1. LDS obtains an access token via **`POST /api/auth/token`** ([Chapter 3](./login_page.md), [Chapter 6](./main_system_integration.md)).
2. LDS calls the runtime integration APIs ([Chapter 6](./main_system_integration.md)).
3. Agent validates request (Pydantic).
4. Agent composes context from `courseInfo`, `form_state`, optional conversation history, LDS Options lookup (Appendix Chapter 7), and optional RAG retrieval (`api/rag_common.py`).
5. Agent calls Azure OpenAI and post-processes output (including handoff actions and ILO diversity rules).
6. Agent returns LDS-compatible JSON (`chat_message_reply` + `actions`), or **HTTP 202** with `job_id` when `BOT_RESPONSE_MODE=async` (`AgentJob` row in `lds_chatbot.db`).
7. LDS renders the response in its own UI; polls `GET /api/jobs/{job_id}` when async.

## 2.3 Optional Admin Portal (non-runtime)

Security positioning:
- The External Agent is **API-first** for LDS; the main **Learning Design Facilitator (LDF)** web UI may be disabled at Nginx while `/api/` stays live ([Chapter 10](./deployment_operations.md)).
- **Admin Portal** (`/admin/`) is for operators (logs, users, monitoring). Protect with **email OTP 2FA** in production ([Chapter 3](./login_page.md)).
- LDS runtime auth uses **`POST /api/auth/token`** on the main backend; credentials are validated against Admin Portal users but **OTP does not apply** to LDS tokens.

### 2.3.1 Backend Architecture
Tech Stack
- FastAPI
- SQLAlchemy (ORM)
- SQLite (default: `admin_portal/backend/admin_portal.db`)
- bcrypt (password hashing)
- PyJWT (JWT; not the unrelated `jwt` package)

Database Models (admin backend)
- **User** — username, password hash, email, role (`admin` \| `user`)
- **LoginOtpChallenge** — email OTP challenges when `ADMIN_2FA_ENABLED=true`
- OAuthClient, AuthorizationCode, AccessToken — OAuth2 client management (optional)

2FA configuration (`/etc/lds-chatbot-admin.env`):
- `ADMIN_2FA_ENABLED` — master switch
- `ADMIN_2FA_ALL_USERS` — default `true`: all portal users require OTP
- `ADMIN_2FA_USERNAME` — used only when `ADMIN_2FA_ALL_USERS=false`
- `EMAIL_ENABLED`, `SMTP_*` — required to deliver OTP emails

Directory Structure (admin backend only)
`admin_portal/backend/` — auth routes (`/api/auth/token`, `/api/auth/verify-otp`), user management, access-log proxy.

User bootstrap: `admin_portal/backend/scripts/create_admin.py` (`--interactive`, `--reset-password`, `--set-email`, `--no-admin` for LDS service accounts). No public self-registration.

Admin Operations Reference
- Chapter 2 keeps architecture-only content.
- Deployment, backups, SQLite permissions: [Chapter 10](./deployment_operations.md).
- Runtime LDS integration API details: [Chapter 6](./main_system_integration.md).
