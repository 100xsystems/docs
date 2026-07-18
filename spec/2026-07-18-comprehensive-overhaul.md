# Comprehensive Overhaul Spec — July 18, 2026

> **Status:** Active Implementation
> **Author:** AI-assisted spec
> **Target completion:** July 18, 2026

---

## 1. Architecture Overview

### Current State (Problems)
- **Module-based hierarchy**: System → Track → Module → Lesson (4 levels)
- **Multiple tables**: users, user_enrollments, user_validations, submissions, badges (5 tables)
- **Clerk auth on website**: UserButton shows Clerk avatar, no custom username
- **CLI has PR automation**: Octokit-based PR submission
- **Portfolio page**: Uses email slug `/portfolio/[email]`
- **Separate reading page**: Settings as popup, not proper route

### Target State
- **Lesson-based hierarchy**: System → Track → Lesson (3 levels, with `lesson_type` metadata)
- **Two tables only**: `users` and `user_progress`
- **Custom username header**: "Hi, github_username" with yellow hover dropdown
- **No PR automation**: Users submit links instead
- **Settings routes**: `/settings/profile`, `/settings/reading`, `/settings/achievements`, `/settings/activity`
- **Sequential learning**: One lesson at a time, localStorage-based navigation

---

## 2. Database Schema Changes

### Current Tables (to be replaced)
- `users` — github_email PK, github_username, display_name, created_at, last_login_at
- `user_enrollments` — github_email, system_slug, track_slug, next_lesson_slug, started_at, completed_at
- `user_validations` — github_email, system_slug, track_slug, module_slug, lesson_slug, status, ...
- `submissions` — github_email, system_slug, track_slug, pr_url, pr_number, pr_status, ...
- `badges` — github_email, system_slug, badge_type, ...

### New Schema (2 tables)

#### `users` table (updated)
```sql
CREATE TABLE users (
  github_email TEXT PRIMARY KEY,
  github_username TEXT,
  display_name TEXT,
  linkedin_url TEXT DEFAULT '',
  github_avatar TEXT DEFAULT '',
  short_bio TEXT DEFAULT '',
  current_profession TEXT DEFAULT '',
  experience_years INTEGER DEFAULT 0,
  created_at TEXT DEFAULT CURRENT_TIMESTAMP,
  last_login_at TEXT
);
```

#### `user_progress` table (new — replaces all 4 old tables)
```sql
CREATE TABLE user_progress (
  github_email TEXT NOT NULL,
  system_slug TEXT NOT NULL,
  track_slug TEXT NOT NULL,
  lesson_slug TEXT NOT NULL,
  lesson_type TEXT DEFAULT 'lesson',
  is_submitted INTEGER DEFAULT 0,
  submission_link TEXT DEFAULT '',
  live_link TEXT DEFAULT '',
  is_validated INTEGER DEFAULT 0,
  positive_validations INTEGER DEFAULT 0,
  negative_validations INTEGER DEFAULT 0,
  liked_number INTEGER DEFAULT 0,
  is_reported INTEGER DEFAULT 0,
  reported_number INTEGER DEFAULT 0,
  achievement TEXT DEFAULT '',
  achievement_type TEXT DEFAULT '',
  created_at TEXT DEFAULT CURRENT_TIMESTAMP,
  updated_at TEXT DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (github_email, system_slug, track_slug, lesson_slug)
);
```

### Migration Plan
1. ALTER TABLE users to add new columns (using ALTER TABLE ADD COLUMN — SQLite/Turso supports this)
2. CREATE TABLE user_progress
3. Migrate data from old tables into user_progress
4. Drop old tables (user_enrollments, user_validations, submissions, badges)

---

## 3. API Endpoints

### New CLI-Facing (POST — Write)

#### `POST /api/v1/login`
Called by: CLI login command

Records/upserts user from GitHub auth.

**Request:**
```json
{
  "github_email": "user@github.com",
  "github_username": "user123",
  "display_name": "User Name",
  "github_avatar": "https://avatars.github.com/..."
}
```

**Response:** `{ "status": "ok", "user": { ... } }`

#### `POST /api/v1/user_progress`
Called by: `100xsystems init` and `100xsystems validate`

**For init (enrollment):**
```json
{
  "github_email": "user@github.com",
  "system_slug": "claude-code",
  "track_slug": "track-typescript",
  "lesson_slug": "lesson-intro-and-setup",
  "lesson_type": "lesson"
}
```

**For validate (update validation):**
```json
{
  "github_email": "user@github.com",
  "system_slug": "claude-code",
  "track_slug": "track-typescript",
  "lesson_slug": "lesson-intro-and-setup",
  "is_validated": true
}
```

**Response:** `{ "status": "recorded" }`

### New Website-Facing (GET — Read)

#### `GET /api/v1/user_progress/{github_email}/{system_slug}`
Returns all progress rows for a user in a system.

**Response:**
```json
{
  "rows": [
    {
      "github_email": "user@github.com",
      "system_slug": "claude-code",
      "track_slug": "track-typescript",
      "lesson_slug": "lesson-intro-and-setup",
      "lesson_type": "lesson",
      "is_submitted": 0,
      "is_validated": 1,
      "positive_validations": 1,
      "negative_validations": 0,
      "achievement": "",
      "achievement_type": "",
      "updated_at": "2026-07-18T12:00:00Z"
    }
  ]
}
```

#### `GET /api/v1/leaderboard/{system_slug}?limit=10`
Returns recent activity for a system leaderboard.

**Response:**
```json
{
  "activities": [
    {
      "github_email": "user@github.com",
      "github_username": "user123",
      "system_slug": "claude-code",
      "lesson_slug": "lesson-intro-and-setup",
      "action": "validated",
      "timestamp": "2026-07-18T12:00:00Z"
    }
  ]
}
```

#### `GET /api/v1/submissions/{system_slug}/{track_slug}/{lesson_slug}?limit=50`
Returns all submissions for a specific lesson.

**Response:**
```json
{
  "submissions": [
    {
      "github_email": "user@github.com",
      "submission_link": "https://github.com/...",
      "live_link": "https://my-app.vercel.app",
      "liked_number": 5,
      "is_reported": 0
    }
  ]
}
```

#### `POST /api/v1/like`
Like a submission.

**Request:**
```json
{
  "github_email": "liker@github.com",
  "system_slug": "claude-code",
  "track_slug": "track-typescript",
  "lesson_slug": "lesson-intro-and-setup",
  "target_email": "submitter@github.com"
}
```

#### `POST /api/v1/report`
Report a submission.

**Request:** Same as like but increments reported_number and sets is_reported.

---

## 4. UI/Routing Changes

### Header Changes
- Remove `<UserButton>` from Clerk
- Show `"Hi, {github_username}"` text instead
- Yellow hover effect (matching storybook `yellowGhost` / accent-yellow)
- Dropdown on click with:
  - **My Profile** → `/settings/profile`
  - **Reading Settings** → `/settings/reading`
  - **My Achievements** → `/settings/achievements`
  - **My Activity** → `/settings/activity`

### Settings Routes

#### `/settings/profile`
- Shows user info from `users` table
- Yellow logout button

#### `/settings/reading`
- Reading settings (font size, theme, etc.)

#### `/settings/achievements`
- Grid of achievements with icons (bronze/silver/gold)

#### `/settings/activity`
- Systems enrolled, tracks enrolled, lessons completed
- Status (submitted/validated) with counts

### System Pages

#### `/systems/[system_slug]`
- Enroll Now button (if not enrolled)
- Already Enrolled + Continue button (if enrolled)
- Single column sequential lessons with black borders
- Track switcher adjacent to enroll button
- Leaderboard on right side (recent activity, max 10)

#### `/systems/[system_slug]/track-[track_slug]/lesson-[lesson_slug]`
- Lesson content display
- Navigation using localStorage
- Popup for incomplete previous lessons
- "View Submissions" button with submissions popup

---

## 5. LocalStorage Strategy

### Key Format
```
{system_slug} → {full API response JSON}
```

### Flow
1. User visits `/systems/claude-code` → calls `GET /api/v1/user_progress/{email}/{slug}`
2. Response stored in localStorage with key `claude-code`
3. Lesson page reads from localStorage
4. If localStorage is empty, makes fresh API call
5. Validation state determined from stored data

---

## 6. CLI Changes

### GitHub Login Enforcement
- Before `init` and `validate`, enforce GitHub login
- Call `POST /api/v1/login` to upsert user

### Remove Octokit
- Remove `@octokit/rest` dependency
- Remove `octokit` from package.json
- Remove all PR automation logic

### Change `submission_pr` to `submission_link`
- CLI submit command takes URL input instead of creating PR
- New fields: `live_link`, `is_reported`, `reported_number`

### Init Command Updates
- Call `POST /api/v1/user_progress` after cloning
- Send default first lesson slug

### Validate Command Updates
- Call `POST /api/v1/user_progress` with `is_validated` field
- Send github_email, system_slug, track_slug, lesson_slug, is_validated
- Server increments positive_validations or negative_validations

---

## 7. Submissions UI

### Lesson Page
- "View Submissions" button at bottom
- Calls `GET /api/v1/submissions/{system}/{track}/{lesson}`
- Shows popup with 1D column layout

### Submission Card
- User email on left
- Like count with heart icon adjacent
- Like button (empty/filled heart)
- Report button
- Click redirects to submission_link
- "View Live" button if live_link exists

### Like/Report APIs
- `POST /api/v1/like` — increments liked_number
- `POST /api/v1/report` — increments reported_number, sets is_reported

---

## 8. Implementation Order

1. ✅ **Write this spec** ← Doing now
2. **Database migration** — ALTER users, CREATE user_progress, push to Turso
3. **Update db.ts** — New helpers for user_progress
4. **New API routes** — v1 API endpoints
5. **Update existing API routes** — CLI routes for new schema
6. **Header component** — Remove Clerk avatar, add username dropdown
7. **Settings routes** — profile, reading, achievements, activity
8. **Systems page redesign** — enrollment, sequential lessons, leaderboard
9. **Lesson viewing** — localStorage navigation, validation popups
10. **CLI updates** — GitHub login enforcement, remove octokit, submission_link
11. **Submissions UI** — View submissions, likes, reports
12. **Review and test** — code-reviewer, typecheck, verify

---

## 9. Design System Tokens Used

Reference: `website/src/presentation/__components/`

### Colors
- `--accent` (#572EFF) — Purple, primary interactive color
- `--accent-yellow` (#facc15) — Yellow, secondary interactive, hover states
- `--accent-bg` (#f0f0ff) — Light purple background
- `--fg` (#0a0a0a) — Primary text
- `--fg-secondary` (#76777d) — Secondary text
- `--fg-muted` (#a3a3a3) — Muted text
- `--border` (#e5e5e5) — Borders
- `--bg-primary` (#ffffff) — Background
- `--bg-secondary` (#f5f5f5) — Secondary background

### Components to use
- `Button` — variants: primary, ghost, ripple, yellowGhost, purpleGhost
- `Dropdown` — borderless bento dropdown with inset shadow
- `Header` — with ghost hover (yellow underline), active = yellow button
- `Footer` — Purple CTA + compact nav
- `AnimatedIcon` — for achievement icons
- `Badge` — for stats/chips
- `TabBar` — for activity page tabs
- `ArticleCard` — for lesson cards

### Button Variants Reference
- `primary`: bg-accent text-white, hover:bg-accent-hover
- `ghost`: transparent, yellow underline on hover
- `ripple`: bg-accent-yellow text-black, ripple effect on click
- `purpleGhost`: transparent, hover:bg-accent
- `yellowGhost`: transparent, hover:bg-accent-yellow hover:text-black
