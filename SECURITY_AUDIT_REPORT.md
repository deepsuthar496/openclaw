# Security Audit Report

**Date:** 2026-02-19
**Scope:** OpenClaw Gateway Codebase (`src/gateway`)
**Focus:** Authentication Bypass and Remote Exploitation

## Executive Summary

A deep analysis of the OpenClaw codebase was performed to identify potential vulnerabilities related to authentication bypass and remote exploitation. The focus was on the Gateway's authentication logic (`auth.ts`), WebSocket connection handling (`ws-connection/message-handler.ts`), and HTTP server endpoints (`server-http.ts`).

**Conclusion:** The codebase is generally secure against authentication bypass and remote exploitation in its default configuration. The authentication mechanisms fail safe (deny access) when unconfigured or when invalid credentials are provided. A potential risk exists only if the user explicitly misconfigures the Gateway to listen on public interfaces (`0.0.0.0`) without understanding the implications for the Control UI.

## Detailed Findings

### 1. Authentication Bypass (Verified: Secure)

**Investigation:**
We investigated whether it is possible to bypass authentication, specifically looking for:
- Logic flaws in `authorizeGatewayConnect`.
- Spoofing of "local" client status to bypass auth checks.
- Exploiting "device pairing" flows to gain unauthorized access.

**Results:**
- `authorizeGatewayConnect` correctly returns `unauthorized` if no authentication mode (token/password) is configured and active.
- `isLocalDirectRequest` correctly identifies local connections and cannot be easily spoofed from a remote IP unless `gateway.trustedProxies` is explicitly misconfigured by the user.
- **Local Pairing:** We attempted to exploit the device pairing flow by connecting as a local user without a token. The Gateway correctly rejected the connection with `unauthorized` before any pairing logic could be triggered. This confirms that even local users cannot pair a new device without possessing the shared secret (Gateway Token) or an existing paired device token.

### 2. Remote Exploitation without Tokens (Verified: Low Risk / Configuration Dependent)

**Investigation:**
We investigated whether any endpoints are accessible without authentication when the Gateway is exposed remotely.

**Results:**
- **Control UI Exposure:** The Control UI endpoints (`/ui/*`, `/__openclaw__/ui/*`) served via `handleControlUiHttpRequest` **do not enforce authentication**.
    - **Impact:** If the Gateway is bound to `0.0.0.0` (remote access enabled) via the `--bind` flag or config, anyone on the network can access the Control UI static assets (HTML/JS/CSS) and the bootstrap config (Agent Name, Agent ID, Avatar URL).
    - **Mitigation:** The Control UI itself is a client-side application. To perform any actions or view chat history, it must connect to the Gateway WebSocket, which **strictly enforces authentication**. Therefore, an attacker can see the UI but cannot control the agent or access private data.
    - **Recommendation:** Users should follow the documentation and avoid binding the Gateway to `0.0.0.0` unless they are using a secure tunnel (Tailscale, SSH) or fully understand that the UI assets are public.

- **Canvas/A2UI:** Access to Canvas endpoints (`/__openclaw__/a2ui`, `/__openclaw__/canvas`) is protected by `authorizeCanvasRequest`, which requires a valid token or a verified local connection. Remote access without a token is blocked.

- **API Endpoints:** Critical endpoints like `/v1/chat/completions` and `/hooks/*` are protected by token authentication.

## Recommendations

1.  **Maintain Defaults:** Continue to default to loopback binding (`127.0.0.1`) to prevent accidental exposure of the Control UI.
2.  **Trusted Proxies:** Users should be cautious when configuring `gateway.trustedProxies` to avoid allowing external attackers to spoof local IP addresses.
3.  **Monitor Control UI:** While low risk, consider adding a basic auth check (e.g., cookie or token query param) to `handleControlUiHttpRequest` if stricter privacy for agent metadata (name/avatar) is desired in exposed environments.

## Reproduction Attempts

A reproduction script (`test/security/repro-local-pairing.ts`) was created to attempt unauthorized local pairing. The Gateway successfully rejected the connection attempts, confirming the security of the pairing flow.
