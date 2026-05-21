# Credential Migration: Plaintext File to Bitwarden Vault

Date: May 20, 2026

Environment: ChromeOS (Linux container / Penguin), Bitwarden web vault, capstone web services

Time spent: 2.5 hours

NOTE: All credential values shown in this document are visibly fake placeholders. No real tokens, keys, or URLs are recorded here. The work itself was performed against real capstone accounts; the documentation is sanitized for public posting.

---

## Goal

Retire a plaintext credentials file used as temporary scaffolding during early capstone setup and migrate all credentials into Bitwarden — a secrets manager with master-password and 2FA-protected encrypted storage. The goal was to take a deliberate "wrong" pattern (plaintext secrets on disk) and move to a correct one (encrypted vault with verified recovery path), without losing access to any service in the process.

This was part of the NextCISO Academy SCADA/OT Threat Intelligence Dashboard capstone. The credentials being migrated were used to access GitHub, Supabase, Vercel, Anthropic/Claude.ai, plus tokens for two new tools introduced during this session (Axiom for logging, Snyk for vulnerability scanning).

---

## Starting state

A single plaintext file at `~/credentials-notes.txt` held the following types of values:

- GitHub Personal Access Token (fine-grained, scoped to one repo)
- Supabase project URL, anon key, service role key, database password, database connection string
- Vercel production URL and account metadata
- Account emails for four services

A nano editor backup file (`credentials-notes.txt.save`) was also present from an earlier paste failure — discovered during the audit and treated as a second copy that needed the same treatment.

Why this state was a problem:

- Plaintext means any process or person reading the file gets every secret instantly. No decryption, no password.
- A single mistaken `git commit` could publish the file to GitHub history.
- The service-role key for Supabase is an admin-level key that bypasses Row Level Security; leaking it would mean total compromise of the capstone database.
- File backups (`.save`, swap files) from editors create additional copies a user may forget about.

---

## Approach

The migration followed a deliberate sequence rather than ad-hoc copying. Plain-English summary of each phase:

1. **Set up the vault first** — Bitwarden account, master password written on physical paper, 2FA via authenticator app, 2FA recovery code also on paper, Secrets Manager activated, project created.
2. **Practice the new habit on brand-new credentials** — created Axiom and Snyk accounts and captured each token directly into Bitwarden Secrets Manager. The token never touched the plaintext file. Two reps of "secret appears, secret goes straight to the vault" before touching the old file.
3. **Migrate every existing credential** using a consistent naming convention (the secret's name in the vault matches the environment-variable name the code uses — e.g., `SUPABASE_SERVICE_ROLE_KEY` not "Supabase secret key").
4. **Sort each credential into the right place**: logins (username + password) went into Password Manager, machine secrets (tokens, keys) went into Secrets Manager assigned to a project, and reference data (URLs, project IDs) went into Notes fields of related secrets.
5. **Verify everything before deleting anything** — five-gate verification before the file was destroyed.
6. **Securely delete** both the main file and the editor backup using `shred -u` rather than `rm`.

---

## Step 1 — Set Up the Vault and Recovery Path

The vault is only as safe as the recovery path. Bitwarden uses zero-knowledge encryption — they cannot reset the master password. If the master password is lost and no recovery code is available, every secret in the vault is gone.

Setup completed:

- Bitwarden account created with the capstone-only Gmail.
- Master password: a 4-word passphrase, written on physical paper, never stored on the computer.
- Two-factor authentication: enabled via TOTP authenticator app (Google Authenticator).
- 2FA recovery code: written on the same physical paper as the master password.
- Bitwarden Secrets Manager activated on the Free tier (provides up to 3 projects and unlimited secrets).
- Project created: `Threat Intelligence System` (one project per capstone, plus a separate `Personal Projects` later for unrelated personal work).

---

## Step 2 — The Naming Convention

A consistent naming rule for every secret:

> The secret name in the vault equals the environment-variable name the code uses. No prefix, no friendly label.

| Vault secret name              | What it is                                   |
|--------------------------------|----------------------------------------------|
| `GITHUB_PAT`                   | GitHub fine-grained Personal Access Token    |
| `SUPABASE_SERVICE_ROLE_KEY`    | Supabase admin key (bypasses Row Level Security) |
| `SUPABASE_DB_PASSWORD`         | Direct Postgres password                     |
| `SUPABASE_DB_URL`              | Full Postgres connection string              |
| `NEXT_PUBLIC_SUPABASE_URL`     | Supabase project URL (safe for browser)      |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY`| Supabase publishable key (safe for browser)  |
| `NEXT_PUBLIC_APP_URL`          | App's deployed Vercel URL                    |
| `AXIOM_TOKEN`                  | Axiom log ingest token                       |
| `SNYK_TOKEN`                   | Snyk vulnerability scanner auth token        |

Why this matters: when the application code reads `process.env.SUPABASE_SERVICE_ROLE_KEY`, and a vault tool later automates secret injection, the names match. No mental translation needed. If the vault calls it "Supabase secret key (the big important one)," every future automation hits a wall.

---

## Step 3 — Sort Each Credential into the Right Container

Bitwarden has two halves and the split matters:

- **Password Manager** — for logins (things you type to sign in: username + password + recovery codes).
- **Secrets Manager** — for machine secrets (things your code uses: tokens, keys, connection strings), assigned to a project.

Reference data (URLs, project IDs, account display names) went into the Notes field of a related secret rather than getting its own item. These values aren't secret but stay useful when grouped with the credentials they relate to.

End-state inventory: 4 login items in Password Manager, 9 secrets in the `Threat Intelligence System` project, 1 secret in a `Personal Projects` project for an unrelated personal token that happened to live in the same file.

---

## Step 4 — The One Credential with Its Own Rule

The Supabase service-role key is the highest-risk credential in the capstone. It bypasses every Row Level Security policy in the database. If leaked, the database is fully compromised regardless of other protections.

The rule applied:

> `SUPABASE_SERVICE_ROLE_KEY` lives in exactly three places, ever:
> 1. Local `.env.local` (working copy, used by dev server — file is gitignored)
> 2. Supabase Edge Secrets (production working copy, encrypted, set via CLI)
> 3. Bitwarden Secrets Manager (backup of record)
>
> NEVER in: Vercel · source code · `CLAUDE.md` · any plaintext file.

Before migrating this key, verified `.env.local` is gitignored:

```bash
$ grep ".env" .gitignore
# env files (can opt-in for committing if needed)
.env*
```

The `.env*` line means git ignores any file starting with `.env` — including `.env.local`. The working copy cannot accidentally end up in repo history.

The Notes field of the vault secret records the rule alongside the key itself, so future-me reading the vault sees the constraint, not just the value.

---

## Step 5 — Verification Gate (Before Deletion)

Five checks before the plaintext file was destroyed. Skipping any of these creates a lockout risk.

1. **Every line of the plaintext file has a corresponding vault entry.** Verified line by line — not "most lines," every line.
2. **Spot-check the two highest-stakes values.** Compared the first 8 and last 8 characters of `SUPABASE_SERVICE_ROLE_KEY` and `GITHUB_PAT` between the file and the vault. A truncated paste always cuts off one of the ends — both-ends match means the middle is overwhelmingly likely to match too.
3. **Inventory check.** Confirmed all 10 secrets present, all assigned to a project (no orphans). Confirmed all 4 logins present in Password Manager.
4. **Independent log-out / log-back-in test.** Fully logged out of Bitwarden. Logged back in using only the master password (from memory + paper) and 2FA code from the authenticator app. No reference to any file on disk. This proves the vault is self-sufficient — that the user is not secretly depending on the file about to be deleted.
5. **Recovery verified.** Master password and 2FA recovery code both confirmed physically written down and stored where they'll be found months later.

---

## Step 6 — Secure Deletion

A normal `rm` does not erase the file's bytes from the disk — it only removes the directory entry pointing to them. For something this sensitive, the right tool is `shred`, which overwrites the file's contents with random data multiple times before unlinking it.

Commands run:

```bash
$ shred -u ~/credentials-notes.txt
$ shred -u ~/credentials-notes.txt.save
$ ls ~/credentials-notes.txt ~/credentials-notes.txt.save
ls: cannot access '/home/brendan/credentials-notes.txt': No such file or directory
ls: cannot access '/home/brendan/credentials-notes.txt.save': No such file or directory
```

What each part does:

- `shred` — overwrites file contents with random data (3 passes by default).
- `-u` — unlinks (removes) the file after overwriting.
- `ls` confirmation — "No such file or directory" is what success looks like.

Then ran follow-up cleanup checks for stray copies:

- Editor backups (`.swp`, `~` suffix files): none found.
- Downloads folder: nothing relevant.
- Bash history (`grep -iE 'github_pat_|sb_secret_|...' ~/.bash_history`): no real credential values found in history.
- Cloud sync: confirmed the ChromeOS Linux container is not synced to Google Drive or any other cloud service.
- Terminal scrollback: closed and reopened the terminal to clear any visible buffer.

---

## Key Takeaways

- The easiest time to handle a secret safely is the first second of its existence. The Axiom and Snyk tokens never touched a text file because the habit was set up in advance — the migration only existed because earlier credentials weren't handled that way.
- The verification gate matters more than the migration. Five honest yes/no checks stand between successful migration and locking yourself out. Skipping them to "save time" risks losing access to every service in the vault.
- `rm` is not deletion. For sensitive files, `shred -u` overwrites the bytes before unlinking; without it, the data may remain recoverable on disk.
- Editor backup files (`.save`, `.swp`, `~` suffix) are an underappreciated leak vector. Always audit the directory before assuming the visible file is the only copy.
- Naming convention pays off later. Vault entries named after their environment-variable names mean code and vault stay aligned without translation.
- The "split brain" between Password Manager (logins) and Secrets Manager (machine secrets) is a real organizational tool, not just Bitwarden's product structure — it matches the mental model real teams use.

---

## Architecture: Where Each Credential Lives After Migration

```
                  ┌───────────────────────────┐
                  │ BITWARDEN (encrypted vault)│
                  │ Master password + 2FA      │
                  │                            │
                  │ Password Manager:          │
                  │   GitHub / Supabase /      │
                  │   Vercel / Anthropic logins│
                  │                            │
                  │ Secrets Manager:           │
                  │   GITHUB_PAT, AXIOM_TOKEN, │
                  │   SNYK_TOKEN,              │
                  │   SUPABASE_SERVICE_ROLE_KEY│
                  │   SUPABASE_DB_PASSWORD,    │
                  │   SUPABASE_DB_URL,         │
                  │   NEXT_PUBLIC_* (URL, ANON,│
                  │   APP_URL)                 │
                  └────────────┬───────────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
       .env.local       Supabase Edge        Vercel env vars
       (laptop)          Secrets              (production)
       gitignored        (prod-only)
       all secrets       service-role         ONLY:
                         key only             NEXT_PUBLIC_*

           ┌────────────────────────────────┐
           │ credentials-notes.txt          │
           │ ❌ DELETED (shred -u)          │
           │ no longer a storage location   │
           └────────────────────────────────┘
```

Rules encoded in this layout:

- Vault is the source of truth. The other locations are working copies populated from the vault.
- The service-role key is in exactly three locations. Never in Vercel, never in code.
- Only `NEXT_PUBLIC_*`-prefixed values are deployed to Vercel because anything `NEXT_PUBLIC_` is sent to every browser that loads the app — these must never be secret values.

---

## Interview Explanation

I migrated a plaintext credentials file into Bitwarden during a capstone setup session, after deliberately using the plaintext file as temporary scaffolding to feel why it's the wrong pattern. The migration covered 14 credentials across GitHub, Supabase, Vercel, Anthropic, and two tools introduced during the session (Axiom and Snyk).

I can explain the difference between Bitwarden's Password Manager and Secrets Manager halves and which credentials belong in each. I can explain why the service-role key for Supabase had to be treated with stricter rules than the other credentials (it bypasses Row Level Security and would mean total database compromise if leaked). I followed a verification gate — proving the new vault was usable without reference to the plaintext file — before securely deleting the file with `shred -u` instead of `rm`.

This was lab-based practice in a capstone learning environment. The principles — secrets manager use, naming conventions, separation of logins from machine secrets, verification before destruction, secure deletion of sensitive files — are real and applied to real services, but the scale is one user, not an enterprise team.
