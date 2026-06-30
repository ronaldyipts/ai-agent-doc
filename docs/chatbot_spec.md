---
sidebar_position: 4
---

# Chapter 4: Learning Design Facilitator (LDF) Specifications

## 4.1 Socratic Nudging Engine
The Learning Design Facilitator employs a Socratic Nudging Engine that adapts its teaching style based on user responses:

- Guided Questioning
  - Instead of direct answers, asks thought-provoking questions
- Dynamic Scaffolding
  - Adjusts support level based on user response length
  - High Scaffolding (User response: under 10 characters)
    - Provides very detailed guidance
    - Offers multiple specific options
    - Includes examples and step-by-step instructions
  - Medium Scaffolding (User response: 10-30 characters)
    - Provides moderate guidance
    - Offers 2-3 options
    - Brief explanations
  - Low Scaffolding (over 30 characters)
    - Provides concise guidance
    - Asks 1-2 key questions
    - Minimal prompts
- Mode Detection
  - The system automatically detects when to switch modes:
    - Guiding Mode (Default)
      - User has not provided enough details
      - Asks question to help user think
      - Encourages reflection
    - Suggested Mode (Automatic)
      - User has provided sufficient information (50+ chars with keywords OR 3+ conversation rounds)
      - Directly provides specific ILO suggestions
      - Offers examples and templates
- Direct Answer Override
  - Users can explicitly request direct answers using keywords:
    - 不要問了 (Don't ask anymore)
    - 直接給答案 (Give direct answer)
    - 直接寫 (Write directly)
    - 不要引導 (Don't guide)
    - 直接回答 (Answer directly)
    - 別問了 (Stop asking)
    - 直接提供 (Provide directly)

## 4.2 Scope Detection
The Learning Design Facilitator only responds to learning design-related queries.

In Scope Keywords
- English
  - learning, learn, teaching, curriculum, lesson, assessment, rubric
  - bloom, taxonomy, ilo, learning outcome, pedagogy, instruction
  - pattern, suggestion, guide, choice, navigation
- Chinese
  - 課程, 教學, 學習, 學習目標, 評量, 教案, 布魯姆, 課綱
  - 單元, 教材, 設計, 教學設計, 課程設計, 教育
  - 模式, 建議, 指引, 選項, 導航

Out of Scope Handling
If a query is out of scope, the Learning Design Facilitator returns a message such as the following (locale may vary; Traditional Chinese is common for HK deployments):
```
{
  "chat_message_reply": {
    "text": "抱歉，我專門協助學習設計和課程規劃。請詢問與您的課程相關的問題。"
  },
  "actions": []
}
```
**English equivalent (reference):** “Sorry, I specialize in learning design and course planning. Please ask questions related to your course.”

Special Handling
- Greetings: Always accepted (你好, hello, hi, etc.)
- Quick Actions: Always accepted (請顯示, please show, etc.)
- Bot-Suggested Questions: Always accepted (bypasses scope check)

## 4.3 Quick Action Detection
The Learning Design Facilitator recognizes quick action requests and responds with appropriate UI actions.

Supported Actions
- show_pattern - display ILO patterns
- show_suggestion - show suggestions
- guide_user - provide guidance
- show_choice - display options
- show_navigation - show navigation menu
- open_specialist_bot - hand off to a specialist bot (e.g. ILO Bot); **primary** handoff action in LDS-Chatbot
- open_ilo_bot - legacy alias for ILO specialist handoff (same UX intent as `open_specialist_bot`)

Trigger Phrases
- 請指引, 請顯示, 請查看, 請提供, 請獲取
- please guide, please show, please view, please provide
- 顯示模式, 顯示建議, 顯示選項, 顯示導航
- show pattern, show suggestion, show choice, show navigation

## 4.4 Suggested Question Generation
After each response, the Learning Design Facilitator may generate 3 suggested follow-up questions when `ENABLE_SUGGESTED_QUESTIONS` is enabled.

**LDS integration (`POST /api/general_bot`):** suggested questions are **disabled** on the LDS code path (`skip_suggested_questions=true`). LDS clients should not expect `suggested_questions` in the response.

Generation Criteria
- Based on conversation history (last 6 messages)
- Analyzes bot response type (guiding vs. suggesting)
- Presented in "user speaking to bot" format
- Each suggestion is 25 characters or fewer
- Specific and actionable

Example (English UI; Traditional Chinese variants are allowed when the app locale is zh-HK)
```
{
  "suggested_questions": [
    "How can I apply Bloom's Taxonomy here?",
    "I want to design effective assessments",
    "I'm considering project-based learning"
  ]
}
```

## 4.5 RAG and Pre-Classification

Retrieval-augmented generation is implemented in LDS-Chatbot (`api/rag_common.py`, `api/ilap_rag.py`). Index file: `rag/ilap_index.json` (override with `ILAP_INDEX_PATH`).

| Setting | Default | Behavior |
|---------|---------|----------|
| `RAG_PRE_CLASSIFY` | `true` | Lightweight LLM classifies the question as **LDS**, **iLAP**, or **both** before bucket retrieval |
| `RAG_PRE_CLASSIFY=false` | — | Keyword-only routing (`_classify_source_keyword`); skips one LLM call |
| `RAG_MIN_SCORE` | env | Minimum cosine similarity for a chunk to be included |
| `RAG_BUCKET_MODE` | `single` | `single` or multi-bucket retrieval strategy |
| `RAG_MAX_BUCKETS`, `RAG_TOP_K_TOTAL`, `RAG_PER_BUCKET_LIMIT` | optional | Tune how many buckets/chunks are retrieved |

### RAG buckets (8)

| Bucket | Typical use |
|--------|-------------|
| `ILO` | Intended learning outcomes |
| `DP` | Disciplinary practices |
| `PA` | Pedagogical approaches |
| `assessment` | Assessment design |
| `activity` | Learning activities |
| `Theory` | Learning theory |
| `general` | General learning-design guidance |
| `ilap` | iLAP programme content |

Pre-classification and `infer_request_type()` choose the primary bucket before retrieval. `get_rag_context_for_prompt()` injects matched chunks into `general_bot` and `ilo_bot` prompts (with per-bucket char caps via `RAG_*_MAX_CHARS`).

Standalone RAG endpoint: **`POST /api/rag/ilap/answer`** (documented in OpenAPI). Ops health: `GET /api/rag/ilap/status` (not in OpenAPI).

Retrieval-count experiments (optimal `RAG_TOP_K_TOTAL` / bucket mix) remain **TBD**; see [Chapter 9](./action_items_progress.md).

## 4.6 Conversation History Management
- Recent history: only last 10 messages included
- Format: `{role: "user"|"assistant", content: "..."}`
- Filtering: Empty messages excluded
- Context preservation: Maintains conversation flow for better responses

## 4.7 ILO Reload Diversity and Output Rules

This section documents the current behavior of `POST /api/ilo_bot`.

- Output count is fixed to **3 ILO suggestions**.
- Suggestion schema includes:
  - `statement` (string)
  - `type_id` (int)
  - `bloom_taxonomy_level_id` (int)
- Backend removes metadata suffixes from `statement` such as:
  - `(Level: ...)`
  - `(Type: ...)`
  - `(Bt Level: ...)`
  These fields must be represented by IDs, not embedded inside statement text.

Reload-specific behavior (`request_type = reload`):

- Uses `reload_metadata.original_suggestions` (if provided) as prior batch context.
- Enforces strict no-repeat on exact statement keys in recent runtime window.
- Enforces strict no-repeat on leading action verbs in recent runtime window.
- Enforces task-type diversification (avoid repeating recent task types and in-batch duplicates).
- Applies semantic dedup and rewrite when suggestions are too similar.
- Prefers diverse `type_id` and `bloom_taxonomy_level_id` across the batch.

Runtime window and tuning:

- Diversity memory is in-process runtime memory, scoped by user+course+referrer key.
- Current window length is about last 5 batches (max 15 suggestions).
- Tune with environment variables:
  - `ILO_RELOAD_SEMANTIC_THRESHOLD`
  - `ILO_RELOAD_GENERATION_TEMPERATURE`
  - `ILO_RELOAD_REWRITE_TEMPERATURE`
  - `ILO_RELOAD_REWRITE_MAX_ATTEMPTS`

## 4.8 LDS context (general_bot / ilo_bot)

When LDS calls **`POST /api/general_bot`** or **`POST /api/ilo_bot`**, the Agent uses request context to align replies with the course and form the user is editing.

| Field | Role |
|-------|------|
| `courseInfo` | Course topic, grade, subjects, and related metadata |
| `referrer_pathname` | Page path (e.g. ILO edit URL); used to infer `context_object_id` |
| `form_state` | Current form values (statement, `type_id`, `bloom_taxonomy_level_id`, …) |
| `conversation_history` | Multi-turn chat (general_bot); aliases accepted |
| `locale` | `zh_HK` or `en_US` for reply language |

IDs in `form_state` may be resolved via the LDS Options API (Appendix [Chapter 7](./lds_rest_api_for_chatbot.md)). See [Chapter 6: Main System Integration](./main_system_integration.md).

## 4.9 Specialist handoff (General Bot → ILO Bot)

When the user’s message is ILO-related and handoff is appropriate, **general_bot** may return **`open_specialist_bot`** (or legacy **`open_ilo_bot`**) so LDS can show a button; on click, LDS calls **`POST /api/ilo_bot`**. If the user only wants to **see** suggestions without writeback, prefer **`show_suggestion`** in the same response when applicable.

Full contract: [Chapter 8: LDS handoff](./lds_handoff_general_bot_open_ilo_bot.md).

## 4.10 Error handling and fallbacks

| Area | Behavior |
|------|----------|
| **Out of scope** | Safe reply text + empty `actions` (§4.2) |
| **ILO JSON parse failure** | Backend may return a minimal `show_suggestion` payload or retry generation; does not crash the API |
| **Async job failure** | `GET /api/jobs/{job_id}` returns `status: "failed"` and `error` string |
| **Invalid Bearer token** | `401` on protected `/api/*` routes |
| **Long conversations** | Last **10** messages sent to the model (`CHAT_HISTORY_MAX_MESSAGES`); no summarization yet — **TBD** |

Access and runtime errors are logged to `access_logs` (SQLite) and `LDS_RUNTIME_LOG_PATH` (file). See [Chapter 10](./deployment_operations.md).
