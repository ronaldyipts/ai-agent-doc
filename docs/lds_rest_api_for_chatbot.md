---
sidebar_position: 7
---

# LDS REST API for Chatbot

本 AI Agent 子系統在需要時會呼叫 **LDS REST API for Chatbot**（Options API），以解析 `form_state` 中的 ID（例如 `grade_level_id`、`subject_ids`）為可讀名稱，再納入生成 context。

- **用途**：當 LDS 傳入 `form_state`（與前端表單同結構）時，Agent 可透過此 API 將各類選項 ID 對應為顯示名稱，提升回覆的情境化與可讀性。
- **詳細端點與用法**：請參考 LDS-Chatbot 專案內 **`docs/LDS-REST-API-for-Chatbot.md`**（或主系統團隊提供的 Swagger / 文件）。本頁僅作簡介；完整規格以主系統文件為準。

相關章節：[主系統可呼叫本子系統的功能](./main_system_integration.md)、[Chatbot Specifications - LDS 上下文](./chatbot_spec.md#46-lds-上下文)。
