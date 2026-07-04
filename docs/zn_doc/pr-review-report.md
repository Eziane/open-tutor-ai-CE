# Backend PR Review Report

> Log of PRs reviewed as backend maintainer, using [`backend-review-checklist.md`](./backend-review-checklist.md)
> as the standard. One section per PR, plus a running tally of common issues at the
> bottom for spotting patterns (useful for the soutenance).

**Severity legend:** 🚫 Blocker · ⚠️ Major · 💡 Minor

---

## PR #286 — fix(auth): move browser sessions to HttpOnly cookies with CSRF origin guard

- **Link:** https://github.com/Open-TutorAi/open-tutor-ai-CE/pull/286
- **Author:** Ilyas-ek
- **What it changes:** Session tokens were sitting in `localStorage`, readable by any
  injected script (XSS). This PR moves them into an HttpOnly cookie instead, adds a
  CSRF Origin check for cookie-based requests, and updates the frontend to call the
  API same-origin so the cookie rides along automatically.
- **Verdict:** Good fix, well tested. Found 2 things worth fixing before merge.

### Issues found

**1. ⚠️ Major — the CSRF check will stop working once this is deployed behind HTTPS**
`gateway/http/app.py`, line ~149 (`csrf_origin_check`)

Spotted that the check compares the request's Origin against
`request.url.scheme + host` to allow same-origin requests through. In production this
app runs behind a reverse proxy that terminates HTTPS, and Uvicorn itself is never
started with proxy-header support (checked `Dockerfile.backend` and `dev.sh` — neither
sets `--proxy-headers`). So `request.url.scheme` will always read `"http"` internally,
even when the real traffic is `https`. Once that mismatch happens, this check can
never pass, and it will start rejecting legitimate requests (password change, etc.)
with a 403, unless someone manually sets `CORS_ALLOW_ORIGIN` to the exact production
URL. Needs either proxy-header handling or a clear deployment note about
`CORS_ALLOW_ORIGIN`.

**2. 💡 Minor — new `AUTH_COOKIE_SECURE` setting isn't documented anywhere**
`config/settings.py`, line 54

This new env var controls whether the cookie requires HTTPS, but it's missing from
both `.env.example` and CLAUDE.md's env var table. As is, nobody deploying this would
know to turn it on. Quick fix — one line in each file.

### Also noted (smaller, not blocking)

- The CSRF check only blocks the request when an `Origin` header is present and
  wrong — if `Origin` is missing entirely, it lets the request through. Low risk
  since `SameSite=Lax` already blocks the real attack case in modern browsers, but
  worth tightening later.
- Socket.IO still allows all origins (`cors_allowed_origins="*"`) — pre-existing, not
  from this PR, but worth revisiting now that the socket handshake also accepts the
  cookie.

### What was good

- New test file (`test_cookie_auth.py`) is thorough — covers cookie issuing, bearer
  vs. cookie priority, CSRF pass/block cases, and the socket cookie parser.
- Bearer token correctly takes priority over the cookie when both are present, and
  that's actually tested, not just assumed.
- Frontend move to same-origin relative URLs is a clean way to make the cookie flow
  without CORS headaches.
- Didn't need any changes to the contract-coverage test — checked and the new
  `/auths/cookie` route is already covered by the existing route mounting.

*(Did not run Black/pytest locally — no Python toolchain on this machine. Confirm the
PR's own CI is green before merging.)*

---

## PR #274 — feat(teacher): teacher section (classes, roster, assignments, exams, messaging, resources) + security hardening

- **Link:** https://github.com/Open-TutorAi/open-tutor-ai-CE/pull/274
- **What it changes:** The whole teacher portal — classrooms, roster/invitations,
  guardians, assignments, exam proctoring, messaging, a resources library — plus new
  security infra (rate limiting, security headers). Large PR (115 files); reviewed by
  checking out the branch in an isolated worktree and going through the backend
  domain by domain, then spot-checking the frontend glue.
- **Verdict:** Strong overall structure (ownership checks, test coverage) but found 3
  things that need fixing before merge — one exploitable stored XSS, and one
  violation of the project's own mandatory architecture rule.

### Issues found

**1. 🚫 Blocker — new domains don't follow the project's mandatory boundary rule**
`classrooms/`, `assignments/`, `exams/`, `guardians/`, `messaging/`, `resources/`
(all at repo root)

CLAUDE.md's "Adding a new domain" section is explicit: every new domain must be
created under one of the six boundaries — `accounts`, `learning`, `ai`, `content`,
`governance`, `system` — as `<boundary>/<domain>/`. This PR creates all six new
domains as top-level packages instead (e.g. `classrooms/service.py` at the repo
root, not `learning/classrooms/service.py`). This isn't a style preference; it's
listed as the project's own architectural convention, which the review checklist
ranks above the checklist itself and above general best practice. Checked #266 (the
other open PR touching classroom management) for comparison — it does this
correctly, nesting under `learning/classrooms`, `learning/attendance`,
`learning/announcements`. Since both PRs modify overlapping ground, this should be
fixed here before merge rather than left for a later cleanup — six new top-level
packages is exactly the kind of thing that becomes the accepted pattern if it lands
first.

**2. 🚫 Blocker — stored XSS through the class-materials upload/download flow**
`gateway/http/routers/resources.py` (upload check ~L79-87, download ~L141-144),
allowlist in `config/settings.py` L60-61

Traced the upload path: the content-type check trusts whatever content-type the
client declares (no verification against the actual file bytes), and the allowlist
permits the entire `"text/"` prefix — which includes `text/html`. Then traced the
download path (`download_material`): it serves the file back with
`Content-Disposition: inline` and that same client-supplied content-type. Put
together: a teacher account can upload a file declared as `text/html` containing a
script, and it will render and execute in the browser of any student who opens it,
in the app's own origin. Compared this against the other two attachment endpoints
added in the same PR (assignments, messaging) — both correctly force
`Content-Disposition: attachment`. Resources is the outlier.

**3. ⚠️ Major — rate limiter's "per-user" key collapses to one shared bucket for everyone**
`gateway/http/rate_limit.py` L53 (`_client_key`)

The per-user rate-limit key takes the first 32 characters of the bearer token.
Checked the JWT structure this app produces: the first ~36 characters of *any*
token issued here are just the fixed, identical header segment
(`{"typ":"JWT","alg":"HS256"}` base64-encoded) — confirmed this by encoding it
directly. So every user produces the same key, meaning the limiter is really one
global bucket, not per-user as the code comments claim. Confirmed via
`test_rate_limit.py` that only the generic counting logic is tested — the
token-to-key derivation itself is never exercised. Also confirmed the limiter is
actually wired into the app (`gateway/http/app.py`, active whenever `DEBUG=false`),
so this isn't dead code — it will misbehave in a real deployment.

### What was good

- Ownership checks are consistent across the whole classrooms domain — every
  service method that touches a classroom goes through the same `_owned()` guard
  before doing anything, checked this across all of `classrooms/service.py`.
- Assignment and messaging attachment downloads correctly use
  `Content-Disposition: attachment` (this is what made the resources one stand out
  as the odd one out).
- Test coverage is substantial for a feature this size — `test_classrooms.py` alone
  is over 1000 lines.
- New security-headers middleware (CSP, X-Frame-Options, nosniff, etc.) and the
  rate-limiting concept are both good additions to the app, independent of the bug
  in its implementation.

*(Did not run Black/pytest locally — no Python toolchain on this machine. Confirm the
PR's own CI is green before merging. Also did not do a deep pass on the exam/
proctoring logic or the Svelte components — the fix commit already on this PR
suggests that area had already been iterated on, and reviewing 115 files pushed
scope choices; flagging that as a gap in this review, not a clean bill of health.)*

**Correction note:** finding #1 (the boundary-rule violation) was missed on the
initial pass and only caught while reviewing #266 for comparison — added here after
the fact rather than silently editing the original verdict. Both PRs also
independently created `gateway/http/rate_limit.py` from scratch with incompatible
implementations; whichever merges first effectively forces a rework of the other —
worth resolving as a team decision rather than letting merge order settle it.

---

## PR #266 — feat: teacher classroom management (CRUD, enrollment, attendance, 3D seating chart)

- **Link:** https://github.com/Open-TutorAi/open-tutor-ai-CE/pull/266
- **Author:** hasnaa88
- **What it changes:** A separate, competing implementation of teacher classroom
  management — CRUD wizard, student enrollment (email/CSV/join code/invite code),
  live attendance sessions with self check-in, announcements, and a Three.js 3D
  seating chart. Correctly placed under `learning/classrooms`, `learning/attendance`,
  `learning/announcements` per the project's boundary convention.
- **Read first:** the PR description and all 3 existing review comments (TarekAi,
  Oumaima-elkhoummassi, Eziane) before reviewing, specifically to avoid repeating
  ground already covered.
- **Already covered by others, confirmed and not repeated:** merge conflicts, missing
  doc sections, French-language docs, stray `.Rhistory` file — all addressed in the
  latest commit; spot-checked `class-diagram.md` is genuinely in English now, and
  confirmed `.Rhistory` is gone from the tree. TarekAi's "reuse the existing 3D
  classroom" point also checked out — there's already a Three.js
  `ui/src/lib/components/classroom/` on main this PR doesn't appear to reuse — but
  since that was already raised, didn't re-raise it.
- **Verdict:** Solid feature, good test coverage. Found 2 new issues, both about gaps
  between what the code assumes and what actually runs in production.

### Issues found

**1. ⚠️ Major — new Alembic migrations are never actually executed**
`alembic/versions/*.py`, `data/database.py::init_database`

This PR adds 4 real migrations (create `classrooms`, add `objectives` to
`class_sessions`, add `ended_at` to `class_sessions`, etc.) — genuine schema changes
for an *existing* database. But confirmed the app still boots via
`Base.metadata.create_all()`, and checked the Dockerfile, devops scripts, and CI —
nothing runs `alembic upgrade head` anywhere. `create_all()` only creates missing
tables; it won't add a column to a table that already exists. Net effect: a fresh
database is fine, but deploying this onto an existing production database won't pick
up the new columns, and the app will error the first time it touches them.

**2. ⚠️ Major — live-session check-in is rate-limited per IP, not per user**
`gateway/http/routers/sessions.py` L128 (`POST /api/sessions/{session_id}/join`,
`@limiter.limit("20/minute")`)

Checked the limiter setup (`gateway/http/rate_limit.py`) — it's slowapi's default
`get_remote_address` key, i.e. capped per IP. Self-check-in for live attendance is
exactly the scenario where a whole class checks in within a minute or two, commonly
from behind the same school network/NAT — meaning many students would share one IP
and collectively hit this cap. This is domain-specific: a generic reviewer without
context on shared-school-network IPs might not flag it, but it directly threatens
the feature's own core use case.

### What was good

- Ownership checks present and consistent in `learning/classrooms/service.py`
  (`caller_id` vs `classroom.owner_id`, checked this pattern across the file).
- JWT dev-secret is now randomly generated (`secrets.token_hex(32)`) instead of the
  old static placeholder string — confirmed, matches the PR description.
- Test coverage is substantial — dedicated integration test file
  (`test_classrooms_integration.py`, 400+ lines) alongside unit tests.
- Correctly uses the project's `learning/<domain>` boundary convention, unlike #274.

*(Did not run Black/pytest locally. Did not deep-dive the 3D seating chart component
or CSV-import edge cases — lower risk surface, and effort went toward the
backend/production-readiness questions above.)*

---

## PR #269 — feat: adaptive diagnostic test (LLM-based level assessment before chat)

- **Link:** https://github.com/Open-TutorAi/open-tutor-ai-CE/pull/269
- **Author:** latifaaznague — **Draft**
- **What it changes:** Generates a 10-question diagnostic from the support's course
  content via an LLM, scores it, and assigns a level (beginner/intermediate/advanced)
  meant to calibrate the tutor's teaching style before the student starts chatting.
- **Read first:** the PR description and both existing comments (Oumaima-elkhoummassi:
  docs in French + undocumented rate limiting; Dakir-Ai: merge conflicts) before
  reviewing.
- **Already flagged by others, checked against the current code:** docs are in
  English now, the 429 case is documented in `feature.md`'s error table, and this
  diffs cleanly against the current upstream `main` (no conflicts as of this
  review) — appears already addressed, not re-raised. Worth a live re-check on
  GitHub's own mergeable status since I'm going off a local diff, not the platform.
- **Verdict:** the specific thing the reporter asked about — does the description
  match the implementation — turned out to have a real answer: the "mandatory"
  framing overstates what's actually enforced. Two other real issues found too, all
  three below.

### Issues found

**1. ⚠️ Major — "mandatory before chat" is only enforced in the frontend**
`ui/src/lib/components/student/pages/SupportDetails.svelte` (`diagnosticCompleted` +
`goto()` redirect)

The description states learners "must complete" the diagnostic before accessing
tutoring. Checked the backend (chat/completions router, supports service, sessions
router) for a matching server-side check — there isn't one. The gate is a client-side
redirect only. A student calling the chat API directly bypasses the diagnostic
entirely. Not a cross-user data exposure (it's their own account), but it does mean
the description's "mandatory" claim isn't actually true today — either the backend
needs the same check, or the docs/description should describe this as a UI flow
rather than a requirement.

**2. ⚠️ Major — malformed LLM output fails late (at submit) instead of at generation**
`learning/diagnostics/service.py` — `generate()` ~L166-183, `submit()` ~L196-203

`generate()` parses the LLM's JSON response and stores it without validating that
each question actually has the keys `submit()` later assumes (`id`,
`correct_answer`). A structurally incomplete but syntactically valid LLM response
sails through `generate()` (which does catch parse failures and maps them to a
clean 502) and gets handed to the student as a normal pending diagnostic. The
`KeyError` only surfaces in `submit()`, which doesn't catch generic exceptions —
an unhandled 500, potentially after the student has answered all 10 questions.

**3. 💡 Minor — determined level is hardcoded French, shown untranslated in the UI**
`learning/diagnostics/service.py` L50-53 (`_LEVEL_THRESHOLDS`),
`DiagnosticTest.svelte` ~L261

`determined_level` is always `"débutant"/"intermédiaire"/"avancé"` regardless of
the course's language, and the frontend renders it raw
(`{$i18n.t('Level')}: {diagnostic.determined_level}`) — the label is translated,
the value isn't. An English or Arabic-locale user sees a French word. Checked
`Chat.svelte`'s level-based prompt logic — it consistently compares against the
same French strings, so the adaptive-teaching behavior itself isn't broken; this is
purely a display/i18n gap, fixable by running the value through `$i18n.t()` with
proper translation keys.

### What was good

- The "37 pytest tests" claim in the description checks out — counted the same
  number of test functions in `test_diagnostics.py`.
- Ownership checks are present and correctly scoped (`support.user_id`,
  `diagnostic.user_id` checked against the caller in both `generate()` and
  `submit()`, plus router-level checks on the two GET routes).
- The rate limiter here is keyed correctly (`current_user.id`), unlike the
  same-shaped bug found in #274's rate limiter — worth noting as the right way to
  do it.
- Resubmission of a `completed` diagnostic is a no-op that returns the stored
  result instead of erroring or re-scoring — sensible idempotency choice.

*(Did not run Black/pytest locally. Didn't chase the "answer must exactly match
correct_answer" scoring logic as a bug — confirmed the frontend sends back the
exact choice string from the same rendered options, so exact-match scoring is fine
for multiple-choice, not a free-text mismatch risk.)*

---

## Common Issues Across Reviewed PRs (running tally)

_Seeded from PR #286, #274, #266, #269 — update as more PRs come in._

| Category | Count | Seen in |
|---|---|---|
| Deployment config gap (works locally, breaks in real deployment) | 2 | #286 — CSRF check breaks behind HTTPS reverse proxy; #266 — Alembic migrations never actually run |
| Missing docs for new env var/setting | 1 | #286 — `AUTH_COOKIE_SECURE` not documented |
| Unsafe file upload/serving (trusting client-supplied content-type) | 1 | #274 — stored XSS via class-materials `inline` download |
| Security control silently broken by an implementation detail (looks right, isn't) | 2 | #274 — rate limiter's per-user key collapses for everyone; #266 — check-in rate limit keyed by IP, hurts shared-network classrooms |
| Two PRs building the same thing differently (coordination gap, not a single-PR bug) | 1 | #274 vs #266 — incompatible domain-boundary placement + two from-scratch `rate_limit.py` implementations |
| "Mandatory"/enforced claim in the description not actually backed by a server-side check | 1 | #269 — diagnostic gate is a frontend redirect only, nothing blocks the chat API server-side |
| Error handling inconsistent between two closely-related endpoints in the same flow | 1 | #269 — `generate()` catches broad exceptions and returns a clean 502; `submit()` doesn't, so a bad stored record crashes as an unhandled 500 |
