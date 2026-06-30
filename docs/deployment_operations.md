---
sidebar_position: 10
---

# Chapter 10: Deployment and Operations

This chapter documents **production and testing** operations for the **Learning Design Facilitator (LDF)** — implementation repo **LDS-Chatbot**. It complements [Chapter 3: Access Control](./login_page.md) and [Chapter 5: Postman](./postman.md).

**Implementation reference:** `LDS-Chatbot/deploy/` (`nginx.conf`, `backup.sh`, `*.service`, `env.*.example`).

---

## 10.1 Environments

| Environment | Public host (example) | Notes |
|-------------|---------------------|--------|
| **Testing** | `http://ronald-test.cite.hku.hk` | Development / UAT (HTTP) |
| **Production** | `https://ideals-ldf.cite.hku.hk` | LDS integration target (HTTPS) |

Internal services (default):

| Service | Port | Unit name |
|---------|------|-----------|
| Main Agent API | `127.0.0.1:5000` | `lds-chatbot` |
| Admin Portal API | `127.0.0.1:5001` | `lds-chatbot-admin` |
| Nginx | `80` / `443` | public entry |

---

## 10.2 Disable LDF web UI while keeping LDS API (Nginx)

To hide the **main Learning Design Facilitator (LDF) web UI** (`/`, `/client1`, …) but keep **LDS** working, change only Nginx `location /`. **Do not** stop `lds-chatbot`.

In `/etc/nginx/sites-available/lds-chatbot.conf`, replace:

```nginx
location / {
    try_files $uri $uri/ /index.html;
}
```

with:

```nginx
location / {
    default_type text/html;
    return 503 '<!DOCTYPE html><html lang="zh-HK"><head><meta charset="UTF-8"><title>Maintenance</title></head><body style="font-family:sans-serif;text-align:center;padding:4rem"><h1>Learning Design Facilitator 網頁暫停使用</h1><p>系統維護中，請稍後再試。<br>The Learning Design Facilitator web interface is temporarily unavailable.</p><p><small>LDS 整合服務不受影響。</small></p></body></html>';
}
```

**Keep unchanged:**

- `location /api/` → main backend (LDS)
- `location /admin/api/` → Admin API (optional)
- `location /admin/` → Admin Portal UI (optional)

Apply:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

Restore: revert `location /` to `try_files` and reload.

Optional: also return `503` on `/docs`, `/redoc`, `/openapi.json` if Swagger should not be public.

---

## 10.3 Authentication architecture (two layers)

| Layer | Endpoint | OTP 2FA? | Used by |
|-------|----------|----------|---------|
| **Main Agent API** | `POST /api/auth/token` | **No** | LDS (`/api/general_bot`, `/api/ilo_bot`) |
| **Admin Portal** | `POST /admin/api/auth/token` | **Yes** (when enabled) | Operators at `/admin/` |

When `ADMIN_PORTAL_URL` is set on the main backend (`/etc/lds-chatbot.env`):

1. LDS calls **`POST /api/auth/token`** (main host).
2. Main backend validates username/password via Admin Portal **`POST /api/auth/validate-credentials`** (internal; uses `ACCESS_LOG_ADMIN_TOKEN` header). **No OTP email is sent.**
3. Main backend issues its **own JWT** for LDS.
4. LDS uses that Bearer token on `/api/*`.

Admin Portal web login (`/admin/api/auth/token`) **does** require email OTP when 2FA is enabled.

### Admin Portal 2FA (current behavior)

Configured in `/etc/lds-chatbot-admin.env`:

| Variable | Default | Purpose |
|----------|---------|---------|
| `ADMIN_2FA_ENABLED` | `false` | Master switch |
| `ADMIN_2FA_ALL_USERS` | `true` | When `true`, **all** portal users require OTP; when `false`, only `ADMIN_2FA_USERNAME` |
| `ADMIN_2FA_USERNAME` | `admin` | Used only when `ADMIN_2FA_ALL_USERS=false` |
| `ADMIN_OTP_EXPIRE_MINUTES` | `5` | OTP validity window |
| `ADMIN_OTP_MAX_ATTEMPTS` | `5` | Failed attempts before challenge is invalidated |
| `EMAIL_ENABLED`, `SMTP_*` | — | Required to send OTP emails |

Flow:

1. `POST /admin/api/auth/token` → `{ status: "otp_required", challenge_id, email_hint }`
2. `POST /admin/api/auth/verify-otp` → `{ access_token }`

Database table: `login_otp_challenges` (SQLite, created on Admin backend startup via `init_db()`).

**User management (terminal):** `admin_portal/backend/scripts/create_admin.py` (`--interactive`, `--reset-password`, `--set-email`, `--no-admin` for LDS service accounts).

---

## 10.4 SQLite permissions (critical)

Services run as **`www-data`**. The Admin DB and its **directory** must be writable (SQLite journal files).

```bash
sudo chown www-data:www-data /path/to/admin_portal.db
sudo chgrp www-data /path/to/admin_portal/backend
sudo chmod 775 /path/to/admin_portal/backend
```

After `git pull`, if you run `chown -R ubuntu:ubuntu` on the repo, **re-apply** the above for DB paths only.

Symptom if wrong: `sqlalchemy.exc.OperationalError: attempt to write a readonly database` on Admin login (500) or OTP.

---

## 10.5 Automated backup

Script: `LDS-Chatbot/deploy/backup.sh` → install as `/usr/local/bin/lds-chatbot-backup.sh`.

Backs up:

- `lds_chatbot.db`, `admin_portal.db`
- `/etc/lds-chatbot.env`, `/etc/lds-chatbot-admin.env`
- Nginx site config

Default retention: **365 days** (`LDS_BACKUP_RETAIN_DAYS`).

Example cron (HKT **03:00**, matches `deploy/backup.sh` header comment):

```cron
CRON_TZ=Asia/Hong_Kong
0 3 * * * /usr/local/bin/lds-chatbot-backup.sh >> /var/log/lds-chatbot-backup.log 2>&1
```

You may use `0 0` (midnight) instead if preferred; keep `CRON_TZ=Asia/Hong_Kong`.

Paths are resolved from env files when possible; see script for `LDS_MAIN_DB` / `LDS_ADMIN_DB` overrides.

---

## 10.6 Production go-live data hygiene

When promoting from **testing** to **production**, clear test dynamic data so IDs and logs start clean:

| Data | Action |
|------|--------|
| `access_logs`, `agent_jobs`, OTP challenges | Delete or fresh DB |
| `users` | Recreate production accounts only |
| Runtime logs | Truncate `/var/log/lds-chatbot/` |
| JWT secrets | **New** values in `/etc/lds-chatbot.env` (do not reuse test secrets) |
| `sqlite_sequence` | Reset if you need auto-increment IDs from 1 |

See `deploy/backup.sh` before destructive steps.

---

## 10.7 Python dependencies (JWT)

Both venvs must use **PyJWT** (not the unrelated `jwt` package):

```bash
/path/to/.venv/bin/pip uninstall -y jwt PyJWT
/path/to/.venv/bin/pip install "PyJWT>=2.8.0"
```

Main backend: `api/auth.py` uses `jwt.PyJWTError`. Wrong package causes **500** on `/api/auth/token`.

---

## 10.8 Health checks

```bash
curl -s http://<host>/api/health
curl -s -X POST http://<host>/api/auth/token -d "username=...&password=..."
```

Production Postman collection: [Chapter 5](./postman.md) → `LDS-AI-Agent-Production.postman_collection.json`, bodies in [`COURSE.json`](./postman/COURSE.json).

---

## 10.9 Initial deployment

Greenfield setup is documented in **`LDS-Chatbot/deploy/RUNBOOK_DEPLOY.md`**. Summary:

1. Clone repo to server (e.g. `/home/ubuntu/LDS-Chatbot`)
2. Create Python venvs; `pip install -r requirements.txt` (main + `admin_portal/backend`)
3. Copy env templates: `deploy/env.backend.example` → `/etc/lds-chatbot.env`, `deploy/env.admin-backend.example` → `/etc/lds-chatbot-admin.env`
4. Set `JWT_SECRET_KEY`, `ADMIN_PORTAL_URL=http://127.0.0.1:5001`, Azure OpenAI, `LDS_BASE` / `LDS_TOKEN`, `ACCESS_LOG_ADMIN_TOKEN` (same in both env files)
5. Install systemd units: `deploy/lds-chatbot-home.service`, `deploy/lds-chatbot-admin-home.service` (or `/opt/lds-chatbot` variants)
6. Build frontends; install `deploy/nginx.conf` (or site under `/etc/nginx/sites-available/lds-chatbot.conf`)
7. Bootstrap Admin user: `admin_portal/backend/scripts/create_admin.py` (see §10.4 SQLite permissions)
8. Install `deploy/backup.sh` → `/usr/local/bin/lds-chatbot-backup.sh` + cron (§10.5)

Reference: `deploy/DEPLOYMENT.txt`, `deploy/env.*.example`.

---

## 10.10 Routine updates

Code updates: **`LDS-Chatbot/deploy/RUNBOOK_UPDATE.md`** and **`deploy/update.sh`**.

Typical flow on EC2:

```bash
cd /home/ubuntu/LDS-Chatbot
git pull
# Re-apply www-data ownership on SQLite DB + backend dir (§10.4)
sudo systemctl restart lds-chatbot lds-chatbot-admin
sudo systemctl reload nginx
```

After `git pull`, never leave `.db` files owned only by `ubuntu` if services run as `www-data`.

---

## 10.11 Key environment variables (main backend)

| Variable | Purpose |
|----------|---------|
| `BOT_RESPONSE_MODE` | `sync` (default) or `async` — only `general_bot` uses async jobs |
| `ADMIN_PORTAL_URL` | e.g. `http://127.0.0.1:5001` — validates LDS service-account passwords |
| `JWT_SECRET_KEY` | Main API token signing |
| `LDS_BASE`, `LDS_TOKEN` | Agent → LDS Options/Patterns API |
| `AZURE_OPENAI_*` | LLM + embeddings for RAG |
| `RAG_PRE_CLASSIFY`, `ILAP_INDEX_PATH` | RAG routing and index path |
| `ACCESS_LOG_ADMIN_TOKEN` | Must match admin backend for log proxy |

Admin backend: `ADMIN_2FA_*`, `ADMIN_OTP_EXPIRE_MINUTES` (default 5), `ADMIN_OTP_MAX_ATTEMPTS` (default 5), `EMAIL_ENABLED`, `SMTP_*`.

Full lists: `deploy/env.backend.example`, `deploy/env.admin-backend.example`.

---

*Last aligned with LDS-Chatbot: Admin Portal all-user 2FA, async `general_bot` jobs, `open_specialist_bot` handoff, `deploy/backup.sh`, RAG eight-bucket retrieval, Nginx UI maintenance pattern.*
