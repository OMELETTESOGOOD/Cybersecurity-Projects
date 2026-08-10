# Linux + Apache Web Server Project — Stage 4: Hardening & Access Control

**Project:** Linux + Apache Web Server
**Stage:** 4 of 5 — Hardening
**Date:** August 2026
**Author:** Marshel Mburu Macharia

---

## Overview

Stage 3 established that log-based detection works, but it also made clear how wide open `apache-web-01` still was — full version disclosure, browsable directories, no firewall, no encryption, and file ownership that only worked by accident of permissive default bits. Stage 4 closes that gap before Stage 5 hands the box over to Kali for live attack simulation: there's little value in validating detection against a target that would fall to the first automated scanner regardless.

The goal was defense-in-depth across five independent layers — information disclosure, module surface, file permissions, network exposure, and transport encryption — verifying each change with a real request/response before moving to the next.

---

## Objectives

- Disable directory listing and symlink-following on the doc root
- Suppress Apache/OS version disclosure in HTTP responses and generated pages
- Remove unnecessary modules (`autoindex`, `status`) rather than relying solely on config-level restriction
- Correct file/directory ownership to `www-data` and remove world-readable access
- Scope UFW to only the ports actually in use, default-deny everything else
- Add TLS via a self-signed certificate and validate HTTPS end-to-end
- Document every failure encountered along the way, since several were as instructive as the fixes themselves

---

## Build Process

### 1. Directory Listing & Symlink Restriction

Added a `<Directory>` block to the existing `*:80` vhost (`/etc/apache2/sites-available/apache-web-01.conf`):

```apache
<Directory /var/www/apache-web-01/public_html>
    Options -Indexes -FollowSymLinks
    AllowOverride None
    Require all granted
</Directory>
```

`-Indexes` stops Apache from auto-generating a file listing when a directory has no `index.html`. `-FollowSymLinks` prevents content from being served through a symlink that resolves outside the doc root.

```bash
sudo apache2ctl configtest
sudo systemctl reload apache2
curl -sI http://127.0.0.1/testdir/
```

Validated: `403 Forbidden` on a directory with no index file.

### 2. Version Banner Suppression

Edited `/etc/apache2/conf-available/security.conf`:

```apache
ServerTokens Prod
ServerSignature Off
```

`ServerTokens Prod` trims the `Server:` response header to `Apache` only — no version, no OS. `ServerSignature Off` removes the version footer Apache appends to generated error/status pages.

```bash
curl -sI http://127.0.0.1/
```

Before: `Server: Apache/2.4.66 (Ubuntu)`. After: `Server: Apache`.

### 3. Module Reduction

Checked loaded modules first:

```bash
apache2ctl -M
```

`status_module` and `autoindex_module` were present; `info_module` was not loaded at all. Disabled the two that were:

```bash
sudo a2dismod autoindex status
sudo apache2ctl configtest
sudo systemctl restart apache2
```

`autoindex` triggered Ubuntu's "essential module" confirmation prompt (`Yes, do as I say!`) since it's assumed most sites want directory listing as a fallback — intentionally overridden here since the opposite is the goal.

Validated:

```bash
apachectl -M | grep -Ei "autoindex|status"    # no output
curl -sI http://127.0.0.1/server-status        # 404 Not Found
```

`404` (not `403`) confirms the module itself is gone, not just access-restricted.

### 4. Ownership & File Permissions

Baseline check before changing anything:

```bash
ls -la /var/www/apache-web-01/public_html
```

Revealed `public_html` owned by `marshel:marshel` and `index.html` owned by `root:root` — Apache (running as `www-data`) had never been the owner or group of anything it served. It had only worked because the "other" permission bits (`755`/`644`) happened to be world-readable.

```bash
sudo chown -R www-data:www-data /var/www/apache-web-01/public_html
sudo find /var/www/apache-web-01/public_html -type d -exec chmod 750 {} \;
sudo find /var/www/apache-web-01/public_html -type f -exec chmod 640 {} \;
```

Validated ownership/mode (`sudo` required, since `marshel` no longer has "other" access):

```bash
sudo ls -la /var/www/apache-web-01/public_html
```
```
drwxr-x--- 2 www-data www-data 4096 Aug  6 18:48 .
-rw-r----- 1 www-data www-data   44 Aug  6 06:49 index.html
```

Confirmed Apache unaffected:

```bash
curl -sI http://127.0.0.1/    # 200 OK
```

### 5. UFW Firewall

Checked active listeners before writing any rules:

```bash
sudo ss -tulpn
```

Confirmed only `sshd` (22), `apache2` (80), and loopback-only `systemd-resolve`/`chronyd` were listening — nothing unexpected to account for.

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw show added        # verified before enabling
sudo ufw enable
sudo ufw status verbose
```

Result: `Status: active`, default `deny (incoming) / allow (outgoing)`, only `22/tcp` and `80/tcp` open (IPv4 + IPv6). SSH session survived the enable — rule was confirmed present beforehand specifically to avoid a lockout.

### 6. TLS (Self-Signed Certificate)

```bash
sudo a2enmod ssl
sudo mkdir -p /etc/apache2/ssl
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/apache2/ssl/apache-web-01.key \
  -out /etc/apache2/ssl/apache-web-01.crt
```

CN set to `apache-web-01.local` to match `ServerName`. Added a second `<VirtualHost *:443>` block to the same vhost file, mirroring the `*:80` block's `<Directory>` restriction and adding:

```apache
SSLEngine on
SSLCertificateFile /etc/apache2/ssl/apache-web-01.crt
SSLCertificateKeyFile /etc/apache2/ssl/apache-web-01.key
```

```bash
sudo apache2ctl configtest
sudo systemctl restart apache2
sudo ufw allow 443/tcp
curl -kI https://127.0.0.1/
```

Validated: `200 OK` over HTTPS (`-k` used since the cert is self-signed and untrusted by default — expected for a lab environment, not a real CA-issued cert).

---

## Validation Summary

| Check | Command | Result |
|---|---|---|
| Directory listing | `curl -sI http://127.0.0.1/testdir/` | `403 Forbidden` |
| Version disclosure | `curl -sI http://127.0.0.1/` | `Server: Apache` (no version/OS) |
| Module removal | `apachectl -M \| grep -Ei "autoindex\|status"` | No output — modules fully unloaded |
| Module removal (live) | `curl -sI http://127.0.0.1/server-status` | `404 Not Found` |
| File ownership | `sudo ls -la public_html` | `www-data:www-data`, `750`/`640` |
| Site still serves post-permissions | `curl -sI http://127.0.0.1/` | `200 OK` |
| Firewall state | `sudo ufw status verbose` | `active`; only 22/80/443 allowed, default deny incoming |
| TLS | `curl -kI https://127.0.0.1/` | `200 OK` over HTTPS |

---

## Troubleshooting Log

| Issue | Root Cause | Resolution |
|---|---|---|
| `<Directory>` block with `-Indexes` appeared to have no effect on first test (`/testdir/` still returned `200` listing) | `configtest` and `reload` were pasted/run as one garbled line; reload likely never actually executed | Re-ran `configtest` and `reload` as two separate, isolated commands; `/testdir/` then correctly returned `403` |
| Version banner still showed full string after editing `security.conf` | `security.conf` contained duplicate, conflicting directives — both `ServerTokens Prod` and `ServerTokens OS` uncommented (same for `ServerSignature Off`/`On`); Apache applies whichever line comes last | Commented out the stale default lines (`ServerTokens OS`, `ServerSignature On`), leaving only the intended values active |
| `a2dismod autoindex` triggered an "essential module" warning | Ubuntu flags `autoindex` as essential by default since most sites rely on it for fallback directory browsing | Confirmed override was intentional (`Yes, do as I say!`) — the opposite behavior is the goal on this box |
| `*:443` vhost block missing from file after first edit attempt | `nano` edit wasn't actually saved/pasted before the file was re-catted | Redid the edit, appending the full `*:443` block after the existing `*:80` block, then verified with `cat` before proceeding |
| `ls -la` on `public_html` returned `Permission denied` after the `chown`/`chmod` step | Directory is now `www-data:www-data` at `750` — the logged-in user (`marshel`) is neither owner nor group member, so falls into the "other" bucket with zero access, same mechanism that previously let Apache serve files it didn't own | Expected behavior, not a fault; confirmed via `sudo ls -la` instead — noted as a permanent tradeoff of tightened permissions (admin now needs `sudo` for routine inspection) |

---

## Outcome

Stage 4 is complete. `apache-web-01` now discloses no version/OS information, blocks directory browsing and symlink traversal, runs with `autoindex`/`status` modules fully removed rather than just restricted, serves all content as its correct owner (`www-data`) with no world-readable access, exposes only the three ports actually in use behind a default-deny firewall, and serves over TLS with a validated self-signed certificate. Every fix was verified against a live request/response rather than assumed from config syntax alone, and five real misconfigurations/gotchas were caught and resolved in the process — a paste/reload race, a duplicate-directive conflict, an intentional "essential module" override, an unsaved edit, and the ownership-vs-"other"-bits permission mechanic — each documented above as findings in their own right.

---

## Next Steps (Stage 5)

1. Forward `apache-web-01-access.log` / `apache-web-01-error.log` into the existing Wazuh SIEM instance, replacing Stage 3's manual grep signatures with maintained detection rules
2. Generate attack traffic from the Kali Linux VM against the now-hardened target
3. Validate detection coverage against MITRE ATT&CK, and confirm the hardening from this stage measurably raises the bar (blocked/failed attempts) versus the Stage 3 baseline
4. Optionally add a second VirtualBox NAT forward (host port → guest 443) if HTTPS needs to be reachable from the Windows host directly, rather than VM-internal only