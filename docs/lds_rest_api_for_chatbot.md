---
sidebar_position: 7
---

# Chapter 7: LDS REST API for Chatbot

This AI Agent sub-system may call the **LDS REST API for Chatbot** (Options API) to resolve IDs in `form_state` (e.g. `grade_level_id`, `subject_ids`) into human-readable names, which are then included in the generation context.

- **Purpose**: When LDS sends `form_state` (same structure as the front-end form), the Agent can use this API to map option IDs to display names, improving contextual and readable replies.
- **Full endpoint list and usage**: See **`docs/LDS-REST-API-for-Chatbot.md`** in the LDS-Chatbot project (or the Swagger/docs provided by the main system team). This page is a short overview; the authoritative specification is the main system documentation.

Related: [Chapter 6: Main System Integration](./main_system_integration.md), [Chapter 4: Chatbot Specifications — LDS context](./chatbot_spec.md#46-lds-context).

---

## LDS (Laravel) vs this Agent (sub-system)

| Layer | Host | Role |
|-------|------|------|
| **LDS REST API for Chatbot** (Options, Patterns, Learning Design GET, …) | LDS main system (Laravel) | The Agent **calls** these endpoints (with LDS OAuth credentials) to resolve IDs and fetch patterns when building context. |
| **POST `/api/general_bot`**, **POST `/api/ilo_bot`** | **This AI Agent** sub-system | LDS **calls** these with a Bearer token; responses use the [External AI Agent JSON shape](https://hkucite.github.io/lds-external-ai-agent/docs/overall-json-structure). |

---

## DRAFT: General Bot → ILO Bot (`open_ilo_bot`)

**Proposal:** When the user’s message is ILO-related, **POST `/api/general_bot`** may include an **`open_ilo_bot`** item in `actions` so LDS can render a button; on click, LDS calls **POST `/api/ilo_bot`** with the same `courseInfo` / `form_state` / `referrer_pathname` pattern as today.

- **Full contract (Chinese), JSON examples, split of responsibilities:** [LDS handoff: `open_ilo_bot`](./lds_handoff_general_bot_open_ilo_bot.md)  
- **OpenAPI** (`/general_bot` examples, `general_bot_action_open_ilo_bot` schema): maintained in the **LDS-Chatbot** repo — `docs/openapi.json`.

**Status:** DRAFT — production Agent may not emit `open_ilo_bot` until implemented; LDS can still design UI against this spec.
