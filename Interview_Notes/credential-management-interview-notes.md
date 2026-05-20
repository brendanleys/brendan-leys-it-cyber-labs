# Credential Management Interview Notes

Prepared: May 20, 2026

Companion file to: GRC_Evidence/credential-migration-bitwarden.md

---

## Q1: Walk me through a time you handled credential hygiene.

As part of a capstone setup, I had a plaintext credentials file with about 14 secrets — GitHub PAT, Supabase service-role key, database password, project URLs, account emails for several services. It was fine as scaffolding during initial setup, but a plaintext credentials file is one accidental git commit away from being published to the internet, so I migrated it to Bitwarden.

I used Bitwarden's Password Manager for logins (the things you type to sign in — username + password) and its Secrets Manager for machine secrets (tokens, keys, the kind of values your code reads as environment variables). I used a naming convention where the vault entry name matches the environment variable name in code — so the Supabase admin key is stored as `SUPABASE_SERVICE_ROLE_KEY`, not "Supabase secret key." That way the vault and the code stay aligned without mental translation.

I then ran a verification gate before deleting anything: confirmed every line of the original file was migrated, spot-checked the highest-stakes values, did an independent log-out and log-back-in test to prove the vault was self-sufficient, and confirmed my recovery path (master password + 2FA recovery code) was physically written down. Then I used `shred -u` to securely overwrite and delete the file, not just `rm`.

---

## Q2: What's the difference between Bitwarden's Password Manager and Secrets Manager?

Password Manager is for human logins — the credentials you personally type or paste into a browser to sign in to a service. Username, password, recovery codes, MFA backup codes.

Secrets Manager is for machine secrets — values that code reads as environment variables. API tokens, database connection strings, service-role keys. Each secret can be assigned to a project, which is a logical grouping.

The split matters because the two categories have different lifecycles. Passwords change when a person resets them. Machine secrets change when they're rotated on a schedule or after a suspected compromise. Mixing them in one item list makes both harder to manage.

---

## Q3: Why isn't `rm` enough for deleting a credentials file?

When you run `rm`, the operating system removes the directory entry pointing to the file's data, but the actual bytes on disk usually remain intact until something else happens to overwrite that exact disk location. Forensic tools can often recover the contents.

`shred` overwrites the file's contents with random data — three passes by default — before unlinking it. The `-u` flag tells it to also remove the file after overwriting. So the byte pattern of the original content is gone, not just the pointer to it.

For something like a plaintext credentials file, where the bytes themselves are the entire risk, the overwrite step is the difference between "deleted" and "actually unrecoverable."

```bash
shred -u ~/credentials-notes.txt
```

---

## Q4: What's a service-role key and why does it need special handling?

In Supabase, the service-role key is the admin key. It bypasses every Row Level Security policy in the database — those are the access rules that normally prevent one user from reading another user's data. The service-role key sees everything and can do everything.

So if the service-role key leaks, the database is fully compromised regardless of every other security control. Authentication, RLS, user permissions — none of it stops a request authenticated with the service-role key.

The rule I follow: it lives in exactly three places.

1. Local `.env.local` (the working copy on a developer machine — and `.env.local` must be in `.gitignore` so it never goes to a repo)
2. Supabase Edge Secrets (the production working copy, set via the Supabase CLI, encrypted at rest)
3. Bitwarden Secrets Manager (the backup of record)

Never in Vercel environment variables, never in source code, never in any AI-readable config file like `CLAUDE.md`, never in a plaintext file. The Notes field of the vault entry records this rule alongside the key itself.

---

## Q5: What does the `NEXT_PUBLIC_` prefix mean and why does it matter?

In Next.js, any environment variable prefixed with `NEXT_PUBLIC_` is included in the JavaScript bundle sent to every browser that loads the app. The prefix is a deliberate signal that the value is **public by design** — every visitor to the site will see it in their browser's developer tools.

So you put `NEXT_PUBLIC_SUPABASE_URL` and `NEXT_PUBLIC_SUPABASE_ANON_KEY` in Vercel env vars — those are safe because they're meant to be public. The Supabase anon key has no admin power; Row Level Security policies in the database protect the actual data.

But you must never give a real secret a `NEXT_PUBLIC_` prefix. If `SUPABASE_SERVICE_ROLE_KEY` accidentally became `NEXT_PUBLIC_SUPABASE_SERVICE_ROLE_KEY`, every visitor to the site would receive the admin key in their browser.

---

## Q6: How would you verify a credentials migration is safe before deleting the original?

I'd use a verification gate — a checklist of yes/no questions that all need to pass before deletion.

1. Has every line of the original been migrated? Not "most" — every. I'd compare line by line.
2. Spot-check the highest-stakes values. Compare the first 8 and last 8 characters between source and vault for the most dangerous credentials. A truncated paste always cuts off an end, so both-ends matching is a strong signal.
3. Inventory check. Confirm every secret is assigned to a project — no orphans floating outside the organization structure.
4. Independent recovery test. Fully log out of the vault and log back in using only the master password and 2FA, without referencing any file on disk. This proves the vault is genuinely self-sufficient.
5. Confirm the recovery path is in place. Master password and 2FA recovery code both written down physically. Without that, a phone failure means losing the vault.

If any of those fail, you fix the gap before deleting anything. The original file isn't hurting anything while it sits there for one more hour.

---

## Q7: What is a secrets manager and why is it better than a plaintext file?

A secrets manager is a tool that stores secrets encrypted behind one master credential, with access control and (typically) audit logging. To read a secret, you unlock the vault — you don't just open a file.

The key differences from a plaintext file:

- Encryption at rest. Even if someone steals the storage, they can't read the contents without the master password.
- Multi-factor authentication. Reading the vault requires both something you know (master password) and something you have (2FA code).
- Logging. Some secrets managers log when a secret was accessed, by whom, and from where — useful for audit trails.
- Structured organization. Secrets are named, grouped, and have metadata (notes, links, related items).
- Programmatic access via API. Code can fetch a secret at runtime instead of having it baked into config files.

For a learning project, the manual benefits matter most. For a real team, the programmatic and audit benefits become essential.

---

## Important Caveat

This was lab-based practice in a single-user capstone learning environment. I'm not claiming professional secrets-management or enterprise vault administration experience. I'm claiming I can explain the principles, the practical patterns (naming conventions, separation of login from machine secrets, the verification gate before deletion, secure file shredding), and apply them to my own work. The next step would be to apply the same patterns at team scale — shared vaults, role-based access, automated rotation — which I have not done yet.
