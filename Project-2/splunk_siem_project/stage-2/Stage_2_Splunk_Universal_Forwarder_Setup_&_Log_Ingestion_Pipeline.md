# SIEM Lab — Stage 1: Splunk Universal Forwarder Setup & Log Ingestion Pipeline

**Project:** SOC Portfolio — Project 2 of 5 (SIEM Lab)
**Date:** August 2026
**Host VMs:** `apache-web-01` (log source), `splunk-siem-01` (Splunk Enterprise indexer)

## Objective

Stand up a working Splunk ingestion pipeline: configure a Universal Forwarder on `apache-web-01` to ship Apache access/error logs and Linux authentication logs to a dedicated Splunk Enterprise indexer (`splunk-siem-01`), with purpose-built indexes and sourcetypes for each data source. This lays the foundation for the three planned detection use cases (Shellshock/exploit-attempt, SSH brute-force, web scanner/recon).

## Environment

- **splunk-siem-01**: Ubuntu Server 26.04 LTS, 2 vCPU, 4GB RAM, 20GB disk, static IP `192.168.208.40` on the host-only segment `192.168.208.0/24`. Splunk Enterprise 10.4.2 installed via `.deb`.
- **apache-web-01**: existing VM from the completed Apache/Wazuh project, static IP `192.168.208.10`, running Apache 2.4.66 with a hardened custom vhost (from Stage 4 of that project).
- Both VMs reachable over the VirtualBox host-only adapter used throughout the broader SOC portfolio lab environment.

## Work Completed

### 1. Splunk Enterprise receiver configuration (splunk-siem-01)

- Enabled receiving on the standard forwarder port:
  ```bash
  sudo /opt/splunk/bin/splunk enable listen 9997 -auth 'marshel:<password>'
  ```
- Staged (but left inactive) a UFW rule permitting the host-only subnet on 9997, deferring full firewall activation to a later hardening pass — consistent with the risk profile of an isolated, non-internet-facing lab segment.

### 2. Index and sourcetype design (splunk-siem-01)

Created two dedicated indexes rather than dumping everything into `main`, to keep web and auth data separately retained and easy to scope in searches:

| Index | Purpose | Sourcetypes |
|---|---|---|
| `apache_web` | Apache access/error logs | `apache:access`, `apache:error` (custom, `props.conf`) |
| `linux_auth` | OS authentication/audit logs | `linux_secure` (Splunk built-in) |

The Apache sourcetypes required custom `TIME_FORMAT`/`TIME_PREFIX` stanzas in `props.conf` to correctly extract event timestamps from the vhost's combined log format. `linux_secure` needed no custom configuration — Splunk ships pre-built parsing rules for standard `auth.log`/`secure` formats.

### 3. Universal Forwarder installation (apache-web-01)

- Installed Splunk Universal Forwarder 10.4.2 (`.deb`, matched to the indexer's version).
- Configured `outputs.conf` (via `add forward-server`) to target `splunk-siem-01:9997`.
- Configured `inputs.conf` with three monitor stanzas:
  - `/var/log/apache2/apache-web-01-access.log` → `apache_web` / `apache:access`
  - `/var/log/apache2/apache-web-01-error.log` → `apache_web` / `apache:error`
  - `/var/log/auth.log` → `linux_auth` / `linux_secure`

## Issues Encountered & Resolutions

Several real-world issues surfaced during setup — documenting them here since the troubleshooting itself is representative SOC/sysadmin work:

1. **Admin username mismatch.** Initial CLI auth attempts against splunk-siem-01 assumed the default `admin` username; the account was actually seeded as `marshel` during first-run setup. Resolved by testing the correct username directly.

2. **Forwarder admin account not created.** `splunk start -seed-passwd` silently failed to create the admin account because the supplied password didn't meet Splunk's minimum complexity requirement — the CLI gave no hard error, just a buried warning. Resolved by seeding the account explicitly via a `user-seed.conf` file instead.

3. **Wrong forward-server target IP.** The forwarder was initially pointed at `192.168.208.20` (the IP of an unrelated VM — the Wazuh manager from a prior project) instead of splunk-siem-01's actual `192.168.208.40`. This produced a clean, diagnosable failure signature: `nc -zv` confirmed port 9997 reachable at the *correct* IP, while splunkd.log showed repeated `Connection ... failed` against the *wrong* one — isolating the fault to configuration, not networking. Resolved by removing and re-adding the correct `forward-server`.

4. **Log read permissions.** Both `apache-web-01-access.log`/`-error.log` (owned `root:adm`) and `auth.log` (owned `syslog:adm`) are `640` — a deliberate result of the Stage 4 hardening pass on apache-web-01. The forwarder runs as a dedicated `splunkfwd` user, which by default is not a member of `adm` and therefore could not read any of them. Resolved with `usermod -aG adm splunkfwd` plus a forwarder restart to pick up the new group membership.

5. **Significant clock skew on both VMs.** apache-web-01's clock had drifted 6 days behind actual time; splunk-siem-01's was about 1 day behind — most likely from extended VM suspension between sessions. This caused two distinct problems: `apt` refused to trust Ubuntu's repository release files ("not valid yet") on both VMs, and — more subtly — ingested Splunk events initially appeared to be missing entirely, because they were timestamped days outside the default "Last 24 hours" search window. Root cause was confirmed by comparing `splunkd.log` timestamps against the actual date. Resolved by manually forcing the clock forward (`date -s`) to unblock `apt`, then installing and enabling `chrony` on both VMs for ongoing NTP sync (Ubuntu Server 26.04 does not ship `systemd-timesyncd` on this image, so the more usual `timedatectl`-based fix wasn't available).

6. **Splunk restarts require `--run-as-root`.** Both Splunk Enterprise and the Universal Forwarder are being run as the root user in this lab (a known simplification, not production practice) — `splunk restart` refuses to proceed without the explicit `--run-as-root` flag under those conditions.

## Validation

- Generated test HTTP traffic via `curl` against apache-web-01; confirmed corresponding events appeared in splunk-siem-01's `apache_web` index with correct `sourcetype=apache:access`, correct `host`/`source` fields, and correctly parsed timestamps.

  [Screenshot: `index=apache_web sourcetype=apache:access` search results in Splunk web UI]

- Attempted a deliberate failed SSH login against apache-web-01; confirmed corresponding `sshd` failure events appeared in the `linux_auth` index under `sourcetype=linux_secure`.

  [Screenshot: `index=linux_auth sourcetype=linux_secure` search results in Splunk web UI]

- Confirmed via `splunk list forward-server` that `192.168.208.40:9997` shows under **Active forwards** (not merely configured).

  [Screenshot: `splunk list forward-server` CLI output showing Active forwards]

  [Screenshot: `indexes.conf` / `inputs.conf` contents, e.g. via `cat`, showing the final working configuration]

## Outcome

End-to-end log ingestion pipeline is fully operational: apache-web-01 → Universal Forwarder → splunk-siem-01, covering both web server activity and OS-level authentication activity, each correctly indexed, sourcetyped, and time-accurate.

## Next Steps

Build out the three planned detection use cases against this pipeline:
1. Shellshock / exploit-attempt detection (Apache access/error logs)
2. SSH brute-force detection (`linux_auth` — note: raw event volume includes routine `sudo`/PAM session activity, not just login attempts, so detection logic needs to filter specifically on `sshd` authentication-failure patterns)
3. Web scanner/recon detection (Apache access logs)