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

## 4.5 RAG and request-type clustering (including LDT / Theory)
The Agent uses **RAG (Retrieval-Augmented Generation)** to improve reply quality by injecting relevant reference content from the iLAP User Guide and the **Learning Design Studio (LDS) User Guide** into prompts. To retrieve the right kind of content for each request, the system uses **clustering**: reference content is grouped into **buckets** by topic, and the bucket used for retrieval is chosen from the user’s message (or from the endpoint, e.g. ILO bot).

### 4.5.1 RAG buckets
Reference chunks are assigned to one of the following buckets at index build time and used at query time:

| Bucket | Description | Example use |
|--------|-------------|-------------|
| **ILO** | Intended Learning Outcomes, Bloom’s taxonomy | “How do I write ILOs?” |
| **DP** | Disciplinary Practices | “Suggest a disciplinary practice” |
| **PA** | Pedagogical Approaches | “What pedagogical approach fits?” |
| **assessment** | Assessment, rubrics, evaluation | “Design an assessment” |
| **activity** | Learning activities, tasks | “Suggest an activity” |
| **Theory** | **LDT (Learning Design Theory)** and theoretical foundations | “What is the theory behind LDS?” |
| **general** | General guidance | Fallback when no other bucket matches |
| **ilap** | iLAP-specific content | Default for iLAP PDF chunks |

### 4.5.2 LDT chapter / Theory bucket
The **Theory** bucket corresponds to the **LDT (Learning Design Theory)** chapter and related theoretical content in the **LDS User Guide**. When users ask about learning design theory, pedagogical theory, or conceptual foundations, the Agent prefers chunks in this bucket so answers align with LDS documentation.

**Trigger keywords (Theory bucket)**  
User messages containing any of the following (case-insensitive) are mapped to the Theory bucket for retrieval:

- **English:** `theory`, `ldt`, `learning design theory`, `learning theory`, `constructivism`, `pedagogy theory`, `theoretical`, `theory behind`, `conceptual framework`
- **Chinese:** `理論`, `學習設計理論`, `教學理論`, `設計理論`, `建構主義`, `理論基礎`, `理論架構`

Example queries that use the Theory bucket: “What is the learning design theory behind LDS?”; “Explain the theoretical basis of the LDS framework.”; “學習設計理論是什麼？” / “LDT 章節講什麼？”

Chunks from the LDS User Guide that discuss LDT or these topics are tagged with the **Theory** bucket during index build and are then retrieved when the inferred request type is Theory.

### 4.5.3 Retrieval flow
1. **Index build**: PDFs (e.g. iLAP + LDS User Guide) are chunked, embedded, and each chunk is assigned a bucket (including **Theory** for LDT-related content).
2. **Request handling**: For each user message, the backend infers a request type (bucket) via keyword matching.
3. **Retrieval**: Chunks are retrieved with the inferred bucket preferred (same-bucket chunks are boosted; others may be down-weighted).
4. **Prompt**: The retrieved text is appended to the prompt as reference material so the model can cite the LDS User Guide (including the LDT chapter) and iLAP where appropriate.

The **ILO bot** endpoint (`/api/ilo_bot`) always uses the **ILO** bucket for RAG. The **general bot** (`/api/general_bot`) uses the inferred bucket (including **Theory** for LDT/theory questions).

## 4.6 LDS context
When the LDS main system sends **`courseInfo`**, **`referrer_pathname`**, and **`form_state`** (e.g. via `/api/general_bot` or `/api/ilo_bot`), the Agent includes them in the generation context so that replies align with the course/form the user is editing.

- **courseInfo**: Course-related information so the Agent understands the current course context.
- **referrer_pathname**: The page path the user is on, to help infer the current editing stage.
- **form_state**: State in the same structure as the LDS front-end form. The Agent may call the **LDS REST API for Chatbot** (Options API) to resolve IDs in form_state (e.g. grade_level_id, subject_ids) to human-readable names before including them in context.

## 4.7 Conversation History Management
- Recent history: only last 10 messages included
- Format: `{role: "user"|"assistant", content: "..."}`
- Filtering: Empty messages excluded
- Context preservation: Maintains conversation flow for better responses
