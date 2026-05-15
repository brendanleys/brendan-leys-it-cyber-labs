
# Linux Practice

## What I Practiced

Linux command-line fundamentals using WSL (Windows Subsystem for Linux): navigating the filesystem, managing file permissions, analyzing log files with grep and text processing tools, and running network diagnostic commands.

## Commands Used

- Navigation: pwd, ls, ls -l, ls -la, cd, mkdir, touch, cp, mv, rm
- File viewing: cat, less, head, tail
- Permissions: chmod, chown, whoami, id, sudo
- Log analysis: grep, grep -i, grep -c, grep -oP, sort, uniq -c, wc -l
- Networking: ip a, ping, nslookup, curl -I, ss -tuln

## Files in This Folder

- linux-navigation.md — filesystem navigation commands with plain-English explanations
- linux-permissions-lab.md — permission failure, chmod walkthrough, sudo practice
- log-filtering-and-networking.md — grep log analysis and network diagnostic commands
- fake_auth.log — sample auth log used for grep practice (fake data — all entries invented)

## What Broke or Confused Me

Running ./hello.sh failed with "Permission denied" until I ran chmod +x hello.sh — this made the concept of execute permissions concrete in a way that reading about it did not.

## How I Fixed It

Read the permission string from ls -l output, identified that execute permission was missing, used chmod +x to add it, and verified with ls -l again before re-running.

## Interview Explanation

I practiced Linux filesystem navigation and permissions by running each command manually and documenting what it does. I can explain pwd, ls, cd, mkdir, touch, cp, mv, rm, cat, head, tail, less, chmod, and sudo, and demonstrate when I would use each one in a troubleshooting scenario. All practice was done in a local WSL environment — this is not production Linux administration.

