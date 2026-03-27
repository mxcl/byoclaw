# BYOClaw Specification

Version 0.2.0-alpha

Bring Your Own Claw (BYOClaw) is a minimal specification that allows a user's
OpenClaw (Claw) agent to temporarily access restricted website APIs using
human-initiated authorization.

BYOClaw lets websites expose agent-specific capabilities without granting full
user credentials or broad API access.

---

# Normative Language

The key words "MUST", "MUST NOT", "SHOULD", "SHOULD NOT", and "MAY" in this
document are to be interpreted as described in RFC 2119.

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
- standardize general-purpose long-lived agent credentials
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
A human-pasted instruction block that tells the Claw how to use a Claw Token.

---

# Architecture Overview

BYOClaw follows a simple trust model.

```text
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

## Threat model

BYOClaw explicitly considers:

- leaked tokens in logs, prompts, or screenshots
- replay of renewal artifacts over observed channels
- malicious or malformed gateway text pasted by a human

## Assumptions

BYOClaw assumes:

- the human controls the Claw implementation they use
- websites provide authentic gateway text over authenticated sessions

These assumptions are not complete controls. Implementations MUST still apply
the controls in this specification.

## Required controls

- Claw Tokens MUST be short-lived.
- Renewal challenges MUST be fresh and single-use.
- Renewal proofs MUST be bound to a specific token and challenge.
- Claw implementations SHOULD parse gateway text as structured data and MUST
  NOT execute arbitrary instructions embedded in that text.

---

# Human-Initiated Gateway Issuance

A Claw Token MUST be created through an explicit human action in an
authenticated browser session.

Typical flow:

1. Human signs into website.
2. Human clicks a "Bring Your Claw" action.
3. Website issues a short-lived token and gateway text.
4. Human pastes gateway text into their Claw.
5. Claw calls the Claw API using the temporary token.

At no point should the Claw receive:

- user session cookies
- login credentials
- the user's primary long-lived account credentials

---

# Claw Tokens

Claw Tokens authorize calls to the Claw API.

## Requirements

Claw Tokens MUST:

- be generated from a cryptographically secure random source
- provide at least 128 bits of entropy
- be transmitted as bearer tokens
- be scoped to the Claw API
- expire automatically no later than 60 minutes after issuance

Websites SHOULD issue tokens with a shorter default lifetime (for example,
10 minutes).

Websites SHOULD store token hashes rather than raw tokens at rest.

## Format and encoding

- Token format is implementation-defined.
- Tokens MUST use an ASCII-safe encoding suitable for headers and copy/paste.
- base64url without padding is RECOMMENDED.
- Deployment-specific prefixes (for example `smbhclaw_`) MAY be used, but MUST
  be documented and MUST NOT reduce entropy requirements.

## Multiple simultaneous tokens

Websites MAY allow multiple active Claw Tokens per user.

If multiple active tokens are supported, websites MUST define and document:

- maximum active tokens per user
- whether limits are global or per Claw integration
- whether rate limits are per token, per user, or both

---

# Token Renewal Protocol

Renewal is OPTIONAL for websites, but if implemented it MUST require
human-confirmed action in an authenticated browser session.

## Security objective

Renewal allows a Claw to prove possession of a prior token over a potentially
logged or observed channel without sending that prior token itself.

## Mandatory renewal properties

A compliant renewal flow MUST satisfy all of the following:

- The website MUST issue a fresh renewal challenge for each attempt.
- A challenge MUST be single-use.
- A challenge MUST be bound to one specific expired token (or token hash), user,
  and renewal attempt context.
- The challenge MUST expire quickly (5 minutes or less is RECOMMENDED).
- Renewal proof verification MUST fail if the challenge is reused, expired,
  malformed, or bound to a different token/user.
- After successful renewal, the previous token MUST be invalidated.

These rules prevent replay of leaked proof material during the grace window.

## Expiry behavior

When renewal is implemented and a token is expired but still in renewal grace,
the API MUST return a self-describing renewal response:

- HTTP 401
- `error: "CLAW_GATEWAY_TOKEN_EXPIRED"`
- `renewal` object with enough information for proof-based renewal without any
  prior renewal knowledge in gateway text

The `renewal` object MUST include at least:

- `challengeToken`
- `challengeExpiresAt`
- `proofAlgorithm`
- `proofFormula` as a literal expression string
- `renewalUrlTemplate` or equivalent human-confirmation endpoint template
- `graceExpiresAt`

Example payload:

```json
{
  "error": "CLAW_GATEWAY_TOKEN_EXPIRED",
  "expiredAt": "2026-03-06T01:46:10.000Z",
  "renewal": {
    "challengeToken": "f7D4...base64url...",
    "challengeExpiresAt": "2026-03-06T01:51:10.000Z",
    "proofAlgorithm": "sha256",
    "proofFormula": "sha256(challengeToken + \":\" + sha256(previousToken))",
    "renewalUrlTemplate": "https://example.com/?clawRenewProof={proof}",
    "graceExpiresAt": "2026-03-06T03:46:10.000Z"
  }
}
```

Implementations MAY use different proof algorithms and response field names,
provided the mandatory renewal properties are met.

## Renewal steps

1. Claw receives `CLAW_GATEWAY_TOKEN_EXPIRED`.
2. Claw computes renewal proof from local token material plus fresh challenge.
3. Claw prepares a human-confirmation renewal URL or request.
4. Human completes renewal while signed into the website.
5. Website verifies challenge binding, proof validity, and grace eligibility.
6. Website rotates token and returns fresh gateway text.

After grace expiry, renewal MUST fail and the token is invalid.

---

# API Requirements

Websites implementing BYOClaw MUST expose a restricted API surface.

Recommended pattern:

```text
/api/claw/*
```

The Claw API SHOULD:

- expose only agent-safe functionality
- avoid high-risk account or security mutations
- avoid sensitive personal data
- keep mutations scoped to the token owner

Websites MUST enforce rate limits on Claw API traffic and MUST document whether
limits are evaluated per token, per user, or both.

## Interoperability behavior

Because gateway text is human-readable, Claw clients may need to infer some
request details.

BYOClaw APIs MUST be strict in what they output and tolerant in what they
accept, without weakening security controls.

Strict output requirements:

- Responses MUST use stable field names, types, and error codes for each
  endpoint.
- Error payloads SHOULD remain machine-parseable and consistent across similar
  failures.

Tolerant input requirements:

- APIs SHOULD accept harmless request-shape variation when intent is clear (for
  example, insignificant whitespace, JSON key ordering, or unknown optional
  fields).
- APIs MAY normalize equivalent, unambiguous input encodings before applying
  business logic.

Security boundary requirements:

- Tolerance MUST NOT expand token scope, endpoint access, allowed methods, or
  ownership checks.
- Ambiguous, conflicting, or out-of-scope requests MUST fail closed with an
  explicit error.

## Discovery document

If discovery is provided, `GET /api/claw` MUST return JSON with the following
minimum schema:

- `byoclawSpecVersion` (string): spec version implemented by the website
- `apiVersion` (string): website's Claw API version
- `basePath` (string): base path for listed endpoints
- `auth` (object): auth details
- `endpoints` (array): endpoint descriptors

`auth` MUST contain:

- `type` (string), currently `"bearer"`
- `header` (string), typically `"Authorization"`

Each `endpoints[]` item MUST contain:

- `name` (string)
- `method` (string)
- `path` (string, relative to `basePath`)

Discovery metadata is descriptive only and MUST NOT broaden permissions beyond
what is listed in gateway text.

Example:

```json
{
  "byoclawSpecVersion": "0.2.0-alpha",
  "apiVersion": "1",
  "basePath": "/api/claw",
  "auth": {
    "type": "bearer",
    "header": "Authorization"
  },
  "endpoints": [
    {
      "name": "me",
      "method": "GET",
      "path": "/me"
    },
    {
      "name": "shelves",
      "method": "GET",
      "path": "/shelves"
    }
  ]
}
```

---

# Error Model

BYOClaw APIs SHOULD return JSON errors with a stable `error` code string.

The following codes are RECOMMENDED as a minimal interoperable vocabulary:

- `CLAW_GATEWAY_TOKEN_MISSING`
- `CLAW_GATEWAY_TOKEN_INVALID`
- `CLAW_GATEWAY_TOKEN_EXPIRED`
- `CLAW_GATEWAY_TOKEN_REVOKED`
- `CLAW_GATEWAY_RATE_LIMITED`
- `CLAW_GATEWAY_SCOPE_FORBIDDEN`
- `CLAW_GATEWAY_RENEWAL_CHALLENGE_INVALID`
- `CLAW_GATEWAY_RENEWAL_PROOF_INVALID`

When rate limiting is triggered, APIs SHOULD return HTTP 429 and include
`retryAfterSeconds`.

---

# Revocation

Websites SHOULD provide a revocation mechanism in authenticated UI or API
controls.

If revocation is supported, revocation MUST immediately invalidate the target
Claw Token and SHOULD invalidate any outstanding renewal challenges derived from
that token.

---

# PRELIMINARY: Long-Term Tokens

This section is PRELIMINARY and non-normative. It documents an expected
integration pattern for background synchronization, but it does not yet
standardize a required issuance flow, response schema, discovery metadata, or
error vocabulary for long-term tokens.

BYOClaw continues to standardize short-lived Claw Tokens as the core
interoperable mechanism. A long-term token, if a website offers one, is a
separate credential used by a client integration after an explicit human
authorization step. It is not a browser session token, and it is not the same
thing as a short-lived Claw Token.

## Motivation

Some client apps need ongoing access after the human has finished the initial
interactive setup. Clawlicious is the motivating example for this section.

Clawlicious polls a website for events over time so it can ingest new activity
in the background. A short-lived interactive Claw Token is not sufficient for
that workflow because the token expires long before the next scheduled polls.
A long-term token lets Clawlicious continue polling and ingesting events until
the token is revoked or rotated.

## Rationale

This preliminary section exists to document the shape of a Clawlicious-style
integration without prematurely freezing the protocol. The immediate need is
background event polling: authorize once in a human-visible flow, then let the
client continue low-risk polling until the integration is revoked, expires, or
is rotated.

## Expected integration pattern

The following flow is an operational example of how a website MAY support
Clawlicious for recurring event polling.

1. Human signs into the website in a browser.
2. Human explicitly authorizes Clawlicious for background event polling.
3. Website issues a short-lived Claw Token for the interactive bootstrap step
   and MAY also issue a separate long-term token scoped only to the polling
   integration.
4. Clawlicious stores the long-term token in secure local storage appropriate
   to the platform, not in prompts, logs, screenshots, or clipboard history.
5. On its recurring schedule, Clawlicious calls the website's event polling
   endpoint with the long-term token and ingests any new events.
6. If the token is revoked, expired, or rotated, polling fails closed and
   Clawlicious prompts the human to reauthorize rather than silently widening
   access or falling back to the browser session.

Worked example:

1. Human clicks "Connect Clawlicious for background sync".
2. Website shows the requested scope, for example "read events for shelf
   updates", plus expiry, rotation, and revoke controls.
3. Human approves.
4. Website returns a long-term token that Clawlicious stores in the system
   keychain.
5. Every 10 minutes, Clawlicious calls `GET /api/claw/events?cursor=...` with
   `Authorization: Bearer <long_term_token>`.
6. Clawlicious stores the returned cursor and continues polling over time.
7. If the website returns a revoked, expired, or rotated-token response,
   Clawlicious stops background polling, marks the integration unhealthy, and
   asks the human to reconnect.

## Security constraints and tradeoffs

Long-term tokens, if implemented, SHOULD follow these constraints:

- least privilege: scope the token narrowly to the specific integration task,
  such as polling events, rather than general account access
- revocability: provide a clear user-facing revoke control that invalidates the
  token quickly
- rotation: allow token replacement without requiring a new browser login for
  every poll and invalidate the previous token after rotation
- secure storage by the client: require storage in platform-appropriate secret
  storage and avoid prompt text, logs, or plaintext config files
- auditability and visibility to the user: show active long-term integrations,
  scopes, creation time, last use, and revoke controls in authenticated UI
- separation of credential types: keep browser session tokens, short-lived
  Claw Tokens, and long-term integration tokens distinct in purpose and UI

Websites SHOULD prefer expiry plus revocation over indefinite validity.
Clients SHOULD treat long-term tokens as high-value secrets because they are
designed for unattended use.

## Example representation

The wire format is not standardized by this section, but a deployment might
represent an issued long-term token like this:

```json
{
  "tokenType": "integration",
  "token": "clawlt_7wYx...base64url...",
  "scope": ["events:read"],
  "integration": "clawlicious",
  "issuedAt": "2026-03-27T14:00:00Z",
  "expiresAt": "2026-06-25T14:00:00Z"
}
```

This example is descriptive only. This section does not standardize field
names, token prefixes, issuance endpoints, polling endpoints, cursor formats,
or rotation semantics beyond the general expectations described above.

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

- adding media to a shelf or library
- linking existing items into a shelf
- reordering shelf items
- removing an item from a shelf without deleting account data

---

# Gateway Text Format

BYOClaw APIs are typically introduced via gateway text pasted by the human.

Gateway text MUST be human-scannable and include:

- API base URL
- authorization header format
- user identity handle (if applicable)
- endpoint summary
- a level-1 title line beginning with `#`
- a short description of the website

Gateway text presented for copy/paste MUST be wrapped in markdown fences with
an opening fence of `\`\`\`md` and a closing fence of `\`\`\``.

The gateway text endpoint list is normative. Discovery responses MUST NOT
expand permissions.

Token expiry SHOULD be displayed in the human issuance UI where the token is
created. Expiry information need not be included in gateway text.

Websites implementing BYOClaw MUST link to `https://byoclaw.dev` either in the
footer of the relevant site UI or in the UI that presents gateway text or
prompt-facing instructions. The link MAY be visually minor, including a
footnote-style treatment.

This link is part of the interoperability surface. It gives humans and future
AI implementers a stable route to the authoritative specification so they can
coordinate on standardized behavior and continue adopting improvements instead
of forking prompt conventions.

Claw implementations SHOULD validate that gateway text contains only expected
BYOClaw fields and MUST ignore or reject unrelated instructions.

Gateway text SHOULD NOT include renewal formulas, challenge material, or
renewal URL templates. Renewal instructions SHOULD be learned from the
authoritative `CLAW_GATEWAY_TOKEN_EXPIRED` response at renewal time.

Gateway text SHOULD be concise. It SHOULD contain only information needed to
call the Claw API safely.

For endpoints that accept JSON parameters, endpoint summaries SHOULD include
parameter hints inline using braces, for example:

`- POST /media/add {sourceKey, target?, shelfId?}`

In this notation, parameters without `?` are required and parameters with `?`
are optional.

For `GET` endpoints that return paginated results, endpoint summaries MUST use
the standardized optional pagination hints `{limit?, page?}`.

## Prompt Guidelines (Non-Normative)

For prompt-facing guidance, BYOClaw favors Ruby on Rails style convention over
configuration.

The Prompt MUST end with:

```md
> Adheres to byoclaw.dev vX.Y.Z
```

Related capabilities should be combined into fewer, cohesive APIs whenever
possible.

Avoid over-engineering edge cases in v1. Focus on the core path, observe real
Claw-reported issues, and handle edge-case expansions in v2.

---

# Gateway Text Example

```md
# Supermassive Book Hole - Temporary Gateway

SMBH is a website where humans curate shelves of books and media.

## Credentials

- Base URL: https://api.example.com/api/claw
- Authorization: Bearer <temporary_token>
- Identity: @mxcl

## Endpoints

- GET /me
- GET /shelves {limit?, page?}
- GET /users/:username/shelves {limit?, page?}
- GET /followers {limit?, page?}
- POST /library/books {sourceKey}
- POST /shelves/:shelfId/books {sourceKey, target?}
- PATCH /shelves/:shelfId/books/reorder {sourceKey, target?, shelfId?}
- DELETE /shelves/:shelfId/books/:bookId
```

Note that the `DELETE` is non-destructive as it *archives* the book from the
shelf rather than deleting it from the account.

---

# Example Flows

## Initial issuance

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

# Appendix A: SMBH v1 Reference Profile (Non-Normative)

The following values are deployment-specific examples from SMBH v1 and are not
normative BYOClaw requirements.

- active TTL: 60 minutes
- renewal grace period: 120 minutes after expiry
- proof formula:
  `sha256(challengeToken + ":" + sha256(previousToken))`
- example endpoints:
  - `GET /api/claw/me`
  - `GET /api/claw/shelves {limit?, page?}`
  - `GET /api/claw/users/:username/shelves {limit?, page?}`
  - `GET /api/claw/followers {limit?, page?}`
  - `POST /api/claw/library/books {sourceKey}`
  - `POST /api/claw/shelves/:shelfId/books {sourceKey, target?}`
  - `PATCH /api/claw/shelves/:shelfId/books/reorder {sourceKey, target?,
    shelfId?}`
  - `DELETE /api/claw/shelves/:shelfId/books/:bookId`

---

# Future Work (Non-Normative)

Potential extensions include:

- scoped permissions
- a standardized profile for long-term integration tokens
- richer discovery metadata

These features are not fully standardized in version 0.2.0-alpha.

---

# Versioning

BYOClaw follows semantic versioning (`MAJOR.MINOR.PATCH`).
