---
sidebar_position: 1
---

# Chapter 1: Introduction

This specification defines the rationale of designing the LDS External AI agent.

**Product name:** The formal name of this system is **Learning Design Facilitator** (**LDF**). In documentation below, *LDF* refers to the product; *LDS-Chatbot* refers to the implementation repository and deployment artifacts (service names, env files, database filenames) where historical naming is unchanged.

The External Agent is a **REST API Server**. It is **not** presented as an end-user frontend application.

**Implementation repository:** The running Agent is built from the **LDS-Chatbot** project (`main.py`, `api/`, `docs/openapi.json`). This **ai-agent-doc** site describes integration and behavior for LDS teams; when in doubt, prefer LDS-Chatbot `docs/openapi.json` for request/response schemas.

**Deployed environments (current):**

| Environment | Host | Role |
|-------------|------|------|
| Production | `https://ideals-ldf.cite.hku.hk` | LDS integration target (HTTPS) |
| Testing | `http://ronald-test.cite.hku.hk` | UAT / development (HTTP) |

**LDS integration**: The main system (LDS) can send `courseInfo` and `form_state` via the **general_bot** and **ilo_bot** endpoints; the Agent uses them to provide contextual replies and ILO suggestions. See [Chapter 6: Main System Integration](./main_system_integration.md).

**Authentication**: LDS obtains tokens via **`POST /api/auth/token`** on the main host (not Admin Portal). Admin Portal (`/admin/`) is for operators and supports email OTP 2FA when enabled. See [Chapter 3](./login_page.md) and [Chapter 10: Deployment and Operations](./deployment_operations.md).

**First-time server setup:** see `LDS-Chatbot/deploy/RUNBOOK_DEPLOY.md` (summarized in [Chapter 10 §10.9](./deployment_operations.md#109-initial-deployment)).

**Response mode:** `BOT_RESPONSE_MODE` in `/etc/lds-chatbot.env` — `sync` (default, HTTP 200) or `async` (HTTP 202 + job polling for `general_bot` only). Confirm with your deployment before LDS implements polling.

**Appendix note**: LDS options/patterns lookup API details are intentionally placed in [Chapter 7 (Appendix)](./lds_rest_api_for_chatbot.md), so the main reading flow starts from integration endpoints first.

*Spec last aligned with LDS-Chatbot: Admin Portal all-user 2FA, async `general_bot` jobs, RAG pre-classification, automated backups, production Postman collection.*