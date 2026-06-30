---
sidebar_position: 9
---

# Chapter 9: Action Items Progress (AI Agent Development and Optimization)

This page maps **Learning Design Facilitator (LDF)** development and optimization items to what is currently covered in documentation and implementation.

**Legend:** ✅ Documented and/or implemented | 🟡 Partially covered | ❌ Not covered

---

## 1. RAG and classification optimization

| Item | Status | Notes |
|------|--------|--------|
| **Eight RAG buckets** (ILO, DP, PA, assessment, activity, Theory, general, ilap) | ✅ | Listed in [Chapter 4 §4.5](./chatbot_spec.md#45-rag-and-pre-classification); env tuning vars in [Chapter 10 §10.11](./deployment_operations.md#1011-key-environment-variables-main-backend) |
| **Test bucket retrieval count** (compare RAG vs zero-shot; optimal `RAG_TOP_K_TOTAL`) | ❌ | Experiment plan not documented |
| **Pre-classification** (LDS vs iLAP before retrieval) | ✅ | `RAG_PRE_CLASSIFY` in `api/rag_common.py`; [Chapter 4 §4.5](./chatbot_spec.md#45-rag-and-pre-classification) |

---

## 2. Prompt engineering and House Rules

| Item | Status | Notes |
|------|--------|--------|
| **Explicit constraints** (e.g. DP and PA usually 1–2 items) | 🟡 | Not yet a dedicated House Rules section; ILO rules in [ilo_prompt_oscar.txt](./prompts/ilo_prompt_oscar.txt) |
| **Cross-bot “ten commandments”** | 🟡 | ILO baseline documented; General Bot commandments TBD |

---

## 3. ILO Bot optimization

| Item | Status | Notes |
|------|--------|--------|
| **UI trigger timing** (Add/Edit ILO only) | 🟡 | LDS UI responsibility; API documented in [Chapter 6](./main_system_integration.md) |
| **Context use** (`courseInfo`, `form_state`) | ✅ | [Chapter 6](./main_system_integration.md), [Chapter 5](./postman.md) |
| **Structured output** (`type_id`, `bloom_taxonomy_level_id`) | ✅ | [Chapter 6](./main_system_integration.md) |
| **Reload diversity** | ✅ | [Chapter 4 §4.7](./chatbot_spec.md), [Chapter 6](./main_system_integration.md) |

---

## 4. General Bot positioning and safeguards

| Item | Status | Notes |
|------|--------|--------|
| **General vs ILO Bot capability copy** | 🟡 | API-level distinction in Ch6; LDS UI copy TBD |
| **Async `general_bot` jobs** | ✅ | `BOT_RESPONSE_MODE`, polling contract in [Chapter 6](./main_system_integration.md), [Chapter 10](./deployment_operations.md) |
| **Handoff** (`open_specialist_bot`) | ✅ | [Chapter 8](./lds_handoff_general_bot_open_ilo_bot.md) |
| **Long-conversation summarization** | ❌ | Last 10 messages only; noted in [Chapter 4 §4.10](./chatbot_spec.md#410-error-handling-and-fallbacks) |

---

## 5. Structured outputs, auth, and operations

| Item | Status | Notes |
|------|--------|--------|
| **LDS JSON Schema / `CHATBOT_SCHEMA`** | 🟡 | [Chapter 2](./app_archi.md); link to full LDS five-level schema TBD |
| **Error / fallback behavior** | 🟡 | Overview in [Chapter 4 §4.10](./chatbot_spec.md#410-error-handling-and-fallbacks) |
| **Admin Portal 2FA (all users)** | ✅ | [Chapter 3](./login_page.md), [Chapter 10](./deployment_operations.md) |
| **LDS auth (no OTP on `/api/auth/token`)** | ✅ | [Chapter 3](./login_page.md), [Chapter 6](./main_system_integration.md) |
| **Automated backups** | ✅ | [Chapter 10 §10.5](./deployment_operations.md#105-automated-backup) |
| **Deploy / update runbooks in ai-agent docs** | ✅ | Summarized in [Chapter 10 §10.9–10.10](./deployment_operations.md#109-initial-deployment) |
| **PyJWT dependency** | ✅ | [Chapter 10 §10.7](./deployment_operations.md#107-python-dependencies-jwt) |

---

## Summary

| Area | Covered | Partial | Not covered |
|------|---------|---------|-------------|
| 1. RAG and classification | 2 | 0 | 1 |
| 2. House Rules / prompt | 0 | 2 | 0 |
| 3. ILO Bot | 3 | 1 | 0 |
| 4. General Bot | 2 | 1 | 1 |
| 5. Schema / ops / auth | 5 | 2 | 0 |

**Suggested priorities:** (1) RAG retrieval-count experiment plan; (2) House Rules (DP/PA caps, cross-bot commandments); (3) long-conversation summarization and full LDS JSON Schema link.

---

*Update this table as documentation and implementation change.*
