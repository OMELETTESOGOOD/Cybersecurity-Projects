# Project 2: SIEM Lab (Splunk) — Log Collection, Forwarding & Detection Engineering

**Author:** Marshel Mburu Macharia
**Status:** Complete
**Stack:** Splunk Enterprise, Splunk Universal Forwarder, Ubuntu Server, VirtualBox (host-only networking), Kali Linux (attack simulation)

## Overview

This project builds a small but functional Security Information and Event Management (SIEM) lab from scratch, using Splunk to collect, index, and analyze logs from a separate Apache web server VM (built in Project 1). The goal was to go beyond "installing Splunk" and actually demonstrate the full SIEM workflow a SOC analyst deals with day to day: getting logs from a source machine into a central system reliably, and then writing detections against that data that catch real attack behavior — not just theoretical ones.

The project was completed in three stages:

1. **Stage 1 — Splunk Enterprise Setup:** Building the SIEM server itself.
2. **Stage 2 — Universal Forwarder & Log Pipeline:** Getting logs from the Apache web server into Splunk reliably.
3. **Stage 3 — Detection Engineering:** Writing and validating three real detections against live attack traffic.

Midway through the project, the host machine this lab was built on failed and wiped every VM, including the fully working Stage 1–2 setup. The lab was rebuilt from a fresh VirtualBox install on a new machine, and that rebuild is what's documented below — including the mistakes and fixes, since those are as useful to a reviewer as the finished product.

---

## Stage 1: Splunk Enterprise Setup

**Goal:** Stand up a dedicated SIEM server VM and get Splunk Enterprise running on it.

**What was built:**
- A new Ubuntu Server VM (`splunk-siem-01`) sized deliberately larger than the original build (2 vCPU, ~5.4GB RAM, ~41GB disk) after the first build ran into resource contention — a direct lesson carried forward into the rebuild.
- Dual network adapters: NAT for internet access/updates, and a host-only adapter so the SIEM server could talk to the other lab VMs (Apache web server, Wazuh manager) on a private `192.168.208.0/24` segment.
- A static IP (`192.168.208.40`) configured via Netplan.
- Splunk Enterprise 10.4.2 installed via the official `.deb` package, and started under Splunk's own dedicated service account rather than root — a security best practice, and again a fix carried over from a mistake made in the original build.

**Problems hit and fixed:**
- A stray, typo'd Netplan config file (`00-installer-cnfig.yaml`, missing the "o") was still being read by the system alongside the correct file, silently breaking `netplan apply`. Found and deleted the bad file to resolve it.
- `openssh-server` wasn't installed by default during the unattended OS setup, so SSH access had to be installed and enabled manually, plus a port-forward rule added so the VM could be reached from the host machine.
- Once Splunk was running, confirmed the web interface, certificate generation, and all background processes (`splunkd`, etc.) were healthy via `splunk status` before moving on.

**Outcome:** A clean, correctly-provisioned SIEM server with Splunk Enterprise live and reachable, ready to receive forwarded logs.

---

## Stage 2: Universal Forwarder & Log Pipeline

**Goal:** Get real logs flowing from the Apache web server VM (Project 1) into Splunk, end to end, reliably and repeatably.

**What was built:**
- Splunk Universal Forwarder installed on the Apache VM (`apache-web-01`) to ship logs to the SIEM server on port 9997.
- `inputs.conf` configured to monitor three log sources: Apache access logs, Apache error logs, and Linux authentication logs (`/var/log/auth.log`).
- Two indexes created on the Splunk server to receive this data: `apache_web` and `linux_auth`.
- Correct file permissions granted so the forwarder's service account could actually read logs it doesn't own by default (adding it to the right Linux group).
- Time synchronization (`chrony`) installed and enabled on both VMs, since accurate timestamps are critical for any SIEM to correlate events correctly.

**Problems hit and fixed (this is the part that mirrors real SOC/sysadmin troubleshooting):**
- **Wrong forwarding target:** the forwarder was initially pointed at the old Wazuh manager's IP instead of the Splunk server's IP, causing repeated silent connection failures. Diagnosed and corrected.
- **Permission gap:** the forwarder's service account couldn't read Apache's logs until it was added to the correct Linux group.
- **Clock skew:** both VMs' clocks had drifted significantly (one VM was 6 days behind) after being suspended for a long period. This wasn't just a display issue — it blocked package installs entirely and would have caused Splunk to hide real events outside the default search time window. Fixed manually, then prevented long-term with `chrony`.
- **Silent data loss on rebuild:** after the host crash and rebuild, logs appeared to be flowing correctly (Apache logging fine, forwarder connected, no errors anywhere) but nothing showed up in search — because the destination indexes hadn't been recreated yet on the new Splunk server. Splunk was silently discarding events sent to a non-existent index. This was traced by working outward from the source of the data to the destination, step by step, rather than guessing.

**Outcome:** A fully validated forwarding pipeline — test HTTP requests generated on the Apache server showed up correctly in Splunk with the right host, source, sourcetype, and timestamp, confirming the pipeline works end to end rather than just "looking" configured.

---

## Stage 3: Detection Engineering

**Goal:** Use the log data now flowing into Splunk to detect real attacks, not just simulate them in theory — using an actual Kali Linux VM to generate live attack traffic against the Apache server.

**Three detections were built, tested against real attack traffic, and validated:**

| Detection | Attack Simulated | Logic | MITRE ATT&CK |
|---|---|---|---|
| SSH Brute Force | Repeated failed SSH logins from a single source | ≥5 failed login attempts within a short time window (8 failed logins in 24 seconds was the test case) | T1110.001 – Brute Force: Password Guessing |
| Web Scanner / Recon | Nikto vulnerability scan against the Apache server | High request volume with an abnormally high 404 (not-found) rate from one source (test case: 7,752 unique paths requested, 96% resulting in 404s in under 6 minutes) — threshold set at >20 paths AND >50% 404 rate | T1595.002 – Active Scanning: Vulnerability Scanning |
| Shellshock Exploit Attempt | Injection of the Shellshock (`() { :; };`) payload signature across multiple HTTP attack vectors | Raw-text pattern match for the Shellshock signature across the User-Agent, Referer, and Cookie header fields (all 4/4 test payloads across these vectors were caught) | T1190 – Exploit Public-Facing Application |

Each detection was written as a saved Splunk search/alert and validated against traffic generated live from a Kali VM on the lab network — not synthetic or sample data. A supporting write-up (`splunk-siem-01-stage3.md`) documents the full SPL (Splunk Search Processing Language) queries, result tables, and MITRE mappings for each.

**A minor operational issue was also caught during this stage:** a Splunk admin password was briefly pasted in plaintext into terminal commands, which left it sitting in shell history and in the captured `linux_auth` logs. This was flagged as a follow-up item — clear shell history and rotate the password — which is itself a small but realistic example of the kind of operational hygiene issue a SOC analyst needs to notice and act on, not just detections against external attackers.

---

## Skills Demonstrated

- SIEM deployment and administration (Splunk Enterprise + Universal Forwarder)
- Log source configuration and centralized log collection across multiple hosts
- Linux system administration: networking (Netplan, static IPs, host-only virtual networks), service accounts, file permissions, time synchronization
- Methodical troubleshooting: diagnosing silent failures (misconfigured forwarding target, missing indexes, permission errors, clock drift) by tracing data from source to destination rather than guessing
- Detection engineering: writing SPL queries with defensible thresholds, based on real attack traffic rather than assumptions
- Mapping detections to the MITRE ATT&CK framework
- Operational security awareness (catching and flagging a credential-hygiene mistake in real time)
- Resilience: fully rebuilding a lab environment from scratch after total data loss, while improving on the original build (better VM sizing, correct service account use from the start, avoided repeat mistakes)

## Notes for Reviewers

This project was built twice — once before a host machine failure, and once after, from a completely fresh environment. The rebuild is documented here because it reached full parity with the original working pipeline and because the process of debugging real issues (wrong IPs, permission errors, clock drift, missing indexes) is a closer representation of real SOC/IT work than a clean, error-free walkthrough would be.