# Key Rotation Exercise

## What This Is

The direct continuation of `credential-migration-bitwarden.md`. The migration moved every secret out of a plaintext file and into Bitwarden; this exercise retired those migrated values and replaced them with fresh ones. After this lab, any copy of the original key values that might have leaked from a plaintext file, a backup, or shell history can no longer authenticate to anything — they are dead at the source.

Completed at the start of the WMCAT/GRCIE 2026 capstone Phase 1 setup, before the Phase 1 security build itself begins.

## Skills Practiced

- **Credential lifecycle:** verify-before-revoke discipline, create-new-then-swap vs. regenerate-in-place, recognizing instant-kill exceptions (Snyk), expiration-setting as a forcing function for future rotation
- **Cloud config across providers:** Supabase (new "API Keys" tab with the modern key class), Axiom (in-place token regeneration), Vercel (Sensitive flag, encrypted-at-rest, Hobby plan two-entry workaround), GitHub Actions secrets, GitHub fine-grained Personal Access Tokens (per-repo access and per-permission scoping)
- **Linux / terminal hygiene:** loading secrets via `read -s` instead of inline command arguments, never including a secret value in any command that goes to shell history, using `git credential reject` to clear stale cached credentials without touching plaintext storage
- **End-to-end verification:** sending a real test event to Axiom via curl using the new ingest token and watching it arrive in the Stream view (not trusting the curl's own success code), confirming a git push landed via `git log origin/main --oneline -3` rather than the push command's success message
- **Process discipline:** working from a structured runbook, pausing to verify the actual state of files and dashboards before acting, recovering from a wrong-folder trap that produced a false-alarm output

## What Was Rotated

Six credentials, in deliberate order from lowest-blast-radius to highest:

1. **Supabase publishable key** — the browser-safe API key (the credential formerly named "anon key"). Method: create-new-then-swap with a safe overlap window. Verified by app restart and clean page load.
2. **Axiom ingest token** — the server-only token that writes events to my observability dataset. Method: regenerate in place. **Verified end-to-end** by sending a test event with the new token via `curl`, then watching it arrive in Axiom's Stream view with the expected payload.
3. **Snyk token** — the CI dependency-scanner credential. Method: revoke & regenerate (instant kill, no overlap window — the one exception to the verify-before-revoke rule). Placed in GitHub Actions secrets for future consumption when the CI workflow is wired up later in the build.
4. **GitHub fine-grained PAT** — used for `git push` over HTTPS. Method: regenerate in place. Verified with a deliberate empty commit + push + upstream log check.
5. **Supabase database password** — direct-Postgres password, not consumed by the app at this stage (the app uses API keys, not the raw DB connection). Method: reset. No verify needed; no current consumer.
6. **Supabase secret key** — the server-only, RLS-bypassing key (the credential formerly named "service_role key"). Most powerful credential in the project, handled with the strictest rules (never pasted into any terminal command, ever). Method: create-new-then-swap. Soft verify via clean app restart; the full end-to-end verify is deferred to when the webhook → Edge Function path consumes it during the Phase 1 build proper.

## What I Caught Along the Way

These are the moments where the discipline of the exercise mattered, not just the mechanics:

**Supabase old-vs-new key-class naming mismatch.** The exercise documentation referenced `NEXT_PUBLIC_SUPABASE_ANON_KEY` and `SUPABASE_SERVICE_ROLE_KEY`, but my project (using Supabase's newer key class) actually used `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY` and `SUPABASE_SECRET_KEY`. The doc was a guide; the project state was the truth. I reconciled by reading `.env.local` and the Vercel dashboard directly rather than blindly editing or adding lines based on the doc's naming.

**Forward-looking variables (no consumer yet).** `AXIOM_TOKEN` wasn't yet present in `.env.local` or Vercel — those touchpoints would come later when the logger and the deploy step actually need them. Instead of forcing the doc's instructions onto a project state that didn't match, I added the variable in both places forward-looking, so the later steps wouldn't trip on a missing env var.

**Vercel's "Rotate" button is not the same thing as rotating.** When configuring `AXIOM_TOKEN` in Vercel, I noticed a "Rotate" option and paused before clicking. After checking Vercel's documentation, I learned it's tied to their April 2026 security-incident response flow and the Marketplace integration API — not a general-purpose rotation tool for manually-managed values. Clicking it on a manually-entered credential can create duplicate orphan entries. The correct path was plain "Edit."

**Vercel Hobby plan + Sensitive flag.** Two of the tokens (`AXIOM_TOKEN`, `SUPABASE_SECRET_KEY`) needed to be added to Vercel as Sensitive (encrypted at rest, hidden after creation). The Hobby-plan UI wouldn't let me select both Production and Preview environments in a single entry. I worked around it by creating two separate entries with the same variable name — one scoped to Production, one to Preview — and noted this in my rotation map so future rotations remember to update both.

**A 403 traced through carefully, not "fixed" by guessing.** After regenerating my GitHub PAT and pushing, I got a 403 — "Write access to repository not granted." The instinctive read is "scope problem." But on inspection, the PAT's Contents permission was already set to Read and Write. The actual cause was a **stale credential cache**: git silently used an older cached PAT for auth, which authenticated successfully but lacked write access on the repo. I diagnosed by bypassing the cache for one push (`git -c credential.helper= push origin main`), which forced an interactive prompt and let the new PAT be tested cleanly. Confirmed the new PAT worked, then cleared the stale cache with `git credential reject` so future pushes would re-seed correctly.

**The wrong-folder reflex.** Twice in this session, a diagnostic was run from the wrong directory and produced an output that looked alarming until verified. The first was `npm run dev` from `~/projects` instead of the project folder. The second — more importantly — was a `cat .gitignore | grep -i env` that returned "No such file or directory" and looked like a missing safety net. But the command had been run from outside the project. From inside the project, `.gitignore` was present and protecting `.env*` correctly the whole time. **The lesson: a check that draws conclusions about your project but isn't run *from* your project isn't actually a check.** The *where* matters as much as the *what*.

## Why This Matters

The exercise had a stated throughline that ran through every step: **"Verify the actual thing, from the environment where it has to be true, and distrust 'it probably worked.'"**

Each of the catches above is a version of that principle. The naming mismatch was caught by reading `.env.local` rather than trusting the doc. The 403 was solved by isolating the credential variable rather than guessing at scopes. The wrong-folder false alarm was resolved by verifying location before drawing conclusions. The Axiom rotation was confirmed by watching a real event arrive in the consumer system, not by reading curl's success code.

This is the same discipline that catches problems in production: it's not about preventing the unexpected; it's about verifying the expected.

## Interview Explanation

I rotated six credentials at the start of a security-build capstone: two Supabase API keys (publishable and secret), the Supabase database password, an Axiom observability ingest token, a Snyk dependency-scanner token, and a GitHub fine-grained personal access token. I used create-new-then-swap with verify-before-revoke wherever the service supported parallel keys (Supabase), regenerate-in-place where the service only supports one active value (Axiom, GitHub PAT), and recognized Snyk as the instant-kill exception. Each rotation was verified at the *consumer* end — not by trusting the rotate command's success code, but by exercising the new credential in the system that uses it. The Axiom verify, for example, used `read -s` to load the new token into a shell variable without exposing it in shell history, then sent a real test event via curl, then visually confirmed the event arrived in Axiom's Stream view. I documented where each credential lives across all storage locations, recorded the last-rotated date, and set 90-day calendar reminders.

I can walk through any specific catch — the 403 I traced to a stale credential cache rather than a scope issue, the Supabase old-vs-new key naming mismatch I reconciled against the doc, or the Vercel Hobby-plan Sensitive-flag workaround.

## Caveat

This is lab-based credential rotation in a coursework project, not production incident response. I am not claiming production rotation experience — I am claiming I understand the verify-before-revoke discipline, the difference between in-place and create-new-then-swap rotation patterns, the principle that secrets should never appear in command-line arguments or shell history, and I can execute the rotation end-to-end across six provider dashboards while keeping the verification rigorous.
