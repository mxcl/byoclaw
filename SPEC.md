# BYOClaw Specification

Version 0.1.0

Bring Your Own Claw (BYOClaw) is a minimal specification that allows a user's
Claw agent to temporarily access restricted website APIs using human-initiated
authorization.

BYOClaw lets websites expose agent-specific capabilities without granting full
user credentials or broad API access.

---

# Goals

BYOClaw is designed to:

- allow user-controlled agents to interact with websites safely
- prevent agents from receiving sensitive authentication material
- provide short-lived, tightly scoped authorization
- reduce prompt injection risk through human-scannable gateway text
- allow websites to expose agent-safe capabilities without full UI automation

---

# Non-Goals

BYOClaw does not attempt to:

- replace OAuth
- provide full identity federation
- provide long-lived agent credentials
- permit high-risk destructive account actions

BYOClaw is intentionally minimal.

---

# Terminology

**Human**
The authenticated user interacting with a website.

**Claw**
A user-controlled AI agent capable of calling APIs.

**Website**
A service that supports BYOClaw and exposes a Claw API.

**Claw API**
A restricted API surface specifically designed for agents.

**Claw Token**
A short-lived token allowing an agent to access the Claw API.

**Gateway Text**
The human-pasted instruction block that tells the Claw how to use the token.

---

# Architecture Overview

BYOClaw follows a simple trust model.

```
Human Browser
  | authenticated session
  v
Website
  | token issuance
  v
Claw Token + Gateway Text
  | copy/paste (or local transport)
  v
User's Claw Agent
  | Authorization: Bearer <token>
  v
Claw API
```

The Claw never receives the user's browser session cookie.

---

# Security Model

BYOClaw assumes:

- The human controls the Claw agent.
- Gateway text is only provided by trusted websites.
- Claw agents do not automatically follow external instructions.
- Tokens may be leaked through agent logs or prompts and must therefore be
  short-lived.

---

# Human-Initiated Gateway Issuance

A Claw Token MUST be created through an explicit human action.

Typical flow:

1. Human signs into website.
2. Human clicks a "Bring Your Claw" action.
3. Website issues a short-lived token and gateway text.
4. Human pastes gateway text into their Claw.
5. Claw calls the Claw API using the temporary token.

At no point should the Claw receive:

- user session cookies
- login credentials
- long-lived credentials

---

# Claw Tokens

Claw Tokens authorize calls to the Claw API.

## Requirements

Claw Tokens MUST:

- be short-lived
- be scoped to the Claw API
- be transmitted as a bearer token
- expire automatically

Recommended lifetime:

```text
10 minutes recommended
60 minutes maximum
```

Current reference deployment profile (SMBH v1):

- active TTL: 60 minutes
- renewal grace period: 120 minutes after expiry
- token value stored as hash at rest

Websites SHOULD store token hashes rather than raw tokens.

---

# Token Renewal Protocol

Renewal is OPTIONAL for websites, but if implemented it MUST require
human-confirmed action in an authenticated browser session.

## Expiry behavior

When a token is expired but still in renewal grace, APIs SHOULD return:

- HTTP 401
- `error: "CLAW_GATEWAY_TOKEN_EXPIRED"`
- renewal details sufficient for proof-based renewal

Example payload:

```json
{
  "error": "CLAW_GATEWAY_TOKEN_EXPIRED",
  "expiredAt": "2026-03-06T01:46:10.000Z",
  "renewalGraceMinutes": 120,
  "renewal": {
    "challengeToken": "f7D4...base64url...",
    "proofAlgorithm": "sha256",
    "proofFormula": "sha256(challengeToken + \":\" + sha256(previousToken))",
    "renewalUrlTemplate": "https://example.com/?clawRenewProof={proof}&clawRenewToken=f7D4...",
    "expiresAt": "2026-03-06T01:46:10.000Z",
    "graceExpiresAt": "2026-03-06T03:46:10.000Z"
  }
}
```

## Renewal steps

1. Claw receives `CLAW_GATEWAY_TOKEN_EXPIRED`.
2. Claw computes proof from the previous token and `challengeToken`.
3. Claw substitutes `{proof}` in `renewalUrlTemplate`.
4. Claw asks the human to open the URL while signed in.
5. Website verifies proof against an expired token in grace for that user.
6. Website rotates token and returns fresh gateway text.

After grace expiry, renewal MUST fail and the token is invalid.

## Human confirmation endpoints (reference profile)

The SMBH v1 integration uses authenticated browser endpoints:

- `GET /api/me/claw/gateway-renewal?proof=...&challengeToken=...`
- `POST /api/me/claw/gateway-renewal/confirm`

On success, confirmation returns a new token/gateway text and invalidates the
previous token.

---

# API Requirements

Websites implementing BYOClaw MUST expose a restricted API surface.

Recommended pattern:

```text
/api/claw/*
```

Endpoint discovery guidance:

`GET /api/claw` SHOULD return a small discovery document, for example:

```json
{
  "version": "1",
  "endpoints": {
    "me": "GET /me",
    "shelves": "GET /shelves"
  }
}
```

The API SHOULD be additive and non-breaking. New versions MAY be introduced,
for example `/api/claw/v2/*`.

Reference profile examples:

```text
GET    /api/claw/me
GET    /api/claw/shelves
GET    /api/claw/users/:username/shelves
GET    /api/claw/followers
POST   /api/claw/library/books
POST   /api/claw/shelves/:shelfId/books
PATCH  /api/claw/shelves/:shelfId/books/reorder
DELETE /api/claw/shelves/:shelfId/books/:bookId
```

The Claw API SHOULD:

- expose only agent-safe functionality
- avoid high-risk account/security mutations
- avoid sensitive personal data
- keep mutations scoped to the token owner

Claw APIs SHOULD enforce rate limits per token.
Suggested baseline: 60 requests per minute.

---

# Safety Guidelines

Claw APIs SHOULD follow these constraints.

## Forbidden actions

Claws SHOULD NOT be allowed to:

- delete accounts
- change authentication factors or passwords
- modify security-critical settings

## Sensitive data

Claw APIs SHOULD NOT expose:

- authentication tokens
- payment information
- private messages
- private contact details unless explicitly required

## Allowed bounded mutations

Low-risk content mutations MAY be supported, for example:

- adding media to a shelf/library
- linking existing items into a shelf
- reordering shelf items
- removing an item from a shelf without deleting account data

---

# Gateway Text Format

BYOClaw APIs are typically introduced via gateway text pasted by the human.

Gateway text MUST be human-scannable and include:

- API base URL
- authorization header format
- token expiry
- user identity handle (if applicable)
- endpoint summary
- expiry/renewal behavior (if renewal exists)

Guidance:

- concise prompts are a SHOULD, not yet a MUST
- endpoint docs should be self-describing
- responses should be predictable JSON
- avoid hidden or indirect instructions

---

# Gateway Text Example

```md
# Supermassive Book Hole - Temporary Gateway

SMBH is a website where humans curate shelves of books and media.

## Credentials

- Base URL: https://api.example.com/api/claw
- Authorization: Bearer smbhclaw_TeStTeStTeStTeStTeStTeStTeStTeSt
- Expires At (UTC): 2026-03-06T01:46:10Z
- Identity: @mxcl

## Renewal

- If API returns CLAW_GATEWAY_TOKEN_EXPIRED, compute:
  sha256(challengeToken + ":" + sha256(previousToken))
- Replace {proof} in renewalUrlTemplate and ask user to click it.

## Endpoints

- GET /me
- GET /shelves
- GET /users/:username/shelves
- GET /followers
- POST /library/books
- POST /shelves/:shelfId/books
- PATCH /shelves/:shelfId/books/reorder
- DELETE /shelves/:shelfId/books/:bookId
```

---

# Example Flows

## Initial Issuance

1. Human clicks BYOClaw button.
2. Website returns token and gateway text.
3. Human pastes gateway text to Claw.
4. Claw calls `/api/claw/*` with bearer token.

## Renewal

1. Claw gets `CLAW_GATEWAY_TOKEN_EXPIRED`.
2. Claw computes proof and prepares renewal URL.
3. Human opens URL while authenticated.
4. Website verifies and issues replacement token.
5. Claw continues with new token.

---

# Future Work

Potential extensions include:

* scoped permissions
* tightly scoped long-term tokens
* discovery endpoints

These features are not part of version 0.1.

---

# Versioning

BYOClaw follows semantic versioning.

Version format:

```
MAJOR.MINOR.PATCH
```

Current version:

```
0.1.0
```

---

# Contributing

The BYOClaw specification is an open source document.

Contributions from humans and AI agents are welcome.

Please check the issue tracker before submitting new proposals.

Low-quality or abusive submissions may be removed.

---

# License

The BYOClaw specification is open and free to implement.

```
