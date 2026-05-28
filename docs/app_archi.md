---
sidebar_position: 2
---

# Chapter 2: Application Architecture

## 2.1 Overall Architecture Pattern
- **This sub-system is a REST API Server (FastAPI)**.
- It does **not** provide an end-user frontend UI as part of the integration contract.
- The LDS main system (Laravel) is the caller and UI host; this Agent only exposes HTTP APIs.
- Optional admin components may exist for operations, but they are not part of the LDS-to-Agent runtime API contract.

## 2.2 Main Application Architecture

### 2.2.1 REST API Server Architecture
Tech Stack
- FastAPI (Python Web Framework)
- Uvicorn (ASGI Server)
- Pydantic (Data Validation)
- Azure OpenAI (AI Service)
- SQLAlchemy (Optional, used for OAuth)

Directory Structure
```
├── main.py          # FastAPI Main Application
├── api/
│   ├── utils.py     # Utility functions
│   └── auth.py      # Authentication module
├── run.py           # Startup script
└── requirements.txt # Python dependencies
```

Endpoint Catalog
- Chapter 2 focuses on architecture only (no endpoint list).
- Runtime integration endpoints are documented in [Chapter 6: Main System Integration](./main_system_integration.md).
- LDS Options/lookup API background is documented in [Chapter 7: LDS REST API for Chatbot (Appendix)](./lds_rest_api_for_chatbot.md).
- Testable endpoint examples are documented in [Chapter 5: API Testing with Postman](./postman.md).

Integration Ports and Links (for LDS caller)

| Item | Default Port | Example Link |
|------|--------------|--------------|
| API public entry (via reverse proxy) | 80/443 | `http(s)://<agent-host>/...` |
| FastAPI app (internal process) | 5000 (default deploy) | `http://127.0.0.1:5000/docs` |
| Admin backend (optional operations only) | 5001 (default deploy) | `http://127.0.0.1:5001/...` |
| Docs site (optional static site) | 3000 (local preview) | `http://127.0.0.1:3000/` |

AI Integration Layer (api/utils.py)
- call_openai() - Calls Azure OpenAI API
- run_chat_with_optional_tools() - Chat supporting tool calls
- validate_response_format() - Validates JSON response format
- generate_suggested_questions() - Generates follow-up questions

LDS API Integration Layer
- make_lds_request() - LDS API Request (supports multiple auth formats)
- call_lds_api() - Call LDS API by name
- build_lds_headers() - Build request headers (Bearer/Token/X-API-Key)

Response Format Validation
- CHATBOT_SCHEMA - JSON Schema definition
- validate_response_format() - Validation function
- Double validation: forced format during API call + validation after response

### 2.2.2 Core Business Logic
See Chapter 4 for chatbot logic details.

### 2.2.3 Data Flow (LDS ↔ Agent API)
1. LDS obtains access token.
2. LDS calls the runtime integration APIs (see Chapter 6).
3. Agent validates request (Pydantic).
4. Agent composes context from `courseInfo`, `form_state`, and optional LDS options lookup.
5. Agent calls Azure OpenAI and post-processes output.
6. Agent returns LDS-compatible JSON (`chat_message_reply` + `actions`).
7. LDS renders the response in its own UI.

## 2.3 Optional Admin Operations API (non-runtime)

Security positioning:
- The External Agent should be treated as an API-only service.
- Avoid exposing additional public login pages because they increase attack surface.
- If a minimal admin access gate is required at server edge, prefer basic server-side protection (e.g. **`.htaccess` prompt password** or equivalent reverse-proxy auth) before exposing any admin route.

### 2.3.1 Backend Architecture
Tech Stack
- FastAPI
- SQLAlchemy (ORM)
- SQLite (Default Database)
- bcrypt (Password Hashing)

Database Models
- User - User table (username, password hash, role)
- OAuthClient - OAuth2 Clients
- AuthorizationCode - Auth Codes
- AccessToken - Access Tokens

Directory Structure (admin backend only)
`admin_portal/backend/` contains auth and client-management APIs for operational maintenance.

Admin Operations Reference
- Chapter 2 keeps architecture-only content.
- For operational auth/client endpoints used by maintainers, refer to the Admin Portal OpenAPI/Swagger in the deployment environment.
- Runtime LDS integration API details are documented in [Chapter 6](./main_system_integration.md).

**Note**: A default admin is no longer created automatically; the first admin must be created via an existing admin or database/bootstrap. Public self-registration is disabled.
