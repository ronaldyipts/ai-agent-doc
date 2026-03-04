---
sidebar_position: 2
---

# Chapter 2: Application Architecture

## 2.1 Overall Architecture Pattern
- Adopts a decoupled frontend-backend architecture, consisting of two independent applications:
  - Main Application (LDS Chatbot): Learning Design Assistant
  - Admin Portal: OAuth2 Authorization Server

## 2.2 Main Application Architecture

### 2.2.1 Frontend Architecture
Tech Stack
- React 18.3.1
- Vite 5.4.8
- No external state management library (uses React Hooks)

Frontend Features
- Responsive design
- Real-time chat interface
- File upload (PDF, DOCX, TXT)
- Quick action buttons
- Conversation suggested question generation

Directory Structure
```
src/
├── App.jsx      # Main application component (contains all page logic)
├── main.jsx     # React entry file
└── styles.css   # Global styles
```

Key Functional Modules
- Chatbot Page
  - Conversation interface, file upload, course information input.
- ILO Generation Page
  - Form for generating learning objectives.
  - Multi-language support: Traditional Chinese (zh_HK) and English (en_US).
- Conversation History Management
  - localStorage persistence, export as .txt, clear conversation.

### 2.2.2 Backend Architecture
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

API Endpoints
- Authentication (6)
  - GET /api/auth/login - OAuth2 Login
  - GET /api/auth/callback - OAuth2 Callback
  - POST /api/auth/token - Get JWT token
  - POST /api/auth/refresh - Refresh token
  - GET /api/auth/me - Get current user info
  - POST /api/auth/logout - Logout
- LDS-compatible (2) — 主系統整合建議優先使用
  - POST /api/general_bot - 一般對話；body 含 courseInfo、可選 referrer_pathname、form_state、學習設計陣列
  - POST /api/ilo_bot - 生成 ILO 建議；body 含 courseInfo、可選 referrer_pathname、form_state、學習設計陣列
- Core Features (6)
  - POST /api/chat - General chat interface
  - POST /api/generate_ilos - Generate Intended Learning Outcomes
  - POST /api/suggest_dp - Suggest Disciplinary Practices
  - POST /api/analyze-document - Analyze teaching documents
  - POST /api/extract-course-info - Extract course info from docs
  - POST /api/save-conversation - Save conversation history
- System & Docs (4)
  - GET / - Health Check
  - GET /api/health - Detailed Health Check
  - GET /docs - Swagger UI (interactive API docs)
  - GET /api/openapi.json - OpenAPI (Swagger) specification

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

### 2.2.3 Core Business Logic
See Chapter 4 for chatbot logic details.

### 2.2.4 Data Flow
Client to server to AI or LDS integration
1. User enters text and clicks Send
2. POST /api/chat (request)
3. Vite proxy forwards request to localhost:5000 to avoid CORS
4. Backend validates request (Pydantic)
5. Send prompt for AI response
6. Return AI completion
7. Fetch external LDS data
8. Return LDS data
9. Response validation (JSON Schema)
10. Return JSON response
11. Frontend parses JSON and updates state
12. Render message bubble

## 2.3 Admin Portal Architecture

### 2.3.1 Frontend Architecture
Tech Stack
- React 18.3.1
- Vite 5.4.8
- Axios (HTTP Client)

Directory Structure
```
admin_portal/frontend/
├── src/
│   ├── App.jsx
│   ├── components/
│   │   ├── Login.jsx        # Login only (no Sign Up / Register)
│   │   ├── ClientList.jsx   # Client List
│   │   ├── ClientForm.jsx   # Client Registration Form
│   │   └── ClientDetails.jsx# Client Details
│   └── styles.css
```

Admin Portal 登入頁僅提供登入，**無**「Sign Up / Register」連結；**無**預設 admin 帳密提示。新用戶須由既有 admin 透過 **POST /api/auth/users** 建立（或日後在 Admin Portal 新增「User Management」頁面）。

### 2.3.2 Backend Architecture
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

Directory Structure
```
admin_portal/backend/
├── app/
│   ├── main.py       # FastAPI App
│   ├── models.py     # Database Models
│   ├── database.py   # Database Connection
│   ├── config.py     # Configuration
│   ├── auth.py       # Auth Utilities
│   └── routes/
│       ├── auth.py       # Auth Routes
│       ├── clients.py    # Client Management
│       ├── token.py      # OAuth2 Token Endpoint
│       └── authorize.py  # OAuth2 Authorization Endpoint
└── run.py
```

API Endpoints
- /api/auth/token - Login（form body: username, password）
- /api/auth/me - User Info
- **POST /api/auth/users** - Create user（**Admin only**；需 Bearer token 且當前用戶為 admin）。Body：username, password, email, full_name，可選 is_admin。**公開註冊已停用**，帳號須由管理員建立。
- /api/clients - Client Management (CRUD)
- /oauth/authorize - OAuth2 Authorization Endpoint
- /oauth/token - OAuth2 Token Endpoint

**說明**：不再自動建立 default admin；首次使用需透過既有 admin 或資料庫／bootstrap 建立首個 admin。原「POST /api/auth/register - Register」已停用（回傳 403）。
