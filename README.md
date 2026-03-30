# Completing the Migration of Alpha One Labs Learn Platform to Cloudflare Python Workers

## Candidate Information

| Field | Details |
|---|---|
| **Name** | Chinmayee Mada |
| **GitHub** | [@chinnu8055](https://github.com/chinnu8055/) |
| **Email** | madhachinmayee@gmail.com |
| **University** | Anurag University, Hyderabad |
| **Degree** | B.Tech in Computer Science and Engineering |
| **Time Zone** | IST (UTC+5:30) |

---

## About Me

I am a third-year Computer Science and Engineering student at Anurag University. I have worked on full-stack projects using Python, JavaScript, and React, and have experience with backend services like Supabase. I am currently learning Cloudflare Workers and edge-based architectures as part of this project.

**Personal projects:**

- **Healthcare Platform** — A role-based web app supporting patients, doctors, and admins, built with React, TypeScript, and Supabase. Handles appointment scheduling and basic user management.
- **AI Chatbot** — A chatbot that connects to LLM APIs (OpenAI, Groq) through a shared abstraction layer, with conversation history stored in a backend database.
- **Carbon Footprint Tracker** — A React Native mobile app that tracks daily activity inputs and visualises estimated carbon output, with streak-based goals.

**Open-source contributions:**
I have had 9 pull requests merged across the Alpha One Labs repositories. These cover bug fixes, accessibility improvements, a quiz option deletion feature, dark mode fixes, navigation corrections, and an automated leaderboard system with GitHub Actions. Working on these PRs has given me familiarity with how the Django codebase is structured, how the team reviews code, and what the platform's current state looks like in practice.

---

## Project Overview

| Attribute | Value |
|---|---|
| **Project Title** | Completing the Migration of Alpha One Labs Learn Platform to Cloudflare Python Workers |
| **Organisation** | Alpha One Labs |
| **Mentor** | Daniel |
| **Estimated Duration** | ~480 hours over 12 weeks |
| **Difficulty** | Medium–Hard |

### Goal

Complete the migration of the existing Django-based Alpha One Labs platform into the Cloudflare Workers-based `learn` repository, close all remaining feature gaps, harden security, and establish the `learn` repo as the single production system.

---

## Abstract

The Alpha One Labs learning platform is mid-migration from a Django monolith to a Cloudflare Workers-based architecture. The Django repository serves as the canonical feature reference while the `learn` repository — using Cloudflare Python Workers, D1 (SQLite at the edge), and KV storage — is the target system.

The `learn` repo has a working foundation: user authentication, activity and session management, enrollments, and a basic dashboard are already implemented. However, a significant number of features present in the Django system remain unported, and several existing implementations have security issues and performance problems that need to be addressed before the system is production-ready.

This proposal outlines a structured, phased approach to:
1. Harden the existing security model (encryption, token management, Google OAuth, CORS, secrets)
2. Refactor the backend into a maintainable modular structure
3. Port the remaining core features (attendance, profiles, full CRUD, role-based access)
4. Implement community features (forums, peer connections, study groups)
5. Deliver frontend pages for all new functionality
6. Add an AI-powered personalised learning path system as an elective feature

The result will be a complete, secure platform running on Cloudflare's edge network, eliminating the need to maintain the Django backend.

---

## Architecture Direction: Modular Design

Rather than continuing to grow a single large file, the backend will be split into domain modules within the `learn` repository. Non-core features like community tools and the AI learning path system can later be extracted into separate Workers if the team decides that makes sense.
```
┌────────────────────────────────────────────────────────────────┐
│                         Cloudflare Edge                        │
├──────────────────┬──────────────────┬──────────────────────────┤
│   learn (core)   │  community-svc   │      ai-learning-svc     │
│  ─────────────   │  ─────────────   │  ────────────────────    │
│  Auth (+ OAuth)  │  Forums          │  Learning Paths          │
│  Activities      │  Study Groups    │  Quiz Generation         │
│  Sessions        │  Peer Connect    │  Progress Tracking       │
│  Enrollments     │                  │                          │
│  Attendance      │                  │                          │
│  Profiles        │                  │                          │
├──────────────────┴──────────────────┴──────────────────────────┤
│                      Cloudflare D1 (SQLite)                    │
│                      Cloudflare KV (Tokens / OAuth state)      │
└────────────────────────────────────────────────────────────────┘
```

The `learn` repo owns all core platform functionality. Community and AI features start inside `learn` and can be extracted later once stable.

---

## Technical Baseline

### What Is Already Working in `learn`

| Feature | Status |
|---|---|
| User registration and login | ✅ Implemented |
| Activity and session creation | ✅ Implemented |
| Enrollment creation | ✅ Implemented |
| Basic dashboard | ✅ Implemented |
| Static frontend pages (HTML/Tailwind/JS) | ✅ Implemented |

### What Is Missing or Incomplete

| Feature | Gap |
|---|---|
| Encryption | XOR placeholder in use; must be replaced with AES-GCM |
| Google Sign-In | Absent; no OAuth flow exists |
| Token management | No expiry or refresh mechanism |
| Secrets management | Secrets committed in `wrangler.toml` |
| CORS policy | Wildcard — overly permissive |
| Protected endpoints | `/init` and `/seed` accessible without auth |
| Update/delete for Activities & Sessions | Missing CRUD operations |
| Enrollment status workflow | No state machine (pending → approved → completed) |
| Attendance system | Table exists in schema but no handler code touches it |
| User profiles | No view/update/password-change endpoints |
| Role-based access control | No host/admin enforcement |
| Community features | Entirely absent |
| Pagination | No cursor-based pagination on list endpoints |
| N+1 query issues | Tags fetched per activity in a loop |

---

## Phase 1: Security Hardening and Authentication (Weeks 1–3)

Security issues are addressed first because they represent real risks to user data if the system is deployed as-is. This phase also includes Google Sign-In, since it is part of the authentication layer and must be built on top of a secure token system.

Given the scope — AES-GCM migration of existing records, KV-backed token rotation, and a full OAuth 2.0 flow — this phase is allocated three weeks rather than two.

### 1.1 Encryption Upgrade

Replace the XOR placeholder with AES-GCM via the Web Crypto API, which is available natively in the Cloudflare Workers runtime without any external dependency.
```python
# AES-GCM encryption using Web Crypto API
async def encrypt_data(plaintext: str, env) -> str:
    key = await crypto.subtle.importKey(
        "raw",
        env.ENCRYPTION_KEY.encode(),
        {"name": "AES-GCM"},
        False,
        ["encrypt"]
    )
    iv = crypto.getRandomValues(bytes(12))  # 96-bit IV for AES-GCM
    ciphertext = await crypto.subtle.encrypt(
        {"name": "AES-GCM", "iv": iv},
        key,
        plaintext.encode()
    )
    # Prepend IV to ciphertext for storage; IV is not secret
    return base64_encode(iv + ciphertext)

async def decrypt_data(stored: str, env) -> str:
    raw = base64_decode(stored)
    iv, ciphertext = raw[:12], raw[12:]
    key = await crypto.subtle.importKey(
        "raw", env.ENCRYPTION_KEY.encode(),
        {"name": "AES-GCM"}, False, ["decrypt"]
    )
    plaintext = await crypto.subtle.decrypt(
        {"name": "AES-GCM", "iv": iv}, key, ciphertext
    )
    return plaintext.decode()
```

Since this changes how stored user data is encrypted, a dedicated one-time migration endpoint will be included:
```
POST /api/admin/migrate-encryption
Auth: admin Basic Auth + env.ENVIRONMENT != "production" guard
```

This endpoint reads each user record, decrypts with XOR, re-encrypts with AES-GCM, and writes back. It is single-use: once run, the endpoint disables itself by writing a completion flag to KV. This avoids forcing users to re-register while ensuring no manual SQL intervention is needed.

Blind indexes for username/email lookup will also be re-derived using HMAC-SHA256 keyed on a separate `env.BLIND_INDEX_SECRET`, replacing the current approach.

### 1.2 Token Expiry and Refresh

Add `iat` (issued-at) and `exp` (expiry) claims to the existing HMAC-signed token payload. Default TTL is 7 days, configurable via `env.TOKEN_TTL_SECONDS`. Token validation rejects expired tokens before verifying the signature to avoid unnecessary HMAC computation.
```python
def create_token(user_id: str, username: str, role: str, env) -> str:
    now = int(time.time())
    payload = {
        "id": user_id,
        "username": username,
        "role": role,
        "iat": now,
        "exp": now + int(env.TOKEN_TTL_SECONDS)
    }
    return sign_hmac(json.dumps(payload), env.JWT_SECRET)

def validate_token(token: str, env) -> dict:
    payload = verify_hmac(token, env.JWT_SECRET)  # raises if invalid
    if int(time.time()) > payload["exp"]:
        raise TokenExpiredError()
    return payload
```

A `/api/auth/refresh` endpoint accepts a valid non-expired token and returns a new one, resetting the TTL. This avoids requiring users to log in again after expiry during an active session.

### 1.3 Google Sign-In (OAuth 2.0 Authorization Code Flow)

Cloudflare Workers have no access to Node.js libraries like Passport.js, so the OAuth 2.0 flow is implemented directly using `fetch`. The flow works as follows:
```
Browser                    Worker                      Google
   │                          │                            │
   │── GET /auth/google ──────►│                            │
   │                          │── redirect_uri + state ───►│
   │◄── 302 to Google ────────│                            │
   │                          │                            │
   │── (user consents) ───────────────────────────────────►│
   │◄── redirect to /auth/google/callback?code=... ────────│
   │                          │                            │
   │── GET /auth/google/callback ────────────────────────► │
   │                          │── POST /token (code) ─────►│
   │                          │◄── access_token ───────────│
   │                          │── GET /userinfo ──────────►│
   │                          │◄── { email, name, sub } ───│
   │                          │                            │
   │                          │ (find or create user)      │
   │◄── 302 + Set HMAC token ─│                            │
```

**Step 1 — Initiate:**
```
GET /api/auth/google
```
The Worker constructs the Google authorization URL with `client_id`, `redirect_uri`, `scope=openid email profile`, and a `state` parameter (random 32-byte value stored in KV with a 10-minute TTL to prevent CSRF). The user is redirected to Google.

**Step 2 — Callback:**
```
GET /api/auth/google/callback?code=...&state=...
```
The Worker:
1. Verifies the `state` value against KV (deletes it on match — one-time use)
2. Exchanges the `code` for tokens via `POST https://oauth2.googleapis.com/token`
3. Fetches user info from `https://www.googleapis.com/oauth2/v3/userinfo`
4. Looks up the user by `google_sub` (Google's stable user ID) in D1
5. If found: issues a platform HMAC token and redirects to dashboard
6. If not found: checks if the email already exists (account linking), or creates a new user record with a null password field (Google-only accounts cannot use password login)
7. Redirects to the frontend with the token as a short-lived query param or sets it via a secure cookie

**Schema change:**
```sql
ALTER TABLE users ADD COLUMN google_sub TEXT;
ALTER TABLE users ADD COLUMN password_hash TEXT; -- nullable for OAuth-only accounts
CREATE UNIQUE INDEX idx_users_google_sub ON users(google_sub) WHERE google_sub IS NOT NULL;
```

**Security considerations:**
- `state` is verified before any token exchange to prevent CSRF
- `redirect_uri` is hardcoded in `env.GOOGLE_REDIRECT_URI` (not taken from the request)
- Google client secret is stored in Wrangler managed secrets only
- OAuth-only accounts are clearly marked; password login is rejected for them

### 1.4 Secrets Management

All credentials will be moved to Cloudflare Workers Secrets (`wrangler secret put`) and accessed via `env.*` bindings at runtime. The `[vars]` block in `wrangler.toml` will hold only non-sensitive configuration (e.g., `ENVIRONMENT`, `TOKEN_TTL_SECONDS`). A `.env.example` file will document every required secret name without values.

Secrets required:
- `ENCRYPTION_KEY` — 32-byte key for AES-GCM
- `BLIND_INDEX_SECRET` — separate key for HMAC blind indexes
- `JWT_SECRET` — HMAC signing key for platform tokens
- `PEPPER` — password hashing pepper
- `GOOGLE_CLIENT_ID` and `GOOGLE_CLIENT_SECRET`
- `ADMIN_PASSWORD` — for Basic Auth admin routes

### 1.5 CORS Hardening

Replace the wildcard `Access-Control-Allow-Origin: *` with an explicit allowlist read from `env.ALLOWED_ORIGINS` (comma-separated). The response echoes back the request's `Origin` header only if it appears in the allowlist.

### 1.6 Bootstrap Endpoint Protection

`/api/init` and `/api/seed` will require admin Basic Auth. In production (`env.ENVIRONMENT == "production"`), the seed endpoint returns `403` unconditionally.

**Phase 1 Deliverables:**
- AES-GCM encryption with one-time migration utility
- Token expiry and refresh endpoint
- Google Sign-In via OAuth 2.0 Authorization Code flow
- Account linking for existing email accounts
- No secrets in source code or `wrangler.toml`
- CORS restricted to trusted origins
- `/api/init` and `/api/seed` protected

---

## Phase 2: Backend Modularization (Week 4)

The current `worker.py` is a single large file. As Phase 1 adds OAuth routes and crypto helpers, this will become unmanageable. This phase restructures the code into domain modules before further feature work begins.

### Proposed File Structure
```
src/
├── worker.py               # Entry point: on_fetch → dispatcher only
├── router.py               # Route table registration
├── auth/
│   ├── handlers.py         # register, login, refresh, google OAuth
│   ├── middleware.py       # require_auth, require_role decorators
│   └── crypto.py           # AES-GCM, HMAC, PBKDF2 wrappers
├── activities/
│   ├── handlers.py         # list, create, get, update, delete
│   └── queries.py          # all D1 queries for activities domain
├── sessions/
│   ├── handlers.py
│   └── queries.py
├── enrollments/
│   ├── handlers.py
│   └── queries.py
├── attendance/
│   ├── handlers.py
│   └── queries.py
├── profiles/
│   ├── handlers.py
│   └── queries.py
├── community/
│   ├── handlers.py
│   └── queries.py
└── shared/
    ├── db.py               # D1 query helpers
    ├── response.py         # success/error JSON wrappers
    ├── validation.py       # input validation and sanitisation
    └── pagination.py       # cursor-based pagination utilities
```

This is a pure structural refactor — no behavior changes. All existing routes are verified against their expected responses before and after using the regression test suite established in Phase 1.

**Deliverables:**
- Codebase split into domain modules
- Shared utilities extracted
- All existing routes passing after refactor

---

## Phase 3: Query Performance and Pagination (Week 5)

### 3.1 Eliminating N+1 Queries

The current activity listing handler fetches tags for each activity in a Python loop — one extra D1 query per activity. At 50 activities this means 51 round-trips per page load. The fix uses a single aggregated JOIN with D1's `JSON_GROUP_ARRAY`:
```sql
-- Before: N+1 — one query per activity for its tags
-- After: single query, tags returned as JSON array per row

SELECT
  a.id, a.title, a.description, a.status,
  a.host_id, a.created_at,
  COALESCE(
    JSON_GROUP_ARRAY(t.name) FILTER (WHERE t.name IS NOT NULL),
    '[]'
  ) AS tags
FROM activities a
LEFT JOIN activity_tags at ON at.activity_id = a.id
LEFT JOIN tags t ON t.id = at.tag_id
WHERE a.status = 'published'
GROUP BY a.id
ORDER BY a.created_at DESC
LIMIT ? OFFSET ?;
```

The handler parses the `tags` field as JSON. The same aggregation pattern is applied to dashboard queries, which have the same problem.

Similarly, the enrollment dashboard currently joins sessions and activities in a loop. This is collapsed into:
```sql
SELECT
  e.id, e.status, e.enrolled_at,
  s.id AS session_id, s.title AS session_title,
  a.id AS activity_id, a.title AS activity_title
FROM enrollments e
JOIN sessions s ON e.session_id = s.id
JOIN activities a ON s.activity_id = a.id
WHERE e.user_id = ?
ORDER BY e.enrolled_at DESC;
```

### 3.2 Cursor-Based Pagination

All list endpoints will support cursor-based pagination. The cursor is the `id` of the last seen record; queries use `WHERE id > ?` rather than `OFFSET`, which is stable under concurrent inserts and avoids the performance degradation of large offsets.
```
GET /api/activities?limit=20&cursor=act_01j8x...&tag=python&search=machine+learning

Response:
{
  "data": [...],
  "pagination": {
    "next_cursor": "act_01j9y...",
    "has_more": true
  }
}
```

A shared `paginate(query, params, limit, cursor, env)` utility in `shared/pagination.py` handles cursor injection and `has_more` detection consistently across all list handlers.

**Deliverables:**
- No N+1 queries in any list endpoint
- Cursor-based pagination on all list endpoints
- Search and filter support on activity listing (by tag, host, date range)

---

## Phase 4: Core Feature Completion (Weeks 6–8)

### 4.1 Activities and Sessions — Full CRUD

| Endpoint | Method | Auth | Description |
|---|---|---|---|
| `/api/activities/:id` | `PUT` | JWT, must be host | Update activity metadata |
| `/api/activities/:id` | `DELETE` | JWT, must be host | Soft-delete (sets `status = 'deleted'`) |
| `/api/sessions/:id` | `PUT` | JWT, must be activity host | Update session details |
| `/api/sessions/:id` | `DELETE` | JWT, must be activity host | Soft-delete session |
| `/api/activities/:id/tags/:tag_id` | `DELETE` | JWT, must be host | Remove a tag from activity |

Soft-delete is used rather than hard-delete so that existing enrollment and attendance records remain valid. Deleted activities are excluded from public listing but visible to the host in their dashboard with a `deleted` status badge.

All mutations verify ownership with a single query before executing the change:
```python
async def assert_activity_owner(activity_id: str, user_id: str, env):
    row = await db.query_one(
        "SELECT id FROM activities WHERE id = ? AND host_id = ?",
        [activity_id, user_id], env
    )
    if not row:
        raise ForbiddenError("You do not own this activity")
```

### 4.2 Enrollment Status Workflow

The Django system enforces a state machine for enrollment. This is ported as server-side transition guards:
```
[PENDING] ──(host approves)──► [APPROVED] ──(session ends)──► [COMPLETED]
    │                                │
    └──(host rejects / user cancels)─┴──► [CANCELLED]
```

Valid transitions are defined as a dictionary and checked before any status update:
```python
VALID_TRANSITIONS = {
    "pending":   ["approved", "rejected", "cancelled"],
    "approved":  ["completed", "cancelled"],
    "rejected":  [],
    "completed": [],
    "cancelled": [],
}

def assert_valid_transition(current: str, next: str):
    if next not in VALID_TRANSITIONS.get(current, []):
        raise ValidationError(f"Cannot transition from {current} to {next}")
```

### 4.3 Attendance System

The `session_attendance` table already exists in the schema but has no handler code. Three endpoints are added:

| Endpoint | Method | Auth | Description |
|---|---|---|---|
| `POST /api/sessions/:id/attendance` | Host only | Submit attendance list for a session |
| `GET /api/sessions/:id/attendance` | Host or enrolled | View attendance for a session |
| `GET /api/users/me/attendance` | JWT (own records) | Own attendance history |

The POST body accepts a batch:
```json
{
  "records": [
    { "user_id": "usr_01...", "status": "present" },
    { "user_id": "usr_02...", "status": "absent" }
  ]
}
```

Records are inserted with `INSERT OR REPLACE` to make the operation idempotent — hosts can resubmit corrected attendance without creating duplicates.

### 4.4 User Profiles

| Endpoint | Method | Description |
|---|---|---|
| `GET /api/users/me` | JWT | Full profile including role, enrollment count |
| `PUT /api/users/me` | JWT | Update display name, bio |
| `POST /api/users/me/password` | JWT | Change password (requires current password; blocked for Google-only accounts) |
| `GET /api/users/:id` | Public | Public profile: display name, bio, hosted activities |

### 4.5 Role-Based Access Control

Currently, any user who calls a host endpoint becomes a de facto host. The fix introduces explicit role enforcement via middleware:
```python
def require_role(*roles):
    def decorator(handler):
        async def wrapper(request, env, ctx, user):
            if user["role"] not in roles:
                return error_response("Forbidden", 403)
            return await handler(request, env, ctx, user)
        return wrapper
    return decorator

# Usage
@require_auth
@require_role("host", "admin")
async def create_session(request, env, ctx, user):
    ...
```

Role assignment is admin-controlled via:
```
POST /api/users/me/role-request   { requested_role: "host" }
PATCH /api/admin/role-requests/:id { status: "approved" | "rejected" }
```

**Phase 4 Deliverables:**
- Full CRUD for activities and sessions with ownership enforcement
- Enrollment state machine with transition guards
- Attendance batch submission and retrieval
- Profile management endpoints
- RBAC middleware applied to all protected routes

---

## Phase 5: Community Features (Weeks 9–10)

Community features will be built inside the `learn` repo and can be extracted to `community-svc` after the GSoC period once stable.

### 5.1 Forums

- Threads: create, read, update (own), soft-delete (own or moderator)
- Replies: up to 2 levels of nesting
- Categories: activity-scoped or global
- Moderation: hosts can pin or close threads within their activity

### 5.2 Peer Connections

- Send, accept, reject connection requests
- List connected peers
- View peer's public profile
- Unique constraint prevents duplicate requests in either direction

### 5.3 Study Groups

- Create groups linked to an activity or standalone
- Invite by user ID, accept/decline invite
- Leave group; creator can remove members
- Member list with roles (creator, member)

**Schema additions:**
```sql
CREATE TABLE forum_threads (
    id TEXT PRIMARY KEY,
    author_id TEXT NOT NULL REFERENCES users(id),
    activity_id TEXT REFERENCES activities(id),
    title TEXT NOT NULL,
    body TEXT NOT NULL,
    is_pinned INTEGER DEFAULT 0,
    is_closed INTEGER DEFAULT 0,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL
);

CREATE TABLE forum_replies (
    id TEXT PRIMARY KEY,
    thread_id TEXT NOT NULL REFERENCES forum_threads(id),
    parent_reply_id TEXT REFERENCES forum_replies(id), -- for 2-level nesting
    author_id TEXT NOT NULL REFERENCES users(id),
    body TEXT NOT NULL,
    created_at TEXT NOT NULL
);

CREATE TABLE peer_connections (
    id TEXT PRIMARY KEY,
    requester_id TEXT NOT NULL REFERENCES users(id),
    addressee_id TEXT NOT NULL REFERENCES users(id),
    status TEXT NOT NULL CHECK(status IN ('pending','accepted','rejected')),
    created_at TEXT NOT NULL,
    UNIQUE(requester_id, addressee_id)
);

CREATE TABLE study_groups (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    creator_id TEXT NOT NULL REFERENCES users(id),
    activity_id TEXT REFERENCES activities(id),
    created_at TEXT NOT NULL
);

CREATE TABLE study_group_members (
    group_id TEXT NOT NULL REFERENCES study_groups(id),
    user_id TEXT NOT NULL REFERENCES users(id),
    role TEXT DEFAULT 'member',
    joined_at TEXT NOT NULL,
    PRIMARY KEY(group_id, user_id)
);
```

**Deliverables:**
- Forum CRUD with moderation controls
- Peer connection flow with duplicate prevention
- Study group management with invite/accept/leave
- Frontend pages for all community features

---

## Phase 6: Frontend Development (Weeks 6–11, parallel)

Frontend work runs in parallel with backend development. The UI continues using HTML, Tailwind CSS, and vanilla JavaScript, consistent with the existing `learn` frontend. No build step is required — assets are served directly from Cloudflare Workers Assets.

### New Pages and Components

| Feature | Page / Component |
|---|---|
| Google Sign-In | Login page button → OAuth redirect; callback handling |
| Attendance marking | Host-facing checklist per session with batch submit |
| Attendance history | User view showing status per session attended |
| User profile | View and edit form; password change (disabled for Google accounts) |
| Enrollment management | Status badges; approve/reject buttons for host |
| Activity/session edit | Edit forms pre-populated with existing data |
| Forums | Thread list, thread detail, reply composer, moderation controls |
| Peer connections | Search by username, request/accept/reject cards |
| Study groups | Group list, detail view, member management |

### Design Principles

- Core functionality works without JavaScript where possible
- ARIA labels and keyboard navigation on all interactive elements
- Mobile-first layouts, consistent with existing UI patterns
- Auth token stored in `sessionStorage` instead of `localStorage` to reduce XSS blast radius (improvement over current implementation)

---

## Phase 7: Elective Feature — AI-Powered Learning Path System (Weeks 11–12)

This feature is built only after migration work is complete. It will be reduced in scope or deferred if earlier phases run over time.

### What It Does

A user provides a topic, difficulty level, and available time. The system generates a sequential learning path where each node must be passed (via a quiz) before the next unlocks. Users can also upload their own notes as the source material, in which case the AI extracts the syllabus from their content rather than generating it from scratch.
```
Topic Input ──► LLM Generates Syllabus ──► User Reviews & Edits ──► Confirm
                                                                         │
                                                             ┌───────────┘
                                                             ▼
                                                  Node 1: Content + Resources
                                                             │
                                                   Pass quiz? ──No──► Retry
                                                             │ Yes
                                                             ▼
                                                  Node 2: Next topic…
                                                             │
                                                             ▼
                                                  Completion summary
```

### Content Sources

| Source | Purpose |
|---|---|
| LLM (configurable provider) | Syllabus generation, explanations, quiz questions |
| Wikipedia REST API | Factual summary per topic node (no auth required) |
| YouTube Data API v3 | Up to 2 relevant videos per node |
| User-uploaded notes | Alternative source material for syllabus extraction |

### LLM Provider Abstraction

Rather than coupling to a single provider, the client is a thin abstraction over the shared chat completions API format (used by OpenAI, Groq, Together AI, and others):
```python
class LLMProvider:
    base_url: str
    api_key: str
    model: str

    async def generate(self, system: str, prompt: str) -> str:
        response = await fetch(f"{self.base_url}/chat/completions", {
            "method": "POST",
            "headers": {
                "Authorization": f"Bearer {self.api_key}",
                "Content-Type": "application/json"
            },
            "body": json.dumps({
                "model": self.model,
                "messages": [
                    {"role": "system", "content": system},
                    {"role": "user", "content": prompt}
                ],
                "temperature": 0.3
            })
        })
        data = await response.json()
        return data["choices"][0]["message"]["content"]

def get_provider(env) -> LLMProvider:
    configs = {
        "openai":   ("https://api.openai.com/v1", env.OPENAI_API_KEY, env.LLM_MODEL),
        "groq":     ("https://api.groq.com/openai/v1", env.GROQ_API_KEY, env.LLM_MODEL),
        "together": ("https://api.together.xyz/v1", env.TOGETHER_API_KEY, env.LLM_MODEL),
    }
    base_url, api_key, model = configs[env.LLM_PROVIDER]
    return LLMProvider(base_url, api_key, model)
```

Switching providers requires only an environment variable change — no code changes.

All prompts instruct the model to return strict JSON only. Responses are validated against an expected schema before being stored. If validation fails, the request is retried once with an explicit correction prompt before returning an error to the user.

### User-Uploaded Notes Processing

When a user uploads their own notes, the content is extracted and processed through the LLM to derive the syllabus rather than generating it from scratch. This is the most technically involved part of the feature:
```
Raw notes (text or PDF)
        │
        ▼
  Text extraction
  (PDF: pdfminer via Workers; plain text: direct)
        │
        ▼
  Chunk into ~400-token segments
  (split on paragraph breaks; avoid mid-sentence cuts)
        │
        ▼
  For each chunk → LLM prompt:
  "Identify the main topic and list 2-3 key concepts.
   Return JSON: { topic: str, concepts: [str] }"
        │
        ▼
  Deduplicate and order topics
  (group by semantic similarity using LLM, not embeddings)
        │
        ▼
  Construct syllabus nodes from extracted topics
        │
        ▼
  Present to user for review and editing
  before path is activated
```

The chunking approach deliberately avoids vector embeddings (which would require Vectorize and add cost) in favour of direct LLM calls per chunk, which is simpler and sufficient for the document sizes this use case involves (personal notes, not textbooks).

### Enrichment Pipeline

After a node's content is generated, Wikipedia and YouTube enrichment is fetched in parallel:
```python
async def enrich_node(title: str, env) -> dict:
    wiki_task = fetch_wikipedia_summary(title)
    youtube_task = search_youtube(title, env.YOUTUBE_API_KEY, max_results=2)
    wiki, videos = await asyncio.gather(wiki_task, youtube_task)
    return {"wikipedia": wiki, "videos": videos}
```

Wikipedia uses the public `/api/rest_v1/page/summary/{title}` endpoint (no API key needed). YouTube uses the Data API v3 `search.list` with `type=video&relevanceLanguage=en`. Both results are stored in `enrichment_json` on the node record and served from there on subsequent requests — no re-fetching on every page load.

### Quiz Generation

After a user marks a node complete, 3–5 multiple-choice questions are generated from the node's content. Questions, options, and correct answers are stored server-side. The user submits their answers; the server scores them (correct answers are never sent to the client).

Failing a quiz triggers a retry with a freshly generated question set to prevent answer memorisation:
```python
async def generate_quiz(node_content: str, llm: LLMProvider) -> list:
    system = "You are a quiz writer. Return ONLY valid JSON. No markdown."
    prompt = f"""
    Based on this content, generate 4 multiple-choice questions.
    Return: {{
      "questions": [{{
        "question": str,
        "options": [str, str, str, str],
        "correct_index": int
      }}]
    }}
    Content: {node_content}
    """
    raw = await llm.generate(system, prompt)
    return validate_quiz_schema(json.loads(raw))
```

### Data Model
```sql
CREATE TABLE learning_paths (
    id TEXT PRIMARY KEY,
    user_id TEXT NOT NULL REFERENCES users(id),
    topic TEXT NOT NULL,
    difficulty TEXT CHECK(difficulty IN ('beginner','intermediate','advanced')),
    mode TEXT CHECK(mode IN ('learn','revise')),
    duration_hours INTEGER,
    syllabus_json TEXT,       -- full generated/extracted syllabus
    status TEXT NOT NULL CHECK(status IN ('draft','active','completed')),
    created_at TEXT NOT NULL
);

CREATE TABLE path_nodes (
    id TEXT PRIMARY KEY,
    path_id TEXT NOT NULL REFERENCES learning_paths(id),
    position INTEGER NOT NULL,
    title TEXT NOT NULL,
    content_json TEXT,        -- LLM-generated explanations and key points
    enrichment_json TEXT,     -- Wikipedia summary + YouTube video links
    duration_minutes INTEGER,
    status TEXT DEFAULT 'locked' CHECK(status IN ('locked','unlocked','completed'))
);

CREATE TABLE quiz_attempts (
    id TEXT PRIMARY KEY,
    node_id TEXT NOT NULL REFERENCES path_nodes(id),
    user_id TEXT NOT NULL REFERENCES users(id),
    questions_json TEXT NOT NULL,  -- stored server-side; never sent to client
    answers_json TEXT,
    score INTEGER,
    passed INTEGER DEFAULT 0,
    attempted_at TEXT NOT NULL
);
```

**Deliverables (if time permits):**
- Provider-agnostic LLM client supporting at least two backends
- Syllabus generation from topic prompt with user review step
- Notes upload and chunked topic extraction
- Node content generation with Wikipedia and YouTube enrichment
- Quiz generation with server-side scoring and retry on failure
- Frontend: topic form, syllabus review, node viewer, quiz widget, progress bar

---

## Detailed Timeline

### Community Bonding Period

- Map every existing route in `learn` against every Django view — produce a gap document shared with mentor
- Set up local development environment: `wrangler dev` with a local D1 instance and seed data
- Write smoke tests for all currently working endpoints as a regression baseline
- Draft all schema additions (OAuth columns, community tables, AI tables) and get mentor sign-off before writing any code
- Research Cloudflare Workers Python runtime limitations that could affect any planned implementation; document findings

### Week 1: Encryption Migration

| Days | Task |
|---|---|
| 1–2 | Implement AES-GCM encrypt/decrypt using `crypto.subtle`; unit tests for crypto layer |
| 3–4 | Implement HMAC-SHA256 blind index regeneration |
| 5–7 | Build and test one-time admin re-encryption endpoint; verify existing user records survive round-trip |

**Milestone:** AES-GCM encryption live; migration utility tested.

### Week 2: Token Security and Secrets

| Days | Task |
|---|---|
| 1–2 | Add `iat`/`exp` to token payload; update validation to reject expired tokens |
| 3–4 | Implement `/api/auth/refresh` endpoint; write expiry/refresh integration tests |
| 5–6 | Move all secrets to Wrangler managed secrets; remove from `wrangler.toml` |
| 7 | CORS allowlist; protect `/api/init` and `/api/seed` |

**Milestone:** Token expiry and refresh working; no secrets in source.

### Week 3: Google Sign-In

| Days | Task |
|---|---|
| 1–2 | Schema changes for `google_sub`; OAuth initiation endpoint with KV state storage |
| 3–4 | OAuth callback: token exchange, userinfo fetch, find-or-create user logic |
| 5–6 | Account linking for existing email addresses; block password login for OAuth-only accounts |
| 7 | Frontend: Google Sign-In button on login page; callback handling; integration tests |

**Milestone:** Google Sign-In fully functional; CSRF protection verified.

### Week 4: Backend Modularization

| Days | Task |
|---|---|
| 1–3 | Split `worker.py` into auth, activities, sessions, enrollments modules |
| 4–5 | Extract shared utilities (db, response, validation, pagination) |
| 6–7 | Community and profiles module stubs; full regression test against all existing routes |

**Milestone:** Modular structure complete; all existing routes passing.

### Week 5: Query Performance and Pagination

| Days | Task |
|---|---|
| 1–2 | Fix N+1 tag queries with `JSON_GROUP_ARRAY` aggregation |
| 3–4 | Fix N+1 enrollment dashboard queries with JOINs |
| 5–7 | Cursor-based pagination utility; apply to all list endpoints; search and filter on activities |

**Milestone:** No N+1 queries; all list endpoints paginated.

### Week 6: Activities, Sessions, Enrollments — Full CRUD

| Days | Task |
|---|---|
| 1–2 | `PUT` and `DELETE` for activities with ownership check and soft-delete |
| 3–4 | `PUT` and `DELETE` for sessions; tag deletion endpoint |
| 5–7 | Enrollment state machine with transition guards; `PATCH /api/enrollments/:id/status` |

**Milestone:** Full CRUD for activities, sessions, and enrollment transitions.

### Week 7: Attendance and Profiles

| Days | Task |
|---|---|
| 1–3 | Attendance batch submission (`POST`), session view (`GET`), user history (`GET`) |
| 4–5 | Profile endpoints: view, update, public profile |
| 6–7 | Password change endpoint; block for Google-only accounts; frontend profile page |

**Milestone:** Attendance and profiles complete.

### Week 8: RBAC and Buffer

| Days | Task |
|---|---|
| 1–3 | `require_role` middleware; apply to all host/admin routes |
| 4–5 | Role request and admin approval endpoints; frontend role-aware components |
| 6–7 | Buffer: clear any slipped items from weeks 6–7; expand integration test coverage |

**Midterm Milestone:** Core migration complete. `learn` deployable as Django replacement. All security, CRUD, attendance, profiles, and RBAC in place.

### Week 9: Community — Forums

| Days | Task |
|---|---|
| 1–2 | D1 schema for threads and replies; thread CRUD with ownership checks |
| 3–4 | Categories; moderation controls (pin, close); reply nesting |
| 5–7 | Frontend: thread list, thread detail, reply composer |

### Week 10: Community — Peers and Study Groups

| Days | Task |
|---|---|
| 1–3 | Peer connection request/accept/reject API; duplicate prevention; frontend |
| 4–7 | Study group CRUD; invite/accept/leave; member management; frontend |

**Milestone:** All community features complete and tested.

### Week 11: AI Learning Path — Backend

| Days | Task |
|---|---|
| 1–2 | LLM provider abstraction; D1 schema; syllabus generation endpoint |
| 3–4 | Notes upload; chunking and per-chunk LLM topic extraction |
| 5–6 | Node content generation; Wikipedia and YouTube enrichment (parallel fetch) |
| 7 | Quiz generation endpoint; server-side scoring; pass/fail gating |

### Week 12: AI Learning Path — Frontend, Testing, Documentation

| Days | Task |
|---|---|
| 1–3 | Frontend: topic form, syllabus review UI, node viewer with enrichment, quiz widget |
| 4–5 | Progress bar; resume functionality; retry on quiz fail with new question set |
| 6–7 | Full end-to-end integration tests; OpenAPI documentation for all endpoints; deployment guide |

**Final Milestone:** Full platform on Cloudflare Workers; AI learning path system operational; documentation complete.

---

## Success Metrics

| Metric | Target |
|---|---|
| Feature parity with Django backend | 100% of identified features ported |
| Route handler test coverage | ≥ 80% |
| N+1 query elimination | 0 N+1 queries in any list endpoint |
| API response time (p95 at edge) | < 100 ms |
| Security issues | Zero critical or high findings in final review |
| Documentation | All endpoints documented with request/response examples |

---

## Testing Strategy

Cloudflare Python Workers do not yet have a mature testing framework, which requires a layered approach tailored to what can and cannot run outside the Worker runtime.

**Unit tests (pure Python, run with `pytest` locally):**
These cover logic that has no dependency on the Workers runtime: state machine transition guards, pagination cursor logic, input validation functions, and JSON schema validators for LLM responses. The AES-GCM crypto layer cannot be unit-tested with `pytest` directly because `crypto.subtle` is a Workers API — this is handled by mocking the crypto interface with a Python equivalent (using the `cryptography` library) for unit test purposes, with the real implementation verified in integration tests.

**Integration tests (run against `wrangler dev` with local D1):**
Route handlers are tested end-to-end by pointing an HTTP client (`httpx` in async mode) at a locally running Worker instance. Each test starts with a known database state (seeded via the admin seed endpoint), exercises a route, and asserts on the response body and status code. These tests cover auth flows, ownership checks, state machine transitions, and pagination behaviour. The Google OAuth callback is tested using a mocked `fetch` that returns a fixed userinfo payload.

**Regression suite (runs on every PR via GitHub Actions):**
A smoke-test suite hits every existing route with a valid request and asserts a non-5xx response. This ensures that refactoring and new features do not silently break working functionality. The suite runs in under 60 seconds against `wrangler dev` to keep PR feedback fast.

---

## Risk Assessment and Mitigation

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Cloudflare Python Workers runtime missing a needed API (e.g., PDF parsing) | Medium | Medium | Identify during Community Bonding; for PDF notes, fall back to plain text extraction or instruct users to paste text directly |
| AES-GCM re-encryption migration fails on edge cases in existing data | Low | High | Migration endpoint is admin-only, single-use, and tested against a copy of production data before running; original records are not deleted until re-encryption is confirmed |
| Google OAuth `redirect_uri` mismatch in production vs development | Medium | Low | `redirect_uri` is stored in `env.GOOGLE_REDIRECT_URI` per environment; verified during Week 3 testing |
| LLM API rate limits block AI feature development | Medium | Low | AI feature is elective; Groq's free tier is sufficient for development; LLM responses are mocked in tests |
| Scope creep from Django feature audit | Medium | Medium | Prioritised backlog maintained with mentor; cosmetic features deferred in favour of functional parity |
| Midterm deadline pressure | Low | Medium | Week 8 includes an explicit buffer; midterm scope is defined conservatively at the end of Week 7 |

---

## Benefits to the Project and Community

| Stakeholder | Benefit |
|---|---|
| **Alpha One Labs** | Single deployable system on Cloudflare edge; no Django infrastructure to maintain; Google Sign-In reduces registration friction |
| **Platform users** | Faster response times from edge deployment; more complete feature set; familiar sign-in option |
| **Contributors** | Modular, documented codebase with clear domain boundaries is easier to contribute to |
| **Future maintainers** | Test coverage and documentation reduce onboarding time |

---

## Previous Contributions to Alpha One Labs

I have had **9 pull requests merged** across the Alpha One Labs repositories:

| PR | Repository | Description | Area |
|---|---|---|---|
| [#992](https://github.com/alphaonelabs/education-website/pull/992) | education-website | Resolved duplicate messaging interfaces | Frontend / UX |
| [#982](https://github.com/alphaonelabs/education-website/pull/982) | education-website | Fixed virtual lab navigation and duplicated Chemistry URL | Routing |
| [#869](https://github.com/alphaonelabs/education-website/pull/869) | education-website | Fixed duplicate locale in Top Contributors link | i18n / Templates |
| [#863](https://github.com/alphaonelabs/education-website/pull/863) | education-website | Updated GSoC '25 link to 2026 in footer | Templates |
| [#843](https://github.com/alphaonelabs/education-website/pull/843) | education-website | Added delete functionality for quiz options | Feature / Backend |
| [#818](https://github.com/alphaonelabs/education-website/pull/818) | education-website | Fixed dark mode colour inconsistencies | CSS / Theming |
| [#807](https://github.com/alphaonelabs/education-website/pull/807) | education-website | Fixed broken Discord and Slack links in footer | Templates |
| [#791](https://github.com/alphaonelabs/education-website/pull/791) | education-website | Improved discoverability of horizontal scrolling on enrollment table | Accessibility |
| [#7](https://github.com/alphaonelabs/gsoc/pull/7) | gsoc | Functional Leaderboard — added dynamic 2026 tab and static 2025 dataset; automated leaderboard generation via GitHub Actions workflow that fetches PR stats and commits updated JSON on a schedule | Feature / Automation |

The leaderboard PR is in a different repository (`alphaonelabs/gsoc`) and involves more than a template change — it includes a Python script for generating leaderboard data from GitHub's API, a GitHub Actions workflow for scheduled regeneration, and frontend changes to load data dynamically from the generated JSON rather than from hardcoded markup.

---

## Why This Project

I have contributed to the Alpha One Labs Django repository across several months, so I am not approaching this migration from a cold read of the codebase. I have explored how the data models are structured and how Django views handle business logic, and I am continuing to deepen my understanding as I work on this migration.

The migration itself is the part I find worth working on. Moving from Django's ORM and synchronous request model to D1's direct SQL interface and Cloudflare Workers' async model is not a mechanical process — it involves deliberate design decisions at each step, and mistakes in the early phases (security, schema) are expensive to fix later. That sequence of decisions is what makes this more interesting than adding a standalone feature.

Google Sign-In is included because it directly serves new users — eliminating the friction of creating yet another account is a real product improvement, not just a checkbox. Building it without a library on a platform that does not support Node.js means the implementation has to be understood end-to-end, which is a reasonable challenge for a GSoC project.

The AI learning path feature is secondary and I will treat it that way. I have worked with LLM APIs and basic integrations before, and I plan to build this feature in a structured and incremental way. But if the migration runs long, the AI feature is the first thing to be cut.

---

## Availability and Communication

- **Availability:** ~40 hours per week during the GSoC period
- **Other commitments:** No internship, part-time job, or academic overlap during this period
- **Communication:** Weekly written status updates; GitHub issues and PR comments for async discussion; mentor syncs as agreed
- **If behind schedule:** AI feature is deprioritised first. Migration is the primary deliverable and will not be sacrificed for the bonus feature.

---

## Post-GSoC Plans

After GSoC I plan to continue contributing. Areas I expect to work on:

- Extracting community features into a standalone `community-svc` Worker once they are stable in `learn`
- Extending the AI learning path system — spaced repetition, instructor-created templates, progress analytics
- Adding structured logging via Cloudflare Analytics Engine for production observability
- Reviewing PRs and helping onboard new contributors to the migrated codebase
