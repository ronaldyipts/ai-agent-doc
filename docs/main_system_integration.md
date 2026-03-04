---
sidebar_position: 6
---

# 主系統可呼叫本子系統的功能

主系統（LDS）持有本 AI Agent 子系統的 **API endpoint、Client ID、Client Secret、Access Key** 後，可透過以下 API 觸發本子系統執行各項功能。主系統需先取得 **access token**（見 [API Testing with Postman](./postman.md)），再以 `Authorization: Bearer <access_token>` 呼叫下列 endpoint。

---

## LDS 專用端點（建議優先使用）

與 LDS 主系統整合時，**建議優先使用**以下兩個 LDS-compatible 端點。Request body 需傳入 **`courseInfo`**，可選 **`referrer_pathname`**、**`form_state`** 及學習設計相關陣列，Agent 會據此提供情境化回覆。

### POST `/api/general_bot` — 一般對話

- **用途**：一般學習設計對話、引導、建議。
- **必填**：`message`, `courseInfo`
- **可選**：`referrer_pathname`, `form_state`, `disciplinaryPractices`, `pedagogicalApproaches`, `intendedLearningOutcomes`, `lessons`
- **回傳**：`{ chat_message_reply: { text }, actions: [...] }`

當 LDS 在表單頁（例如 Course Information、ILO 編輯頁）觸發請求時，可傳入 **`form_state`**（與前端表單同結構），Agent 會用來組 context；必要時可透過 LDS Options API 將 ID 解析為可讀名稱。

### POST `/api/ilo_bot` — 生成 ILO 建議

- **用途**：直接取得 AI 建議的 Intended Learning Outcomes（按鈕觸發「AI 建議 ILO」時使用）。
- **必填**：`courseInfo`
- **可選**：`referrer_pathname`, `form_state`, `disciplinaryPractices`, `pedagogicalApproaches`, `intendedLearningOutcomes`, `lessons`
- **回傳**：`{ chat_message_reply: { text }, actions: [ show_suggestion, ... ] }`

#### 按鈕觸發「AI 建議 ILO」流程（主系統 → 子系統）

1. **主系統 UI**：在課程／單元編輯頁加按鈕（例如「由 AI 建議 ILO」）。
2. **撳按鈕後**：主系統以 access token 呼叫 **POST** `{子系統 base URL}/api/ilo_bot`，body 含 `courseInfo`（及可選 `form_state`, `referrer_pathname`）。
3. **本子系統**：回傳 ILO 建議（`chat_message_reply` + `actions`）。
4. **主系統**：將回傳內容顯示在彈窗／側欄／表單，供教師揀選或編輯後儲存。

---

## 其他 API（仍可使用）

以下端點仍可使用；**LDS 整合建議以 `/api/general_bot` 與 `/api/ilo_bot` 為主**。

### 1. 一般對話與學習成果

| 功能 | 方法 | Endpoint | 說明 |
|------|------|----------|------|
| **對話 / 問答** | POST | `/api/chat` | 傳入 `message`、可選 `conversation_id`。LDS 整合建議使用 `/api/general_bot`。 |
| **生成 ILO** | POST | `/api/generate_ilos` | 依 `topic`、`level`、`language` 生成 ILO。LDS 整合建議使用 `/api/ilo_bot`。 |
| **建議學科實踐（DP）** | POST | `/api/suggest_dp` | 依 `topic`、`context` 建議 Disciplinary Practices。 |

### 2. 文件分析與課程資訊

| 功能 | 方法 | Endpoint | 說明 |
|------|------|----------|------|
| **分析教學文件** | POST | `/api/analyze-document` | 上傳檔案（PDF、DOCX、TXT），分析教學文件內容。 |
| **擷取課程資訊** | POST | `/api/extract-course-info` | 上傳檔案，擷取課程相關資訊。 |

### 3. 對話紀錄

| 功能 | 方法 | Endpoint | 說明 |
|------|------|----------|------|
| **儲存對話紀錄** | POST | `/api/save-conversation` | 傳入 `conversation_id`、`messages`，儲存對話歷史。 |

### 4. 系統與文件

| 功能 | 方法 | Endpoint | 說明 |
|------|------|----------|------|
| **簡易健康檢查** | GET | `/` | 確認子系統是否在線。 |
| **詳細健康檢查** | GET | `/api/health` | 取得子系統健康狀態細節。 |
| **Swagger 文件** | GET | `/docs` | 互動式 API 文件（Swagger UI）。 |
| **OpenAPI 規格** | GET | `/api/openapi.json` | OpenAPI (Swagger) JSON。 |

---

## 小結

- **LDS 主系統**：優先使用 **POST `/api/general_bot`**（一般對話）與 **POST `/api/ilo_bot`**（ILO 建議），並在 body 傳入 **`courseInfo`**、可選 **`referrer_pathname`**、**`form_state`** 及學習設計陣列。
- **form_state**：若在表單頁觸發請求，可傳入與前端表單同結構的 `form_state`，Agent 會用來組 context，並可選用 LDS Options API 解析 ID 為可讀名稱。
- 其餘 endpoint（`/api/chat`、`/api/generate_ilos` 等）仍可使用，詳見 [Application Architecture](./app_archi.md) 與 [Postman 集合](./postman.md)。
