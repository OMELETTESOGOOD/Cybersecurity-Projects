# Stage 5: Log Monitoring & Attack Detection with Wazuh + Kali

**Project:** apache-web-01 — Linux + Apache Web Server
**Stage:** 5 of 5 (Final Stage)
**Status:** Complete

## What This Stage Was About

The first four stages built and locked down a web server (apache-web-01). But a hardened server that nobody is watching is only half a security setup — if someone attacks it, who finds out?

Stage 5 answers that question. The goal was to:

1. Stand up **Wazuh**, a free security monitoring tool, on its own dedicated server (a "manager")
2. Connect apache-web-01 to it as a monitored "agent," so its logs get sent over for analysis
3. Launch real (but safe, lab-only) attacks from a Kali Linux machine against apache-web-01
4. Confirm Wazuh actually noticed the attacks, and map each one to the industry-standard **MITRE ATT&CK** framework — a shared vocabulary security teams use to describe attacker behavior

This turns the project from "I can build and secure a server" into "I can build, secure, *and monitor* a server, and prove that monitoring works against real attack traffic."

## Environment Setup

Three virtual machines were networked together on a private, isolated network (192.168.208.0/24) so they could talk to each other without touching the internet:

| Machine | Role | IP Address |
|---|---|---|
| apache-web-01 | The web server being protected | 192.168.208.10 |
| wazuh-manager | The security monitoring server | 192.168.208.20 |
| Kali | The "attacker" machine, used for testing | 192.168.208.30 |

Getting all three to see each other reliably, and getting Wazuh itself installed, took real troubleshooting — full detail is in the "Challenges" section below, since that problem-solving is as much a part of this project as the end result.

## What Got Built

### 1. A Dedicated Monitoring Server

Rather than cramming Wazuh onto apache-web-01 itself (which would mean the server was watching itself — not great practice), a separate VM was built just to run Wazuh. Given the limited resources on the host laptop, the full Wazuh stack (which normally includes a dashboard and search engine) had to be scaled back to just the core **manager** service — the part that actually receives, stores, and analyzes logs. No pretty dashboard, but all the real detection logic is present and working.

### 2. Connecting apache-web-01 to Wazuh

A small piece of software called a **Wazuh agent** was installed on apache-web-01. Its job is to watch specific files and events on the server and forward anything interesting to the manager. Agent and manager authenticate to each other using a shared secret key — similar to a password, but longer and randomly generated.

### 3. Pointing Wazuh at the Right Logs

apache-web-01's website logs its traffic to two files — one for successful requests, one for errors. By default, Wazuh's agent expects those logs to live at Ubuntu's *default* file locations, but Stage 2 of this project deliberately used *custom* file names for better organization. The agent's configuration had to be manually updated to point at the correct files. Once that was fixed, real website traffic began flowing into Wazuh.

### 4. Simulated Attack Traffic

With monitoring confirmed working, **Nikto** — a well-known open-source web vulnerability scanner — was run from the Kali VM against apache-web-01. Nikto automatically probes a web server with thousands of requests designed to find outdated software, dangerous file paths, common exploit attempts, and misconfigurations. It's a standard first step a real attacker (or a legitimate security tester) would take against any web server.

## Results: What Wazuh Caught

Without writing a single custom detection rule, Wazuh's built-in rule set caught the Nikto scan and correctly categorized it into 14 different types of suspicious behavior, generating over 6,000 individual alerts. A summary of the most notable ones:

| What Was Detected | Alerts | Severity | What It Means (MITRE ATT&CK) |
|---|---|---|---|
| Repeated bad/malformed requests from one source | 472 | High | Active Scanning — a tool systematically probing the server |
| Common web attack patterns | 246 | Medium | Exploiting a Public-Facing Application |
| Cross-Site Scripting (XSS) attempts | 243 | Medium | Attempted script-injection attack |
| **Shellshock exploit attempts** | 59 | Medium | Attempted use of a well-known, serious 2014 vulnerability |
| Multiple XSS attempts from one source | 27 | High | Active Scanning (aggregated pattern) |
| SQL Injection attempts | 4 | Medium-High | Attempted database attack |
| **Shellshock attack — full detection** | 2 | **Critical** | Confirmed exploit attempt, highest alert level in the system |
| Forbidden file/directory access attempts | 2 | Medium | Reconnaissance — probing for hidden files |

The two "critical" Shellshock alerts stand out — that's the highest severity Wazuh can assign, and it correctly flagged an attempt to exploit a serious, real-world vulnerability, even though apache-web-01 was never actually vulnerable to it (the hardening from Stage 4 already closed that door). This is exactly the outcome a monitoring system is supposed to deliver: **it doesn't matter if the attack would have worked — the point is that it was seen and logged.**

## Challenges & Troubleshooting

This stage had more real problem-solving than any before it. Documenting it honestly, because working through it *is* the skill:

- **Hardware limits forced a scaled-down design.** The full Wazuh stack (manager + dashboard + search engine) repeatedly crashed the host machine — running out of disk space, freezing under CPU/memory pressure, and failing DNS lookups mid-install. After several failed attempts, the decision was made to install just the manager component. This isn't a shortcut — a manager-only setup is a legitimate, real-world deployment pattern for smaller environments, and it still does full log analysis and alerting.

- **A single mistyped character broke the entire connection.** The agent and manager communicate using a long, randomly generated key. When that key was typed in manually (character by character, since the lab VM didn't support copy-paste at the time), a lowercase "L" got typed as the digit "1" in several places — and one character got dropped entirely. The result: the manager kept rejecting the agent's messages as "wrong key," with no more specific error to go on. This took methodical, line-by-line comparison of the key on both machines to catch. The permanent fix was setting up SSH access from the Windows host so the key could be copied and pasted cleanly instead of retyped — a good reminder that the "boring" infrastructure fix (proper remote access) often prevents the harder debugging problem downstream.

- **Logs were "connected" but silent — because Wazuh was watching the wrong files.** After the key issue was resolved, the agent was online and traffic was flowing — but no website activity was showing up. The cause: Wazuh's default configuration expected Apache to log to Ubuntu's standard file names, but Stage 2 had intentionally set up custom log file names for the site. Wazuh wasn't broken; it was just watching empty files. Once the configuration was corrected to point at the actual log files, real traffic appeared immediately.

- **A recurring clock-sync issue.** Similar to a problem hit back in Stage 2, the wazuh-manager VM's internal clock kept drifting out of sync after being powered off and on. This didn't block any of the work, but it's worth resolving before doing any timeline-based analysis later, since mismatched timestamps between machines make correlating events confusing.

## Skills Demonstrated

- Deploying and configuring a SIEM-style monitoring tool (Wazuh) in a resource-constrained environment
- Agent-to-manager authentication and troubleshooting a real cryptographic key mismatch
- Diagnosing a "silent" log pipeline by tracing configuration back to its actual file paths
- Running and interpreting an automated vulnerability scan (Nikto) as an attacker would
- Reading raw security alerts and mapping real detected behavior to the MITRE ATT&CK framework
- End-to-end thinking: build → harden → monitor → validate, rather than stopping at "it's secure on paper"

---

# Project Wrap-Up: apache-web-01 (Stages 1–5)

This project took a single Ubuntu server from a blank virtual machine to a hardened, monitored, and attack-tested web server — the full lifecycle a real web asset goes through in a security-conscious organization.

| Stage | What Happened |
|---|---|
| **1. Build** | Created the base VM (Ubuntu Server), configured remote access |
| **2. Deploy** | Installed Apache, configured a custom website (vhost), verified it was reachable |
| **3. Baseline** | Learned the server's normal log behavior and practiced spotting suspicious patterns manually |
| **4. Harden** | Locked down file permissions, disabled risky features, added a firewall, enabled HTTPS |
| **5. Monitor & Validate** | Connected the server to a security monitoring system and proved it detects real attacks |

**Why this matters:** anyone can follow a tutorial to install Apache. What this project demonstrates is the full defensive mindset — building something, assuming it *will* be attacked, proving the defenses hold up under hardening review, and then proving that if something ever does get through, someone (or something) is watching. The troubleshooting throughout — clock drift, disk exhaustion, a single mistyped character breaking authentication, a "connected but silent" log pipeline — is arguably the most valuable part of the write-up, since it's the kind of real-world debugging that doesn't show up in a clean tutorial.

**Next steps for this server (optional, future work):** add the missing HTTP security headers Nikto flagged, and build a small Wazuh dashboard component if hardware allows, for a visual view of alerts rather than raw log files.