---
sidebar_position: 3
---

# Chapter 3: Access Control, Authentication, and Attack Surface

This chapter defines security posture for the External Agent deployment and how it differs between **LDS runtime API** and **Admin Portal** access.

## 3.1 Positioning

- The External Agent is primarily a **REST API Server** for LDS integration.
- The **main LDF web UI** (SPA at `/`) is optional and may be **disabled at Nginx** in production while `/api/` remains available ([Chapter 10](./deployment_operations.md)).
- **Admin Portal** (`/admin/`) is an optional operations UI for logs, users, and monitoring—not part of the LDS runtime contract.

## 3.2 LDF web UI vs API

| Surface | Path | Production typical use |
|---------|------|----------------------|
| LDF web UI (SPA) | `/` | Often **disabled** (Nginx 503) |
| LDS APIs | `/api/*` | **Always on** |
| Admin Portal | `/admin/` | Operators only; protect with 2FA |
| Swagger | `/docs` | Optional; may be disabled publicly |

Avoid exposing unnecessary public login pages on the main LDF SPA when LDS is the only consumer. See [Chapter 10 §10.2](./deployment_operations.md#102-disable-ldf-web-ui-while-keeping-lds-api-nginx).

## 3.3 LDS runtime authentication

- LDS obtains a Bearer token via **`POST /api/auth/token`** on the **main** host (not `/admin/api/`).
- Body: `application/x-www-form-urlencoded` with `username` and `password`.
- When `ADMIN_PORTAL_URL` is configured, the main backend validates credentials via Admin Portal **`/api/auth/validate-credentials`** (password only; **no OTP email**), then issues its **own JWT**.
- **Email OTP applies only to Admin Portal UI login** (`/admin/api/auth/token` + `verify-otp`), not to LDS `POST /api/auth/token`.
- Use a dedicated **service account** (e.g. `lds`, `--no-admin`) for LDS; reserve human admin accounts for Portal access.

See [Chapter 6](./main_system_integration.md) and [Chapter 5: Postman](./postman.md).

## 3.4 Admin Portal authentication (operators)

When `ADMIN_2FA_ENABLED=true` (recommended for production):

| Setting | Current default | Effect |
|---------|-----------------|--------|
| `ADMIN_2FA_ALL_USERS` | `true` | Every portal user must complete **email OTP** after password |
| `ADMIN_2FA_ALL_USERS=false` | — | Only `ADMIN_2FA_USERNAME` (default `admin`) requires OTP |
| `ADMIN_OTP_EXPIRE_MINUTES` | `5` | OTP code validity |
| `ADMIN_OTP_MAX_ATTEMPTS` | `5` | Wrong codes before challenge is locked |

Portal login flow:

1. `POST /admin/api/auth/token` → `otp_required` + `challenge_id`
2. User enters code from email
3. `POST /admin/api/auth/verify-otp` → `access_token`

Requirements:

- Each portal user needs a valid **`email`** in `admin_portal.db`.
- SMTP must be configured (`EMAIL_ENABLED`, `SMTP_*` in `/etc/lds-chatbot-admin.env`).
- SQLite file and `admin_portal/backend/` directory must be **writable by `www-data`** ([Chapter 10 §10.4](./deployment_operations.md#104-sqlite-permissions-critical)).

Account lifecycle: `admin_portal/backend/scripts/create_admin.py` (`--interactive`, `--reset-password`, `--set-email`, `--no-admin` for LDS service accounts). When 2FA is enabled, every user needs a valid **email** in the database. Prefer the script over `POST /api/auth/users` for bootstrap and password resets. No public self-registration.

## 3.5 Recommended edge protection

- Restrict `/admin/` by network policy or reverse-proxy auth where possible.
- Rotate `JWT_SECRET_KEY` separately in `/etc/lds-chatbot.env` and `/etc/lds-chatbot-admin.env` for production (invalidates existing tokens).
- Keep `ACCESS_LOG_ADMIN_TOKEN` **identical** in both env files if Admin Portal reads main-system access logs.

## 3.6 Summary

| Caller | Login endpoint | 2FA email OTP |
|--------|----------------|---------------|
| **LDS** | `POST /api/auth/token` | No |
| **Admin Portal UI** | `/admin/api/auth/token` + `verify-otp` | Yes (when enabled) |

Treat the Agent as **API-first** for LDS; use Admin Portal only for operations, with 2FA and SMTP properly configured.
