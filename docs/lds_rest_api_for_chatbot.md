---
sidebar_position: 7
---

# Chapter 7: LDS REST API for Learning Design Facilitator (Appendix)

> **Appendix:** This chapter documents outbound calls from the **AI Agent** to the **LDS main system (Laravel)**. It is placed after the main integration flow so readers start with [Chapter 6: Main System Integration](./main_system_integration.md), not LDS internal APIs.

**Implementation reference:** **LDS-Chatbot** project (`api/lds_form_state.py`, `api/utils.py` → `make_lds_request`). Canonical Agent API schemas: `docs/openapi.json` in that repo.

---

## LDS (Laravel) vs this Agent (sub-system)

| Layer | Host | Role |
|-------|------|------|
| **LDS REST API for Learning Design Facilitator** (Options, Patterns, …) | LDS main system (Laravel) | The Agent **calls** these with LDS credentials to resolve IDs and fetch patterns when building context. **LDS integrators do not call these from the browser.** |
| **POST `/api/general_bot`**, **POST `/api/ilo_bot`** | **This AI Agent REST API Server** | LDS **calls** these with a Bearer token; responses use the [External AI Agent JSON shape](https://hkucite.github.io/lds-external-ai-agent/docs/overall-json-structure). |

---

## Base URL and authentication

| Setting | Purpose |
|---------|---------|
| `LDS_BASE` / `LARAVEL_HOST_API` | LDS API base (e.g. `https://lds.cite.hku.hk/api`) |
| `LDS_TOKEN` | Bearer or API token for LDS |
| `LDS_CLIENT_ID`, `LDS_CLIENT_SECRET` | OAuth client credentials (scope default `chatbot`) |

The Agent uses `make_lds_request()` in `api/utils.py` to call LDS with the configured auth.

---

## Options endpoints (ID → display name)

Used when resolving `form_state` and `courseInfo` IDs. Defined in `api/lds_form_state.py` → `OPTIONS_ENDPOINTS`:

**Note on LDS API paths:** LDS Options and Patterns endpoints use the `/chatbot/` URL prefix (legacy LDS naming). This is unrelated to the product name **Learning Design Facilitator (LDF)**.

| Key | Method | Path (under LDS base) |
|-----|--------|------------------------|
| Grade levels | `POST` | `/chatbot/options/courses/grade-levels` |
| Subjects | `POST` | `/chatbot/options/courses/subjects` |
| Bloom taxonomy levels | `POST` | `/chatbot/options/intended-learning-outcomes/bloom-taxonomy-levels` |
| ILO types | `POST` | `/chatbot/options/intended-learning-outcomes/types` |

**Typical request body:**

```json
{
  "course_ids": [1490],
  "per_page": 100,
  "locale": "zh_HK"
}
```

**Response handling:** list or `{ "data": [...] }`; normalized to compact `{ "id", "name" }` for prompts.

---

## Patterns endpoints (learning-design reference)

Defined in `api/lds_form_state.py` → `PATTERNS_ENDPOINTS`:

| Key | Method | Path (under LDS base) |
|-----|--------|------------------------|
| Disciplinary practices | `POST` | `/chatbot/patterns/disciplinary-practices` |
| Pedagogical approaches | `POST` | `/chatbot/patterns/pedagogical-approaches` |
| Intended learning outcomes | `POST` | `/chatbot/patterns/intended-learning-outcomes` |

The Agent also exposes a proxy: **POST `/api/chatbot/patterns/intended-learning-outcomes`** (Bearer auth) for callers that need ILO patterns via the Agent.

---

## `options/aggregate` (LDS main system only)

Some LDS documentation and Postman collections describe a combined **`options/aggregate`** call (batching multiple option paths in one request). **The Agent implementation does not call aggregate**; it uses the individual Options/Patterns endpoints above.

For aggregate testing, use the official LDS collection (see [Chapter 5: API Testing with Postman](./postman.md)).

---

## When the Agent calls LDS Options

- **`form_state`** or **`courseInfo`** contains IDs (`grade_level_id`, `subject_ids`, `type_id`, `bloom_taxonomy_level_id`, …).
- **ILO Bot** needs bloom levels and ILO types to return valid `type_id` and `bloom_taxonomy_level_id`.
- **General Bot** may fetch options/patterns when building course context (can be skipped via `GENERAL_BOT_SKIP_DYNAMIC_OPTIONS`).

**Related env vars** (`deploy/env.backend.example` in LDS-Chatbot):

| Variable | Purpose |
|----------|---------|
| `LDS_OPTIONS_CACHE` / `LDS_OPTIONS_CACHE_TTL_SEC` | Cache option lookups |
| `LDS_RESOLVE_FORM_STATE_IDS` | Resolve IDs in `form_state` |
| `GENERAL_BOT_SKIP_DYNAMIC_OPTIONS` | Skip dynamic option fetch in general_bot |
| `LDS_COURSE_CONTEXT_MAX_CHARS` | Cap injected course context size |

---

## Handoff and specialist bots

General Bot → ILO Bot handoff (`open_specialist_bot`, legacy `open_ilo_bot`) is documented in [Chapter 8](./lds_handoff_general_bot_open_ilo_bot.md). That flow is **implemented** in LDS-Chatbot `main.py`; OpenAPI examples are in `docs/openapi.json`.

---

## Further reading

- [Chapter 6: Main System Integration](./main_system_integration.md) — what LDS calls on the Agent
- [Chapter 8: LDS handoff](./lds_handoff_general_bot_open_ilo_bot.md) — General Bot specialist handoff
- LDS-Chatbot `docs/LLM_LOGIC_OVERVIEW.md` — non-technical LLM flow for operators (also at `/docs/llm-logic-overview` on the Agent host)
