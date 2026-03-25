---
sidebar_position: 8
---

# LDS handoff: General Bot suggests opening ILO Bot (`open_ilo_bot` action)

**Status: DRAFT (proposal)**  
**Goal:** When a user in **General Bot** expresses intent related to **ILOs (Intended Learning Outcomes)**, the backend may include an **`open_ilo_bot`** entry in the `actions` array so **LDS UI** can show a button (or equivalent). After the user confirms, LDS calls **`POST /api/ilo_bot`**.

> **Important:** At the time of writing, the **Agent backend may not yet emit this action in production**. This document fixes the **JSON contract** and **division of work** so LDS can design UI and flows in parallel. Backend delivery can be scheduled separately (e.g. feature flag plus rules or model output).

**Mirrored copy (implementation repo):** Keep in sync with **`docs/LDS_HANDOFF_GENERAL_BOT_OPEN_ILO_BOT.md`** in the **LDS-Chatbot** project; OpenAPI examples are in that repo’s **`docs/openapi.json`** (`POST /general_bot` response examples).

---

## 1. Responsibilities

| Owner | Work |
|--------|------|
| **Agent / Chatbot backend** | Under agreed conditions, include **`action_type: "open_ilo_bot"`** in `actions` on **`POST /api/general_bot`** responses (with `payload` as below); keep **`POST /api/ilo_bot`** stable; provide test environment and examples after rollout. |
| **LDS (frontend / product)** | Parse `actions`; detect `open_ilo_bot`; render a button (labels from `payload` or your own i18n); on click, build the **`ilo_bot` request** aligned with the last **general_bot** call (`courseInfo`, `referrer_pathname`, `form_state`, etc.); call **`POST /api/ilo_bot`** and handle loading, errors, and tokens. |

---

## 2. JSON contract (`open_ilo_bot`)

### 2.1 Placement in `POST /api/general_bot` responses

Same overall shape as the existing LDS External AI Agent:

- Top level: `chat_message_reply.text` (normal reply text; may explain that the user can open ILO Bot below).
- Top level: `actions` (array; **multiple** entries allowed; `open_ilo_bot` may coexist with others; LDS decides priority).

### 2.2 Single action fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `action_type` | string | Yes | Must be **`"open_ilo_bot"`** |
| `target` | object | No | Align with `ilo_bot`: `context: "ILO"`, `context_object_id` = current ILO id being edited, or **0** if none |
| `payload` | object | Yes | See table below |
| `ui` | object | No | `presentation`: `inline` \| `sidebar` \| `popup` (suggest **`inline`** under the message); `highlight_target` optional |

#### Suggested `payload` fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `intent` | string | Yes | **`generate_ilos`** (batch ILO draft) \| **`refine_ilos`** (tune existing draft) \| **`explain_ilos`** (explain ILO / writing tips; still routes to ILO Bot for optional generation) |
| `trigger_reason` | string | No | For LDS logging / analytics (machine-readable, e.g. `user_asked_generate_ilo`) |
| `button_label_zh` | string | No | Default Traditional Chinese button label; LDS may override with your i18n key |
| `button_label_en` | string | No | Default English button label |

---

## 3. Example JSON

### Example A: User wants to generate ILOs (`open_ilo_bot` only)

```json
{
  "chat_message_reply": {
    "text": "You can use ILO Bot to draft several adjustable learning outcomes from your current course context. If you prefer, we can keep refining the topic and grade focus in this chat."
  },
  "actions": [
    {
      "action_type": "open_ilo_bot",
      "target": {
        "context": "ILO",
        "context_object_id": 0
      },
      "payload": {
        "intent": "generate_ilos",
        "trigger_reason": "user_asked_generate_ilo",
        "button_label_zh": "開啟 ILO Bot",
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

### Example B: Same response with another action (illustrative)

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
        "button_label_zh": "用 ILO Bot 優化此條 ILO",
        "button_label_en": "Refine with ILO Bot"
      },
      "ui": { "presentation": "inline", "highlight_target": "" }
    }
  ]
}
```

### Example C: Empty `actions` (still common today)

```json
{
  "chat_message_reply": {
    "text": "(Text-only reply; no UI action.)"
  },
  "actions": []
}
```

---

## 4. After the user clicks: call `ilo_bot`

- **Method / path:** `POST /api/ilo_bot` (same as existing docs).
- **Header:** same **Bearer access token** as for `general_bot`.
- **Body:** align with the latest **general_bot** request where possible, at least:
  - `courseInfo` (required)
  - Optional: `disciplinaryPractices`, `pedagogicalApproaches`, `intendedLearningOutcomes`, `lessons`
  - Optional: `referrer_pathname`, `form_state`
- **Response:** reuse the existing **`show_suggestion`** ILO list structure; LDS can reuse current ILO form integration.

---

## 5. Relationship to public specifications

- Overall message shape should stay aligned with **[LDS External AI Agent — overall JSON structure](https://hkucite.github.io/lds-external-ai-agent/docs/overall-json-structure)**.
- **`open_ilo_bot`** is a **proposed new `action_type`** for this project; if HKU CITE central specs use different names or fields, revise this doc and OpenAPI after alignment.

---

## 6. Revision history

| Date | Change |
|------|--------|
| 2026-03-25 | First draft: contract, examples, responsibilities (documentation only; backend emission tracked separately) |
| 2026-03-25 | Copied into ai-agent docs site (keep in sync with LDS-Chatbot repo) |
| 2026-03-25 | Narrative translated to English; sample `button_label_zh` values retained as optional i18n examples |
