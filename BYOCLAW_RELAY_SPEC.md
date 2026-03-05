# BYOClaw Relay Specification (v0.2)

**Status:** Draft (MVP-oriented)  
**Audience:** Relay/backend engineers, OpenClaw connector engineers, Website SDK engineers, security reviewers

---

## 1) Overview

BYOClaw enables websites to connect to user-owned OpenClaw instances through a cloud relay **without granting the relay authority to read or forge command payloads**.

This system has three components:

1. **Relay Service** (cloud-hosted, eg AWS)
2. **OpenClaw Connector** (local plugin/integration running next to OpenClaw)
3. **Website SDK** (npm package embedded by partner websites)

### Core principles

- **No inbound ports** required on user machines.
- Connector maintains **outbound persistent connection** to relay.
- Payloads are **end-to-end encrypted and signed** between Website SDK and Connector.
- Relay performs authn/authz for routing eligibility but is **not trusted for payload confidentiality/integrity**.
- Connector is final policy enforcement point for all execution.

---

## 2) Goals and Non-Goals

## 2.1 Goals

- NAT/firewall-friendly connectivity via outbound connector socket.
- Multi-tenant routing between websites and claws.
- Strong security posture with E2EE + capability enforcement.
- Fast user experience: immediate ack + asynchronous completion.
- Auditable and revocable trust relationships.

## 2.2 Non-Goals (MVP)

- Metadata privacy against relay (timing, packet sizes, route IDs are visible).
- Full anonymity of users from websites.
- P2P connectivity without relay.

---

## 3) High-Level Architecture

```text
Website (SDK)  <---TLS--->  Relay  <---TLS--->  Connector (local) -> OpenClaw runtime
      |                                         |
      |<==== E2EE signed encrypted envelope ===>|  (relay cannot decrypt)
```

### 3.1 Data planes

- **Control plane** (relay-visible): hosted auth redirects, rotating handoff
  code exchange, pairing states, route setup, heartbeats, quotas.
- **Message plane** (relay-blind): encrypted envelopes carrying user messages, tool intents, and results.

---

## 4) Actors and Identities

## 4.1 Identities

- **Connector Device ID (`device_id`)**: stable identity for local claw endpoint.
- **Connector Keypair (`Kc_pub`, `Kc_priv`)**: generated locally; private key never leaves device.
- **Website Client ID (`client_id`)**: website app identity issued by BYOClaw platform.
- **Website Session Keypair (`Kw_pub`, `Kw_priv`)**: ephemeral per-browser-session keys.
- **User Account ID (`user_id`)**: platform identity that authorizes pairing between website and claw.
- **Handoff Code (`handoff_code`)**: one-time short-lived code minted after
  `byoclaw.dev` login and redirected back to the partner website.

## 4.2 Trust anchors

- Platform signing keys for issuing routing/capability tokens.
- Connector trust store with paired website identities and policy grants.

---

## 5) Threat Model Summary

Relay is assumed potentially malicious for content handling. Must not be able to:

- decrypt payloads,
- forge accepted commands,
- escalate capabilities,
- replay old side-effecting commands successfully.

Controls:

- E2EE envelope encryption,
- end-to-end signatures,
- nonce/counter replay protection,
- short-lived scoped capability tokens,
- connector-local policy enforcement.

---

## 6) Protocols

## 6.1 Transport

- WebSocket over TLS (`wss`) for both connector and website SDK.
- JSON envelopes for control-plane frames.
- Binary or base64 payload for encrypted message-plane frames.

### Recommended endpoints

- `wss://relay.example.com/v1/connector`
- `wss://relay.example.com/v1/client`
- `https://relay.example.com/v1/*` for REST management/handoff exchange
- `https://byoclaw.dev/connect/*` for hosted login + consent redirect UX

---

## 6.2 Connector session lifecycle

1. Connector authenticates with device credential.
2. Relay validates and establishes session.
3. Connector sends heartbeat every `N` seconds.
4. Relay marks connector online/offline based on heartbeat timeout.

### Connector HELLO frame

```json
{
  "type": "connector.hello",
  "device_id": "dev_123",
  "connector_version": "1.0.0",
  "pubkey": "base64(Kc_pub)",
  "nonce": "...",
  "ts": 1772559123,
  "sig": "base64(signature_over_canonical_payload)"
}
```

---

## 6.3 Website client lifecycle

1. SDK initiates hosted auth at `byoclaw.dev` with `client_id`, `origin`, and
   return URL.
2. User logs in at `byoclaw.dev` and selects/approves a target claw.
3. `byoclaw.dev` redirects back to the original website with a one-time
   `handoff_code` and `state`.
4. SDK exchanges `handoff_code` (+ PKCE `code_verifier`) with relay REST API.
5. Relay validates exchange and returns `pairing_token` + resolved `device_id`.
6. SDK opens client socket to relay and presents `pairing_token` + `Kw_pub`.
7. Relay checks pairing and scope eligibility, then emits `route.ready` or
   denial.

### Client HELLO frame

```json
{
  "type": "client.hello",
  "client_id": "site_abc",
  "pairing_token": "...",
  "device_id": "dev_123",
  "origin": "https://partner.example",
  "session_pubkey": "base64(Kw_pub)",
  "nonce": "...",
  "ts": 1772559124
}
```

---

## 6.4 Pairing flow (hosted login redirect, no manual code entry)

Pairing binds a website identity and user account to a target connector.

### Recommended flow

1. Website asks SDK to connect a user claw.
2. SDK redirects browser to `https://byoclaw.dev/connect/start` with
   `client_id`, `origin`, `return_url`, `state`, and PKCE challenge.
3. User authenticates at `byoclaw.dev` and grants consent for origin/scopes.
4. `byoclaw.dev` mints a one-time rotating `handoff_code` bound to
   `user_id+client_id+origin+device_id+pkce_challenge`.
5. Browser is redirected back to `return_url` on partner website with
   `handoff_code` and original `state`.
6. SDK exchanges code at `POST /v1/auth/handoff/exchange` with
   `client_id+origin+code_verifier`.
7. Relay validates code signature, TTL, rotation epoch, one-time use, origin,
   and PKCE binding.
8. Relay upserts pairing record and returns short-lived `pairing_token`.

### Handoff code requirements

- One-time use only (single successful exchange).
- Very short validity window (eg 60-120 seconds).
- Rotates frequently (eg every 30 seconds issuance epoch).
- Bound to originating `client_id`, `origin`, and PKCE challenge.
- Relay stores only hashed code material for replay prevention/audit.

### Pairing record

```json
{
  "pair_id": "pair_...",
  "user_id": "usr_...",
  "client_id": "site_abc",
  "device_id": "dev_123",
  "origin": "https://partner.example",
  "scopes": ["chat.send"],
  "verified_via": "hosted_login_redirect",
  "last_handoff_at": "2026-03-03T16:00:00Z",
  "created_at": "2026-03-03T16:00:00Z",
  "expires_at": null,
  "revoked": false
}
```

---

## 6.5 Encrypted message envelope

Relay forwards opaque message payloads. Connector and SDK handle cryptography.

### Outer frame (relay-visible)

```json
{
  "type": "msg.envelope",
  "route_id": "rte_...",
  "message_id": "msg_...",
  "from": "client|connector",
  "to": "connector|client",
  "ciphertext": "base64(...)",
  "meta": {
    "content_type": "application/byoclaw+json",
    "bytes": 2048
  }
}
```

### Inner plaintext (E2EE, example)

```json
{
  "kind": "chat.request",
  "request_id": "req_...",
  "cap_token": "...",
  "nonce": "...",
  "counter": 42,
  "ts": 1772559125,
  "payload": {
    "text": "summarize this page",
    "context": {"url": "https://partner.example/x"}
  }
}
```

### Crypto requirements (MVP recommendation)

- Key agreement: X25519
- Symmetric encryption: XChaCha20-Poly1305 (or AES-GCM if ecosystem-constrained)
- Signatures: Ed25519 on canonical inner payload
- Replay protection: monotonic counter + nonce cache + timestamp skew checks

---

## 7) Capability and Policy Model

## 7.1 Capability token claims

Capability token (JWT/PASETO/macaroon) must include:

- `iss`, `sub`, `aud` (audience must equal `device_id`)
- `client_id`, `user_id`, `origin`
- `api_base` (exact allowed API base path, eg `/api/claw`)
- `scopes` (least privilege)
- `exp`, `nbf`, `iat`, `jti`
- optional caveats: rate limits, max tool risk level

## 7.2 Suggested scope taxonomy

- `chat.send`
- `context.read`
- `memory.read`
- `tool.invoke:<toolname>`
- `action.external.send` (high risk)
- `fs.read`, `fs.write` (high risk)
- `exec.run` (very high risk)

## 7.3 Enforcement

Connector **must** validate token and enforce scope locally. Relay checks are additional, not sufficient.

Connector **must** reject requests whose HTTP target/path is outside the token's
`api_base` grant, even when the requested scope would otherwise permit action.

## 7.4 Token lifetime requirements

- Capability token lifetime **must** be short-lived.
- `exp - iat` **must not exceed 60 minutes**.
- Recommended default lifetime remains 15 minutes.
- Connectors and relay should reject tokens with excessive issued-at skew.

## 7.5 Token renewal via user session

When a capability token is near expiry (or expired), the connector may return a
renewal requirement response that includes a hosted renewal URL.

- Renewal URL **must** be derived from a valid previous token context
  (eg token `jti` + pairing context), not from arbitrary caller input.
- Renewal URL **must** resolve to a BYOClaw hosted auth endpoint that requires
  an active, valid user login session.
- If the user is not logged in, renewal **must** fail and require user
  authentication before issuing a new token.
- A successful renewal issues a new short-lived token with the same or reduced
  privileges unless the user explicitly re-consents to broader scope.

---

## 8) Execution Semantics

## 8.1 Non-blocking behavior

- Relay/client returns immediate ack (`request.accepted`) when message enqueued.
- Connector emits lifecycle events:
  - `run.started`
  - `run.progress` (optional)
  - `run.completed`
  - `run.failed`

## 8.2 Idempotency

- All side-effecting requests require `idempotency_key`.
- Connector keeps bounded dedupe cache (eg 24h).

## 8.3 Cancellation

- Client may send `run.cancel` with `request_id`.
- Connector should best-effort cancel if not yet committed to irreversible side effect.

---

## 9) Relay Responsibilities

- Authenticate connector/client sessions.
- Validate routing eligibility (pairing exists, not revoked, handoff satisfied).
- Maintain route table and online presence.
- Enforce quotas/rate limits/abuse controls.
- Forward envelopes without decryption.
- Persist minimal operational telemetry and delivery status.

### Relay must NOT

- Decrypt payload content.
- Alter encrypted payloads.
- Issue privileged commands to connector.

---

## 10) Data Storage and Retention

## 10.1 Relay storage (minimum)

- accounts, clients, devices, pairing records, handoff exchange audit
- route/session metadata
- delivery receipts and error codes
- billing/usage counters

## 10.2 Avoid storing

- plaintext user prompts/responses
- decrypted tool outputs
- raw handoff codes

## 10.3 Retention defaults (example)

- session metadata: 30 days
- usage aggregates: 12 months
- pairing/audit records: until revoked + policy window
- handoff exchange records: 30 days

---

## 11) Audit and Observability

## 11.1 Connector local audit log (recommended mandatory)

Record:

- timestamp
- origin/client_id/user_id
- requested scope/action
- allow/deny decision
- result code
- hash-chain pointer (tamper-evident)

## 11.2 Relay observability

- connection success/failure rates
- latency percentiles (p50/p95/p99)
- queue depth and drop rate
- route churn and reconnect frequencies

---

## 12) Operational Requirements

## 12.1 Availability

- Stateless relay workers behind LB.
- Shared low-latency state store for route mapping (eg Redis).
- Multi-AZ deployment.

## 12.2 Backpressure

- Per-device and per-client queue limits.
- Configurable drop/retry policies.
- Explicit overload responses.

## 12.3 Heartbeats and timeouts

- Connector heartbeat interval: 15–30s.
- Offline threshold: ~3 missed heartbeats.
- Route idle timeout configurable.

---

## 13) API/Message Types (MVP)

### Control plane

- `connector.hello`
- `client.hello`
- `route.ready`
- `route.denied`
- `session.ping` / `session.pong`
- `auth.handoff.issued` / `auth.handoff.exchanged`
- `pair.upserted` / `pair.revoked`

### Message plane

- `msg.envelope`
- `request.accepted`
- `run.started`
- `run.progress`
- `run.completed`
- `run.failed`
- `run.cancel`

### Errors

Use stable error codes, eg:

- `AUTH_INVALID`
- `PAIRING_REQUIRED`
- `HANDOFF_REQUIRED`
- `HANDOFF_INVALID`
- `HANDOFF_EXPIRED`
- `HANDOFF_REPLAYED`
- `SCOPE_DENIED`
- `DEVICE_OFFLINE`
- `REPLAY_DETECTED`
- `RATE_LIMITED`

---

## 14) Security Requirements Checklist

- [ ] TLS for all connections
- [ ] E2EE envelope encryption implemented
- [ ] End-to-end signatures verified at connector
- [ ] Nonce/counter replay prevention
- [ ] Short-lived capability tokens
- [ ] Capability token max lifetime <= 60 minutes
- [ ] Capability token bound to exact `api_base`
- [ ] Renewal URL requires active authenticated user session
- [ ] Per-origin consent + revocation
- [ ] Sensitive scopes default-deny
- [ ] Local audit log with tamper evidence
- [ ] No plaintext payload logging at relay
- [ ] Incident response + key rotation procedures

---

## 15) Monetization-Ready Boundaries

Design relay billing around value-added services, not raw packet forwarding:

- API usage/MAU by website client
- policy/governance features
- reliability SLA tiers
- analytics and conversion attribution
- enterprise controls (SSO, region pinning, dedicated tenancy)

This keeps user trust high while monetizing commercial integrations.

---

## 16) Open Questions (to resolve before v1)

1. Token format choice: JWT vs PASETO vs macaroons?
2. Default cryptographic suite across JS + connector runtimes?
3. How much offline buffering is allowed at relay?
4. Should websites receive streamed partial outputs by default?
5. Minimal mandatory consent UX standard for partner sites?

---

## 17) Versioning

- Spec version field required in all control-plane hello frames.
- Backward-compatible additions only within minor versions.
- Breaking protocol changes require major version bump and dual-stack transition window.

---

## Appendix A: Minimal sequence (happy path)

1. Connector connects to relay (`connector.hello`).
2. Website redirects user to `byoclaw.dev` login/consent.
3. `byoclaw.dev` redirects back with rotating one-time `handoff_code`.
4. SDK exchanges `handoff_code` for `pairing_token` and `device_id`.
5. Client connects (`client.hello`) with `pairing_token`.
6. Relay verifies pairing/scope eligibility; emits `route.ready`.
7. Client sends `msg.envelope` (E2EE).
8. Relay forwards to connector.
9. Connector decrypts, validates capability, executes allowed action.
10. Connector sends lifecycle events/result in encrypted envelope.
11. Relay forwards to client.

---

## Appendix B: Recommended MVP defaults

- Token TTL: 15 minutes
- Token TTL hard maximum: 60 minutes
- Pairing token TTL: 5 minutes
- Handoff code TTL: 90 seconds
- Handoff code rotation interval: 30 seconds
- Max clock skew: ±60 seconds
- Nonce cache horizon: 10 minutes
- Idempotency retention: 24 hours
- Default scope on new pairing: `chat.send` only
