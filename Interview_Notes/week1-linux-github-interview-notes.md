# Week 1 Interview Notes — Linux and GitHub

Prepared: May 17, 2026

---

## Q1: Tell me about your GitHub portfolio.

I built a public GitHub repository to document my hands-on IT and cybersecurity lab practice. The repository is organized into folders by skill area — Linux, Python, Networking, Windows Admin, SOC Log Analysis, and GRC Evidence. Each folder has a README explaining what is inside, and every change was committed from the terminal using git add, git commit, and git push. The goal was to create visible, navigable evidence of consistent practice rather than just claiming skills on a resume.

My repository is at: https://github.com/brendan-leys/brendan-leys-it-cyber-labs

---

## Q2: What Linux commands are you comfortable with?

For filesystem navigation: pwd, ls, ls -l, ls -la, cd, mkdir -p, touch, cp, mv, and rm. I know what each one does and when to use it.

For file viewing: cat for short files, less for long ones, head and tail for the first or last N lines. tail -f follows a live file, which is useful for watching logs in real time.

For permissions: I understand the rwx permission model and can read a permission string like -rwxr-xr-x. I can use chmod with symbolic notation (chmod +x) and numeric notation (chmod 755 or 644). I understand what sudo means and why it requires careful use.

For log analysis: grep, grep -i, grep -c, grep -oP with regex, sort, uniq -c, wc -l, and pipeline combinations of these.

For networking: ip a, ping, nslookup, curl -I, and ss -tuln.

All of this was practiced in a WSL (Windows Subsystem for Linux) environment as part of a structured study plan.

---

## Q3: How would you troubleshoot a permission denied error on Linux?

First I would run ls -l on the file to read the permission string. I would identify whether the issue is with the owner, group, or others column. Then I would run whoami and id to confirm which user I am running as and which groups I belong to. If I need to add execute permission to run a script, I would use chmod +x filename. If the file is owned by root and I need to access it legitimately, I would use sudo — but only if I have sudo access and a legitimate reason.

---

## Q4: What is the difference between cat, head, tail, and less?

cat prints the entire file at once. Use it for short files.
head shows the first N lines (default 10). Use it to quickly see what a file looks like.
tail shows the last N lines. Use tail -f to watch a log file update in real time.
less opens the file in a scrollable viewer. Use it for long files — press q to quit.

In a real troubleshooting scenario, I would use tail -f /var/log/auth.log to watch for new authentication events live.

---

## Q5: What does the grep pipeline do?

  grep 'Failed password' auth.log | grep -oP 'd+.d+.d+.d+' | sort | uniq -c | sort -rn

It extracts just the IP addresses from failed login lines, counts how many times each IP appears, and sorts by frequency with the highest count first. This is a core pattern for identifying brute-force attempts in auth logs — the attacker IP shows up with the most failures.

---

## Important Caveat

All of this practice was done in a local WSL lab environment. I am not claiming professional Linux administration experience. I am claiming that I can navigate the command line, explain what each command does, and apply these skills in a structured troubleshooting scenario.
