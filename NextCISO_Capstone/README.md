# NextCISO Academy Capstone

This folder documents the SurgicalWatch capstone project —
a threat intelligence dashboard for the Stryker Mako surgical
robotics fleet, built as part of the NextCISO Academy WMCAT 2026 program.

## Completed Labs

- **phase1-security-hardening.md** — Full writeup of hardening a
  Next.js application before building features: fail-closed proxy gate,
  sanitizing logger, auth-event forwarder, Snyk CI, open-redirect
  prevention, and a 28-check production verification script.

- **phase2-threat-intel-dashboard.md** — Full writeup of building the
  threat intelligence pipeline and dashboard: four automated Edge
  Functions ingesting CISA KEV/CVE/news feeds, 10-table Postgres schema
  with RLS, six-page Next.js dashboard, triage workflow, and pg_cron
  scheduling.

## What This Folder Demonstrates

End-to-end application security — from hardening a deployment before
launch, to building a self-sustaining data pipeline with defense-in-depth
database access controls and a working analyst workflow on top of it.
