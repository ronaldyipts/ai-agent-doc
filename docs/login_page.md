---
sidebar_position: 3
---

# Chapter 3: Access Control and Attack Surface

This chapter defines security posture for the External Agent deployment.

## 3.1 Positioning

- The External Agent is a **REST API Server**, not a frontend product.
- Do not present this service as having a public login page.
- Keep the public surface focused on API endpoints only.

## 3.2 Why avoid login pages here

- A web login page introduces extra attack surface (UI routes, session/cookie flows, brute-force vectors, form attacks).
- For this integration, LDS is the caller and UI host; this Agent should remain API-first.

## 3.3 Recommended protection for admin access

If a minimal operations/admin entry is necessary:

- Prefer server-level gate first (for example, **`.htaccess` prompt password** or equivalent reverse-proxy authentication).
- Do not expose admin functions without a protection layer.
- Keep admin APIs private/network-restricted whenever possible.

## 3.4 API authentication (runtime)

- Runtime API calls (for LDS integration) use Bearer token authentication.
- Main integration endpoints remain:
  - `POST /api/general_bot`
  - `POST /api/ilo_bot`
- See [Chapter 6: Main System Integration](./main_system_integration.md) for the API contract.

## 3.5 Summary

- Treat External Agent as API-only.
- Avoid public login pages on this service.
- If admin access is required, gate it at server edge (e.g. `.htaccess` prompt password) and keep admin routes tightly controlled.
