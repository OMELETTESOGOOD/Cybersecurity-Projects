# SIEM Lab (Splunk) — Part 1: Installing Splunk on splunk-siem-01

**Project:** SIEM Lab (Splunk) — Part 1
**Date:** Week of 2026-08-17 to 2026-08-23
**Goal of this step:** Get Splunk Enterprise installed and running properly on the lab server `splunk-siem-01`, so it's ready to start collecting and analyzing logs.

## Why this matters

A SIEM (Security Information and Event Management) tool is basically a central place where all the logs from different computers and devices get collected and watched. Instead of checking each machine one by one, a security analyst uses a SIEM to see everything in one dashboard and get alerted when something suspicious happens.

Splunk is one of the most widely used SIEM tools in the industry, and having hands-on experience with it is something almost every SOC (Security Operations Center) job posting asks for. This project builds that experience from the ground up — starting with the install.

## What I did

### 1. Installed Splunk Enterprise

I downloaded the Splunk Enterprise `.deb` installer (a standard package format for Ubuntu/Debian-based Linux systems) directly onto the lab server:

```
wget -O splunk-10.4.2-33c3bf42cd73-linux-amd64.deb \
  "https://download.splunk.com/products/splunk/releases/10.4.2/linux/splunk-10.4.2-33c3bf42cd73-linux-amd64.deb"
```

Then installed it:

```
sudo dpkg -i splunk-10.4.2-33c3bf42cd73-linux-amd64.deb
```

The install completed without errors. There was one message during the process about a missing file path (`python3.7/site-packages`), but this turned out to be a harmless leftover check from an older version of Splunk's installer script — it didn't affect anything and the install still finished successfully.

I double-checked the install worked by running:

```
dpkg -l | grep splunk
```

This showed `ii` next to the Splunk package name, which is Linux's way of confirming a package is fully installed and working correctly.

### 2. Fixed a permissions warning (running Splunk as root)

When I first tried to start Splunk with `sudo /opt/splunk/bin/splunk start --accept-license`, it gave a warning that running Splunk as the "root" user (the most powerful account on a Linux system) is being phased out and won't be supported in future versions.

This matters for security reasons: running any service as root means that if something ever went wrong with that service, an attacker could potentially get full control of the whole machine. The safer practice — and the one real companies use — is to run services like Splunk under their own limited, dedicated user account instead. That way, even if something goes wrong, the damage is contained.

I fixed this by creating a dedicated `splunk` service account, handing it ownership of the install directory, and starting Splunk under that account instead of root:

```
sudo useradd -r -m -d /opt/splunk -s /bin/bash splunk
sudo chown -R splunk:splunk /opt/splunk
sudo -u splunk /opt/splunk/bin/splunk start --accept-license
```

This let Splunk start up cleanly without the warning, and follows better security practice — which is also a good thing to be able to point to when I write this project up (it shows I'm not just following steps blindly, but understanding *why* they matter).

### 3. Found and fixed a resource problem

Once Splunk was running, its built-in Health panel (Settings → Health Report) flagged two problems:

- **Not enough free disk space** — Splunk needs room to store its own internal logs and indexes, and the lab server was running low.
- **High "IOWait"** — this is a technical way of saying the processor was spending a lot of time waiting on the hard disk to catch up, which usually happens when a machine doesn't have enough memory (RAM) to work with.

I checked the server's actual resources:

```
df -h
free -m
```

This confirmed the problem: the VM only had ~21 GB of disk space (with ~9 GB free) and ~3.4 GB of RAM — both too low for Splunk to run comfortably, and there was no swap space configured either.

**The fix:**
1. Shut the VM down safely (`sudo shutdown now`)
2. Increased VM RAM from ~3.4 GB to ~5.4 GB in VirtualBox Settings → System
3. Resized the virtual hard disk from ~21 GB to 40 GB in VirtualBox's Medium Manager
4. Extended the actual Linux partition and filesystem to use the new space:

```
sudo growpart /dev/sda 1
sudo resize2fs /dev/sda1
```

After restarting Splunk, the Health panel's warning icon turned into a green checkmark, confirming both issues were resolved.

### 4. Set up remote access (SSH)

Instead of controlling the lab server through the VirtualBox console window every time, I set up SSH so I can drive it from my Windows host's command line:

```
sudo systemctl enable --now ssh
sudo systemctl status ssh
```

Then connected from Windows `cmd` using the built-in OpenSSH client:

```
ssh marshel@192.168.208.40
```

This makes future work faster and easier, since I can now copy and paste commands directly instead of typing everything manually inside the VM window.

## Outcome

By the end of this step, `splunk-siem-01` has:
- A clean, working install of Splunk Enterprise 10.4.2
- Splunk running securely under its own dedicated user account (not root)
- Enough disk space and memory to run reliably (40 GB disk, ~5.4 GB RAM)
- Remote SSH access set up for easier ongoing work

## What's next

With Splunk installed and the server running smoothly, the next steps in Part 1 are:
- Set up a Universal Forwarder on the `apache-web-01` server so its logs get sent to Splunk
- Configure indexes and sourcetypes (basically, telling Splunk how to organize and label the incoming log data)
- Build the first detection rule — a search and alert that catches a real attack technique called Shellshock

## Skills demonstrated in this step

- Linux server administration (package installation, service/user management)
- Reading and interpreting system diagnostics (disk, memory, service health)
- Applying basic security best practice (avoiding running services as root)
- Virtual machine resource management (VirtualBox disk/RAM configuration, partition/filesystem resizing)
- Remote server access via SSH