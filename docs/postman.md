---
sidebar_position: 5
---

# Chapter 5: API Testing with Postman

A curated **Postman Collection** is provided for streamlined testing. **Import only one JSON file** (the collection). Set **username** and **password** (the AI agent's Admin Portal credentials, not LDS) in the collection variables, run **1. Retrieve Access Token**, then call any API.

**Auth 說明**：Admin Portal **不再提供公開註冊**；帳號須由既有 admin 透過 **POST /api/auth/users** 建立（或首次部署時以其他方式建立首個 admin）。取得 token 前請先由管理員建立帳號。

## Workflow (4 steps)

1. **Import the collection**
   - Open Postman → **Import** → select or drag the collection file, or use the link below.
   - **Download:** [**LDS-AI-Agent.postman_collection.json**](https://ronaldyipts.github.io/ai-agent-doc/postman/LDS-AI-Agent.postman_collection.json) (click to download, then Import in Postman).
   - No separate environment file needed; variables are built into the collection.

2. **Set variables**
   - Click the collection **LDS External AI Agent** → **Variables** tab.
   - Set **`username`** and **`password`** (the **AI agent's Admin Portal** username and password, not LDS credentials).
   - Set **`client_id`** and **`client_secret`** (from the AI agent's Admin Portal).
   - **`scope`** is pre-set to `chatbot`; **`base_url`** to `http://ronald-test.cite.hku.hk`. Adjust if needed. Save.

3. **Retrieve access token**
   - Open **0 - Auth** → send **1. Retrieve Access Token**.
   - On success, `access_token` (and `refresh_token` if returned) is saved automatically into the collection variables.

4. **Call the APIs**
   - All requests in **2 - Core** (and **Me** / **Logout** in Auth) use the saved Bearer token.
   - **LDS 整合建議**：使用 **General Bot**、**ILO Bot**（body 含 `courseInfo`、可選 `referrer_pathname`、`form_state`）。
   - 其餘：Chat, Generate ILOs, Suggest Disciplinary Practices, Analyze Document, Extract Course Info, Save Conversation。

## File

| File | Purpose |
|------|--------|
| [**LDS-AI-Agent.postman_collection.json**](https://ronaldyipts.github.io/ai-agent-doc/postman/LDS-AI-Agent.postman_collection.json) | All API requests (Auth, System, Core) and built-in variables. **1. Retrieve Access Token** sends **x-www-form-urlencoded** body: `grant_type`, `client_id`, `client_secret`, `scope`, `username`, `password`. Click the link to download, then Import in Postman. |

## Collection structure

- **0 - Auth**: **1. Retrieve Access Token** (form body: grant_type, client_id, client_secret, scope, username, password → saves token), OAuth2 code exchange, Refresh Token, Me, Logout. **無公開註冊**；帳號由 admin 建立（POST /api/auth/users）。
- **1 - System**: Health check (root and `/api/health`).
- **2 - Core**: **General Bot**、**ILO Bot**（LDS 建議優先使用；body 含 `message`/`courseInfo`、可選 `referrer_pathname`、`form_state`）、Chat, Generate ILOs, Suggest Disciplinary Practices, Analyze Document, Extract Course Info, Save Conversation.

### LDS 專用端點（General Bot / ILO Bot）

| 請求 | Endpoint | Body 必填 | Body 可選 |
|------|----------|-----------|-----------|
| **General Bot** | POST `/api/general_bot` | `message`, `courseInfo` | `referrer_pathname`, `form_state`, `disciplinaryPractices`, `pedagogicalApproaches`, `intendedLearningOutcomes`, `lessons` |
| **ILO Bot** | POST `/api/ilo_bot` | `courseInfo` | `referrer_pathname`, `form_state`, 同上學習設計陣列 |

回傳格式：`{ chat_message_reply: { text }, actions: [...] }`。若 collection 尚未包含此兩支，可參考 [主系統可呼叫本子系統的功能](./main_system_integration.md) 自行新增。

## Collection variables

Edit the collection → **Variables** tab. No separate environment file needed.

| Variable | Required for token flow | Description |
|----------|--------------------------|-------------|
| `base_url` | Yes | API base URL. Default: `http://ronald-test.cite.hku.hk` (deployed); use `http://localhost:5000` for local. |
| `username` | Yes | AI agent's Admin Portal username (not LDS) |
| `password` | Yes | AI agent's Admin Portal password (not LDS) |
| `client_id` | Yes | OAuth2 client ID (for token request) |
| `client_secret` | Yes | OAuth2 client secret (for token request) |
| `scope` | Yes | Pre-set to `chatbot` |
| `access_token` | — | Set automatically by **1. Retrieve Access Token** (Bearer token) |
| `refresh_token` | — | Set automatically by **1. Retrieve Access Token** |
| `conversation_id` | — | Optional; for Chat / Save Conversation |
| `auth_code`, `redirect_uri` | No | For OAuth2 authorization-code flow only |

## Troubleshooting: 422 "username / password required"

If **1. Retrieve Access Token** returns `422 Unprocessable Entity` with "Field required" for username and password:

1. **Use the Body tab (not Params)**  
   Click the **Body** tab for **1. Retrieve Access Token**. Ensure **x-www-form-urlencoded** is selected and the table includes: `grant_type`, `client_id`, `client_secret`, `scope`, `username`, `password`. If Body is set to "none", the API returns 422. Re-import the collection to restore the default body.

2. **Set the *Current value* for username and password**  
   In the Variables table, Postman has two columns: **Initial value** (for sharing) and **Current value** (used when sending the request). The request uses **Current value**. If you only filled Initial value, Current value can be blank and the server will get empty credentials → 422.  
   Fill the **Current value** column for both `username` and `password`, then Save and send again.

3. **Environment vs collection**  
   If an **environment** is selected (top-right), its variables override the collection. Either:
   - Choose **No Environment** so the collection variables are used, or  
   - Open that environment → **Variables** and set **username** and **password** in the **Current value** column there as well.

4. **Re-import the collection**  
   If the Body was never set correctly, re-import **`LDS-AI-Agent.postman_collection.json`** (replace existing) and set username/password in the collection Variables again.

For file uploads (**Analyze Document**, **Extract Course Info**), choose a file in the request **Body** → **form-data** in Postman.
