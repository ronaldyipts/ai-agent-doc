---
sidebar_position: 4
---

# Chapter 4: Chatbot Specifications

## 4.1 Socratic Nudging Engine
The chatbot employs a Socratic Nudging Engine that adapts its teaching style based on user responses:

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
The chatbot only responds to learning design-related queries.

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
If a query is out of scope, the chatbot returns a message such as the following (locale may vary; Traditional Chinese is common for HK deployments):
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
The chatbot recognizes quick action requests and responds with appropriate UI actions.

Supported Actions
- show_pattern - display ILO patterns
- show_suggestion - show suggestions
- guide_user - provide guidance
- show_choice - display options
- show_navigation - show navigation menu

Trigger Phrases
- 請指引, 請顯示, 請查看, 請提供, 請獲取
- please guide, please show, please view, please provide
- 顯示模式, 顯示建議, 顯示選項, 顯示導航
- show pattern, show suggestion, show choice, show navigation

## 4.4 Suggested Question Generation
After each response, the chatbot generates 3 suggested follow-up questions.

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

## 4.5 Conversation History Management
- Recent history: only last 10 messages included
- Format: `{role: "user"|"assistant", content: "..."}`
- Filtering: Empty messages excluded
- Context preservation: Maintains conversation flow for better responses

## 4.6 ILO Reload Diversity and Output Rules

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
