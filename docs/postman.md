---
sidebar_position: 5
---

# Chapter 5: API Testing with Postman

A curated **Postman Collection** is provided for streamlined testing. **Import only one JSON file** (the collection). Set **username** and **password** (the AI agent's Admin Portal credentials, not LDS) in the collection variables, run **1. Retrieve Access Token**, then call any API.

**Auth**: LDS integration uses **`POST /api/auth/token`** on the **main** host (`{{base_url}}/api/auth/token`), not `/admin/api/`. Use an Admin Portal **service account** (e.g. `lds` created with `create_admin.py --no-admin`) — these are not LDS main-system login credentials. Email OTP does **not** apply to this token ([Chapter 3](./login_page.md)). Create accounts via **`create_admin.py`** ([Chapter 10](./deployment_operations.md)); no public registration.

## Workflow (4 steps)

1. **Import the collection**
   - Open Postman → **Import** → select or drag the collection file, or use the link below.
   - **Download:** [**LDS-AI-Agent.postman_collection.json**](https://ronaldyipts.github.io/ai-agent-doc/postman/LDS-AI-Agent.postman_collection.json) (click to download, then Import in Postman).
   - No separate environment file needed; variables are built into the collection.

2. **Set variables**
   - Click the collection **Learning Design Facilitator (LDF)** → **Variables** tab.
   - Set **`username`** and **`password`** (the **AI agent's Admin Portal** username and password, not LDS credentials).
   - **`client_id`**, **`client_secret`**, **`scope`** are optional (used only for the OAuth2 authorization-code request in Auth; **1. Retrieve Access Token** uses only username and password).
   - **`scope`** is pre-set to `chatbot`; **`base_url`** defaults to **`https://ideals-ldf.cite.hku.hk`** (production). For testing, set **`http://ronald-test.cite.hku.hk`**. Save.

### Environments (`base_url`)

| Environment | `base_url` | Protocol |
|-------------|------------|----------|
| **Production** | `https://ideals-ldf.cite.hku.hk` | HTTPS |
| **Testing** | `http://ronald-test.cite.hku.hk` | HTTP |
| **Local** | `http://127.0.0.1:5000` or `http://localhost:5000` | HTTP |

Optional environment files: **`LDS-AI-Agent-Production.postman_environment.json`** (production) and **`LDS-AI-Agent-Testing.postman_environment.json`** (testing) — download from `static/postman/` after deploy.

3. **Retrieve access token**
   - Open **0 - Auth** → send **1. Retrieve Access Token**.
   - On success, `access_token` (and `refresh_token` if returned) is saved automatically into the collection variables.

4. **Call the APIs**
   - All requests in **2 - Core** (and **Me** / **Logout** in Auth) use the saved Bearer token.
   - **For LDS integration**: Prefer **General Bot** (body: `message`, `courseInfo`, optional `referrer_pathname`, `form_state`) and **ILO Bot** (body: `courseInfo`, optional `referrer_pathname`, `form_state`).
   - Others: Chat, Generate ILOs, Suggest Disciplinary Practices, Analyze Document, Extract Course Info, Save Conversation.

## File

| File | Purpose |
|------|--------|
| [**LDS-AI-Agent.postman_collection.json**](https://ronaldyipts.github.io/ai-agent-doc/postman/LDS-AI-Agent.postman_collection.json) | All API requests (Auth, System, Core) and built-in variables. **1. Retrieve Access Token** sends **x-www-form-urlencoded** body: `username`, `password`. Click the link to download, then Import in Postman. |
| [**LDS-AI-Agent-Production.postman_collection.json**](https://ronaldyipts.github.io/ai-agent-doc/postman/LDS-AI-Agent-Production.postman_collection.json) | **Production smoke test** for `https://ideals-ldf.cite.hku.hk`. Bodies use [**COURSE.json**](./postman/COURSE.json). |
| [**COURSE.json**](./postman/COURSE.json) | Canonical sample `courseInfo`, `form_state`, and full `general_bot` / `ilo_bot` request bodies. |
| [**LDS-AI-Agent-Testing.postman_environment.json**](https://ronaldyipts.github.io/ai-agent-doc/postman/LDS-AI-Agent-Testing.postman_environment.json) | **Testing** `base_url`: `http://ronald-test.cite.hku.hk` |
| [**LDS-AI-Agent-Production.postman_environment.json**](https://ronaldyipts.github.io/ai-agent-doc/postman/LDS-AI-Agent-Production.postman_environment.json) | **Production** `base_url`: `https://ideals-ldf.cite.hku.hk` |

### Production site (`https://ideals-ldf.cite.hku.hk`)

1. Import **`docs/postman/LDS-AI-Agent-Production.postman_collection.json`** (or download from the link above).
2. Optional: import **`LDS-AI-Agent-Production.postman_environment.json`** (`base_url` pre-set to production).
3. Set **`username`** / **`password`** (LDS service account or Admin Portal user).
4. Run in order: **1. Retrieve Access Token** → **Health Check** → **Get Agent Info** → **General Bot** → (if HTTP **202**, **General Bot Job Status**) → **ILO Bot**.
5. Request bodies are also documented in **`docs/postman/COURSE.json`** (`general_bot_request`, `ilo_bot_request_initial`, `ilo_bot_request_reload`).

### Testing site (`http://ronald-test.cite.hku.hk`)

1. Import the main or Production collection (same requests).
2. Set **`base_url`** to **`http://ronald-test.cite.hku.hk`**, or import **`LDS-AI-Agent-Testing.postman_environment.json`**.
3. Use the same token → health → bot flow as production.

**Admin Portal (operators):** `https://ideals-ldf.cite.hku.hk/admin/` (production) or `http://ronald-test.cite.hku.hk/admin/` (testing). Runtime LDS integration still uses **`POST /api/auth/token`** on the main host, not `/admin/api/auth/token`.

### Collection split (important)

- **LDS options/patterns testing** (main system Laravel API): use the official LDS collection:  
  [**LDS-Rest-API-for-Chatbot.postman_collection.json**](https://hkucite.github.io/lds-external-ai-agent/lds/LDS-Rest-API-for-Chatbot.postman_collection.json) *(LDS main-system API; filename uses legacy “Chatbot” label)*  
  Includes individual option paths and optional **`options/aggregate`** (aggregate is **not** called by the Agent backend; see [Chapter 7 Appendix](./lds_rest_api_for_chatbot.md)).
- **AI Agent endpoint testing**: use this project’s collection (`LDS-AI-Agent.postman_collection.json`), especially:
  - `POST /api/general_bot`
  - `POST /api/ilo_bot`
  - `GET /api/info` (agent capabilities for LDS UI)
  - `GET /api/jobs/{job_id}` (when `BOT_RESPONSE_MODE=async`)

**Maintainers:** The download link serves the file from **`static/postman/LDS-AI-Agent.postman_collection.json`**. After editing the source collection in **`postman/LDS-AI-Agent.postman_collection.json`**, copy it to **`static/postman/`** before building/deploying so the public link serves the latest version.

## Collection structure

- **0 - Auth**: **1. Retrieve Access Token** (form body: `username`, `password` → saves token), OAuth2 code exchange, Refresh Token, Me, Logout. **No public registration**; accounts are created by an admin.
- **1 - System**: Health check (root and `/api/health`), **Get Agent Info** (`/api/info`).
- **2 - Core**: **General Bot**, **General Bot Job Status (Async)** (when General Bot returns `202`), **ILO Bot** (initial), **ILO Bot — reload**, Chat, Generate ILOs, Suggest Disciplinary Practices, Analyze Document, Extract Course Info, Save Conversation.

### LDS-dedicated endpoints (General Bot / ILO Bot)

| Request | Endpoint | Required body | Optional body |
|---------|----------|---------------|---------------|
| **General Bot** | POST `/api/general_bot` | `message`, `courseInfo` | `referrer_pathname`, `form_state`, learning-design arrays, `conversation_history`, `chat_session_id`, `locale` |
| **ILO Bot** | POST `/api/ilo_bot` | `courseInfo` | `referrer_pathname`, `form_state`, learning-design arrays; `request_type` (`initial` \| `reload`); for `reload`, `reload_metadata.original_suggestions` (last 3 suggestions); `locale` (`zh_HK` \| `en_US`). CamelCase aliases supported. |

Response format: `{ chat_message_reply: { text }, actions: [...] }`. The collection includes **ILO Bot — reload** for the `request_type` + `reload_metadata` contract; canonical field definitions live in **LDS-Chatbot** `docs/openapi.json` (`ilo_request`). See [Chapter 6: Main System Integration](./main_system_integration.md).

### Postman JSON examples (current behavior)

General Bot currently follows a **handoff-first** pattern for apply/update-oriented ILO requests.  
For these requests, expect a handoff action (`open_specialist_bot`, legacy `open_ilo_bot`) so the **LDS client app** can trigger ILO Bot.
This example is also synced into `postman/LDS-AI-Agent.postman_collection.json` and `static/postman/LDS-AI-Agent.postman_collection.json`.

**General Bot request body** (`POST /api/general_bot`)

```json
{
  "message": "Please help apply these ILO updates in LDS.",
  "courseInfo": {
    "topic": "Sustainable City",
    "subject": "General Studies",
    "grade": "Primary 5"
  },
  "referrer_pathname": "/designstudio/123/intended-learning-outcome/456/edit/",
  "form_state": {
    "statement": "",
    "type_id": 0,
    "bloom_taxonomy_level_id": 0
  }
}
```

**Expected response shape** (handoff action for ILO Bot)

```json
{
  "chat_message_reply": {
    "text": "Here are suggested ILOs based on your course context."
  },
  "actions": [
    {
      "action_type": "open_specialist_bot",
      "target": {
        "context": "ILO",
        "context_object_id": 456
      },
      "payload": {
        "specialist_type": "ilo_bot",
        "intent": "refine_ilos",
        "trigger_reason": "user_requested_apply_ilo_changes",
        "confirm_required": true,
        "button_label_zh": "生成預期學習成果建議",
        "button_label_en": "Refine with ILO Bot"
      },
      "ui": {
        "presentation": "inline",
        "highlight_target": ""
      }
    }
  ]
}
```

## Collection variables

Edit the collection → **Variables** tab. No separate environment file needed.

| Variable | Required for token flow | Description |
|----------|--------------------------|-------------|
| `base_url` | Yes | **Production:** `https://ideals-ldf.cite.hku.hk` · **Testing:** `http://ronald-test.cite.hku.hk` · **Local:** `http://127.0.0.1:5000` |
| `username` | Yes | AI agent's Admin Portal username (not LDS) |
| `password` | Yes | AI agent's Admin Portal password (not LDS) |
| `client_id` | No | OAuth2 client ID (for authorization-code exchange only; token request uses only username/password) |
| `client_secret` | No | OAuth2 client secret (for authorization-code exchange only) |
| `scope` | No | Pre-set to `chatbot`; used in OAuth2 code exchange |
| `access_token` | — | Set automatically by **1. Retrieve Access Token** (Bearer token) |
| `refresh_token` | — | Set automatically by **1. Retrieve Access Token** |
| `conversation_id` | — | Optional; for Chat / Save Conversation |
| `auth_code`, `redirect_uri` | No | For OAuth2 authorization-code flow only |

## Troubleshooting: 422 "username / password required"

If **1. Retrieve Access Token** returns `422 Unprocessable Entity` with "Field required" for username and password:

1. **Use the Body tab (not Params)**  
   Click the **Body** tab for **1. Retrieve Access Token**. Ensure **x-www-form-urlencoded** is selected and the table includes: **`username`**, **`password`**. If Body is set to "none", the API returns 422. Re-import the collection to restore the default body.

2. **Set the *Current value* for username and password**  
   In the Variables table, Postman has two columns: **Initial value** (for sharing) and **Current value** (used when sending the request). The request uses **Current value**. If you only filled Initial value, Current value can be blank and the server will get empty credentials → 422.  
   Fill the **Current value** column for both `username` and `password`, then Save and send again.

3. **Environment vs collection**  
   If an **environment** is selected (top-right), its variables override the collection. Either:
   - Choose **No Environment** so the collection variables are used, or  
   - Open that environment → **Variables** and set **username** and **password** in the **Current value** column there as well.

4. **Re-import the collection**  
   If the Body was never set correctly, re-import **`LDS-AI-Agent.postman_collection.json`** (replace existing) and set username/password in the collection Variables again.

For file uploads (**Analyze Document**, **Extract Course Info**), choose a file in the request **Body** → **form-data** in Postman.
