# BYOClaw Specification

Version 0.1.0

Bring Your Own Claw (BYOClaw) is a minimal specification that allows a user’s
OpenClaw agent to temporarily access restricted website APIs using
human-initiated authorization.

BYOClaw enables websites to expose functionality designed specifically for
AI agents without granting agents full user credentials or unrestricted API
access.

---

# Goals

BYOClaw is designed to:

- Allow **user-controlled agents** to interact with websites safely
- Prevent agents from receiving sensitive authentication material
- Provide **short-lived, tightly scoped authorization**
- Reduce prompt injection risk through simple prompt design
- Allow websites to expose agent-specific capabilities without building full UI

---

# Non-Goals

BYOClaw does **not** attempt to:

- Replace OAuth
- Provide full identity federation
- Enable destructive automation
- Provide long-lived agent credentials

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

---

# Architecture Overview

BYOClaw follows a simple trust model.

```

Human Browser
│
│ authenticated session
▼
Website
│
│ token exchange
▼
Claw Token
│
▼
User's Claw Agent
│
▼
Claw API

```

The Claw never receives the user's authentication cookie.

Token exchange occurs entirely in the user's browser.

---

# Human-Initiated Token Exchange

A Claw Token MUST only be created through a human-initiated flow.

Typical flow:

1. Human logs into website
2. Human clicks **Bring Your Claw**
3. Website generates a temporary Claw Token
4. Token is provided to the Claw via copy/paste or local transport
5. Claw uses the token to call the Claw API

At no time should the Claw receive:

- user session cookies
- login credentials
- long-lived access tokens

---

# Claw Tokens

Claw Tokens authorize requests to the Claw API.

## Requirements

Claw Tokens MUST:

- be short-lived
- be scoped to the Claw API
- identify the user
- expire automatically

Suggested lifetime:

```

10 minutes recommended
60 minutes maximum

```

Websites MAY implement token renewal.

Renewal MUST require a human action such as clicking a link in an authenticated
browser session.

---

# API Requirements

Websites implementing BYOClaw MUST expose a restricted API surface.

Recommended pattern:

```

/api/claw/*

```

Examples:

```

GET /api/claw/:username
GET /api/claw/:username/followers
POST /api/claw/:shelf-id/book

````

The Claw API SHOULD:

- avoid destructive operations
- avoid sensitive personal data
- expose only agent-safe functionality

---

# Safety Guidelines

Claw APIs SHOULD follow these safety constraints.

## No Destructive Actions

Claws SHOULD NOT be allowed to perform destructive actions such as:

- deleting accounts
- deleting data
- modifying security settings

## No Sensitive Data

Claw APIs SHOULD NOT expose:

- email addresses
- authentication tokens
- payment information
- private messages

---

# Prompt Format

BYOClaw APIs are typically introduced to the Claw via a prompt the human
provides.

Prompts MUST be designed to be **human-scannable**.

Goals:

- allow a human to quickly verify the prompt
- reduce prompt injection risks
- minimize hidden instructions

Guidelines:

- prompts SHOULD be concise
- endpoints SHOULD be self-describing
- additional documentation SHOULD be minimal
- responses SHOULD be self-explanatory JSON

---

# Prompt Example

```md
# Supermassive Book Hole — Temporary Gateway

SMBH is a website where humans curate shelves of books and media.

## Credentials

Base URL:
https://api.example.com/api/claw

Authorization:
Bearer smbhclaw_TeStTeStTeStTeStTeStTeStTeStTeSt

Expires At:
2026-03-06T01:46:10Z

User:
@mxcl

## Endpoints

GET /:username

GET /:username/following

GET /:username/followers

POST /:shelf-id/:media-type
Fields: {title, author, shelf}

### media-type

["book", "movie", "show", "game", "album", "single"]
````

---

# Example Flow

Example interaction with Supermassive Book Hole.

1. Human clicks **Bring Your Claw**
2. Website generates a Claw Token and prompt
3. Human pastes the prompt to their Claw
4. Claw begins calling the Claw API
5. Token expires after the configured duration

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
