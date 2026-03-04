---
sidebar_position: 3
---

# Chapter 3: Login Page

本章描述 **主應用（LDS Chatbot）** 的登入頁。主應用登入 endpoint 為 **POST /api/auth/login**（form: username, password）。**Admin Portal** 登入為 **POST /api/auth/token**（form body: username, password），且無公開註冊；帳號由管理員透過 **POST /api/auth/users** 建立，詳見 [Application Architecture](./app_archi.md)。

## 3.1 Overview
- Entry point for **LDS Chatbot**（主應用）user authentication.
- Supports Traditional Chinese and English.

## 3.2 Features

### 3.2.1 User Authentication
- Username and password login
- Form validation (required fields)
- Loading state indicator
- Error message display
- Auto-redirect to chatbot page on success

### 3.2.2 Branding
- Displays IDEALS Logo
- Displays Learning Design Studio Logo
- Shows "聊天機器人" (Chatbot) badge

### 3.2.3 Navigation Links
- Forgot Password: Navigate to password reset page (still under development)
- Sign Up：若主應用有提供註冊入口則顯示「Navigate to registration page」；**Admin Portal 無公開註冊**，帳號由管理員建立。
- Responsive design

## 3.3 Technical Details

### 3.3.1 Frontend
- Component: `src/components/Login.jsx`
- Framework: React (Functional Component)
- State Management: React Hooks (useState)
- Endpoint: POST /api/auth/login
- Authentication: Form Data (username, password)
- Response: JSON with access_token, refresh_token, and user object
- Tokens stored in localStorage (access_token, refresh_token)
- User info stored in localStorage (user)
- Language preference stored in localStorage (app_language)

Flow
1. User enters username and password
2. Clicks "Login" button
3. System sends POST request to /api/auth/login
4. Backend validates credentials
5. On success, returns JWT tokens and user info
6. Frontend stores tokens in localStorage
7. Updates authentication state and redirects to chatbot page

### 3.3.2 Backend API
- Password hashing with bcrypt
- JWT token authentication
- Token expiration: 30 minutes for access token
- Generic error messages (security best practice)
- HTTPS recommended for production

### 3.3.3 Data Storage (Under Development)
- Database: SQLite (default) or PostgreSQL
- User table with: username, hashed_password, is_active fields
- JWT configuration: JWT_SECRET_KEY required

## 3.4 User Flow
See 3.3.1 Flow.

## 3.5 Security Features
- bcrypt password hashing
- JWT authentication
- Generic error messaging
- HTTPS recommended

## 3.6 Backend Requirements
- SQLite (default) or PostgreSQL
- JWT_SECRET_KEY configured
