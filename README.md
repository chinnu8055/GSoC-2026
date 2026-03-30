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

I am a third-year Computer Science and Engineering student at Anurag University with hands-on experience in full-stack development, serverless architectures, and open-source contribution. My technical stack includes **React**, **TypeScript**, **Python**, **Supabase**, and **Cloudflare Workers**, enabling me to work across both the frontend and backend layers of modern web applications.

**Selected personal projects:**

- **Healthcare Platform** — A role-based access control system supporting patients, doctors, and admins, built with React, TypeScript, and Supabase. Handles appointment scheduling, medical records, and user management.
- **AI Chatbot** — Integrated multiple LLM providers (OpenAI, Groq) via a unified API abstraction layer, with conversation history stored in a persistent backend.
- **Carbon Footprint Tracker (Mobile)** — A React Native application that calculates and visualises a user's carbon footprint based on daily activity inputs, featuring streak tracking and gamified goals.

**Open-source experience:**  
I have been an active contributor to the Alpha One Labs repository with **8 merged pull requests** spanning UI bug fixes, accessibility improvements, feature additions (quiz management, dark mode, leaderboard), and navigation consistency fixes. These contributions have given me a thorough understanding of the existing codebase structure, review processes, and the team's engineering standards—directly relevant to the work proposed here.

---

## Project Overview

| Attribute | Value |
|---|---|
| **Project Title** | Completing the Migration of Alpha One Labs Learn Platform to Cloudflare Python Workers |
| **Organisation** | Alpha One Labs |
| **Estimated Duration** | ~480 hours over 12 weeks |
| **Difficulty** | Medium–Hard |

### Goal

Complete the migration of the existing Django-based Alpha One Labs platform into the Cloudflare Workers-based `learn` repository, close all remaining feature gaps, harden security, and establish the `learn` repo as the single production system.

---

## Abstract

The Alpha One Labs learning platform is mid-migration from a Django monolith to a Cloudflare Workers-based architecture. The Django repository serves as the canonical feature reference while the `learn` repository—using Cloudflare Workers with Python, D1 (SQLite at the edge), and KV storage—is the target system.

The `learn` repo has a working foundation: user authentication, activity and session management, enrollments, and a basic dashboard are already implemented. However, a significant number of features present in Django remain unported, and several existing implementations require security hardening and performance optimisation before the system can be considered production-ready.

This proposal details a structured, phased approach to:
1. Harden the existing security model (encryption, token management, CORS, secrets)
2. Refactor the backend into a maintainable modular architecture
3. Port the remaining core features (attendance, profiles, full CRUD, role-based access)
4. Implement community features (forums, peer connections, study groups)
5. Deliver a polished frontend for all new functionality
6. Optionally build an AI-powered personalised learning path system

The result will be a complete, secure, and performant platform running entirely on Cloudflare's edge network—eliminating the need for the Django backend and giving Alpha One Labs a scalable foundation for future development.

---

## Architecture Direction: Modular Repository Design

### Design Principle

Rather than extending an already-large monolith, the platform will adopt a **modular, multi-repository architecture** where each functional domain lives in an independently deployable Cloudflare Worker. All modules communicate through well-defined API contracts and share a single Cloudflare D1 database (with schema ownership per module).

```
┌────────────────────────────────────────────────────────────────┐
│                         Cloudflare Edge                        │
├──────────────────┬──────────────────┬──────────────────────────┤
│   learn (core)   │  community-svc   │      ai-learning-svc     │
│  ─────────────   │  ─────────────   │  ────────────────────    │
│  Auth            │  Forums          │  Learning Paths          │
│  Activities      │  Study Groups    │  Quiz Generation         │
│  Sessions        │  Peer Connect    │  Progress Tracking       │
│  Enrollments     │                  │                          │
│  Attendance      │                  │                          │
│  Profiles        │                  │                          │
├──────────────────┴──────────────────┴──────────────────────────┤
│                      Cloudflare D1 (SQLite)                    │
│                      Cloudflare KV (Sessions/Cache)            │
└────────────────────────────────────────────────────────────────┘
```

### Core Repository (`learn`)

The `learn` repo will own all essential platform functionality:
- Authentication and user management
- Activities, sessions, and enrollments
- Attendance tracking
- User profiles and role-based access control
- Dashboard and participation metrics

### Separate Service Repositories

Non-core features will graduate to their own Workers once stable, keeping the core lean:
- `community-svc` — forums, peer connections, study groups
- `ai-learning-svc` — AI-generated learning paths (elective feature)

### Rationale for This Approach

| Concern | Benefit |
|---|---|
| **Maintainability** | Each module has a single responsibility and an independent deployment pipeline |
| **Contributor experience** | New contributors can onboard to a single domain without understanding the entire system |
| **Independent deployability** | A bug in the community service does not require redeploying the authentication system |
| **Scalability** | Each Worker scales independently based on actual load |
| **Extensibility** | New integrations (e.g., payment processing, notifications) can be added as new modules |

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
| Encryption | MD5/SHA-1 in use; must be replaced with AES-GCM (Web Crypto API) |
| Token management | No expiry or refresh mechanism |
| Secrets management | Hardcoded or unprotected credentials |
| CORS policy | Overly permissive |
| Protected endpoints | `/init` and `/seed` accessible without auth |
| Update/delete for Activities & Sessions | Missing CRUD operations |
| Enrollment status workflow | No state machine (pending → approved → completed) |
| Attendance system | Entirely absent |
| User profiles | No view/update/password-change endpoints |
| Role-based access control | No host/admin differentiation |
| Community features | Absent |
| Pagination | No cursor-based pagination |
| N+1 query issues | Multiple list endpoints perform per-row queries |

---

## Phase 1: Security Hardening (Weeks 1–2)

Security is the highest priority because vulnerabilities in the current implementation could be exploited immediately upon production deployment.

### 1.1 Encryption Upgrade

Replace the current weak hashing scheme with **AES-GCM via the Web Crypto API**, which is natively available in the Cloudflare Workers runtime without any external library.

```python
# Pseudocode for AES-GCM password storage
async def hash_password(password: str, env) -> str:
    key = await crypto.subtle.importKey(
        "raw", env.ENCRYPTION_KEY, {"name": "AES-GCM"}, False, ["encrypt"]
    )
    iv = crypto.getRandomValues(bytes(12))
    ciphertext = await crypto.subtle.encrypt(
        {"name": "AES-GCM", "iv": iv}, key, password.encode()
    )
    return base64(iv + ciphertext)
```

### 1.2 JWT Token Expiry and Refresh

Introduce short-lived access tokens (15 minutes) and long-lived refresh tokens (7 days) stored in Cloudflare KV:

- `POST /auth/refresh` — accepts a valid refresh token and issues a new access token
- Refresh tokens are rotated on each use (rotation prevents silent refresh token theft)
- Invalidated tokens are tombstoned in KV to prevent replay attacks

### 1.3 Secrets Management

All credentials (encryption keys, JWT secrets, API keys) will be moved to **Cloudflare Workers Secrets** (`wrangler secret put`) and accessed via `env.*` bindings. No secrets will appear in `wrangler.toml` or source code.

### 1.4 CORS Hardening

Implement an explicit allowlist: only the production frontend origin and, in development, `localhost` will be permitted. All other origins will receive `403 Forbidden`.

### 1.5 Protected Admin Endpoints

`/init` and `/seed` will require a valid admin JWT. In production, these will be further restricted by an IP allowlist or a one-time secret header to prevent accidental invocation.

**Deliverables:**
- All passwords stored with AES-GCM
- JWT refresh flow with KV-backed token rotation
- Zero hardcoded secrets in source
- CORS restricted to trusted origins
- `/init` and `/seed` protected

---

## Phase 2: Backend Refactoring (Week 3)

The current backend logic is concentrated in a single entry-point file, making it difficult to test, review, and extend. This phase restructures the codebase into a clean module layout.

### Proposed File Structure

```
src/
├── index.py              # Entry point — routes requests to handlers
├── auth/
│   ├── handlers.py       # Route handlers (login, register, refresh)
│   ├── middleware.py     # JWT validation middleware
│   └── models.py        # User schema, password hashing
├── activities/
│   ├── handlers.py
│   └── models.py
├── sessions/
│   ├── handlers.py
│   └── models.py
├── enrollments/
│   ├── handlers.py
│   └── models.py
├── attendance/
│   ├── handlers.py
│   └── models.py
├── profiles/
│   ├── handlers.py
│   └── models.py
├── community/
│   ├── handlers.py
│   └── models.py
└── shared/
    ├── db.py             # D1 query helpers and connection pooling
    ├── response.py      # Standardised JSON response helpers
    ├── validation.py    # Input validation and sanitisation
    └── pagination.py    # Cursor-based pagination utilities
```

**Benefits:**
- Each module can be reviewed and tested in isolation
- New contributors can understand a single module without reading thousands of lines
- Reduces merge conflicts when multiple contributors work in parallel

**Deliverables:**
- Full codebase split into domain modules
- Shared utilities extracted and documented
- All existing tests passing after refactor
- Module-level README for each domain

---

## Phase 3: Query Performance and Pagination (Weeks 3–4)

### 3.1 Eliminating N+1 Queries

Several list endpoints currently issue one database query per row to fetch related data. These will be rewritten using SQL `JOIN`s and aggregation:

```sql
-- Before (N+1): one query per enrollment to get session title
SELECT * FROM enrollments WHERE user_id = ?;
-- Then for each row: SELECT title FROM sessions WHERE id = ?;

-- After (single query with JOIN):
SELECT e.id, e.status, e.enrolled_at,
       s.id AS session_id, s.title AS session_title,
       a.id AS activity_id, a.name AS activity_name
FROM enrollments e
JOIN sessions s ON e.session_id = s.id
JOIN activities a ON s.activity_id = a.id
WHERE e.user_id = ?
ORDER BY e.enrolled_at DESC;
```

### 3.2 Cursor-Based Pagination

All list endpoints will support cursor-based pagination to ensure consistent performance as data grows:

```
GET /activities?limit=20&cursor=<opaque_cursor>

Response:
{
  "data": [...],
  "pagination": {
    "next_cursor": "<next_opaque_cursor>",
    "has_more": true
  }
}
```

### 3.3 Search and Filtering

- Activities: filter by tag, date range, host
- Sessions: filter by status, upcoming/past
- Enrollments: filter by status (pending/approved/completed)

**Deliverables:**
- No N+1 queries in any list endpoint (verified by query logging)
- Cursor-based pagination on all list endpoints
- Search and filter parameters documented in API reference

---

## Phase 4: Core Feature Completion (Weeks 4–7)

### 4.1 Activities and Sessions — Full CRUD

Implement the missing update and delete operations with proper ownership enforcement:

| Endpoint | Method | Auth | Description |
|---|---|---|---|
| `/activities/:id` | `PUT` | Host (owner) | Update activity metadata |
| `/activities/:id` | `DELETE` | Host (owner) | Soft-delete activity |
| `/sessions/:id` | `PUT` | Host (owner) | Update session details |
| `/sessions/:id` | `DELETE` | Host (owner) | Soft-delete session |

All mutations verify that the authenticated user is the resource owner before proceeding. Non-owners receive `403 Forbidden`.

### 4.2 Enrollment Status Workflow

Implement a formal state machine for enrollment lifecycle:

```
[PENDING] ──(host approves)──► [APPROVED] ──(session ends)──► [COMPLETED]
    │                                │
    └──(host rejects / user cancels)─┴──► [CANCELLED]
```

State transitions will be enforced server-side; clients cannot set arbitrary statuses.

### 4.3 Attendance System

A new attendance module will provide:

| Endpoint | Method | Description |
|---|---|---|
| `POST /sessions/:id/attendance` | Host only | Mark a list of users as present/absent |
| `GET /sessions/:id/attendance` | Host or enrolled user | View attendance for a session |
| `GET /users/me/attendance` | Authenticated user | View own attendance history across all sessions |

Attendance records are immutable once submitted by the host; corrections require an admin action (logged for audit).

### 4.4 User Profiles

| Endpoint | Method | Description |
|---|---|---|
| `GET /users/me` | Authenticated | Retrieve own profile |
| `PUT /users/me` | Authenticated | Update display name, bio, avatar URL |
| `POST /users/me/password` | Authenticated | Change password (requires current password) |
| `GET /users/:id` | Public | View another user's public profile |

### 4.5 Role-Based Access Control

Introduce a two-tier role model:

| Role | Permissions |
|---|---|
| `student` | Enroll, view sessions, mark own attendance, manage profile |
| `host` | All student permissions + create/update/delete own activities and sessions, manage enrollments, record attendance |

Role assignment will be controlled by admins via a dedicated endpoint. Host privileges are not self-service. Middleware will enforce role checks consistently across all protected routes.

**Deliverables (Phase 4):**
- Full CRUD for activities and sessions with ownership checks
- Enrollment state machine with valid transitions
- Complete attendance recording and retrieval
- User profile view and update endpoints
- RBAC middleware integrated across all routes

---

## Phase 5: Community Features (Weeks 8–9)

Community features will be implemented to match the Django reference implementation. These will later be extracted into `community-svc` but initially live in the `learn` repo.

### 5.1 Forums

- **Threads** — create, read, update (own), soft-delete (own or moderator)
- **Replies** — nested up to 2 levels, with threaded display
- **Categories** — activity-scoped or global
- **Moderation** — hosts can pin or close threads within their activity

### 5.2 Peer Connections

- Send/accept/reject connection requests
- List connected peers
- View peer's public profile and shared activity history

### 5.3 Study Groups

- Create groups linked to an activity or independent
- Invite/join/leave groups
- Group discussion thread (backed by the forum module)
- Group membership management by the group creator

**Schema additions (D1):**

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

CREATE TABLE peer_connections (
    id TEXT PRIMARY KEY,
    requester_id TEXT NOT NULL REFERENCES users(id),
    addressee_id TEXT NOT NULL REFERENCES users(id),
    status TEXT NOT NULL CHECK(status IN ('pending','accepted','rejected')),
    created_at TEXT NOT NULL
);

CREATE TABLE study_groups (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    creator_id TEXT NOT NULL REFERENCES users(id),
    activity_id TEXT REFERENCES activities(id),
    created_at TEXT NOT NULL
);
```

**Deliverables:**
- Forum CRUD with moderation controls
- Peer connection request/accept/reject flow
- Study group management
- Frontend views for all community features

---

## Phase 6: Frontend Development (Weeks 6–10, parallel)

Frontend work will proceed in parallel with backend development. The UI will continue using **HTML, Tailwind CSS, and vanilla JavaScript** to stay consistent with the existing frontend.

### New Pages and Components

| Feature | UI Component |
|---|---|
| Attendance marking | Host-facing checklist view per session |
| Attendance history | Calendar-style heatmap for users |
| User profile | View/edit form with avatar upload |
| Enrollment status | Status badge and action buttons (for host: approve/reject) |
| Community — Forums | Thread list, thread detail, reply composer |
| Community — Peers | Peer search, connection cards |
| Community — Study Groups | Group list, group detail, member list |

### Design Principles

- **Progressive enhancement** — core functionality works without JavaScript
- **Accessibility** — ARIA labels, keyboard navigation, focus management
- **Performance** — minimal JavaScript, no heavy frameworks, assets served from Cloudflare CDN
- **Responsive** — mobile-first layouts consistent with the existing UI

---

## Phase 7: Elective Feature — AI-Powered Learning Path System (Weeks 11–12)

This feature will be built only after all migration work is complete. It represents a significant value-add for the platform.

### Concept

A user specifies a topic (e.g., "Introduction to Machine Learning") and the system automatically constructs a personalised, sequential learning path:

```
Topic Input ──► LLM-Generated Curriculum ──► User Review
                                               │
                            ┌──────────────────┘
                            ▼
             Step 1: Concept explanation + resources
                            │
                      Pass quiz? ──No──► Retry / Review resources
                            │ Yes
                            ▼
             Step 2: Next concept…
                            │
                            ▼
             Completion Certificate + Progress Summary
```

### Content Sources

| Source | Use Case |
|---|---|
| LLM (OpenAI / Groq) | Concept explanations, quiz generation, plan structuring |
| Wikipedia API | Authoritative background information and definitions |
| YouTube Data API | Curated video content per step |
| User-uploaded notes | Personalised supplementary material |

### LLM Provider Abstraction

To avoid vendor lock-in, an abstraction layer will unify different LLM providers:

```python
class LLMProvider:
    async def generate(self, prompt: str, schema: dict) -> dict: ...

class OpenAIProvider(LLMProvider): ...
class GroqProvider(LLMProvider): ...

# Configured via Cloudflare Worker environment variable
provider = get_provider(env.LLM_PROVIDER)  # "openai" | "groq"
```

### Data Model

```sql
CREATE TABLE learning_paths (
    id TEXT PRIMARY KEY,
    user_id TEXT NOT NULL REFERENCES users(id),
    topic TEXT NOT NULL,
    status TEXT NOT NULL CHECK(status IN ('draft','active','completed')),
    created_at TEXT NOT NULL
);

CREATE TABLE learning_steps (
    id TEXT PRIMARY KEY,
    path_id TEXT NOT NULL REFERENCES learning_paths(id),
    step_order INTEGER NOT NULL,
    title TEXT NOT NULL,
    content TEXT NOT NULL,
    quiz_json TEXT NOT NULL,  -- JSON array of questions
    is_unlocked INTEGER DEFAULT 0,
    completed_at TEXT
);
```

### User Flow

1. User enters a topic and optional difficulty preference
2. LLM generates a structured plan (title, steps, learning objectives)
3. User previews and confirms the plan
4. Learns step-by-step; each step unlocks after passing the associated quiz
5. Progress is persisted; users can resume any time
6. On completion, a shareable progress summary is generated

**Deliverables (if time permits):**
- LLM provider abstraction supporting at least OpenAI and Groq
- Learning path generation, storage, and retrieval
- Quiz generation and pass/fail progression logic
- Wikipedia and YouTube content integration
- Frontend for the full learning path experience

---

## Detailed Timeline

### Community Bonding Period (Pre-Week 1)

- Deep dive into the `learn` codebase — map every existing route, model, and migration
- Audit the Django repository to catalogue every feature not yet in `learn`
- Set up local development environment; write end-to-end smoke tests for existing features
- Draft the initial D1 schema additions and get mentor sign-off
- Establish communication cadence and milestone check-in schedule with mentor

### Week 1: Security Hardening

| Day | Task |
|---|---|
| 1–2 | Implement AES-GCM password hashing with Web Crypto API |
| 3–4 | Implement JWT access token + refresh token with KV storage |
| 5 | Move all secrets to Cloudflare Workers Secrets; CORS hardening |
| 6–7 | Protect `/init` and `/seed`; write security-focused integration tests |

**Milestone:** All security improvements merged; no plaintext secrets in codebase.

### Week 2: Backend Restructuring

| Day | Task |
|---|---|
| 1–3 | Split monolithic handler file into domain modules (auth, activities, sessions) |
| 4–5 | Extract shared utilities (db helpers, response formatters, validation) |
| 6–7 | Remaining modules (enrollments, profiles, community); update all imports; full regression test |

**Milestone:** Modular codebase structure complete; existing tests passing.

### Week 3: Query Optimisation and Pagination

| Day | Task |
|---|---|
| 1–3 | Identify and fix all N+1 queries using JOINs; verify with query logging |
| 4–5 | Implement cursor-based pagination utility; apply to activity and session list endpoints |
| 6–7 | Apply pagination to enrollment/attendance endpoints; implement search and filter parameters |

**Milestone:** No N+1 queries; all list endpoints paginated.

### Week 4: Activities, Sessions, and Enrollments — Full CRUD

| Day | Task |
|---|---|
| 1–2 | `PUT /activities/:id`, `DELETE /activities/:id` with ownership checks |
| 3–4 | `PUT /sessions/:id`, `DELETE /sessions/:id` with ownership checks |
| 5–7 | Enrollment state machine; `PUT /enrollments/:id/status` endpoint; validation and tests |

**Milestone:** Complete CRUD for activities, sessions, and enrollment state transitions.

### Week 5: Attendance System

| Day | Task |
|---|---|
| 1–3 | D1 schema for attendance; `POST /sessions/:id/attendance` (host only) |
| 4–5 | `GET /sessions/:id/attendance`; `GET /users/me/attendance` |
| 6–7 | Attendance summary statistics; write integration tests; frontend attendance checklist |

**Milestone:** Attendance system fully functional with frontend UI.

### Week 6: User Profiles and Role-Based Access Control

| Day | Task |
|---|---|
| 1–2 | `GET /users/me`, `PUT /users/me`, public profile endpoint |
| 3–4 | Password change endpoint; RBAC middleware |
| 5–7 | Apply RBAC to all existing routes; host privilege assignment; role-aware frontend components |

**Milestone:** Profiles and RBAC complete; all protected routes enforcing correct role checks.

### Week 7: Buffer, Testing, and Midterm Review

- Address any items that slipped from Weeks 1–6
- Expand integration test coverage to ≥80% of route handlers
- Prepare midterm evaluation report with metrics (lines migrated, endpoints completed, test coverage)
- Demo of the migrated platform to mentors

**Midterm Milestone:** Core migration complete. All Django features replicated in `learn`. Platform deployable as production replacement for Django backend.

### Week 8: Community Features — Forums

| Day | Task |
|---|---|
| 1–2 | D1 schema for forum threads and replies; CRUD handlers |
| 3–4 | Category support; moderation controls (pin, close) |
| 5–7 | Frontend: thread list, thread detail, reply composer |

### Week 9: Community Features — Peers and Study Groups

| Day | Task |
|---|---|
| 1–3 | Peer connection request/accept/reject API and UI |
| 4–7 | Study group CRUD; member management; group discussion integration with forums |

**Milestone:** All community features complete and tested.

### Week 10: Frontend Polish and Accessibility Pass

- Audit all new frontend pages for accessibility (ARIA, keyboard navigation)
- Responsive layout review across breakpoints
- Performance audit (Lighthouse) and optimisation
- Ensure visual consistency with existing UI patterns

### Week 11: AI Learning Path System (Elective)

| Day | Task |
|---|---|
| 1–2 | LLM provider abstraction; D1 schema for learning paths and steps |
| 3–5 | Learning path generation endpoint; quiz generation from step content |
| 6–7 | Wikipedia and YouTube integration; user flow (create → review → start) |

### Week 12: AI System Frontend, Testing, and Documentation

| Day | Task |
|---|---|
| 1–3 | Frontend: topic input, plan preview, step-by-step learning UI |
| 4–5 | Quiz pass/fail progression; progress persistence and resume functionality |
| 6–7 | Final integration testing; API reference documentation; deployment guide |

**Final Milestone:** Full platform running on Cloudflare Workers; AI learning path system operational; documentation complete.

---

## Success Metrics

| Metric | Target |
|---|---|
| Feature parity with Django backend | 100% of identified features ported |
| Route handler test coverage | ≥ 80% |
| N+1 query elimination | 0 N+1 queries in list endpoints |
| API response time (p95 at edge) | < 100 ms |
| Security audit | Zero critical or high vulnerabilities in final review |
| Documentation | All endpoints documented with request/response examples |

---

## Testing Strategy

Given that Cloudflare Workers Python support does not yet have a mature testing framework, the following layered approach will be used:

1. **Unit tests** — Pure Python logic (state machines, validation functions, pagination utilities) tested with `pytest` locally, isolated from the Worker runtime.
2. **Integration tests** — The Cloudflare Workers test environment (via `wrangler dev` with a local D1 instance) will be used to test route handlers end-to-end. Tests will be written using `pytest` with an HTTP client pointed at the local Worker.
3. **Regression tests** — A smoke-test suite will run on every PR to verify that all existing functionality is unbroken after changes.

---

## Risk Assessment and Mitigation

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Cloudflare Python Workers runtime limitations discovered mid-project | Medium | High | Identify blockers during Community Bonding; have fallback implementation strategies per feature |
| D1 schema migrations introduce data loss in development | Low | High | All migrations run through versioned migration files; changes tested against a copy of production data before applying |
| LLM API costs or rate limits block AI feature development | Medium | Low | AI feature is elective; use Groq's generous free tier for development; mock LLM responses in tests |
| Scope creep from Django feature audit | Medium | Medium | Maintain a prioritised feature backlog with mentor; deprioritise cosmetic features in favour of functional parity |
| Midterm deadline pressure | Low | Medium | Week 7 is intentionally a buffer week; midterm scope is defined conservatively |

---

## Benefits to the Project and Community

| Stakeholder | Benefit |
|---|---|
| **Alpha One Labs** | Eliminates Django infrastructure costs; single deployable system on Cloudflare edge |
| **Platform users** | Significantly faster response times (edge deployment vs. centralised server); improved feature set |
| **Contributors** | Modular, well-documented codebase with clear domain boundaries is far easier to contribute to |
| **Future maintainers** | Comprehensive test coverage and documentation reduce onboarding time |
| **Open-source ecosystem** | The migration serves as a publicly accessible reference implementation for Django-to-Cloudflare Workers migrations |

---

## Previous Contributions to Alpha One Labs

I have had **8 pull requests merged** into the Alpha One Labs repository, demonstrating sustained engagement and familiarity with the codebase:

| PR | Description | Area |
|---|---|---|
| [#992](https://github.com/alphaonelabs/education-website/issues/992) | Resolved duplicate messaging interfaces causing inconsistent user experience | Frontend / UX |
| [#982](https://github.com/alphaonelabs/education-website/issues/982) | Fixed virtual lab navigation inconsistency and duplicated Chemistry URL | Routing / Navigation |
| [#869](https://github.com/alphaonelabs/education-website/issues/869) | Fixed duplicate locale in Top Contributors link | i18n / Templates |
| [#863](https://github.com/alphaonelabs/education-website/issues/863) | Updated GSoC '25 link to 2026 in website footer | Content / Templates |
| [#843](https://github.com/alphaonelabs/education-website/issues/843) | Added delete functionality to quiz options | Feature / Backend |
| [#818](https://github.com/alphaonelabs/education-website/issues/818) | Fixed dark mode colour inconsistencies | CSS / Theming |
| [#807](https://github.com/alphaonelabs/education-website/issues/807) | Fixed broken Discord and Slack links in website footer | Content / Templates |
| [#791](https://github.com/alphaonelabs/education-website/issues/791) | Improved discoverability of horizontal scrolling on course enrollment table | Accessibility / UI |

These contributions span the full Django stack (templates, views, static assets), which gives me a strong foundation for the migration work. Crucially, I have already navigated the review process with the Alpha One Labs team and understand their standards and expectations.

---

## Why I Am the Right Person for This Project

**I know the codebase.** My contributions span both the Django repository (the source of truth for feature parity) and the `learn` repository (the migration target). I am not starting from a cold read—I already understand the data models, the team's code style, and the quirks of both systems.

**I have directly relevant technical skills.** The migration requires Cloudflare Workers (Python), D1 (SQLite), KV storage, and TypeScript/JavaScript frontend work. I have shipped production code on all of these technologies in personal projects.

**My background aligns with the elective AI feature.** I have built an LLM-integrated application using provider abstractions, which maps directly to the AI learning path system proposed in Phase 7.

**I am reliable and communicative.** With 8 merged PRs, I have demonstrated that I can deliver consistent, review-quality work on this specific project—not just in general. I respond to feedback quickly and iterate.

**The migration is technically challenging in exactly the right ways.** Moving from Django's ORM and synchronous request handling to D1's SQL interface and Cloudflare Workers' async model requires careful design decisions. This is not a mechanical port—it is an engineering problem, and that is what motivates me.

---

## Availability and Communication

- **Availability:** ~40 hours per week throughout the GSoC period
- **Other commitments:** No internships, part-time jobs, or other programmes during this period
- **Preferred communication:** Async via GitHub issues/PR comments (primary); synchronous check-ins with mentor as needed (at least weekly)
- **Progress reporting:** Weekly written status updates with completed items, blockers, and next steps posted to the project's communication channel

---

## Post-GSoC Plans

The work described here lays a foundation, not a ceiling. After GSoC I intend to:

- **Extract and stabilise `community-svc`** — once the community features are stable in `learn`, migrate them to their own Worker with a proper CI/CD pipeline
- **Extend the AI learning path system** — add support for spaced repetition, collaborative paths, and instructor-created templates
- **Improve observability** — integrate Cloudflare Analytics Engine for structured logging and performance monitoring
- **Continue contributing** — remain an active maintainer, review PRs, mentor new contributors, and help grow the Alpha One Labs community

---
