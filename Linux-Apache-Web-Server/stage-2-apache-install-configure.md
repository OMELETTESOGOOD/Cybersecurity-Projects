# Linux + Apache Web Server Project — Stage 2: Apache Install & Configuration

**Project:** Linux + Apache Web Server
**Stage:** 2 of 5 — Service Installation & Configuration
**Date:** August 2026
**Author:** Marshel Mburu Macharia

---

## Overview

Building on Stage 1's provisioned VM, this stage installs and configures Apache HTTP Server, establishes a working knowledge of `systemctl` for service management, and replaces the default Apache configuration with a dedicated virtual host — the pattern used for the remainder of this project's log integration and hardening stages.

---

## Objectives

- Patch the base OS and resolve any update blockers
- Install Apache HTTP Server (`apache2`)
- Build fluency with `systemctl` (status, start/stop, restart/reload, enable/disable)
- Validate the installation locally and from the host machine via NAT port forwarding
- Replace the default site with a dedicated virtual host, document root, and log paths
- Confirm the service persists across reboots

---

## Build Process

### 1. System Update & Clock Sync Issue

Running `apt update && apt upgrade` initially failed with repository "Release file is not valid yet" errors. Root cause: the VM's system clock had drifted roughly two days behind actual time (a common artifact of VM suspend/resume cycles), causing APT to distrust repository signing timestamps.

**Resolution:** Cycled NTP sync via `timedatectl`:
```bash
sudo timedatectl set-ntp off
sudo timedatectl set-ntp on
```
Confirmed via `timedatectl status` (`System clock synchronized: yes`). Update then proceeded normally.

A subsequent `apt upgrade` briefly queued behind `unattended-upgrades` holding the dpkg lock — resolved by waiting for the background process to complete rather than force-clearing the lock (which risks corrupting the package database).

The upgrade also flagged a pending kernel update (`7.0.0-28-generic` → `7.0.0-29-generic`). The VM was rebooted to apply it before proceeding, confirmed post-reboot via `uname -r`.

### 2. Apache Installation

```bash
sudo apt install apache2 -y
```

Installed Apache 2.4.66 along with standard dependencies (`apache2-bin`, `apache2-utils`, `apache2-data`, supporting `libapr`/`libaprutil` libraries). The installer automatically enabled Apache's default module set (`mpm_event`, `authz_core`, `alias`, `dir`, `mime`, `deflate`, `status`, and others) and registered `apache2.service` with systemd, linking it into `multi-user.target.wants` — meaning the service was enabled for boot persistence immediately on install.

### 3. systemctl Fundamentals

Practiced the core service-management commands against the live `apache2` unit:

```bash
sudo systemctl status apache2
sudo systemctl is-enabled apache2
sudo systemctl is-active apache2
sudo systemctl reload apache2
```

Key distinctions reinforced:
- **`reload` vs `restart`** — `reload` (via `apachectl graceful`) re-reads configuration without dropping active connections; `restart` fully stops and starts the process. Confirmed in practice via the `ExecReload=/usr/sbin/apachectl graceful` line in `systemctl status` output after a config change.
- **`enable` vs `start`** — orthogonal settings: one controls current running state, the other controls boot-time behavior. Relevant here since this NAT-based lab VM is routinely rebooted between sessions.
- **Reading `systemctl status` output** — unit file location and enabled state (`Loaded:`), current run state (`Active:`), tracked process tree (`CGroup:`), and live resource accounting (`Tasks:`, `Memory:`, `CPU:`).

Also resolved a benign `AH00558` warning (Apache unable to determine its own fully-qualified domain name) by setting a global `ServerName` directive via a new file under `conf-available/`, enabled with `a2enconf`.

### 4. Validating the Installation

Confirmed Apache was serving locally from inside the VM:
```bash
curl http://127.0.0.1
```
Returned the default "Apache2 Ubuntu Default Page."

To validate from the host machine, added a second VirtualBox NAT port-forwarding rule (host `8080` → guest `80`), following the same pattern established for SSH in Stage 1 (host `2222` → guest `22`). Confirmed via a Windows browser request to `http://127.0.0.1:8080`, rendering the same default page.

### 5. Custom Virtual Host

Rather than continuing to serve from the shared default document root, configured a dedicated vhost to mirror production practice and to establish clean, identifiable log paths ahead of Stage 4's SIEM integration.

**Document root:**
```bash
sudo mkdir -p /var/www/apache-web-01/public_html
sudo chown -R $USER:$USER /var/www/apache-web-01/public_html
sudo chmod -R 755 /var/www/apache-web-01
```

**Vhost configuration** (`/etc/apache2/sites-available/apache-web-01.conf`):
```apache
<VirtualHost *:80>
    ServerAdmin admin@apache-web-01.local
    ServerName apache-web-01.local
    DocumentRoot /var/www/apache-web-01/public_html

    ErrorLog ${APACHE_LOG_DIR}/apache-web-01-error.log
    CustomLog ${APACHE_LOG_DIR}/apache-web-01-access.log combined
</VirtualHost>
```

**Enabled the new site, disabled the Ubuntu default:**
```bash
sudo a2ensite apache-web-01.conf
sudo a2dissite 000-default.conf
sudo systemctl reload apache2
```

Validated with:
```bash
curl -H "Host: apache-web-01.local" http://127.0.0.1
```
Returned the custom page content, confirming the new vhost was live and the default site was no longer being served.

### 6. Boot Persistence

```bash
sudo systemctl enable apache2
sudo systemctl is-enabled apache2
```
Confirmed `enabled` — Apache will start automatically on VM boot without manual intervention.

---

## Validation: Log Output

With the default site disabled, `apache-web-01` is the sole vhost handling all traffic to this Apache instance regardless of entry point. Inspecting the custom access log after requests from both inside the VM and from the Windows host confirmed distinct source IPs per origin:

```
127.0.0.1 - - [06/Aug/2026:07:01:17 +0000] "GET / HTTP/1.1" 200 271 "-" "curl/8.18.0"
10.0.2.2 - - [06/Aug/2026:07:08:32 +0000] "GET / HTTP/1.1" 200 327 "-" "Mozilla/5.0 ..."
```

- `127.0.0.1` — request originated from `curl` inside the VM itself
- `10.0.2.2` — VirtualBox's standard NAT gateway address, representing the Windows host

This distinction — being able to attribute log entries to their true origin — is the same mechanic that will be used to identify Kali-originated attack traffic in Stage 5.

---

## Troubleshooting Log

| Issue | Root Cause | Resolution |
|---|---|---|
| `apt update` reported repo release files "not valid yet" | VM system clock had drifted ~2 days behind actual time | Cycled NTP sync via `timedatectl set-ntp off/on` |
| `apt upgrade` stalled waiting on dpkg lock | `unattended-upgrades` background service held the lock concurrently | Waited for the background process to release the lock rather than force-clearing it |
| Apache logged `AH00558` FQDN warning on start | No global `ServerName` directive set | Added `ServerName apache-web-01.local` via a new `conf-available` file, enabled with `a2enconf` |

---

## Outcome

Stage 2 is complete. Apache 2.4.66 is installed, enabled for boot persistence, and serving a dedicated virtual host (`apache-web-01`) from a custom document root with isolated access/error logs — reachable both locally within the VM and externally from the host machine via NAT port forwarding.

---

## Next Steps (Stage 3 and beyond)

1. Harden the web server: disable directory listing, suppress Apache/PHP version banners, restrict unnecessary modules
2. Configure UFW firewall rules scoped to required ports only
3. Add TLS (self-signed certificate for lab purposes)
4. Forward `apache-web-01-access.log` / `apache-web-01-error.log` into the existing Wazuh SIEM instance
5. Generate attack traffic from the Kali Linux VM and validate detection coverage against MITRE ATT&CK