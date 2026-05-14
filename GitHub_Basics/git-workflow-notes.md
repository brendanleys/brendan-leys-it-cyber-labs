# Git Workflow Reference Notes

These are my plain-English definitions for the core Git concepts I use every day in this study plan.

---

## Core Terms

**repository** — A project folder stored on GitHub that contains all your files plus a full history of every change ever made to them. Think of it as a project binder that also has a logbook showing every time anyone added, changed, or removed a page. My repository is at: https://github.com/brendan-leys/brendan-leys-it-cyber-labs

**clone** — Downloading a complete copy of a GitHub repository to your own computer so you can edit files locally. Like checking out a library book — you get your own copy to work with, but the original still exists online. Command: git clone https://github.com/brendan-leys/brendan-leys-it-cyber-labs.git

**stage (git add)** — Telling Git which files to include in your next save. Nothing is saved automatically — you choose what goes in. Like placing items in a box before sealing it. Command: git add filename.md or git add . to stage everything.

**commit (git commit)** — Taking a permanent snapshot of your staged files, with a short message describing what you did. This is one entry in your history log. Command: git commit -m "Your message here"

**push (git push)** — Sending your committed snapshots from your local computer up to GitHub so they are visible online. Like shipping the sealed, labeled box to a warehouse where anyone with the address can see it. Command: git push

**pull (git pull)** — Downloading the latest changes from GitHub to your local machine. Used when you make changes directly on the GitHub website and need to sync them locally. Command: git pull

**branch** — A separate line of development that lets you work on changes without affecting the main version. For this portfolio I work directly on the main branch. Command to see current branch: git branch

---

## My Daily Workflow

Every study session ends with these three commands in order:

  git add .
  git commit -m "Describe what you did specifically"
  git push

---

## Useful Diagnostic Commands

  git status          # See what has changed and what is staged
  git log --oneline   # See commit history in compact form
  git diff            # See exactly what changed in unstaged files
