# Phase 2: Mako Threat Intelligence Dashboard

Date: June 6, 2026

Environment: ChromeOS (Linux container / Penguin), Supabase Free tier, Vercel Hobby, Next.js 16

Time spent: ~8 hours across two sessions

NOTE: All credential values, secret keys, and tokens referenced in this document are either
placeholders or public-safe values (publishable keys). No service-role keys, database
passwords, or private tokens are recorded here. Real credentials were handled via Bitwarden
and Supabase Edge Secrets as documented in the credential-migration writeup.

---

## Goal

Build a self-sustaining threat intelligence application on top of the Phase 1 security
foundation — a live dashboard that automatically ingests vulnerability data from public
threat feeds, matches that data against a real medical device asset inventory, and gives
an analyst a working triage workflow. The target: reduce the Monday morning vulnerability
review from ~300 minutes to under 30 minutes.

The application is scoped to the Stryker Mako SmartRobotics fleet — surgical robotic
systems used in orthopedic procedures. The persona is "Maya Chen," an HTM (Healthcare
Technology Management) Cybersecurity Analyst managing FDA 21 CFR Part 11 and IEC 62443
compliance for a health system.

---

## What Was Built

A full-stack threat intelligence application with:

- **4 automated Edge Functions** pulling from public threat feeds on a schedule
- **10-table Postgres database** with Row Level Security on every table
- **6-page Next.js dashboard** for analyst triage workflow
- **18 seeded assets** representing the Mako surgical robotics fleet
- **pg_cron schedules** running the pipeline automatically 24/7

Live at: `scada-threat-dashboard-ten.vercel.app`

---

## Tech Stack

| Layer | Technology | Why |
|-------|------------|-----|
| Frontend | Next.js 16 (App Router) | Server Components enable server-side auth checks without client-side exposure |
| Database | Supabase Postgres 15 | Row Level Security, Edge Functions, and pg_cron in one managed service |
| Backend | Supabase Edge Functions (Deno) | Serverless, co-located with the database, secrets stored separately from code |
| Scheduling | pg_cron + pg_net | Database-native scheduling; no external cron service needed |
| Hosting | Vercel Hobby | Zero-config Next.js deployment |
| Auth | Supabase Auth (Phase 1) | Already hardened in Phase 1; Phase 2 builds on top |

---

## Database Architecture

### 10 tables + 1 admin view, all with Row Level Security

```
track_config          — surgical-robotics track identity and keyword arrays
assets                — 18 Mako fleet assets with criticality ratings
critical_processes    — named clinical processes (Robotic Surgical Procedure Execution, etc.)
asset_critical_processes  — many-to-many: which assets serve which processes
threat_events         — every CVE, KEV entry, and news article ingested
asset_threat_events   — computed matches: (asset, threat_event) pairs from the cascade
recommendations       — per-match remediation guidance
triage_state          — analyst workflow state per match (open/acknowledged/mitigated/etc.)
disputes              — analyst challenges to low-confidence matches
ingest_log            — one row per feed run with status, count, and cadence

v_admin_system_health — read-only view aggregating ingest_log by source
```

### Two-layer anonymous write defense

Every table has RLS enabled (Layer 1) AND has all anonymous privileges revoked at the
Postgres GRANT level (Layer 2). The principle: if RLS is ever misconfigured, the database
privilege wall still holds. Two independent enforcement points.

```sql
-- Layer 1: RLS default-deny (no policy for anon = zero access)
ALTER TABLE public.<table> ENABLE ROW LEVEL SECURITY;

-- Layer 2: Revoke all table-level privileges from the anon role
REVOKE ALL ON ALL TABLES IN SCHEMA public FROM anon;

-- Grant only what authenticated users need
GRANT SELECT ON public.track_config, public.assets, ... TO authenticated;
GRANT SELECT, INSERT, UPDATE ON public.triage_state TO authenticated;
```

---

## The Data Pipeline

### 4 Edge Functions, automated by pg_cron

| Function | Schedule | Source | What it does |
|----------|----------|--------|--------------|
| `ingest-cisa` | Every 6h | CISA KEV JSON | Pulls ~1,600+ Known Exploited Vulnerabilities |
| `ingest-cvelist` | Every 1h | CVE Project GitHub Releases | Downloads delta zip, unzips in memory, parses CVE JSON |
| `ingest-news` | Every 6h | 6 RSS/Atom feeds | BleepingComputer, Krebs, CyberScoop, SANS ISC, The Hacker News, The Register |
| `compute-asset-mappings` | Every 30min | Internal | Runs the 4-method matching cascade against all assets |

Note: A ransomware ingest function was planned but not built — the free public API was
withdrawn mid-project. Ransomware is surfaced as a news-filter feature instead.

### The 4-method matching cascade

For each threat event, the matcher evaluates each asset in priority order and stops at
the first match:

```
1. CPE 2.3 exact match          → confidence: high
2. vendor + product + version   → confidence: high
3. vendor + product             → confidence: medium
4. Keyword fallback             → confidence: low
```

Match method and confidence are stored in `asset_threat_events`. Medium and low confidence
matches show a "Dispute" option in the UI, letting analysts challenge the match.

---

## The Secret-Key Model

The Supabase CLI reserves the `SUPABASE_*` namespace for its own managed variables, which
means a custom Edge Secret named `SUPABASE_SECRET_KEY` gets rejected in production. The
solution is a two-name split:

```
Local (.env.local):     SUPABASE_SECRET_KEY=sb_secret_...
Edge Secret:            EDGE_DB_SECRET_KEY=sb_secret_...
Code reads:             EDGE_DB_SECRET_KEY ?? SUPABASE_SECRET_KEY
```

The Edge name wins in production. The local name works in dev. Only one function in the
entire codebase reads the secret — `getServiceRoleClient()` in `_shared/supabase.ts` for
Edge Functions, and `getServiceRoleClient()` in `src/lib/supabase/server.ts` for the
Next.js server. Grep-confirmed before every commit.

---

## The Dashboard

### 6 pages, all server-rendered

**Dashboard** (`/dashboard`) — Tabbed threat feed:
- New This Week: matches from the last 7 days that are open
- Critical Alerts: KEV-flagged or CVSS ≥ 9.0
- Triaged: all actioned matches
- Each row has an inline triage dropdown — click to update status without a page reload

**Fleet** (`/mako-units`) — Asset inventory table:
- 18 assets sorted by criticality
- Shows open threat count, KEV count, highest CVSS per asset
- Criticality badges: Critical (red), High (amber), Medium (yellow), Low (gray)

**Vulnerabilities** (`/vulnerabilities`) — Filterable vulnerability table:
- Filter by source (KEV / CVE List), severity (Critical/High/Medium/Low), and triage status
- Server-side filtering via URL search params — no client state needed
- "Fleet Secure" empty state when filters return no matches

**Reports** (`/reports`) — Threat posture summary:
- 4 stat cards: total assets, active threats, KEV-flagged, mitigated/accepted
- Per-asset breakdown table with open count, KEV count, highest CVSS, status summary
- Disabled Export CSV button (placeholder for future implementation)

**Audit Ledger** (`/reports/audit`) — Compliance trail:
- Chronological log of every triage decision with analyst ID, justification, and timestamp
- Populates automatically as analysts work through the queue

**Admin** (`/admin`) — Admin-only:
- System health cards for all 4 feed functions (freshness status, 24h record count, 7d failures)
- User management table (lists all profiles via service-role client to bypass RLS)
- Role-gated: non-admin users are redirected to `/dashboard` server-side

---

## Triage Workflow

The triage action is a Next.js Server Action — no API route needed:

```typescript
'use server';
export async function setTriageStatus(matchId: string, status: string): Promise<void> {
  // Authenticate first — server actions run server-side but must verify the caller
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) throw new Error('Not authenticated');

  // Upsert: creates a triage_state row if none exists, updates if one does
  await supabase.from('triage_state').upsert(
    { asset_threat_event_id: matchId, status, analyst_id: user.id,
      decided_at: status !== 'open' ? new Date().toISOString() : null },
    { onConflict: 'asset_threat_event_id' }
  );

  // Revalidate both pages so the UI reflects the change immediately
  revalidatePath('/dashboard');
  revalidatePath('/vulnerabilities');
}
```

The analyst picks a status from the inline dropdown. The server action fires, upserts the
row, and Next.js pushes refreshed server component data back to the client. No full page
reload. The status appears in the Audit Ledger automatically.

---

## Key Technical Decisions

### Why server components, not client components?

Server Components handle auth checks and data fetching before anything reaches the browser.
A malicious user cannot inspect network requests to extract query logic because the query
runs on the server. The client receives rendered HTML.

### Why pg_cron instead of an external scheduler?

pg_cron runs inside the database. No external service, no separate credentials, no network
hop. The schedule is version-controlled as a SQL migration. If the database goes down, the
schedule goes down too — which is the correct failure mode (no point calling an Edge
Function if the DB isn't there to write to).

### Why exclude news from the asset matcher?

News articles are ingested into `threat_events` with `source='news'` but are excluded from
`asset_threat_events` at the query layer. News items describe events, not vulnerabilities
with CPE strings or version information. Running the cascade against news would produce
thousands of low-confidence keyword matches that bury real CVE matches. News surfaces on
the dashboard via its own filter path.

### Why LEFT JOIN for fleet inventory?

PostgREST (Supabase's REST API layer) uses an INNER JOIN by default when embedding related
tables. An asset with zero threat matches would be silently excluded. The `!left` hint
forces a LEFT JOIN so all assets appear regardless of match count. This was a production
bug discovered by seeing an empty fleet table despite 18 seeded assets.

---

## Problems Encountered

**PostgREST INNER JOIN silently dropped assets with no threat matches.**
The fleet page showed empty even though 18 assets existed. Fixed with `!left` in the
embedded select query.

**Supabase CLI reserved namespace rejected SUPABASE_SECRET_KEY as an Edge Secret.**
The CLI manages its own `SUPABASE_*` variables. A custom secret with the same prefix is
rejected at deploy time. Fixed with the two-name split documented above.

**pg_cron SQL migration had URL truncation in the Claude Code interface.**
Long lines in the SQL were silently truncated when written through the Claude Code file
interface, producing broken Edge Function URLs that would have caused every cron job to
silently fail. Detected by grepping the actual file after writing and comparing against
expected values. Fixed by writing the migration manually in nano and verifying with grep.

**Security definer view flagged as "unrestricted" in Supabase dashboard.**
`v_admin_system_health` was flagged because views don't have RLS by default. Fixed with a
follow-up migration setting `security_invoker = true`, which makes the view execute under
the caller's role instead of the view owner's role, so RLS on the underlying table applies.

---

## Architecture Diagram

```
                        ┌─────────────────────────────┐
                        │   Public Threat Feeds        │
                        │   CISA KEV / CVE Project /   │
                        │   RSS News (6 sources)        │
                        └──────────────┬───────────────┘
                                       │ HTTP (every 1–6h)
                        ┌──────────────▼───────────────┐
                        │   Supabase Edge Functions     │
                        │   (Deno runtime)              │
                        │                               │
                        │  ingest-cisa    (every 6h)   │
                        │  ingest-cvelist (every 1h)   │
                        │  ingest-news    (every 6h)   │
                        │  compute-asset-mappings       │
                        │                   (every 30m) │
                        └──────────────┬───────────────┘
                                       │ service role (bypasses RLS)
                        ┌──────────────▼───────────────┐
                        │   Supabase Postgres 15        │
                        │                               │
                        │  threat_events                │
                        │  asset_threat_events          │
                        │  triage_state                 │
                        │  ingest_log                   │
                        │  ... (10 tables total)        │
                        │                               │
                        │  pg_cron: triggers Edge       │
                        │  Functions via pg_net         │
                        └──────────────┬───────────────┘
                                       │ authenticated role (RLS enforced)
                        ┌──────────────▼───────────────┐
                        │   Next.js 16 (Vercel)         │
                        │                               │
                        │  /dashboard   — threat feed   │
                        │  /mako-units  — fleet view    │
                        │  /vulnerabilities — filtered  │
                        │  /reports     — posture sum.  │
                        │  /reports/audit — audit trail │
                        │  /admin       — admin-only    │
                        │                               │
                        │  Server Actions (triage)      │
                        │  Phase 1 auth gate (proxy.ts) │
                        └──────────────────────────────┘
                                       │
                        ┌──────────────▼───────────────┐
                        │   Analyst / Admin (browser)   │
                        │   scada-threat-dashboard-     │
                        │   ten.vercel.app              │
                        └──────────────────────────────┘
```

---

## Key Takeaways

- **Server Components are the right default for security-sensitive pages.** Auth checks and
  data fetching happen before anything touches the browser. There is no client-side
  equivalent of a server-side redirect.

- **Defense in depth means two independent enforcement points.** RLS alone is not enough —
  RLS policies can have bugs. The GRANT revocation layer means the database itself refuses
  anon writes even if every RLS policy were wiped.

- **Verify the actual file, not the display.** The Claude Code interface truncated long SQL
  lines, making broken URLs look correct on screen. Grepping the actual file on disk caught
  the problem before it reached the database.

- **PostgREST defaults (INNER JOIN) are not always right.** An asset inventory page that
  silently omits assets with no matches is worse than wrong — it looks correct until someone
  notices a missing asset. Always test with assets in edge-case states (zero matches, all
  statuses).

- **A two-name secret split is the correct pattern for Supabase + Next.js.** The CLI's
  namespace reservation is real and will silently reject your secret. Test secret injection
  in production before depending on it.

- **pg_cron makes the pipeline self-sustaining.** Once the schedules are in the database,
  nothing needs to be manually triggered. The application is live and ingesting data 24/7
  without developer intervention.

---

## Interview Explanation

I built a full-stack threat intelligence application for medical device security as part of
the NextCISO Academy capstone. The application automatically pulls from three public threat
feeds — CISA's Known Exploited Vulnerabilities catalog, the CVE Project's delta releases,
and six security news RSS feeds — on a rolling schedule using pg_cron and Supabase Edge
Functions.

I can explain the Row Level Security model: every table has RLS enabled as the first layer,
and I separately revoked all anonymous Postgres privileges as the second layer. Both layers
must fail for an anonymous write to succeed, which is defense in depth applied to database
access control.

I can explain the secret-key architecture: the Supabase CLI reserves its own namespace,
so a custom secret with the same prefix gets rejected. I implemented a two-name split where
the Edge Functions read `EDGE_DB_SECRET_KEY` in production and fall back to
`SUPABASE_SECRET_KEY` in local dev, with only one function in the entire codebase calling
`getServiceRoleClient()`.

I can explain the 4-method matching cascade: CPE exact match → vendor+product+version →
vendor+product → keyword fallback, with confidence levels (high/medium/low) stored
alongside the match method. This lets the analyst see why a match was made and dispute it
if the confidence is low.

This was a solo build in a capstone learning environment using real public threat data
against a real database schema. The pipeline runs continuously; the dashboard shows live
data against the 18 seeded assets every time it loads.
