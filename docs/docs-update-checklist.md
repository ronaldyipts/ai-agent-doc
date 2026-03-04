# Docusaurus 文檔更新清單（對應 LDS-Chatbot 實作）

以下根據 LDS-Chatbot 專案現況，列出 ai-agent-doc 內需要更新或新增的內容。

---

## 1. `main_system_integration.md`（主系統可呼叫本子系統的功能）

**建議更新：**

- **LDS 專用 endpoint**：主系統（LDS）應優先使用以下兩個 LDS-compatible 端點，並在 body 傳入 `courseInfo`、可選 `referrer_pathname`、`form_state`：
  - **POST `/api/general_bot`**  
    一般對話。必填：`message`, `courseInfo`；可選：`referrer_pathname`, `form_state`, `disciplinaryPractices`, `pedagogicalApproaches`, `intendedLearningOutcomes`, `lessons`。  
    回傳：`{ chat_message_reply: { text }, actions: [...] }`。
  - **POST `/api/ilo_bot`**  
    生成 ILO 建議。必填：`courseInfo`；可選：`referrer_pathname`, `form_state`, 同上 LD 陣列。  
    回傳：`{ chat_message_reply: { text }, actions: [ show_suggestion, ... ] }`。
- **form_state**：若 LDS 在表單頁（例如 Course Information、ILO 編輯頁）觸發請求，可傳入 `form_state`（與前端表單同結構），Agent 會用來組 context，並可選用 LDS Options API 解析 ID 為可讀名稱。
- **原有 endpoint**：`/api/chat`、`/api/generate_ilos` 等仍可保留說明，但註明「LDS 整合建議使用 `/api/general_bot` 與 `/api/ilo_bot`」。

---

## 2. `app_archi.md`（Application Architecture）

**建議更新：**

- **Main Application API**：  
  - 補充 **POST `/api/general_bot`**、**POST `/api/ilo_bot`**（LDS-compatible，request body 含 `courseInfo`、可選 `form_state`、`referrer_pathname`）。  
  - 補充 **GET `/docs`**、**GET `/api/openapi.json`**（Swagger / OpenAPI）。
- **Admin Portal**：  
  - **移除或修正**「POST /api/auth/register - Register」：公開註冊已停用（回傳 403）。  
  - **新增**「POST /api/auth/users - Create user（Admin only）」：需 Bearer token 且當前用戶為 admin；body 含 `username`, `password`, `email`, `full_name`, 可選 `is_admin`。  
  - 說明：**不再自動建立 default admin**；首次使用需透過既有 admin 或資料庫/bootstrap 建立首個 admin。
- **Admin Portal Frontend**：  
  - Login 頁面：僅登入，**無**「Sign Up / Register」；無預設 admin 帳密提示。  
  - 建立新用戶：需由 admin 登入後透過 API（如 Postman）呼叫 `POST /api/auth/users`，或日後在 Admin Portal 加「User Management」頁面。

---

## 3. `login_page.md`（Chapter 3: Login Page）

**建議釐清與更新：**

- 標題或開頭註明是 **主應用（LDS Chatbot）登入頁** 還是 **Admin Portal 登入頁**；兩者 endpoint 不同（主應用可能為 `/api/auth/login`，Admin Portal 為 **POST `/api/auth/token`**，form body: username, password）。
- 若本章描述的是 **Admin Portal**：  
  - 移除「Sign Up: Navigate to registration page」（已無公開註冊）。  
  - 移除任何「預設 admin 帳密」說明。  
  - 說明帳號由管理員透過 **POST /api/auth/users** 建立。
- 若本章描述的是 **主應用**：保持與主應用實作一致（endpoint、儲存 token 方式等）。

---

## 4. `chatbot_spec.md`（Chapter 4: Chatbot Specifications）

**建議補充（可為新小節）：**

- **RAG 與檢索**  
  - Agent 使用 RAG 提升回覆品質；來源包括 LDS 用戶指南、iLAP 用戶指南等。  
  - 使用 **clustering（buckets）**：資料按類型分桶（如 ILO, DP, PA, assessment, activity, general, ilap），依請求類型（或訊息推斷）檢索對應類型內容。
- **LDS 上下文**  
  - 當 LDS 傳入 `courseInfo`、`referrer_pathname`、`form_state` 時，Agent 會將這些納入生成 context，使回覆與用戶當前編輯的課程/表單一致。

---

## 5. `postman.md`（API Testing with Postman）

**建議更新：**

- **Auth 說明**：  
  - 註明 **Admin Portal 不再提供公開註冊**；帳號須由既有 admin 透過 **POST /api/auth/users** 建立（或首次部署時以其他方式建立首個 admin）。  
  - 若 Postman collection 內有「Register」請求，建議標註為 deprecated 或移除，改為說明「取得 token 前請先由管理員建立帳號」。
- **Core 請求**：  
  - 若有 **general_bot**、**ilo_bot** 請求，補充說明 body 欄位（`message`, `courseInfo`, `referrer_pathname`, `form_state`）。  
  - 若 collection 尚未包含這兩支，建議加入範例請求並在文檔中列出。

---

## 6. 可選：新增一頁「LDS REST API for Chatbot」

- 可簡短說明：Agent 必要時會呼叫 **LDS REST API for Chatbot**（Options API）以解析 form_state 中的 ID（如 grade_level_id, subject_ids）。  
- 詳細端點與用法可連結至 LDS-Chatbot 專案內 **`docs/LDS-REST-API-for-Chatbot.md`**（或將該內容摘要複製到 Docusaurus）。

---

## 7. `intro.md`

- 若希望簡短提及「LDS 整合方式」，可加一兩句：主系統可透過 **general_bot / ilo_bot** 傳入 courseInfo 與 form_state，Agent 會據此提供情境化回覆與 ILO 建議。

---

## 總結表

| 檔案 | 建議動作 |
|------|----------|
| `main_system_integration.md` | 加入 general_bot / ilo_bot、courseInfo、form_state、referrer_pathname；註明 LDS 優先使用此二 endpoint |
| `app_archi.md` | 補充 general_bot、ilo_bot、/docs、openapi；Admin：移除公開 register、加入 POST /api/auth/users、取消 default admin 說明 |
| `login_page.md` | 釐清主應用 vs Admin Portal；若為 Admin：移除 Sign Up、預設帳密，改為說明由 admin 建帳號 |
| `chatbot_spec.md` | 新增 RAG + clustering、LDS 上下文（courseInfo / form_state）說明 |
| `postman.md` | Auth：說明無公開註冊、須由 admin 建帳號；Core：補充 general_bot / ilo_bot 與 body 欄位 |
| `intro.md` | 可選：一句提及 general_bot / ilo_bot 與 LDS 傳入資料 |
| 新頁（可選） | 「LDS REST API for Chatbot」簡介並連結 LDS-Chatbot 的 LDS-REST-API-for-Chatbot.md |

以上為建議清單；實際可依你發佈節奏分批更新。若需要，我可以直接幫你改某一個 .md 的具體段落內容。
