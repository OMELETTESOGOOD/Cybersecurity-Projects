# Apache Web Server Project — Stage 5: Wazuh Log Integration + Attack Simulation (In Progress)

## Objective

Extend the `apache-web-01` project with a dedicated Wazuh manager for log
monitoring and detection, ultimately validated by a Kali-based attack
simulation mapped to MITRE ATT&CK. This stage builds directly on the hardened
Apache server from [Stage 4](./apache-web-01-stage4.md).

## Environment

Three VMs networked together on a dedicated VirtualBox Host-only Adapter
(`192.168.208.0/24`, DHCP disabled, static IPs):

| VM | Role | IP | Adapter |
|---|---|---|---|
| apache-web-01 | Target web server | 192.168.208.10 | enp0s8 |
| wazuh-manager | Log monitoring / SIEM | 192.168.208.20 | enp0s8 |
| kali | Attack platform | 192.168.208.30 | eth1 |

Host: Windows laptop, Intel Core i5-7300U (2 physical cores / 4 logical
threads), 16GB RAM — a genuinely resource-constrained lab environment that
shaped several decisions below.

## Part 1 — Networking

- Built the `wazuh-manager` VM (Ubuntu Server, initially 4GB RAM / 2 vCPU,
  40GB dynamic disk) alongside the existing `apache-web-01` and `kali` VMs.
- Re-imported Kali as a native VirtualBox VM (previously VMware) to avoid
  cross-hypervisor networking issues, and confirmed it reachable on the
  host-only segment.
- Enabled the host-only Adapter 2 on wazuh-manager and configured a static
  IP via netplan:

  ```yaml
  network:
    ethernets:
      enp0s3:
        dhcp4: true
        dhcp6: true
        match:
          macaddress: 08:00:27:72:cf:12
        set-name: enp0s3
      enp0s8:
        dhcp4: false
        addresses:
          - 192.168.208.20/24
    version: 2
  ```

- Verified full triangle connectivity (wazuh-manager ↔ apache-web-01,
  wazuh-manager ↔ kali) with `ping`. First attempt at reaching
  apache-web-01 failed with "Destination Host Unreachable" — root cause was
  simply that the apache-web-01 VM was powered off, not a network fault.

## Part 2 — Wazuh Installation: Troubleshooting the All-in-One Install

Attempted the official all-in-one installer (`wazuh-install.sh -a`, indexer +
manager + dashboard on a single node) multiple times. Each attempt surfaced a
different real-world infrastructure constraint:

1. **Hardware minimum warning** — script flagged the VM as under Wazuh's
   recommended 4GB RAM minimum; proceeded past the check with `-i` (ignore
   minimum requirements).
2. **Disk exhaustion mid-install** — `dpkg-deb: error: paste subprocess was
   killed by signal (13: EPIPE): No space left on device` during dashboard
   package extraction. The install's automatic rollback partially failed,
   leaving the `wazuh-manager` package in a broken, unremovable state
   (`dpkg` errors, exit status 127) with its ports (1515, 55000) still held
   by orphaned processes.
3. **Manual recovery of a broken package** — resolved by:
   - Killing orphaned `ossec`/`wazuh` processes directly (`pkill -9`)
   - Neutralizing the package's broken `prerm`/`postrm` maintainer scripts
     (both referenced `/var/ossec` paths that no longer existed) so `dpkg
     --purge` could complete
   - Confirming a clean `dpkg -l | grep wazuh` before retrying
4. **Root cause of the disk issue** — the VM's LVM logical volume
   (`ubuntu--vg-ubuntu--lv`) was sized at 19GB despite the underlying disk
   and volume group having ~38GB available; the extra space was never
   allocated. Fixed with:

   ```bash
   sudo lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
   sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
   ```

5. **CPU/RAM contention** — subsequent retries hit kernel soft-lockups and
   RCU stall warnings (`watchdog: BUG: soft lockup - CPU#N stuck for Ns!`)
   during the indexer and dashboard install steps. Root-caused to running all
   three lab VMs (apache-web-01, kali, wazuh-manager) simultaneously on a
   2-core host, compounded by the host itself running low on free RAM
   (Chrome, VS Code, and other apps consuming ~1.6GB+ of a 16GB host with
   only ~2.7GB actually free at the time).
6. **DNS resolution failure under load** — one attempt failed at
   `curl: (6) Could not resolve host: raw.githubusercontent.com` while
   downloading a Wazuh template file; the underlying cause was
   `systemd-resolved` itself hitting a watchdog timeout from the same CPU
   starvation, not a DNS configuration issue.

## Part 3 — Scope Decision: Manager-Only Deployment

After multiple resource-constrained failures at the indexer/dashboard
install steps specifically, made the engineering call to **deploy Wazuh
manager only**, skipping the indexer and dashboard:

- Removes the OpenSearch/JVM-based indexer and Node.js-based dashboard —
  the two heaviest, most failure-prone components on this hardware.
- Alerts and events are inspected via the manager's own logs/CLI
  (`/var/ossec/logs/alerts/`, `/var/ossec/bin/wazuh-control`,
  `/var/ossec/bin/agent_control`) rather than a web dashboard.
- This is a legitimate, defensible SIEM-lite deployment pattern for
  constrained lab infrastructure, not a shortcut — and reflects a real
  trade-off a SOC engineer might make against limited hardware.

**Manual install steps:**

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | \
  sudo gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import
sudo chmod 644 /usr/share/keyrings/wazuh.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | \
  sudo tee /etc/apt/sources.list.d/wazuh.list
sudo apt update
sudo apt install wazuh-manager -y
sudo systemctl enable wazuh-manager
sudo systemctl start wazuh-manager
```

**Result:** clean install of `wazuh-manager 4.14.7-1`, no errors, service
`active (running)`. All core daemons confirmed up via
`systemctl status wazuh-manager`:
`wazuh-authd`, `wazuh-db`, `wazuh-execd`, `wazuh-analysisd`,
`wazuh-syscheckd`, `wazuh-remoted`, `wazuh-logcollector`,
`wazuh-monitord`, `wazuh-modulesd`, plus the Wazuh API.

## Where I've Reached

- Networking across all three VMs fully verified.
- `wazuh-manager` running a lean, manager-only Wazuh deployment on
  192.168.208.20, healthy and enabled at boot.
- Multiple real infrastructure failure modes diagnosed and resolved:
  disk exhaustion, broken package state recovery, LVM under-allocation,
  CPU/RAM contention on constrained hardware, and a cascading DNS failure.

## Next Steps

1. Deploy a Wazuh agent on `apache-web-01` and register it against the
   manager (key generation + agent authentication).
2. Configure the agent to ship Apache access/error logs to the manager.
3. Validate alert ingestion via `/var/ossec/logs/alerts/alerts.json`.
4. Run a Kali-based attack simulation against `apache-web-01` (e.g. scanner
   traffic, brute-force attempts, known web exploitation patterns).
5. Map detected alerts to MITRE ATT&CK techniques, closing out Stage 5 with
   a full attacker-plus-defender narrative for the portfolio.