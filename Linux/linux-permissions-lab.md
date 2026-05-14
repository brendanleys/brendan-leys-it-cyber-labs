# Linux File Permissions Lab

Date: May 15, 2026
Environment: WSL2 (Ubuntu) on Windows
Time spent: 2 hours

---

## Goal

Practice understanding and changing Linux file permissions by intentionally creating permission failures, diagnosing them with ls -l, and fixing them with chmod.

---

## Step 1 — Create a Script and Try to Run It

  $ echo '#!/bin/bash' > hello.sh
  $ echo 'echo Hello from my script' >> hello.sh
  $ cat hello.sh
  #!/bin/bash
  echo Hello from my script

  $ ./hello.sh
  -bash: ./hello.sh: Permission denied

Why it failed: The file does not have execute permission. When you create a file, it gets default permissions of rw-r--r-- (read and write for owner, read only for everyone else). Execute permission is not granted by default.

---

## Step 2 — Read the Permission String

  $ ls -l hello.sh
  -rw-r--r-- 1 brendan brendan 32 May 15 10:04 hello.sh

How to read the permission string -rw-r--r--:

  Position 1:    -        File type. - = regular file, d = directory
  Positions 2-4: rw-      Owner permissions: read=yes, write=yes, execute=no
  Positions 5-7: r--      Group permissions: read=yes, write=no, execute=no
  Positions 8-10: r--     Others permissions: read=yes, write=no, execute=no

---

## Step 3 — Add Execute Permission and Re-run

  $ chmod +x hello.sh

  $ ls -l hello.sh
  -rwxr-xr-x 1 brendan brendan 32 May 15 10:04 hello.sh

  $ ./hello.sh
  Hello from my script

chmod +x adds execute permission for everyone (owner, group, and others). The permission string changed from rw-r--r-- to rwxr-xr-x.

---

## Step 4 — Numeric Permission Notation

  $ chmod 755 hello.sh
  $ ls -l hello.sh
  -rwxr-xr-x 1 brendan brendan 32 May 15 10:04 hello.sh

  $ chmod 644 hello.sh
  $ ls -l hello.sh
  -rw-r--r-- 1 brendan brendan 32 May 15 10:04 hello.sh

How numeric permissions work:
  7 = rwx  (read + write + execute = 4+2+1)
  6 = rw-  (read + write = 4+2)
  5 = r-x  (read + execute = 4+1)
  4 = r--  (read only)
  0 = ---  (no permissions)

Each digit sets permissions for owner, group, others (in that order).
  755 = owner gets rwx, group gets r-x, others get r-x. Used for scripts.
  644 = owner gets rw-, group gets r--, others get r--. Used for regular files.

---

## Step 5 — whoami, id, and sudo

  $ whoami
  brendan

  $ id
  uid=1000(brendan) gid=1000(brendan) groups=1000(brendan),4(adm),27(sudo)

  $ sudo whoami
  root

whoami  prints your current username.
id      prints your user ID, primary group, and all group memberships.
sudo    runs the next command as root. Root can read, write, and execute any file.

---

## Step 6 — Root-Owned File Permission Failure

  $ sudo touch /tmp/root_file.txt
  $ ls -l /tmp/root_file.txt
  -rw-r--r-- 1 root root 0 May 15 10:22 /tmp/root_file.txt

  $ rm /tmp/root_file.txt
  rm: cannot remove '/tmp/root_file.txt': Permission denied

  $ sudo rm /tmp/root_file.txt
  (succeeds — no output means success)
  $ ls /tmp/root_file.txt
  ls: cannot access '/tmp/root_file.txt': No such file or directory

Why the first rm failed: The file is owned by root. Even though others can read it, they cannot write to it or delete it. sudo rm worked because sudo temporarily gives root-level privileges.

---

## Key Takeaways

- Every file has three permission sets: owner, group, others.
- Each set has three bits: read (4), write (2), execute (1).
- chmod +x adds execute. chmod 755 sets exact permissions numerically.
- sudo runs a command as root. Use it carefully and only when necessary.
- In corporate environments, most users do not have sudo access.
- Permission denied errors are common: user can't read a log, script can't write output, web server can't read a config file.
