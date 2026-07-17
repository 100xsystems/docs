# API Integration Spec — July 17, 2026

> **Status:** Implemented
> **Architecture:** CLI writes, Website reads

---

## 1. Architecture Overview

```
┌─────────────┐         ┌──────────────┐         ┌─────────────────┐
│   CLI Tool   │ ──POST──▶  Website API  │ ────▶  Turso Database   │
│  (Writer)    │ ◀──resp─│  (Proxy)     │         │   (State)       │
└─────────────┘         └──────┬───────┘         └─────────────────┘
                               │
                        ┌──────▼───────┐
                        │  GitHub OAuth │
                        │  (Identity)   │
                        └──────────────┘
                               │
                        ┌──────▼───────┐
                        │  Website UI  │
                        │  (Reader)     │
                        └──────────────┘
```

### Key Principle
- **CLI is the writer** — it calls POST APIs for all mutations (enroll, validate, submit)
- **Website is the reader** — it calls GET APIs for all reads (portfolio, progress, recent)
- **GitHub OAuth is the identity layer** — CLI authenticates via PKCE, passes token to API
- **API verifies identity** — CLI-facing routes verify the GitHub token before allowing writes

---

## 2. Authentication

### CLI → API: Bearer Token
Every POST request from the CLI includes an `Authorization: Bearer <github_token>` header.
The API verifies the token by calling `https://api.github.com/user`.

### Auth Middleware
A shared `verifyGitHubToken()` helper:
1. Extracts `Bearer` token from `Authorization` header
2. Calls `GET https://api.github.com/user` with the token
3. Returns `{ login, email }` if valid, throws 401 if invalid
4. Also upserts the user in the `users` table (creates on first login)

---

## 3. CLI-Facing APIs (POST — Write)

### `POST /api/cli/enroll`
Called by: `100xsystems init <system>`

Records a user enrolling in a system track.

**Request:**
```json
{
  "github_email": "alice@github.com",
  "system_slug": "claude-code",
  "track_slug": "track-typescript"
}
```

**Response:**
```json
{
  "status": "enrolled",
  "enrollment": {
    "github_email": "alice@github.com",
    "system_slug": "claude-code",
    "track_slug": "track-typescript",
    "next_lesson_slug": null,
    "started_at": "2026-07-17T12:00:00Z"
  }
}
```

**DB action:** `INSERT INTO user_enrollments ... ON CONFLICT DO NOTHING`

---

### `POST /api/cli/validate`
Called by: `100xsystems validate`

Records a lesson validation result and optionally advances enrollment.

**Request:**
```json
{
  "github_email": "alice@github.com",
  "system_slug": "claude-code",
  "track_slug": "track-typescript",
  "lesson_slug": "lesson-intro-and-setup",
  "module_slug": "module-1-cli-foundations",
  "status": "passed",
  "passed_count": 6,
  "failed_count": 0,
  "validation_result": "{ ... }",
  "next_lesson_slug": "lesson-build-cli"
}
```

`next_lesson_slug` is optional — if provided and status is `passed`, the enrollment advances.

**Response:**
```json
{
  "status": "recorded",
  "validation": {
    "lesson_slug": "lesson-intro-and-setup",
    "status": "passed"
  },
  "enrollment_advanced": true,
  "next_lesson_slug": "lesson-build-cli"
}
```

**DB action:**
1. `INSERT INTO user_validations ... ON CONFLICT DO UPDATE`
2. If `status === 'passed'` and `next_lesson_slug` provided: `UPDATE user_enrollments SET next_lesson_slug`

---

### `POST /api/cli/submit`
Called by: `100xsystems submit`

Records a submission (PR) when a user completes a system.

**Request:**
```json
{
  "github_email": "alice@github.com",
  "system_slug": "claude-code",
  "track_slug": "track-typescript",
  "pr_url": "https://github.com/100xsystems/submissions/pull/42",
  "pr_number": 42
}
```

**Response:**
```json
{
  "status": "submitted",
  "submission": {
    "pr_url": "https://github.com/100xsystems/submissions/pull/42",
    "pr_number": 42
  },
  "badge_awarded": true
}
```

**DB action:**
1. `INSERT INTO submissions ... ON CONFLICT DO UPDATE`
2. Auto-award `'completed'` badge: `INSERT INTO badges ... ON CONFLICT DO NOTHING`
3. Mark enrollment as completed: `UPDATE user_enrollments SET completed_at`

---

## 4. Website-Facing APIs (GET — Read)

### `GET /api/portfolio/[email]`
Returns a user's complete portfolio.

**Response:**
```json
{
  "user": {
    "github_email": "alice@github.com",
    "github_username": "alice",
    "display_name": "Alice"
  },
  "enrollments": [
    {
      "system_slug": "claude-code",
      "track_slug": "track-typescript",
      "next_lesson_slug": "lesson-build-cli",
      "completed_at": null
    }
  ],
  "validations": {
    "claude-code/track-typescript": [
      {
        "lesson_slug": "lesson-intro-and-setup",
        "status": "passed",
        "validated_at": "2026-07-17T12:00:00Z"
      }
    ]
  },
  "submissions": [
    {
      "system_slug": "microservices",
      "pr_url": "https://github.com/100xsystems/submissions/pull/10",
      "pr_status": "merged"
    }
  ],
  "badges": [
    {
      "system_slug": "microservices",
      "badge_type": "completed",
      "awarded_at": "2026-07-16T12:00:00Z"
    }
  ]
}
```

---

### `GET /api/progress/[email]/[system]`
Returns progress summary for a specific system.

**Response:**
```json
{
  "user_email": "alice@github.com",
  "system_slug": "claude-code",
  "enrollments": [
    {
      "track_slug": "track-typescript",
      "next_lesson_slug": "lesson-build-cli",
      "started_at": "2026-07-17T10:00:00Z",
      "completed_at": null,
      "passed_validations": 1,
      "total_lessons": null
    }
  ],
  "recent_validations": [
    {
      "lesson_slug": "lesson-intro-and-setup",
      "module_slug": "module-1-cli-foundations",
      "status": "passed",
      "validated_at": "2026-07-17T12:00:00Z"
    }
  ]
}
```

---

### `GET /api/recent`
Returns recent validations and submissions across all users (for activity feed).

**Query params:** `limit` (default 20), `type` (optional: 'validate' | 'submit' | 'all')

**Response:**
```json
{
  "activities": [
    {
      "type": "validate",
      "github_email": "alice@github.com",
      "github_username": "alice",
      "system_slug": "claude-code",
      "track_slug": "track-typescript",
      "lesson_slug": "lesson-intro-and-setup",
      "status": "passed",
      "created_at": "2026-07-17T12:00:00Z"
    },
    {
      "type": "submit",
      "github_email": "bob@github.com",
      "github_username": "bob",
      "system_slug": "microservices",
      "pr_url": "https://github.com/100xsystems/submissions/pull/10",
      "pr_status": "open",
      "created_at": "2026-07-17T11:00:00Z"
    }
  ]
}
```

---

## 5. Auth Middleware

Located at `website/src/lib/auth.ts`.

```typescript
export async function verifyGitHubToken(request: Request): Promise<{ login: string; email: string }>
```

- Extracts `Bearer` token from `Authorization` header
- Calls `GET https://api.github.com/user` 
- Also calls `GET https://api.github.com/user/emails` to get primary email
- Upserts user in `users` table
- Returns `{ login, email }`
- Throws `Error` with status 401 if invalid

---

## 6. Error Handling

All API routes return consistent error shapes:

```json
{
  "error": "error_code",
  "message": "Human-readable description"
}
```

HTTP status codes:
- `401` — Missing or invalid GitHub token
- `400` — Missing required fields
- `500` — Database or server error
