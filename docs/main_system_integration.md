---
sidebar_position: 6
---

# Chapter 6: Main System Integration

This page describes what the main system (LDS) can ask this sub-system (AI Agent) to do. Once the main system has the **API endpoint, Client ID, Client Secret, and Access Key**, it can trigger the following APIs. The main system must obtain an **access token** first (see [Chapter 5: API Testing with Postman](./postman.md)), then call the endpoints with `Authorization: Bearer <access_token>`.

---

## LDS-Dedicated Endpoints (Recommended for Integration)

When integrating with the LDS main system, **use these two LDS-compatible endpoints first**. The request body must include **`courseInfo`**; optional fields include **`referrer_pathname`**, **`form_state`**, and learning-design arrays. The Agent uses them to provide contextual replies.

### POST `/api/general_bot` — General conversation

- **Purpose**: General learning-design conversation, guidance, and suggestions.
- **Required**: `message`, `courseInfo`
- **Optional**: `referrer_pathname`, `form_state`, `disciplinaryPractices`, `pedagogicalApproaches`, `intendedLearningOutcomes`, `lessons`
- **Response**: `{ chat_message_reply: { text }, actions: [...] }`

When LDS triggers the request from a form page (e.g. Course Information, ILO edit page), it can send **`form_state`** (same structure as the front-end form). The Agent uses it to build context; IDs can be resolved to human-readable names via the LDS Options API if needed.

### POST `/api/ilo_bot` — Generate ILO suggestions

- **Purpose**: Get AI-suggested Intended Learning Outcomes (e.g. when the user clicks “AI suggest ILO”).
- **Required**: `courseInfo`
- **Optional**: `referrer_pathname`, `form_state`, `disciplinaryPractices`, `pedagogicalApproaches`, `intendedLearningOutcomes`, `lessons`
- **Response**:  
  - Top level: `{ chat_message_reply: { text }, actions: [ ... ] }`  
  - For each `show_suggestion` action, `payload.suggestions[]` contains items with:  
    - `statement` — ILO text (string)  
    - `type_id` — **LDS ILO category/type ID (integer, mandatory)**  
    - `bloom_taxonomy_level_id` — **LDS Bloom taxonomy level ID (integer, mandatory)**  

LDS can use `type_id` and `bloom_taxonomy_level_id` to **pre-fill the ILO category and Bloom level fields** when writing the suggestion back into the ILO form.

#### Button-triggered “AI suggest ILO” flow (main system → sub-system)

1. **Main system UI**: Add a button (e.g. “AI suggest ILO”) on the course/unit edit page.
2. **On click**: The main system calls **POST** `{sub-system base URL}/api/ilo_bot` with an access token; body includes `courseInfo` (and optionally `form_state`, `referrer_pathname`).
3. **Sub-system**: Returns ILO suggestions (`chat_message_reply` + `actions`).
4. **Main system**: Displays the response in a modal, sidebar, or form for the teacher to select or edit, then save.

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
- **form_state**: When triggering from a form page, send `form_state` (same structure as the front-end form). The Agent uses it to build context and may use the LDS Options API to resolve IDs to readable names.
- Other endpoints (`/api/chat`, `/api/generate_ilos`, etc.) remain available; see [Chapter 2: Application Architecture](./app_archi.md) and [Chapter 5: API Testing with Postman](./postman.md) for details.
