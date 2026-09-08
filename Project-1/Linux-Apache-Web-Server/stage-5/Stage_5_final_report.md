# Stage 5: Wazuh SIEM Integration & Attack Simulation

**Project:** apache-web-01 — Linux + Apache Web Server (SOC Portfolio, Project 1 of 5)
**Stage:** 5 of 5 — Log Integration, Attack Simulation, Detection Validation
**Environment:** VirtualBox lab — apache-web-01 (Ubuntu Server 26.04), wazuh-manager (Ubuntu Server 26.04), Kali Linux — networked on an isolated host-only segment (192.168.208.0/24)

---

## 1. Objective

With the web server built (Stage 2) and hardened (Stage 4), Stage 5 closes the loop from a blue-team perspective: get Apache's logs into a SIEM, generate real attack traffic against the server, and confirm that traffic is detected, alerted on, and mappable to adversary tradecraft (MITRE ATT&CK). This stage also documents a full infrastructure rebuild after a host failure mid-project — a good-faith account of what breaks in a home lab and how to recover it.

---

## 2. Architecture

Three VMs on a dedicated VirtualBox host-only network (`192.168.208.0/24`, DHCP disabled, static IPs):

| Host | Role | IP | Specs |
|---|---|---|---|
| apache-web-01 | Web server under test | 192.168.208.10 | 2 vCPU, 2.5GB RAM, Ubuntu Server 26.04 |
| wazuh-manager | SIEM / log analysis | 192.168.208.20 | 2 vCPU, 4GB RAM, Ubuntu Server 26.04 |
| Kali | Attacker VM | 192.168.208.30 | Re-imported into VirtualBox (originally VMware, switched to avoid cross-hypervisor networking issues) |

Wazuh was deployed **manager-only** (no indexer/dashboard stack) — a deliberate scoping decision to fit the host's 2-core/16GB hardware after the all-in-one installer repeatedly hit disk exhaustion, LVM under-allocation, and CPU/RAM contention during the original build attempt.

---

## 3. Mid-Project Rebuild

Partway through this project the host machine broke down, wiping all four VMs and their configuration. Stages 1–4 were re-completed from scratch on a replacement machine with identical specs (same hardware ceiling, same constraints). Stage 5 — Wazuh + Kali — was rebuilt as follows:

- **Networking rebuilt first.** The host-only network itself didn't survive the rebuild; a new VirtualBox host-only adapter was created (`192.168.208.1/24`, DHCP disabled) since the default adapter used a different subnet. Static IPs were reassigned to all three VMs (netplan on the Ubuntu boxes, `nmcli` on Kali since it uses NetworkManager, not netplan). Connectivity was confirmed in both directions before proceeding.
- **wazuh-manager reinstalled** manager-only via `apt` (skipping the all-in-one installer, learning from the original build's resource contention) and confirmed active with all core daemons running (`authd`, `remoted`, `analysisd`, `logcollector`, `syscheckd`, `modulesd`).
- **Agent enrollment improved.** The original build required manually copying `client.keys` between VMs, which introduced a transcription error (digits substituted for lowercase-`l`/capital-`I` in a manually typed key). The rebuild used the `WAZUH_MANAGER` environment variable at install time instead, auto-enrolling the agent and avoiding the manual key-copy step entirely. Confirmed via `ossec.log`: `Connected to the server ([192.168.208.20]:1514/tcp)`.
- **Log path correction.** Wazuh's default `ossec.conf` on the agent pointed at Ubuntu's stock Apache log paths, not the custom vhost log filenames from Stage 2 (`apache-web-01-access.log` / `apache-web-01-error.log`). Corrected via `sed` and confirmed `logcollector` was reading the right files.

---

## 4. Attack Simulation

With the pipeline confirmed live, a Nikto web vulnerability scan was launched from Kali against apache-web-01, both over HTTPS (`nikto -h https://192.168.208.10 -ssl`) and HTTP (`nikto -h http://192.168.208.10`), to generate a realistic volume and variety of malicious/probing traffic.

### 4.1 Initial HTTPS pass

The TLS-wrapped scan confirmed the certificate and vhost from Stage 4 (`CN=apache-web-01`, TLS_AES_256_GCM_SHA384) and flagged expected informational findings — missing security response headers (CSP, HSTS, X-Content-Type-Options, etc.) and the hostname/CN mismatch from testing by IP rather than hostname. No exploitable findings, consistent with the hardening already applied in Stage 4.

### 4.2 HTTP pass — full detection validation

The unencrypted pass generated substantially more signal and was where Wazuh's default ruleset was properly exercised. Alerts observed, by rule:

| Rule ID | Level | Description | Example trigger |
|---|---|---|---|
| 31101 | 5 | Web server 400 error code | Bulk of Nikto's directory/file brute-force probing (thousands of hits) |
| 31104 | 6 | Common web attack | Path traversal (`../../../etc/hosts`, `\Windows\win.ini`), PHP object injection payloads, SSRF probes against cloud metadata endpoints |
| 31103 | 7 | SQL injection attempt | UNION-based SQLi payload targeting a Joomla content-history endpoint |
| 31105 | 6 | XSS (Cross Site Scripting) attempt | Reflected XSS payload against an actuator/jolokia endpoint |
| 31516 | 6 | Suspicious URL access | Probes for SSH private keys (`.ssh/id_rsa`, `id_dsa`, `id_ed25519`) and backup config files (`.wp-config.php.swp`) |
| 31151 | 10 | Multiple web server 400 error codes from same source IP | Wazuh's correlation engine aggregating Nikto's rapid-fire 400s into a higher-severity composite alert |
| 31153 | 10 | Multiple common web attacks from same source IP | Same correlation pattern, aggregating repeated attack-class hits |

No Shellshock-specific detection occurred on this run (the standout finding from the original, pre-rebuild Stage 5 pass) — Nikto's plugin ordering and exact path selection varies between runs, so this isn't guaranteed every time. The absence doesn't indicate a detection gap; the correlation and individual attack-signature rules above demonstrate the same underlying capability.

Notably, **all of this fired against Wazuh's out-of-the-box ruleset — no custom rules were required.** This mirrors the original Stage 5 finding: default OSSEC/Wazuh web rules provide meaningful coverage against common scanner and exploitation traffic without additional tuning.

---

## 5. MITRE ATT&CK Mapping

| Technique | ID | Detection Evidence |
|---|---|---|
| Active Scanning: Vulnerability Scanning | T1595.002 | Rule 31151/31153 correlation alerts on aggregated scan traffic |
| Exploit Public-Facing Application | T1190 | Rule 31103 (SQLi), 31104 (traversal / injection attempts) |
| File and Directory Discovery | T1083 | Rule 31104 path traversal payloads (`/etc/hosts`, `win.ini`) |
| Command and Scripting Interpreter | T1059 | `/shell?cat /etc/hosts` and similar command-injection probe paths |
| Unsecured Credentials: Private Keys | T1552.004 | Rule 31516 alerts on `.ssh/id_rsa` and related key-file probes |
| Cloud Instance Metadata API (SSRF) | T1552.005 | Probes against `169.254.169.254` and equivalent cloud metadata endpoints |

---

## 6. Stage 5 — Complete

All planned objectives for this stage were met:

- Wazuh manager deployed and stable (manager-only, appropriately scoped for lab hardware)
- Agent enrolled and correctly ingesting the custom vhost logs from Stage 2
- Nikto-generated attack traffic detected across multiple rule categories using Wazuh's default ruleset
- Findings mapped to MITRE ATT&CK for blue-team documentation purposes
- Full mid-project infrastructure rebuild documented, including root causes and fixes for each failure encountered (host-only network loss, agent key mismatch, log path misconfiguration)

**Project apache-web-01 is now COMPLETE — all 5 stages finished.**

---

## Stages Summary

1. **VM Build** — VirtualBox VM, Ubuntu Server 26.04, SSH access via NAT port forwarding
2. **Apache Install & Vhost Config** — Custom vhost, dedicated logs, validated locally and remotely
3. **Log-Based Troubleshooting** — Baselined access/error logs, validated detection via `awk`/`grep`, simulated scanner traffic
4. **Hardening** — Disabled directory listing/symlinks/version banners, corrected file ownership/permissions, scoped UFW firewall rules, self-signed TLS
5. **Wazuh Integration & Attack Simulation** — SIEM log ingestion, Nikto-driven attack simulation, MITRE ATT&CK mapping (this document)