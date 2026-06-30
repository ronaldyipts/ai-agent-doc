---
sidebar_position: 6
---

# Chapter 6: Main System Integration

This page describes what the main system (LDS) can ask the **Learning Design Facilitator (LDF)** to do. LDS needs:

1. **Base URL** — e.g. `https://ideals-ldf.cite.hku.hk` (production) or `http://ronald-test.cite.hku.hk` (testing)
2. **Service account** — username/password created in the Admin Portal user DB (e.g. `lds` via `create_admin.py --no-admin`)
3. **Bearer token** — from **`POST /api/auth/token`** on the **main** host ([Chapter 3](./login_page.md))

Then call integration endpoints with `Authorization: Bearer <access_token>`. For manual testing, see [Chapter 5: Postman](./postman.md).

---

## Integration Quick Start (Ports + Links)

This Agent is a **REST API Server only**. LDS should call API endpoints directly.

| Purpose | Port | Link to call |
|---|---:|---|
| Production API entry (recommended) | 443 | `https://ideals-ldf.cite.hku.hk/api/general_bot` / `https://ideals-ldf.cite.hku.hk/api/ilo_bot` |
| Testing API entry | 80 | `http://ronald-test.cite.hku.hk/api/general_bot` / `http://ronald-test.cite.hku.hk/api/ilo_bot` |
| FastAPI internal service (default deployment) | 5000 | `http://127.0.0.1:5000/api/general_bot` / `http://127.0.0.1:5000/api/ilo_bot` |
| OpenAPI docs | same as API entry | `https://ideals-ldf.cite.hku.hk/docs` (production) or `http://<testing-host>/docs` |

If your environment uses different ports (e.g. `8000`), keep the same paths and replace host/port accordingly.

**Production host:** `https://ideals-ldf.cite.hku.hk`  
**Testing host:** `http://ronald-test.cite.hku.hk`  

The public **LDF web UI** at production root may show a maintenance page; **LDS API integration** (`/api/general_bot`, `/api/ilo_bot`, `/api/auth/token`, etc.) remains available over HTTPS.

---

## Authentication for LDS (main API)

LDS must **not** call `/admin/api/auth/token` for runtime integration. Use the **main** Agent token endpoint:

| Step | Request |
|------|---------|
| 1 | `POST /api/auth/token` — `application/x-www-form-urlencoded`: `username`, `password` |
| 2 | Response: `{ access_token, token_type: "bearer" }` |
| 3 | All `/api/*` calls: `Authorization: Bearer <access_token>` |

When `ADMIN_PORTAL_URL` is set (production), the main backend validates credentials via Admin Portal **`POST /api/auth/validate-credentials`** (internal; `X-Admin-Token` = `ACCESS_LOG_ADMIN_TOKEN`). **No OTP email is sent.** The main backend then issues its **own JWT**. **Email OTP applies only to `/admin/api/auth/token`** (Portal UI).

Use a dedicated **service account** (e.g. `lds`) created with `create_admin.py --no-admin`. Human admin accounts are for Portal access only.

Postman: [Chapter 5](./postman.md) — production collection `LDS-AI-Agent-Production.postman_collection.json`, sample bodies in [`COURSE.json`](./postman/COURSE.json).

---

## LDS-Dedicated Endpoints (Recommended for Integration)

When integrating with the LDS main system, **use these two LDS-compatible endpoints first**. The request body must include **`courseInfo`**; optional fields include **`referrer_pathname`**, **`form_state`**, and learning-design arrays. The Agent uses them to provide contextual replies.

### POST `/api/general_bot` — General conversation

- **Purpose**: General learning-design conversation, guidance, and suggestions.
- **Required**: `message`, `courseInfo`
- **Optional**: `referrer_pathname`, `form_state`, `disciplinaryPractices`, `pedagogicalApproaches`, `intendedLearningOutcomes`, `lessons`, `conversation_history` (aliases: `conversationHistory`, `chatHistory`), `chat_session_id`, `locale` (`zh_HK` \| `en_US`)
- **LDS path behavior**: the dedicated LDS handler sets `skip_suggested_questions=true`; responses do **not** include `suggested_questions` (faster, LDS UI does not use them).
- **Response mode**:
  - **Sync mode** (`BOT_RESPONSE_MODE=sync`, default): `200` with `{ chat_message_reply, actions }`
  - **Async mode** (`BOT_RESPONSE_MODE=async`): `202` with `{ job_id, status, check_url }`, then poll `GET /api/jobs/{job_id}`

When LDS triggers the request from a form page (e.g. Course Information, ILO edit page), it can send **`form_state`**. The Agent uses it to build context; IDs can be resolved to human-readable names via LDS options lookup when needed.

#### Async Contract (current production)

When async mode is enabled:

1. LDS calls `POST /api/general_bot` as usual.
2. API returns `202 Accepted` with:
   - `job_id` (string)
   - `status` (`pending`)
   - `check_url` (for example: `/api/jobs/<job_id>`)
3. LDS polls `GET /api/jobs/{job_id}` with the same Bearer token.
4. Job status handling:
   - `pending` / `running` → keep polling (recommended interval **1–2 s**; max wait **~120 s** before timeout/retry UI)
   - `completed` → read `result` and render normally (`chat_message_reply`, `actions`)
   - `failed` → read `error` and show retry UI

**HTTP errors on `GET /api/jobs/{job_id}`:**

| Code | Meaning |
|------|---------|
| `403` | Bearer token user does not own this job |
| `404` | Unknown `job_id` (expired, wrong host, or typo) |

Jobs are stored in the main SQLite DB (`agent_jobs` table). There is no automatic TTL cleanup today—plan retention as part of go-live hygiene ([Chapter 10 §10.6](./deployment_operations.md#106-production-go-live-data-hygiene)).

**Detecting async mode:** set `BOT_RESPONSE_MODE=async` in `/etc/lds-chatbot.env`, or observe `202 Accepted` from `POST /api/general_bot`. `POST /api/ilo_bot` is always synchronous.

`GET /api/jobs/{job_id}` response shape:

```json
{
  "job_id": "f79051035cc3468ab2393417e3e7c5e0",
  "status": "completed",
  "endpoint": "/api/general_bot",
  "result": {
    "chat_message_reply": { "text": "..." },
    "actions": []
  },
  "error": null
}
```

#### LDS Integration Changes Required (after async switch)

- Update `POST /api/general_bot` handling to accept both `200` and `202`.
- Add polling flow to `GET /api/jobs/{job_id}`.
- Keep request body and Bearer auth unchanged.
- Keep rendering logic unchanged once `result` is received.
- Keep `POST /api/ilo_bot` as synchronous (no polling required for ILO today).

#### Specialist handoff (General Bot → ILO Bot)

The Agent returns **`open_specialist_bot`** (recommended) or legacy **`open_ilo_bot`** when the user is ILO-focused and handoff is appropriate. **Contract, examples, responsibilities:** [Chapter 8: LDS handoff](./lds_handoff_general_bot_open_ilo_bot.md). **Status:** Implemented in LDS-Chatbot; see `docs/openapi.json`.

### POST `/api/ilo_bot` — Generate ILO suggestions

- **Purpose**: Get AI-suggested Intended Learning Outcomes (e.g. when the user clicks “AI suggest ILO”).
- **Required**: `courseInfo`
- **Optional**: `referrer_pathname`, `form_state`, `disciplinaryPractices`, `pedagogicalApproaches`, `intendedLearningOutcomes`, `lessons`, `request_type` (`initial` \| `reload`, default `initial`), `reload_metadata` (for `reload`: include `original_suggestions` with the last three UI suggestions so the model diversifies), `locale` (`zh_HK` \| `en_US`). Snake_case or camelCase accepted.
- **Response**:
  - Top level: `{ chat_message_reply: { text }, actions: [ ... ] }`
  - For each `show_suggestion` action, `payload.suggestions[]` items include:
    - `statement` — ILO text (string)
    - `type_id` — **LDS ILO category/type ID (integer, mandatory)**
    - `bloom_taxonomy_level_id` — **LDS Bloom taxonomy level ID (integer, mandatory)**

LDS can use `type_id` and `bloom_taxonomy_level_id` to **pre-fill the ILO category and Bloom level fields** when writing suggestions back into the ILO form.

#### Current production behavior (authoritative)

- The API always returns **exactly 3 suggestions**.
- `statement` text is sanitized by backend: trailing metadata such as `(Level: ...)`, `(Type: ...)`, `(Bt Level: ...)` is removed.
- `type_id` and `bloom_taxonomy_level_id` are mandatory integer fields in each suggestion.
- For reload, use:
  - `request_type: "reload"`
  - `reload_metadata.original_suggestions` (recommended: pass the 3 suggestions currently shown in LDS UI)

#### Reload Diversity Rules (current system)

When `request_type = reload`, backend applies additional diversity constraints:

1. **Statement rule (strict)**  
   New statements cannot exactly repeat recent statements (same user/course/referrer window).

2. **Action verb rule (strict)**  
   Leading action verbs cannot repeat recent verbs in the same window.

3. **Task type rule (strict)**  
   Task types are diversified across the batch and against recent usage (e.g. avoid cycling `identify` / `compare` / `reflect`).

4. **Semantic similarity rule**  
   Backend checks semantic similarity and rewrites overly similar suggestions.

5. **Category / Bloom diversity rule**  
   Backend prefers diverse `type_id` and `bloom_taxonomy_level_id`, and tries to avoid previous batch IDs when valid alternatives exist.

> Diversity memory window: last 5 batches (up to 15 suggestions) per user+course+referrer runtime key.

#### Reload Environment Variables

- `ILO_RELOAD_SEMANTIC_THRESHOLD` (default `0.86`)
- `ILO_RELOAD_GENERATION_TEMPERATURE` (default `0.55`)
- `ILO_RELOAD_REWRITE_TEMPERATURE` (default `0.78`)
- `ILO_RELOAD_REWRITE_MAX_ATTEMPTS` (default `2`)

Lower threshold = stricter semantic dedup (more rewrites).

**ILO prompt:** ILO generation follows **`docs/prompts/ILO_Prompt.txt`** in LDS-Chatbot (mirrored in this doc site as [ILO Prompt (Oscar)](./prompts/ilo_prompt_oscar.txt)): four ILO categories, Bloom’s Taxonomy verbs, quality criteria, cognitive progression, alignment with course information. Placeholders `{COURSE_INFORMATION}` and `{ILO_GUIDELINES}` are filled at runtime (`api/ilo_prompt.py`).

#### Button-triggered “AI suggest ILO” flow (main system → sub-system)

1. **Main system UI**: Add a button (e.g. “AI suggest ILO”) on the course/unit edit page.
2. **On click**: The main system calls **POST** `{sub-system base URL}/api/ilo_bot` with an access token; body includes `courseInfo` (and optionally `form_state`, `referrer_pathname`). For a **Reload** action, set `request_type` to `reload` and pass the previous three suggestions in `reload_metadata.original_suggestions`.
3. **Sub-system**: Returns ILO suggestions (`chat_message_reply` + `actions`).
4. **Main system**: Displays the response in a modal, sidebar, or form for the teacher to select or edit, then save.

#### Integration acceptance checklist

- Caller can reach:
  - `POST /api/general_bot`
  - `GET /api/jobs/{job_id}` (when `BOT_RESPONSE_MODE=async`)
  - `POST /api/ilo_bot`
  - `GET /docs` or `GET /openapi.json`
- In async mode, `POST /api/general_bot` returns 202 with `job_id` and `check_url`.
- `GET /api/jobs/{job_id}` reaches `completed` and returns `result.chat_message_reply` + `result.actions`.
- `POST /api/ilo_bot` returns 200 and includes `actions[].payload.suggestions[3]`.
- `statement` does not include `(Level:..., Type:...)`.
- Reload request with `original_suggestions` returns a diversified batch (verb/type/semantic differences).

---

## Other APIs (Still Available)

The endpoints below remain available; **for LDS integration, prefer `/api/general_bot` and `/api/ilo_bot`**.

### 1. General conversation and learning outcomes

| Function | Method | Endpoint | Description |
|----------|--------|----------|-------------|
| **Conversation / Q&A** | POST | `/api/chat` | Send `message`, optional `conversation_id`. For LDS integration, use `/api/general_bot` instead. |
| **Generate ILO** | POST | `/api/generate_ilos` | Generate ILO by `topic`, `level`, `language`. For LDS integration, use `/api/ilo_bot` instead. |
| **Suggest Disciplinary Practices (DP)** | POST | `/api/suggest_dp` | Suggest DPs by `topic`, `context`. |

### 2. Document analysis and course information

| Function | Method | Endpoint | Description |
|----------|--------|----------|-------------|
| **Analyze teaching document** | POST | `/api/analyze-document` | Upload a file (PDF, DOCX, TXT) for analysis. |
| **Extract course info** | POST | `/api/extract-course-info` | Upload a file to extract course-related information. |

### 3. Conversation history

| Function | Method | Endpoint | Description |
|----------|--------|----------|-------------|
| **Save conversation** | POST | `/api/save-conversation` | Send `conversation_id`, `messages` to store history. |

### 4. System and documentation

| Function | Method | Endpoint | Description |
|----------|--------|----------|-------------|
| **Agent capabilities** | GET | `/api/info` | Returns `chatbot_paths` and `tasks` (id, name, description, path) for LDS UI discovery |
| **Simple health check** | GET | `/` | Check if the sub-system is up. |
| **Detailed health check** | GET | `/api/health` | Get detailed health status (includes LDS connectivity probe). |
| **Swagger UI** | GET | `/docs` | Interactive API documentation. |
| **OpenAPI spec** | GET | `/openapi.json` | OpenAPI (Swagger) JSON (also in repo: `docs/openapi.json`) |

---

## Summary

- **LDS main system**: Prefer **POST `/api/general_bot`** (general conversation) and **POST `/api/ilo_bot`** (ILO suggestions), with **`courseInfo`** in the body and optional **`referrer_pathname`**, **`form_state`**, and learning-design arrays.
- **form_state**: When triggering from a form page, send `form_state`. The Agent uses it to build context and may use LDS options lookup to resolve IDs to readable names.
- Other endpoints (`/api/chat`, `/api/generate_ilos`, etc.) remain available; see [Chapter 5: API Testing with Postman](./postman.md) and LDS-Chatbot `docs/openapi.json`.
- LDS options/patterns lookup API details are in **Appendix**: [Chapter 7: LDS REST API for Learning Design Facilitator (Appendix)](./lds_rest_api_for_chatbot.md).
