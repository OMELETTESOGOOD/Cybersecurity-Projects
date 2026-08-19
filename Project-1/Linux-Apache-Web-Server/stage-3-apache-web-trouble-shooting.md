# Linux + Apache Web Server Project — Stage 3: Log-Based Troubleshooting & Baseline Anomaly Detection

**Project:** Linux + Apache Web Server
**Stage:** 3 of 5 — Log Analysis & Anomaly Detection
**Date:** August 2026
**Author:** Marshel Mburu Macharia

---

## Overview

Building on Stage 2's dedicated vhost (`apache-web-01`) with isolated access/error logs, this stage establishes a manual log-analysis baseline and validates detection methodology ahead of Stage 4's SIEM (Wazuh) integration. The goal was to answer a simple but foundational question for any SOC-style workflow: *what does normal traffic look like on this box, and how do I recognize when something deviates from it?*

Rather than jumping straight to tooling, this stage was deliberately done by hand — raw `awk`/`grep`/`sort`/`uniq` pipelines against the access and error logs — to build fluency with the underlying log structure before automating any of it.

---

## Objectives

- Establish a known-good traffic baseline for `apache-web-01` (source IPs, status codes, methods)
- Practice core log-analysis one-liners: frequency counts, raw review, and signature-based grep
- Understand Apache's `combined` log format field-by-field, including how field-splitting behavior changes with delimiter choice
- Distinguish benign anomalies (cache validation, forgotten manual traffic) from genuinely suspicious patterns
- Simulate lightweight scanner-like traffic to validate that detection methods actually catch deviations
- Identify the limitations of hand-written signature lists

---

## Build Process

### 1. Source IP Frequency Baseline

```bash
sudo awk '{print $1}' /var/log/apache2/apache-web-01-access.log | sort | uniq -c | sort -rn
```

`awk` splits each log line on whitespace and prints field 1 (source IP) for every request; piping through `sort | uniq -c | sort -rn` is a standard "count and rank" idiom — `sort` groups identical values together, `uniq -c` collapses and counts adjacent duplicates, and the second `sort -rn` orders the counts numerically, highest first.

Initial baseline showed only the two expected sources from Stage 2: `127.0.0.1` (local curl) and `10.0.2.2` (VirtualBox NAT gateway, representing the Windows host).

**Anomaly caught in practice:** after a single `curl 127.0.0.1` from inside the VM, the count for `10.0.2.2` unexpectedly increased as well. Rather than dismissing it, the discrepancy was traced using `tail`/`cat` on the raw log, which showed a genuine second browser request from the Windows host that had been forgotten. This confirmed the detection method works — an unexplained count change is exactly the kind of signal worth investigating, even when the eventual cause turns out to be benign.

### 2. Status Code Breakdown

```bash
sudo awk '{print $9}' /var/log/apache2/apache-web-01-access.log | sort | uniq -c | sort -rn
```

Field 9 in the `combined` log format is the HTTP status code. Results showed a mix of `200` (successful GET), `304` (Not Modified), and later `404` (Not Found) once deliberately generated.

- **`304` explained:** not an error — it reflects the browser sending a conditional request (`If-Modified-Since`/`If-None-Match`) after a prior load, with Apache confirming the cached copy is still current and returning an empty body instead of re-sending content. Logged as expected/benign traffic to filter out during anomaly review.
- **`404` validation:** deliberately triggered via `curl http://127.0.0.1/this-path-does-not-exist` to confirm the detection method correctly surfaces failed path lookups — the core signal used to spot directory/file enumeration.

### 3. Error Log Behavior

```bash
sudo cat /var/log/apache2/apache-web-01-error.log
```

Returned empty even after generating a `404`. This is expected Apache behavior at the default `LogLevel warn` — plain client-side 404s are treated as normal HTTP responses and are recorded only in `access.log`; `error.log` is reserved for server-level issues (permissions, config problems, application errors) unless `LogLevel` is raised (e.g. to `debug`).

### 4. Scanner/Enumeration Signature Grep

```bash
sudo grep -Ei "wp-admin|\.env|\.git|phpmyadmin|/etc/passwd|\.\./" /var/log/apache2/apache-web-01-access.log
```

`-E` enables extended regex (allowing `|` as OR without escaping), `-i` makes matching case-insensitive. The pattern targets common scanner/recon targets: WordPress admin paths, exposed `.env`/`.git` files, database admin panels, and path-traversal indicators.

Initial baseline: zero matches, as expected on an untouched lab instance.

**Simulated traffic generated to validate detection:**
```bash
for path in wp-admin .env .git phpmyadmin "../../../etc/passwd" nonexistent-admin-panel; do
  curl -s -o /dev/null -w "%{http_code} - $path\n" "http://127.0.0.1/$path"
  sleep 1
done
```
This does not invoke any scanning tool — it manually reproduces the *shape* of scanner behavior (a rapid sequence of requests to known-interesting paths) using plain `curl`, so the log contains real entries to test detection against. All 6 requests returned `404`, including the path-traversal attempt — Apache resolves requested paths relative to `DocumentRoot` and does not follow `../` outside it by default, so no file was exposed.

**Result:** 5 of 6 simulated paths matched the grep pattern. `nonexistent-admin-panel` correctly did not match, since it wasn't part of the hand-written signature list — demonstrating a real limitation of manual signature-based detection: it only catches what it's explicitly told to look for, which is part of the motivation for moving to a maintained rule set (Wazuh) in Stage 4.

Also noted: the `../../../etc/passwd` request appeared in the log as `GET /etc/passwd`, with the traversal sequence normalized away client-side before the request was sent — so this particular match was caught via the `/etc/passwd` pattern, not the `\.\./` pattern.

### 5. User-Agent Breakdown

```bash
sudo awk -F'"' '{print $6}' /var/log/apache2/apache-web-01-access.log | sort | uniq -c | sort -rn
```

Using `-F'"'` splits each line on the double-quote character instead of whitespace — necessary because the request line, referrer, and User-Agent fields in the `combined` format can contain spaces and are wrapped in quotes. Under this delimiter, field 6 lands on the User-Agent string.

Result: 9 `curl/8.18.0` (all curl-based testing) and 2 Chrome/Windows entries (browser-based testing) — no unexpected or blank User-Agents, which would otherwise be a red flag (real scanners often identify as `Nikto`, `sqlmap`, `python-requests`, or omit the UA entirely).

### 6. HTTP Method Breakdown

```bash
sudo awk '{print $6}' /var/log/apache2/apache-web-01-access.log | tr -d '"' | sort | uniq -c
```

Same field number (`$6`) as the User-Agent check above, but a completely different result — because this command uses the *default* whitespace delimiter rather than `-F'"'`. Under whitespace splitting, field 6 lands on the HTTP method (with a leading quote character attached, e.g. `"GET`), not the User-Agent. `tr -d '"'` strips the stray quote characters before counting. This comparison was a useful concrete lesson in why field numbers are meaningless without knowing the active delimiter.

Result: 100% `GET` across all 11 requests — expected for a static site with no forms or write endpoints; any `POST`/`PUT`/`DELETE`/`TRACE` here would be worth investigating.

---

## Validation Summary

| Check | Command pattern | Result |
|---|---|---|
| Source IP frequency | `awk '{print $1}'` | 2 legitimate sources; one count discrepancy traced to forgotten manual browser activity |
| Status codes | `awk '{print $9}'` | `200` (normal), `304` (cache validation, benign), `404` (confirmed detection works) |
| Raw log review | `cat` | Confirmed exact request lines behind aggregated counts |
| Error log | `cat`/`tail` | Empty — plain 404s don't log to error.log at default LogLevel |
| Scanner signatures | `grep -Ei "wp-admin\|\.env\|\.git\|phpmyadmin\|/etc/passwd\|\.\./"` | Clean baseline; 5/6 simulated paths matched after generating test traffic |
| User-Agent | `awk -F'"' '{print $6}'` | 9 curl, 2 Chrome — no unexpected/blank UAs |
| HTTP methods | `awk '{print $6}'` + `tr -d '"'` | 100% GET — no anomalous methods |

---

## Troubleshooting Log

| Issue | Root Cause | Resolution |
|---|---|---|
| `10.0.2.2` request count increased unexpectedly after a single local `curl` test | Forgotten manual browser activity from the Windows host, not a bug in the counting method | Traced via `cat`/`tail` on raw log, confirmed by timestamp and User-Agent |
| Expected `404` for a deliberately mistyped browser path never appeared in the log | Request never reached Apache — client-side DNS/hosts resolution failure against `apache-web-01.local` rather than a server-side error | Reproduced a genuine 404 via `curl` from inside the VM against `127.0.0.1`, bypassing hostname resolution as a variable |
| `error.log` empty after generating a real `404` | Apache's default `LogLevel warn` does not log plain client-side 404s to `error.log`; they are considered normal HTTP responses | Confirmed as expected behavior, not a fault; noted that `LogLevel debug` would be required to also capture 404s there |
| One simulated scanner path (`nonexistent-admin-panel`) did not match the grep signature list | Path wasn't included in the hand-written pattern | Documented as an inherent limitation of manual signature-based detection, motivating Stage 4 SIEM rule sets |

---

## Outcome

Stage 3 is complete. A clean baseline was established and validated for `apache-web-01`'s access and error logs across source IPs, status codes, scanner signatures, User-Agents, and HTTP methods. Simulated scanner-like traffic was generated and successfully detected using hand-written `grep` signatures, while also surfacing a concrete limitation of that approach. Field-splitting behavior in `awk` (delimiter-dependent field meaning) was worked through directly by comparing two commands that both referenced `$6` but returned entirely different data.

---

## Next Steps (Stage 4 and beyond)

1. Harden the web server: disable directory listing, suppress Apache/PHP version banners, restrict unnecessary modules
2. Configure UFW firewall rules scoped to required ports only
3. Add TLS (self-signed certificate for lab purposes)
4. Forward `apache-web-01-access.log` / `apache-web-01-error.log` into the existing Wazuh SIEM instance, replacing manual grep signatures with maintained detection rules
5. Generate attack traffic from the Kali Linux VM and validate detection coverage against MITRE ATT&CK