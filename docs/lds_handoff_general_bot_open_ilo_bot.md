---
sidebar_position: 8
---

# Chapter 8: LDS handoff: General Bot specialist handoff (`open_ilo_bot` and generic schema)

**Status: DRAFT (proposal)**  
**Goal:** Define a consistent handoff pattern from **General Bot** to a specialist bot. For ILO-related intent, backend can emit a handoff action in `actions`; LDS UI renders the trigger (button or equivalent), then LDS calls the mapped specialist API (for ILO, **`POST /api/ilo_bot`**).

> **Important:** At the time of writing, the **Agent backend may not yet emit this action in production**. This document fixes the **JSON contract** and **division of work** so LDS can design UI and flows in parallel. Backend delivery can be scheduled separately (e.g. feature flag plus rules or model output).

**Mirrored copy (implementation repo):** Keep in sync with **`docs/LDS_HANDOFF_GENERAL_BOT_OPEN_ILO_BOT.md`** in the **LDS-Chatbot** project; OpenAPI examples are in that repo’s **`docs/openapi.json`** (`POST /general_bot` response examples).

---

## 1. Responsibilities

| Owner | Work |
|--------|------|
| **Agent / Chatbot backend** | Under agreed conditions, include **`action_type: "open_ilo_bot"`** in `actions` on **`POST /api/general_bot`** responses (with `payload` as below); keep **`POST /api/ilo_bot`** stable; provide test environment and examples after rollout. |
| **LDS client application** | Parse `actions`; detect `open_ilo_bot`; render a trigger in LDS UI (labels from `payload` or your own i18n); on click, build the **`ilo_bot` request** aligned with the last **general_bot** call (`courseInfo`, `referrer_pathname`, `form_state`, etc.); call **`POST /api/ilo_bot`** and handle loading, errors, and tokens. |

---

## 2. Decision rule (`show_suggestion` vs handoff)

To reduce unnecessary clicks, use this rule before deciding action output.

| User intent in General Bot | Preferred behavior | Why |
|-------|------|------|
| User asks to **see suggestions only** (no explicit apply/update intent) | Return normal text plus **`show_suggestion`** when suggestions are ready; no handoff button required | Lowest-friction UX; user gets immediate draft content |
| User asks to **generate and apply/update LDS items** (writeback-oriented) | Return handoff action (ILO today: `open_ilo_bot`) so LDS can ask for confirmation and then call specialist API | Write/update flow needs explicit LDS-side confirmation |
| Intent is ambiguous | Ask one short clarifying question in chat, then follow one of the two paths above | Avoid wrong tool jump or accidental write flow |

Notes:
- If model confidence is low, prefer the safer `show_suggestion` path first.
- LDS should continue to ignore unknown `action_type` values for forward compatibility.

---

## 3. JSON contract (generic + ILO compatibility)

### 3.1 Placement in `POST /api/general_bot` responses

Same overall shape as the existing LDS External AI Agent:

- Top level: `chat_message_reply.text` (normal reply text; may explain that the user can open ILO Bot below).
- Top level: `actions` (array; **multiple** entries allowed; handoff action may coexist with others; LDS decides priority).

### 3.2 Action shape: generic handoff

Recommended new generic action:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `action_type` | string | Yes | **`"open_specialist_bot"`** (recommended generic name) |
| `target` | object | No | Specialist context object; same semantics as today (`context`, `context_object_id`) |
| `payload` | object | Yes | See table below |
| `ui` | object | No | `presentation`: `inline` \| `sidebar` \| `popup` (suggest **`inline`**) |

#### Suggested `payload` fields (generic)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `specialist_type` | string | Yes | Target specialist, e.g. `ilo_bot`, `mc_bot` |
| `intent` | string | Yes | Machine-readable intent, e.g. `generate_ilos`, `refine_ilos`, `generate_mc_items` |
| `trigger_reason` | string | No | For LDS logging / analytics (e.g. `user_asked_generate_ilo`) |
| `confirm_required` | boolean | No | Whether LDS should request explicit user confirmation before API call (default recommended: `true` for write/update) |
| `button_label_zh` | string | No | Default Traditional Chinese label; LDS may override with i18n key |
| `button_label_en` | string | No | Default English label |

#### API routing recommendation

| `payload.specialist_type` | LDS calls |
|-------|------|
| `ilo_bot` | `POST /api/ilo_bot` |
| `mc_bot` | `POST /api/mc_bot` (or project-approved MC endpoint) |

### 3.3 Backward compatibility (`open_ilo_bot`)

To avoid breaking LDS integrations already reading `open_ilo_bot`, Agent may keep emitting:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `action_type` | string | Yes | Must be **`"open_ilo_bot"`** |
| `target` | object | No | Align with `ilo_bot`: `context: "ILO"`, `context_object_id` = current ILO id being edited, or **0** if none |
| `payload` | object | Yes | See table below |
| `ui` | object | No | `presentation`: `inline` \| `sidebar` \| `popup` (suggest **`inline`** under the message); `highlight_target` optional |

`open_ilo_bot` can be interpreted as a specialization of `open_specialist_bot` with fixed `specialist_type = "ilo_bot"`.

#### Suggested `payload` fields (legacy-compatible)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `intent` | string | Yes | **`generate_ilos`** (batch ILO draft) \| **`refine_ilos`** (tune existing draft) \| **`explain_ilos`** (explain ILO / writing tips; still routes to ILO Bot for optional generation) |
| `trigger_reason` | string | No | For LDS logging / analytics (machine-readable, e.g. `user_asked_generate_ilo`) |
| `button_label_zh` | string | No | Default Traditional Chinese button label; LDS may override with your i18n key |
| `button_label_en` | string | No | Default English button label |

---

## 4. Example JSON

### Example A: User wants to generate/apply ILOs (generic handoff)

```json
{
  "chat_message_reply": {
    "text": "You can use ILO Bot to draft several adjustable learning outcomes from your current course context. If you prefer, we can keep refining the topic and grade focus in this chat."
  },
  "actions": [
    {
      "action_type": "open_specialist_bot",
      "target": {
        "context": "ILO",
        "context_object_id": 0
      },
      "payload": {
        "specialist_type": "ilo_bot",
        "intent": "generate_ilos",
        "trigger_reason": "user_asked_generate_ilo",
        "confirm_required": true,
        "button_label_zh": "生成預期學習成果建議",
        "button_label_en": "Open ILO Bot"
      },
      "ui": {
        "presentation": "inline",
        "highlight_target": ""
      }
    }
  ]
}
```

### Example B: Backward-compatible ILO action (`open_ilo_bot`)

```json
{
  "chat_message_reply": {
    "text": "Here is a short note on aligning ILOs with your course; you can also open ILO Bot to generate a draft."
  },
  "actions": [
    {
      "action_type": "open_ilo_bot",
      "target": { "context": "ILO", "context_object_id": 456 },
      "payload": {
        "intent": "refine_ilos",
        "trigger_reason": "user_on_ilo_edit_page",
        "button_label_zh": "生成預期學習成果建議",
        "button_label_en": "Refine with ILO Bot"
      },
      "ui": { "presentation": "inline", "highlight_target": "" }
    }
  ]
}
```

### Example C: MC handoff using generic action

```json
{
  "chat_message_reply": {
    "text": "I can route you to MC Bot to draft question items aligned with your current unit outcomes."
  },
  "actions": [
    {
      "action_type": "open_specialist_bot",
      "target": { "context": "MC", "context_object_id": 0 },
      "payload": {
        "specialist_type": "mc_bot",
        "intent": "generate_mc_items",
        "trigger_reason": "user_asked_generate_mc",
        "confirm_required": true,
        "button_label_zh": "開啟 MC Bot",
        "button_label_en": "Open MC Bot"
      },
      "ui": { "presentation": "inline", "highlight_target": "" }
    }
  ]
}
```

### Example D: Empty `actions` (still common today)

```json
{
  "chat_message_reply": {
    "text": "(Text-only reply; no UI action.)"
  },
  "actions": []
}
```

---

## 5. After the user clicks: call specialist API

- **Method / path:** route by `specialist_type` (ILO today: `POST /api/ilo_bot`).
- **Header:** same **Bearer access token** as for `general_bot`.
- **Body:** align with the latest **general_bot** request where possible, at least:
  - `courseInfo` (required)
  - Optional: `disciplinaryPractices`, `pedagogicalApproaches`, `intendedLearningOutcomes`, `lessons`
  - Optional: `referrer_pathname`, `form_state`
  - Optional: `request_type` — `initial` (default) or `reload` when the user asks for a new batch of suggestions
  - Optional: `reload_metadata.original_suggestions` — when `request_type` is `reload`, pass the **last three** suggestions previously shown (same shape as response items: `statement`, `type_id`, `bloom_taxonomy_level_id`) so the Agent can steer toward different wording. CamelCase variants (`requestType`, `reloadMetadata`, `originalSuggestions`, etc.) are accepted by the Agent.
- **Response:** reuse existing specialist response structures (ILO: current `show_suggestion` list shape).

---

## 6. Relationship to public specifications

- Overall message shape should stay aligned with **[LDS External AI Agent — overall JSON structure](https://hkucite.github.io/lds-external-ai-agent/docs/overall-json-structure)**.
- `open_specialist_bot` (generic) and `open_ilo_bot` (compatibility alias) are project-level proposals; if HKU CITE central specs use different names/fields, revise this doc and OpenAPI after alignment.

---

## 7. Revision history

| Date | Change |
|------|--------|
| 2026-03-25 | First draft: contract, examples, responsibilities (documentation only; backend emission tracked separately) |
| 2026-03-25 | Copied into ai-agent docs site (keep in sync with LDS-Chatbot repo) |
| 2026-03-25 | Narrative translated to English; sample `button_label_zh` values retained as optional i18n examples |
| 2026-03-27 | Added decision rule (`show_suggestion` vs handoff); introduced generic `open_specialist_bot` schema; retained `open_ilo_bot` compatibility and added MC example |
| 2026-03-30 | Documented `request_type` / `reload_metadata` on `POST /api/ilo_bot` for reload flows (see LDS-Chatbot `docs/openapi.json` → `ilo_request`) |
