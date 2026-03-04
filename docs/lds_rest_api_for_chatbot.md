---
sidebar_position: 7
---

# Chapter 7: LDS REST API for Chatbot

This AI Agent sub-system may call the **LDS REST API for Chatbot** (Options API) to resolve IDs in `form_state` (e.g. `grade_level_id`, `subject_ids`) into human-readable names, which are then included in the generation context.

- **Purpose**: When LDS sends `form_state` (same structure as the front-end form), the Agent can use this API to map option IDs to display names, improving contextual and readable replies.
- **Full endpoint list and usage**: See **`docs/LDS-REST-API-for-Chatbot.md`** in the LDS-Chatbot project (or the Swagger/docs provided by the main system team). This page is a short overview; the authoritative specification is the main system documentation.

Related: [Chapter 6: Main System Integration](./main_system_integration.md), [Chapter 4: Chatbot Specifications — LDS context](./chatbot_spec.md#46-lds-context).
