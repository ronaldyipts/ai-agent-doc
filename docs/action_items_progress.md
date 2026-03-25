---
sidebar_position: 9
---

# Chapter 9: Action Items Progress (AI Agent Development and Optimization)

This page maps the main **AI Agent development and optimization** items to what is currently covered in documentation and implementation.

**Legend:** ✅ Documented and/or implemented | 🟡 Partially covered | ❌ Not covered

---

## 1. RAG and classification optimization

| Item | Status | Notes |
|------|--------|--------|
| **Test bucket retrieval count** (e.g. 2 / 3 / all buckets; compare RAG vs zero-shot / one-shot; find optimal retrieval count) | ❌ | Docs do not describe eight buckets, retrieval-count experiments, or RAG vs zero/one-shot comparisons. Add RAG architecture and an experiment plan to the spec or a technical note. |
| **Pre-classification**: before RAG, use a lightweight LLM prompt to classify the user question (e.g. LDS vs ILEC), then route retrieval | ❌ | Docs do not mention pre-classification or LDS/ILEC routing. Define classification flow and triggers in the spec. |

**Recommendations:** add to [Chapter 4: Chatbot Specifications](./chatbot_spec.md) or a new section:

- RAG bucket list and purpose (if eight buckets exist, name them and map use cases).
- Retrieval count / strategy (how many buckets per request, multi-bucket mixing).
- If pre-classification is adopted: dimensions (LDS/ILEC, etc.), when it runs, and how it connects to RAG.

---

## 2. Prompt engineering and House Rules

| Item | Status | Notes |
|------|--------|--------|
| **Explicit constraints** in the prompt (e.g. DP and PA usually 1–2 items, not 5–6) | 🟡 | [Chapter 4](./chatbot_spec.md) covers Socratic nudging, scope, quick actions, and suggested questions, but not explicit DP/PA count caps as House Rules. |
| **Core rules (“ten commandments”)**: a one-page point-form baseline drafted with Oscar | 🟡 | **ILO rules:** Oscar’s ILO prompt is in the repo: [docs/prompts/ilo_prompt_oscar.txt](./prompts/ilo_prompt_oscar.txt). It covers four ILO categories, Bloom verbs, quality criteria, cognitive progression, alignment with course information, and suggested counts per category (e.g. about four). Cross-bot “ten commandments” can be added as a separate doc and linked here. |

**Recommendations:**

- Add a **House Rules** section listing business baselines (DP/PA counts, prohibitions, etc.).
- Point to Oscar-assisted rule lists (appendix or standalone file) from the prompt-design chapter.

---

## 3. ILO Bot optimization

| Item | Status | Notes |
|------|--------|--------|
| **UI/UX and trigger timing**: prefer triggering mainly on **Add** or **Edit** ILO, not arbitrary generation from an empty state | 🟡 | [Chapter 6: Main System Integration](./main_system_integration.md) describes the “AI suggest ILO” button and `courseInfo` in the body, but does not state that the control should only appear or be enabled in Add/Edit ILO contexts; main-system UI must enforce this. |
| **Context use**: Agent reads and uses existing **Course Information** for better ILO statements | ✅ | Docs require `courseInfo` and optional `form_state`, `referrer_pathname`; see [Chapter 6](./main_system_integration.md) and [Chapter 5](./postman.md). |
| **Structured output**: ILO Bot returns a full JSON shape that matches LDS (type, Bloom level, etc.) for direct form fill | ✅ | [Chapter 6](./main_system_integration.md) and [Postman](./postman.md) document `statement`, `type_id`, `bloom_taxonomy_level_id`, and pre-filling the ILO form. |

**Recommendation:** in the integration or UI spec, state explicitly that the “AI suggest ILO” control should be shown or enabled only in **Add ILO** or **Edit ILO** flows to avoid accidental use from a fully blank state.

---

## 4. General Bot positioning and safeguards

| Item | Status | Notes |
|------|--------|--------|
| **Clear capability copy**: UI explains when to use General Bot vs ILO Bot | 🟡 | [Chapter 6](./main_system_integration.md) distinguishes `general_bot` (general dialogue and guidance) and `ilo_bot` (ILO suggestions) at the API level. It does not prescribe main-system **UI** copy; product or design should add it. |
| **Context management**: long threads may exceed the context window; need summarization | ❌ | [Chapter 4](./chatbot_spec.md) only notes “recent history: last 10 messages”; no summarization, truncation, or sliding-window policy for long conversations. |

**Recommendations:**

- Add suggested one-line descriptions for General Bot vs ILO Bot in integration or design docs.
- In the chatbot spec, document behavior when conversation length or token limits are exceeded (summary, sliding window, etc.), and mark as TBD if not implemented.

---

## 5. Structured outputs and error handling

| Item | Status | Notes |
|------|--------|--------|
| **Strict JSON Schema**: model output fully matches LDS five-level-depth JSON Schema | 🟡 | [Chapter 2: Application Architecture](./app_archi.md) mentions `CHATBOT_SCHEMA`, `validate_response_format()`, and double validation, but not explicit “five-level depth” or a link to the full LDS schema. |
| **Fallback**: on invalid AI output or validation failure, handle errors or degrade gracefully so the system does not crash | ❌ | Docs do not describe fallback (retry, safe default payload, error codes, user-facing messages). |

**Recommendations:**

- State the required response JSON Schema (or link to LDS five-level definition) and when validation runs (request/response).
- Add an **Error handling** subsection: retries, safe defaults, logging, and reporting.

---

## Summary

| Area | Covered | Partial | Not covered |
|------|---------|---------|-------------|
| 1. RAG and classification | 0 | 0 | 2 |
| 2. House Rules / prompt | 0 | 2 | 0 |
| 3. ILO Bot | 2 | 1 | 0 |
| 4. General Bot | 0 | 1 | 1 |
| 5. Structured output / errors | 0 | 1 | 1 |

**Suggested priorities:** (1) RAG architecture (buckets, retrieval count, pre-classification) and experiment plan; (2) House Rules (including DP/PA counts and cross-bot commandments); (3) long-conversation summarization and structured-output fallback.

---

*Update this table as documentation and implementation change.*
