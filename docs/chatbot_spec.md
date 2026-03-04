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
- High Scaffolding (User response: less than 10 characters)
    - Provides very detailed guidance
    - Offers multiple specific options
    - Includes examples and step-by-step instructions
  - Medium Scaffolding (User response: 10-30 characters)
    - Provides moderate guidance
    - Offers 2-3 options
    - Brief explanations
- Low Scaffolding (more than 30 characters)
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
If a query is out of scope, the chatbot returns:
```
{
  "chat_message_reply": {
    "text": "抱歉，我專門協助學習設計和課程規劃。請詢問與您的課程相關的問題。"
  },
  "actions": []
}
```

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

Example
```
{
  "suggested_questions": [
    "我想了解如何應用 Bloom's Taxonomy",
    "我的目標是設計有效的評量方式",
    "我考慮使用項目式學習方法"
  ]
}
```

## 4.5 RAG 與檢索
Agent 使用 **RAG（Retrieval-Augmented Generation）** 提升回覆品質，檢索來源包括 LDS 用戶指南、iLAP 用戶指南等。

- **Clustering（分桶）**：資料按類型分桶（buckets），例如 ILO、DP（Disciplinary Practices）、PA（Pedagogical Approaches）、assessment、activity、general、ilap 等。
- **檢索策略**：依請求類型或訊息推斷，檢索對應類型內容，再組合成 context 供生成使用。

## 4.6 LDS 上下文
當 LDS 主系統傳入 **`courseInfo`**、**`referrer_pathname`**、**`form_state`** 時（例如透過 `/api/general_bot` 或 `/api/ilo_bot`），Agent 會將這些納入生成 context，使回覆與用戶當前編輯的課程／表單一致。

- **courseInfo**：課程相關資訊，供 Agent 理解當前課程情境。
- **referrer_pathname**：用戶所在頁面路徑，可協助判斷當前編輯階段。
- **form_state**：與 LDS 前端表單同結構的狀態；Agent 可選用 **LDS REST API for Chatbot**（Options API）將 form_state 中的 ID（如 grade_level_id, subject_ids）解析為可讀名稱，再納入 context。

## 4.7 Conversation History Management
- Recent history: only last 10 messages included
- Format: `{role: "user"|"assistant", content: "..."}`
- Filtering: Empty messages excluded
- Context preservation: Maintains conversation flow for better responses
