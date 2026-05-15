# Linux Log Filtering and Network Commands

Date: May 15, 2026
Environment: WSL2 (Ubuntu) on Windows
Time spent: 1 hour

NOTE: The fake_auth.log file in this folder contains invented SSH log entries for practice only. No real system data was used.

---

## Part 1 — Log Filtering with grep

### Extract Only Failed Login Lines

  $ grep 'Failed password' fake_auth.log
  May 15 02:14:01 server sshd[1201]: Failed password for invalid user admin from 45.33.32.156 port 51234 ssh2
  May 15 02:14:12 server sshd[1202]: Failed password for invalid user admin from 45.33.32.156 port 51235 ssh2
  May 15 14:32:01 server sshd[2201]: Failed password for user jdoe from 192.168.1.50 port 52341 ssh2
  (output continues...)

What grep does: Searches each line and prints only lines containing the search string. Case-sensitive by default.

### Case-Insensitive Search

  $ grep -i 'failed' fake_auth.log

The -i flag matches "Failed", "FAILED", "failed", etc.

### Count Matching Lines

  $ grep -c 'Failed password' fake_auth.log
  8

The -c flag returns a count instead of the matching lines.

### Count Total Lines

  $ wc -l fake_auth.log
  25 fake_auth.log

### Find Unique IPs and Count Per IP

  $ grep 'Failed password' fake_auth.log | grep -oP 'd+.d+.d+.d+' | sort | uniq -c | sort -rn
        6 45.33.32.156
        2 192.168.1.50

Breaking down the pipeline:
  1. grep 'Failed password' fake_auth.log  — filter to failure lines only
  2. grep -oP 'd+.d+.d+.d+'         — extract just the IP address (-o = print match only, -P = Perl regex)
  3. sort                                   — sort alphabetically so duplicates are adjacent
  4. uniq -c                                — collapse duplicates and add a count
  5. sort -rn                               — sort numerically in reverse (highest count first)

Result: IP 45.33.32.156 had 6 failed attempts. IP 192.168.1.50 had 2.

---

## Part 2 — Network Diagnostic Commands

### ip a — View Network Interfaces

  $ ip a
  1: lo: <LOOPBACK,UP> mtu 65536
      inet 127.0.0.1/8 scope host lo
  2: eth0: <BROADCAST,UP,LOWER_UP> mtu 1500
      inet 192.168.1.100/24 brd 192.168.1.255 scope global eth0

What to look for: Your main interface (eth0 or ens33 for wired, wlan0 for Wi-Fi) and its IPv4 address (the inet line). 169.254.x.x means DHCP has failed. 127.0.0.1 is loopback only.

### ping — Test Connectivity

  $ ping 8.8.8.8 -c 4
  PING 8.8.8.8 56(84) bytes of data.
  64 bytes from 8.8.8.8: icmp_seq=1 ttl=118 time=12.4 ms
  64 bytes from 8.8.8.8: icmp_seq=2 ttl=118 time=11.9 ms
  64 bytes from 8.8.8.8: icmp_seq=3 ttl=118 time=12.1 ms
  64 bytes from 8.8.8.8: icmp_seq=4 ttl=118 time=12.3 ms
  4 packets transmitted, 4 received, 0% packet loss

The -c 4 flag sends 4 packets and stops. Key diagnostic: if ping 8.8.8.8 works but ping google.com fails, you have a DNS problem, not a connectivity problem.

### nslookup — Test DNS Resolution

  $ nslookup google.com
  Server:  127.0.0.53
  Address: 127.0.0.53#53

  Non-authoritative answer:
  Name:    google.com
  Address: 142.250.80.46

Shows which DNS server answered and what IP was returned. If this fails while ping 8.8.8.8 works, the problem is DNS.

### curl -I — Check HTTP Response Headers

  $ curl -I https://example.com
  HTTP/2 200
  content-type: text/html; charset=UTF-8
  date: Fri, 15 May 2026 14:45:22 GMT
  server: ECS
  content-length: 1256

curl -I sends an HTTP HEAD request — fetches headers only, not the page body. Useful for checking if a web server is alive and whether HTTPS is enforced. HTTP codes: 200=OK, 301/302=redirect, 403=forbidden, 404=not found, 500=server error.

### ss -tuln — View Listening Ports

  $ ss -tuln
  Netid  State   Local Address:Port
  tcp    LISTEN  0.0.0.0:22
  tcp    LISTEN  127.0.0.1:631
  tcp    LISTEN  0.0.0.0:80

Flags: -t=TCP, -u=UDP, -l=listening only, -n=show numbers. Port 22=SSH, Port 80=HTTP, Port 631=CUPS printer. An unexpected port here is a flag for investigation.

---

## Key Takeaways

- grep + sort + uniq -c + sort -rn is the core pipeline for log frequency analysis.
- ping 8.8.8.8 works but ping google.com fails = DNS problem.
- nslookup shows which DNS server you are using and what IP it returned.
- ss -tuln shows all listening services on the machine.
- These commands together cover most first-response network troubleshooting scenarios.
