# Project: Building, Securing, and Monitoring a Web Server (apache-web-01)

**Type:** Home lab / mini Security Operations Center (SOC) project
**Status:** Complete — 5 stages (rebuilt once from scratch after a hardware failure)

## The Big Picture

The goal of this project was simple to state but takes real work to actually do: **build a website from nothing, lock it down the way a security professional would, and then prove that if someone attacks it, it gets noticed.**

Most beginner projects stop after step one — "I installed a web server." This project went all the way through the full lifecycle a real company's server goes through: build it, deploy it, understand its normal behavior, harden it against attackers, connect it to a monitoring system, and then actually attack it (safely, in a private lab) to prove the whole thing works.

Everything was built from scratch using free tools inside VirtualBox on a personal laptop — no cloud costs, no shortcuts. Partway through, the laptop itself failed and wiped every VM in the lab. The whole thing was rebuilt from the ground up on a replacement machine — which turned out to be one of the more valuable parts of the project, described below.

## The Environment

Three separate virtual machines were built and connected on their own private network, isolated from the internet:

| Machine | Job |
|---|---|
| **apache-web-01** | The website/server being protected |
| **wazuh-manager** | The security monitoring system watching apache-web-01 |
| **Kali** | A separate "attacker" machine, used to safely test the defenses |

This mirrors how a real small security setup looks: a protected system, a tool watching it, and a way to test that the watching actually works.

## Stage 1: Building the Server

The project started with a completely blank virtual machine — no operating system, nothing installed. Ubuntu Server was installed from scratch, and remote access (SSH) was set up so the server could be managed without needing a screen and keyboard plugged directly into it — the same way real servers are managed in the real world.

**Outcome:** a working, reachable Linux server, ready to have a website put on it.

## Stage 2: Deploying the Website

Apache, one of the world's most widely-used web server programs, was installed and configured to serve a real (simple) website. Rather than using default settings, a custom configuration was set up — its own dedicated folder for website files and its own dedicated log files, which matters more later. The site was tested and confirmed reachable both from within the lab and from the everyday laptop being used to do the work.

A small but realistic hiccup came up here too: the server's internal clock had drifted out of sync, which broke software updates until it was manually re-synced — a small taste of the kind of "boring" issue that causes real headaches in the field.

**Outcome:** a live, working website, running on infrastructure fully built and configured by hand.

## Stage 3: Learning What "Normal" Looks Like

Before trying to defend anything, it's important to know what everyday, harmless traffic looks like in the logs. This stage involved studying the server's normal access and error logs, and practicing spotting patterns using basic command-line search tools — then simulating some scanner-style traffic to see how it appeared in the logs by comparison.

**Outcome:** a baseline understanding of the server's normal behavior, and hands-on practice reading raw log data — a foundational skill for any security-monitoring role.

## Stage 4: Hardening the Server

This stage was about closing doors an attacker might otherwise walk through:

- Turned off directory listing, so nobody could browse the site's file structure directly
- Hid the server's software version from being advertised to visitors (attackers often use this to pick known exploits)
- Fixed file permissions so only the necessary accounts could read or write website files
- Set up a firewall, only allowing the specific types of traffic the server actually needs (web traffic and remote management), blocking everything else by default
- Added HTTPS (encrypted traffic) support

**Outcome:** the same website from Stage 2, now meaningfully harder to attack — following real hardening practices, not just "it works."

## Mid-Project: A Total Rebuild

Partway through Stage 5, the laptop running the entire lab failed and wiped all four virtual machines — the web server, the monitoring system, the attacker box, and every bit of configuration on them. Nothing was salvageable except the documentation already written for Stages 1 through 4.

Rather than treating this as a setback to gloss over, it was treated as what it actually was: the same situation any real infrastructure team eventually faces, whether from hardware failure, a bad update, or a botched migration. The entire lab — all three VMs, all networking, all monitoring — was rebuilt from scratch on a replacement machine, following the same documented process. Every stage from 1 through 4 was redone and re-validated against the original build to confirm it matched. Stage 5 (monitoring and attack simulation) was rebuilt as well, and this second attempt is what's described below — including a couple of fixes that came directly from lessons learned the first time around (see "Real Problems, Real Debugging").

**Outcome:** a fully working lab, rebuilt from nothing a second time, with the rebuild itself becoming a demonstration of resilience and process discipline rather than just a story about bad luck.

## Stage 5: Monitoring and Proving It Works

This was the most involved stage, and the one that ties the whole project together.

**Setting up monitoring:** A separate server was built specifically to run **Wazuh**, a free, widely-used security monitoring tool. apache-web-01 was connected to it as a "watched" system, sending its activity over for analysis. This is a standard real-world pattern — the system doing the watching should not be the same system being watched.

**A real hardware constraint, handled properly:** The laptop running all of this doesn't have unlimited resources. The full version of Wazuh (which includes a visual dashboard) repeatedly crashed the lab during setup. Rather than giving up or faking it, the deployment was scaled back to just the core monitoring engine — no dashboard, but all the real detection and alerting logic fully working. This is a legitimate, real deployment pattern for smaller environments, and recognizing *when* to make that trade-off is itself a practical skill. This decision carried over unchanged into the rebuild.

**Attacking the server on purpose:** Once monitoring was confirmed working, the Kali machine was used to run **Nikto**, a well-known tool that automatically scans a website for outdated software, hidden files, and common attack patterns — exactly the kind of first move a real attacker (or a professional penetration tester) would make. Both an encrypted (HTTPS) and unencrypted (HTTP) pass were run against the rebuilt server.

**The result:** Wazuh caught it. Without any custom configuration, its built-in detection rules identified and categorized the scan across multiple distinct types of suspicious activity, including:

- Repeated 400-error probing consistent with automated directory/file brute-forcing
- Path traversal attempts (probing for files like `/etc/hosts` and Windows system files)
- A SQL injection attempt against a simulated vulnerable endpoint
- A cross-site scripting (XSS) attempt
- Probes for exposed SSH private keys and backup configuration files
- Wazuh's own correlation rules, which recognized the *pattern* of repeated attacks from a single source and escalated them into higher-severity combined alerts on top of the individual detections

This run didn't reproduce the specific Shellshock exploit-attempt detection seen on the very first (pre-rebuild) run — automated scans don't always hit the exact same paths in the exact same order, so some variation between runs is expected and doesn't indicate anything was missed. What matters is the consistent result across both runs: **the monitoring system saw the attack happen, using nothing but its default, out-of-the-box detection rules.**

Each type of detected activity was also mapped to **MITRE ATT&CK**, a shared framework security professionals use worldwide to describe and categorize attacker behavior — turning raw alert logs into the kind of structured findings a real security team would document.

**Outcome:** a server that isn't just built and hardened, but actively monitored, with real proof — twice now, on two separate builds — that the monitoring works against genuine attack traffic.

## Real Problems, Real Debugging

The parts of this project that took the most effort weren't the "follow the instructions" parts — they were the moments things quietly broke and had to be tracked down:

- **The entire lab was lost and rebuilt from scratch**, described above — the single biggest "real problem" of the whole project, and the clearest demonstration that the process was documented well enough to actually repeat it under pressure.
- **A single mistyped character broke security authentication (first build).** Agent and monitoring server talk to each other using a long, random secret key. Typing it in by hand (character by character, without copy-paste access at the time) caused a lowercase "L" to become the digit "1" in a few places, and dropped one character entirely. The result was a vague "wrong key" error with no obvious cause — solved by comparing the key on both machines line by line, and worked around at the time by setting up proper remote access so the key could be copied cleanly instead of retyped.
- **That same class of problem was engineered out entirely on the rebuild.** Rather than repeating the manual key-copy process a second time, the agent was enrolled using a single environment variable at install time, which handles the key exchange automatically. No manual key entry, no possibility of a repeat typo — a direct example of a first-time workaround becoming a permanent fix once the root cause was understood.
- **The monitoring system was "connected" but silent (both builds).** Even after the key issue was fixed, no real website activity was showing up in the monitoring alerts. The cause: the monitoring tool was, by default, looking for log files at Ubuntu's *standard* file names — but Stage 2 had deliberately used custom file names. Nothing was broken; it was just looking in the wrong place. Fixing the configuration to point at the real files resolved it immediately — and this exact same fix had to be reapplied during the rebuild, since a fresh install reverts to the same defaults.
- **The private lab network itself had to be recreated from nothing during the rebuild** — it wasn't just the VMs that were gone, but the virtual network connecting them. This had to be diagnosed and rebuilt before any of the other Stage 5 work could even begin again.
- **Hardware limits forced real engineering decisions**, not shortcuts — the manager-only Wazuh deployment was a direct, sensible response to repeated crashes, not a corner cut.
- **A recurring clock-sync issue** on more than one VM, a good reminder that timestamps matter a lot in security work, since mismatched clocks make it hard to line up events across different machines.

## Skills Demonstrated

- Building and configuring Linux servers and web services from scratch
- Real-world server hardening (firewalls, permissions, encryption, minimizing exposed information)
- Reading and interpreting raw system and web server logs
- Deploying and troubleshooting a security monitoring (SIEM-style) tool under real hardware constraints
- Diagnosing authentication and configuration issues methodically, rather than guessing
- Rebuilding a full multi-machine lab environment from documentation after a total infrastructure loss
- Recognizing a recurring problem and engineering a permanent fix rather than repeating the same workaround
- Running and interpreting the results of an automated vulnerability scan
- Mapping real detected activity to the MITRE ATT&CK framework, the same language used in professional security teams
- End-to-end thinking — treating "it's secure" as a claim that has to be tested and proven, not just assumed

## Why This Project Matters

Anyone can follow a tutorial to install a web server. This project demonstrates the full defensive mindset that security roles actually require: build something real, harden it properly, connect it to monitoring, and then go prove — with a real attack — that the whole system actually works end-to-end. Losing the entire lab partway through and rebuilding it from scratch wasn't part of the plan, but it ended up being the most realistic part of the project: real infrastructure fails, and what separates a hobbyist from someone ready for the job is whether the work can survive that and come back working. The troubleshooting throughout wasn't a distraction from the project; it *is* the project, and it's honestly the most valuable part to show off.ss