# Phase 1: Security Hardening — SurgicalWatch Application

Date: June 4-5, 2026

Environment: ChromeOS (Linux container / Penguin), Next.js 16, Supabase Free tier,
Vercel Hobby, Axiom (log observability), Snyk (dependency scanning)

Time spent: ~2 days across multiple sessions

NOTE: All credential values shown in this document are visibly fake placeholders.
No real tokens, keys, or URLs are recorded here. The work itself was performed
against real capstone accounts; the documentation is sanitized for public posting.

---

## Goal

Harden a freshly scaffolded Next.js application before any features are built —
treating security as the foundation rather than something added later. The application
is the SurgicalWatch capstone: a threat intelligence dashboard for the Stryker Mako
surgical robotics fleet.

The goal was to end Phase 1 with a live, password-protected application on Vercel
where: anonymous visitors cannot reach any page; every login attempt is observable in
a security log; secrets are handled correctly from the start; and a verification script
can prove all of this is true after any future change.

This was part of the NextCISO Academy capstone program. Phase 1 is the security
foundation that Phase 2 (the threat intelligence pipeline and dashboard) builds on top
of. The principle: build the walls before furnishing the house.

---

## Starting state

A working Next.js 16 application scaffolded with `create-next-app`, deployed to Vercel,
connected to Supabase for auth and database. No authentication gate, no security logging,
no secret-handling controls, no CI scanning. A visitor could reach any page without
signing in. There was no way to observe who was attempting to access the application.

---

## Approach

1. **Install pre-commit hooks** — block secrets from ever entering git history, and
   block dangerous terminal commands from running in the project directory.
2. **Build the sanitizing logger** — a server-side logging module that sends security
   events to Axiom with all secret-shaped values scrubbed from the payload before
   transmission.
3. **Wire up the auth-event forwarder** — a Supabase database webhook feeding an Edge
   Function that reports authentication events (signups, logins) to Axiom independently
   of the application code.
4. **Add Snyk to CI** — dependency vulnerability scanning running on every push to
   GitHub Actions.
5. **Build the authentication foundation** — Supabase SSR auth with cookie handling,
   a fail-closed proxy gate, and open-redirect prevention.
6. **Deploy and configure** — production Vercel deployment with correct environment
   variables, Supabase site URL and redirect allow-list locked down.
7. **Verify in production** — run the verification script against the live deployment
   and check each observability path manually before declaring Phase 1 complete.

---

## Step 1 — Pre-commit Hooks

Two hooks installed in `.git/hooks/`:

**`pre-commit`** runs gitleaks on every commit attempt. If any file being staged contains
a pattern that looks like a secret (API key format, token prefix, connection string), the
commit is rejected before it can be made. This is the last line of defense before a
secret enters git history — once it's in history, it's effectively leaked even if
immediately removed, because history can be cloned.

**`pre-commit-dangerous-commands`** blocks a specific list of terminal commands from
running while in the project directory. This was added because some common developer
habits (running `env`, piping output to files, certain curl patterns) can accidentally
expose secrets in the terminal or file system.

Why hooks rather than relying on discipline: a hook fires every time regardless of how
tired or rushed the developer is. The rule doesn't depend on memory.

```bash
# Confirm both hooks are present and executable
$ ls -la .git/hooks/pre-commit*
-rwxr-xr-x 1 brendan brendan 1842 May 20 .git/hooks/pre-commit
-rwxr-xr-x 1 brendan brendan  894 May 20 .git/hooks/pre-commit-dangerous-commands
```

---

## Step 2 — The Sanitizing Logger

`src/lib/logger.ts` is a server-side module that sends structured security events to
Axiom. Two design requirements drove the implementation:

**Requirement 1: Secrets must never appear in a log payload.** The logger runs a redact
function over every value before transmission. The redact function matches patterns that
look like secrets (token prefixes, key shapes, connection strings) and replaces them
with `[REDACTED]`. A leaked log is almost as bad as a leaked secret — logs are often
stored in less-controlled places than the secrets themselves.

**Requirement 2: The logger must not crash the application if Axiom is unreachable.**
All transmission is wrapped in a try/catch. A failed log write is recorded to console
but does not propagate as an error to the caller. Logging is observability
infrastructure; it should never be on the critical path.

The logger also validates the Axiom endpoint URL shape before attempting to send. If the
URL is not configured (missing environment variable), it falls back to console logging.
This means the application works in local dev without an Axiom account.

A specific test: logging a failed login must not include the attempted password anywhere
in the Axiom payload. This was verified by checking the actual event fields in Axiom
after triggering a failed login in production.

---

## Step 3 — The Auth-Event Forwarder

The logger covers application-layer events. But some auth events happen at the Supabase
layer — signups, email confirmations, password resets — and never touch the application
code at all. If those events aren't captured separately, there are blind spots in the
security log.

The solution: a Supabase database webhook that fires on auth events and calls a Supabase
Edge Function, which forwards the event to Axiom. This path is entirely independent of
the Next.js application.

Why independence matters: if the application is down, the auth-event log still works.
If there is a bug in the application logger, the webhook path still works. Three
independent paths into Axiom (app logger, webhook forwarder, Snyk CI) mean three
independent chances to catch something — one working tells you nothing about the others.

The Edge Function strips credential-shaped fields from the webhook payload before
forwarding, for the same reason the application logger redacts: Supabase database
webhooks can include internal fields that should not leave the Supabase boundary.

Verification: after wiring up the webhook, performed a real signup in production and
confirmed the event appeared in Axiom with all expected fields present and no
credential-token fields included.

---

## Step 4 — Snyk in CI

Snyk scans the project's npm dependencies against a vulnerability database on every
push to GitHub. The scan runs as a GitHub Actions workflow step.

Why CI rather than local: a local scan only runs when a developer remembers to run it.
A CI scan runs on every push, catches vulnerabilities introduced by dependency updates
that no developer explicitly requested, and creates a record in the GitHub Actions log.

Snyk is the third path into the security observability picture — its findings go to the
Snyk dashboard rather than Axiom, but the principle is the same: independent detection,
not dependent on the application being healthy.

---

## Step 5 — The Authentication Foundation

### The proxy gate (`src/proxy.ts`)

Every request to any page passes through the proxy gate before reaching the application.
The gate reads the Supabase session from cookies and makes one decision: is this request
from an authenticated user? If yes, continue. If no, redirect to `/login`.

Critical design choice: **fail closed**. If the session check itself fails — network
error, cookie parse error, Supabase timeout — the gate denies access rather than allowing
it. "I don't know if you're authenticated" is treated the same as "you are not
authenticated." This prevents a class of bugs where a malfunctioning auth check silently
lets everyone through.

The gate uses `getClaims()` with a null-safe destructure rather than `auth.getSession()`.
The difference: `getSession()` trusts the session data from the client-side cookie
without re-validating it with Supabase's servers. `getClaims()` validates. An attacker
who can forge a cookie would fool `getSession()` but not `getClaims()`.

### Open-redirect prevention (`src/lib/safe-redirect.ts`)

The login flow accepts a `?redirect=` URL parameter so users land back where they were
after signing in. Without validation, an attacker can craft a link like
`/login?redirect=https://evil.com` and redirect authenticated users to a phishing page.

`safe-redirect.ts` validates the redirect path against a strict allowlist before using
it. External URLs, protocol-relative URLs (`//evil.com`), and backslash variants
(`/\evil.com`) are all rejected. Only paths that start with `/` and don't escape the
origin are allowed.

The verification script tests seven cases: four that must be rejected and three that
must be accepted. All seven must pass.

### Cookie handling (`src/lib/supabase/server.ts`)

Supabase SSR requires the server client to read cookies from the incoming request and
write updated cookies to both the request and the response. Getting this wrong produces
subtle auth bugs — sessions that appear valid but expire prematurely, or refresh tokens
that never propagate. The implementation uses `getAll`/`setAll` with a try/catch in
`setAll`, so a cookie-write failure doesn't crash the request.

---

## Step 6 — Deploy and Configure

Production deployment on Vercel required configuring environment variables correctly:

- `NEXT_PUBLIC_SUPABASE_URL` — public-safe, browser-accessible
- `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY` — public-safe, browser-accessible
- Axiom URL and tokens — server-only, never `NEXT_PUBLIC_` prefixed

The Supabase project required two configuration changes:
- **Site URL** locked to the production Vercel URL — Supabase uses this to validate where
  auth redirects are allowed to go
- **Redirect allow-list** set to include only the production URL's `/auth/callback` path

Both of these prevent an attacker from registering their own site as a valid Supabase
auth redirect destination for the capstone project.

---

## Step 7 — Verify in Production

`scripts/verify-hardening.sh` is a shell script that runs against the live production
deployment and checks 28 specific conditions across five categories:

1. **Static / code checks** — confirms no service-role key references in application
   code, no password keys in log calls, correct public paths configuration
2. **Deployed HTTP surface** — confirms the live site redirects unauthenticated requests
   to `/login`, login and signup pages return 200
3. **Config via CLI / Management API** — confirms Vercel environment variables are set,
   Supabase site URL and redirect allow-list are correct, local HEAD matches origin/main
4. **Pipeline** — confirms production build passes, both pre-commit hooks are executable
5. **Axiom event-field checks** — confirms a real `login_failed` event exists in Axiom
   within the last 24 hours, has the correct fields, and contains no password or token
   values; confirms the webhook event exists with no credential fields

Final result: **28 PASS, 0 FAIL, 0 SKIP** with 2 known soft warnings (documented in
the script).

The script is the answer to "how do I know Phase 1 is still intact after I change
something?" Run it after any change that touches auth, logging, secrets, or RLS. A
regression shows up immediately.

---

## Problems Encountered

**The Cache-Control header on the root redirect didn't match the expected value.**
The proxy sets the header on the base response object, but `NextResponse.redirect()`
returns a separate response object, so the header doesn't transfer. This was documented
as a known soft issue — it affects caching behavior but not security. The verification
script flags it as a WARN rather than a FAIL.

**Axiom URL misconfiguration silently fell back to console logging.**
The logger is designed to fall back to console if the Axiom URL isn't set, which is
useful in local dev. In production this masked a missing environment variable for
several sessions before the `AXIOM_URL` check in the verification script caught it.
The fix was adding the variable to Vercel. The lesson: silent fallbacks in logging
infrastructure are hard to notice until you look for them explicitly.

---

## Key Takeaways

- **Fail closed is the correct default for authentication.** "I can't verify you" should
  produce the same result as "you are not verified." A gate that fails open is not a gate.

- **`getSession()` trusts client-supplied data; `getClaims()` validates it.** This is
  not a minor implementation detail — it's the difference between a gate that can be
  bypassed with a forged cookie and one that can't.

- **Three independent observability paths are stronger than one.** If the application
  logger breaks, the webhook path still fires. If both break, Snyk still scans. Independence
  means you're more likely to catch something even when part of the system is degraded.

- **A verification script is how you keep guarantees.** A passing test suite says the
  system behaved correctly when the tests were written. A verification script that runs
  against production says the system is behaving correctly *right now*.

- **Secrets in logs are almost as dangerous as secrets in code.** A log payload that
  includes a token value gets stored in log infrastructure, possibly with broader access
  controls than the secret itself. Redact before transmitting.

- **Open-redirect vulnerabilities are easy to introduce and easy to miss.** Any time a
  URL parameter is used to construct a redirect target, it needs to be validated. The
  test matrix (four reject cases, three accept cases) should be explicit and automated.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     REQUEST FLOW                                 │
│                                                                  │
│  Browser ──► proxy.ts (getClaims, fail-closed) ──► App pages    │
│                │                                                 │
│                └── unauthenticated ──► /login                   │
│                                            │                    │
│                                    safe-redirect.ts             │
│                                    (validate redirect param)    │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                  THREE OBSERVABILITY PATHS INTO AXIOM           │
│                                                                  │
│  Path 1: App logger (src/lib/logger.ts)                         │
│    App events ──► redact() ──► Axiom                            │
│                                                                  │
│  Path 2: DB webhook forwarder                                    │
│    Supabase auth event ──► DB webhook ──► Edge Function ──►     │
│    strip credential fields ──► Axiom                            │
│                                                                  │
│  Path 3: Snyk CI                                                 │
│    git push ──► GitHub Actions ──► Snyk scan ──► Snyk dashboard │
│                                                                  │
│  Each path is independent. One working tells you nothing         │
│  about the others.                                               │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                  VERIFICATION                                    │
│                                                                  │
│  scripts/verify-hardening.sh                                     │
│    28 checks · 5 categories · runs against live production       │
│    Rerun after any change touching auth, logging, or secrets     │
│                                                                  │
│  Final result: 28 PASS · 0 FAIL · 0 SKIP                        │
└─────────────────────────────────────────────────────────────────┘
```

---

## Interview Explanation

I hardened a Next.js application before building any features on top of it, treating
security controls as the foundation of the project rather than something retrofitted
later. The work covered three areas: preventing secrets from entering version control,
making authentication and login activity observable, and proving the controls are
working via a repeatable verification script.

I can explain the proxy gate design: it uses `getClaims()` rather than `getSession()`
because `getSession()` trusts client-supplied cookie data without re-validating with
Supabase's servers. A fail-closed gate treats an inconclusive auth check the same as
a failed one — "I can't tell" produces a redirect to login, not a pass-through.

I can explain why there are three independent observability paths into Axiom rather
than one. The application logger, the database webhook forwarder, and Snyk are
deliberately separate so that a failure in one doesn't create a blind spot. Each path
was verified individually in production, not just assumed to be working because the
others were.

The verification script was a deliberate output of Phase 1 — not just something that
passes on the day it was written, but a 28-check suite that can be re-run after any
future change to confirm nothing regressed. That's the difference between a security
control you implement once and a security property you maintain.

This was solo lab work in a capstone learning environment, not enterprise-scale
deployment. The tools (Supabase, Vercel, Axiom, Snyk) are real managed services, and
the controls are real, but the scale is a single-developer project.
