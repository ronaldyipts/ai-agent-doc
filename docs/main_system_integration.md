---
sidebar_position: 6
---

# Chapter 6: Main System Integration

This page describes what the main system (LDS) can ask this sub-system (AI Agent) to do. Once the main system has the **API endpoint, Client ID, Client Secret, and Access Key**, it can trigger the following APIs. The main system must obtain an **access token** first (see [Chapter 5: API Testing with Postman](./postman.md)), then call the endpoints with `Authorization: Bearer <access_token>`.

---

## Integration Quick Start (Ports + Links)

This Agent is a **REST API Server only**. LDS should call API endpoints directly.

| Purpose | Port | Link to call |
|---|---:|---|
| Production API entry (recommended) | 80/443 | `http(s)://<agent-host>/api/general_bot` / `http(s)://<agent-host>/api/ilo_bot` |
| FastAPI internal service (default deployment) | 5000 | `http://127.0.0.1:5000/api/general_bot` / `http://127.0.0.1:5000/api/ilo_bot` |
| OpenAPI docs | same as API entry | `http(s)://<agent-host>/docs` |

If your environment uses different ports (e.g. `8000`), keep the same paths and replace host/port accordingly.

---

## LDS-Dedicated Endpoints (Recommended for Integration)

When integrating with the LDS main system, **use these two LDS-compatible endpoints first**. The request body must include **`courseInfo`**; optional fields include **`referrer_pathname`**, **`form_state`**, and learning-design arrays. The Agent uses them to provide contextual replies.

### POST `/api/general_bot` — General conversation

- **Purpose**: General learning-design conversation, guidance, and suggestions.
- **Required**: `message`, `courseInfo`
- **Optional**: `referrer_pathname`, `form_state`, `disciplinaryPractices`, `pedagogicalApproaches`, `intendedLearningOutcomes`, `lessons`
- **Response**: `{ chat_message_reply: { text }, actions: [...] }`

When LDS triggers the request from a form page (e.g. Course Information, ILO edit page), it can send **`form_state`**. The Agent uses it to build context; IDs can be resolved to human-readable names via LDS options lookup when needed.

#### DRAFT: `open_ilo_bot` action (General Bot → ILO Bot)

The Agent **may** (once implemented) return an action with **`action_type: "open_ilo_bot"`** when the user is ILO-focused, so LDS can show a button that then calls **POST `/api/ilo_bot`**. **Contract, examples, responsibilities:** [LDS handoff: General Bot → ILO Bot](./lds_handoff_general_bot_open_ilo_bot.md). **Status:** DRAFT; production may not emit this yet.

### POST `/api/ilo_bot` — Generate ILO suggestions

- **Purpose**: Get AI-suggested Intended Learning Outcomes (e.g. when the user clicks “AI suggest ILO”).
- **Required**: `courseInfo`
- **Optional**: `referrer_pathname`, `form_state`, `disciplinaryPractices`, `pedagogicalApproaches`, `intendedLearningOutcomes`, `lessons`, `request_type` (`initial` \| `reload`, default `initial`), `reload_metadata` (for `reload`: include `original_suggestions` with the last three UI suggestions so the model diversifies). Snake_case or camelCase accepted.
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

**ILO prompt (Oscar):** ILO generation follows the rules in [ILO Prompt (Oscar)](./prompts/ilo_prompt_oscar.txt): four ILO categories (Disciplinary Knowledge, Disciplinary Skills, Generic Skills, Values and Attitudes), Bloom’s Taxonomy verbs, quality criteria (student-centred, measurable, specific, appropriate difficulty), cognitive progression, alignment with course information, and suggested counts per category (e.g. about four). Placeholders `{COURSE_INFORMATION}` and `{ILO_GUIDELINES}` are filled from the request body and RAG or guideline content.

#### Button-triggered “AI suggest ILO” flow (main system → sub-system)

1. **Main system UI**: Add a button (e.g. “AI suggest ILO”) on the course/unit edit page.
2. **On click**: The main system calls **POST** `{sub-system base URL}/api/ilo_bot` with an access token; body includes `courseInfo` (and optionally `form_state`, `referrer_pathname`). For a **Reload** action, set `request_type` to `reload` and pass the previous three suggestions in `reload_metadata.original_suggestions`.
3. **Sub-system**: Returns ILO suggestions (`chat_message_reply` + `actions`).
4. **Main system**: Displays the response in a modal, sidebar, or form for the teacher to select or edit, then save.

#### Contract Acceptance Checklist (for Ronson)

- Caller can reach:
  - `POST /api/general_bot`
  - `POST /api/ilo_bot`
  - `GET /docs`
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
| **Simple health check** | GET | `/` | Check if the sub-system is up. |
| **Detailed health check** | GET | `/api/health` | Get detailed health status. |
| **Swagger UI** | GET | `/docs` | Interactive API documentation. |
| **OpenAPI spec** | GET | `/api/openapi.json` | OpenAPI (Swagger) JSON. |

---

## Summary

- **LDS main system**: Prefer **POST `/api/general_bot`** (general conversation) and **POST `/api/ilo_bot`** (ILO suggestions), with **`courseInfo`** in the body and optional **`referrer_pathname`**, **`form_state`**, and learning-design arrays.
- **form_state**: When triggering from a form page, send `form_state`. The Agent uses it to build context and may use LDS options lookup to resolve IDs to readable names.
- Other endpoints (`/api/chat`, `/api/generate_ilos`, etc.) remain available; see [Chapter 2: Application Architecture](./app_archi.md) and [Chapter 5: API Testing with Postman](./postman.md) for details.
- LDS options/patterns lookup API details are in **Appendix**: [Chapter 7: LDS REST API for Chatbot (Appendix)](./lds_rest_api_for_chatbot.md).
