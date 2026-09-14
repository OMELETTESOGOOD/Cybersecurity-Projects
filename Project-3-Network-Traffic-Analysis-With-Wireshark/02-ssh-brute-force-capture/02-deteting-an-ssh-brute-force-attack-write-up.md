# Project 3: Network Traffic Analysis with Wireshark
## Scenario 2 — Detecting an SSH Brute-Force Attack

**Date:** September 14, 2026
**Environment:** Kali Linux (attacker, 192.168.208.30) → apache-web-01 (target, 192.168.208.10), host-only lab network 192.168.208.0/24
**Tools used:** Wireshark, Hydra

---

## 1. Goal

The goal of this scenario is to run an SSH brute-force attack against apache-web-01 and capture the traffic in Wireshark, to see what this kind of attack looks like at the packet level. This scenario also revisits the same attack used to build the SSH brute-force detection in the earlier Splunk project, but this time from the network's point of view instead of the log's point of view.

---

## 2. Building the Password List

I created a small wordlist with a handful of common wrong guesses plus the real password mixed in partway through the list, rather than first, so the attack would take multiple attempts before succeeding:

```
nano wordlist.txt
```

This gave Hydra 8 passwords to try against a single username, matching the attempt count used in the earlier Splunk detection (which had an 8-attempt threshold).

---

## 3. Running the Attack

With Wireshark capturing on eth1 and filtered to `tcp.port == 22`, I ran:

```
$ hydra -l marshel -P wordlist.txt ssh://192.168.208.10

Hydra v9.7 (c) 2023 by van Hauser/THC & David Maciejak
[DATA] max 8 tasks per 1 server, overall 8 tasks, 8 login tries (l:1/p:8)
[DATA] attacking ssh://192.168.208.10:22/
[22][ssh] host: 192.168.208.10   login: marshel   password: 049977
1 of 1 target successfully completed, 1 valid password found
Hydra finished at 2026-09-14 05:48:48
```

Hydra tried 8 passwords and found the correct one (`049977`) in about 5 seconds.

---

## 4. What the Traffic Looks Like

### 4.1 Repeated connection pattern

The clearest sign of a brute-force attack in the packet capture is the repeated pattern of short-lived connections, one after another, each using a new source port. Each attempt opens a connection, does the SSH exchange, then closes with FIN/ACK before the next one starts:

![Repeated short-lived SSH connections, one per password attempt](assets/09-repeated-connections-pattern.png)

This rapid, regular pattern of connect → authenticate → disconnect → repeat is the main signature of a brute-force attack at the network layer — even without seeing any password, the *behavior* itself is suspicious.

### 4.2 Full handshake sequence (first attempt)

Looking at the very first connection from start to finish shows the full SSH negotiation process:

1. **TCP handshake** — SYN, SYN-ACK, ACK
2. **SSH version banner exchange** — client and server each announce their SSH version, in plaintext
3. **Key exchange (KEX)** — client and server agree on encryption algorithms and generate a shared secret
4. **New Keys** — encryption is switched on
5. **Encrypted packets** — the actual username/password submission happens here, fully encrypted

![Full SSH handshake sequence from TCP handshake through encrypted authentication](assets/10-full-handshake-sequence.png)

### 4.3 The plaintext banner — the one readable part

Before encryption starts, the SSH version banners are sent as plain, readable text. This is normal and not a vulnerability — it's just how SSH identifies which version/implementation each side is running before agreeing on encryption.

![SSH banner packets visible in the Info column, before encryption starts](assets/12-ssh-banner-packet-detail.png)
![Banner exchange, zoomed view](assets/15-banner-info-column-zoom.png)

### 4.4 Confirming credentials are not visible

Using **Follow → TCP Stream** on the first session shows exactly where the plaintext ends and encryption begins:

![Follow TCP Stream: plaintext SSH banners followed by unreadable encrypted data](assets/14-follow-tcp-stream-banners-and-ciphertext.png)

At the top, the two banners are clearly readable:
```
SSH-2.0-libssh_0.12.0
SSH-2.0-OpenSSH_10.2p1 Ubuntu-2ubuntu3.5
```

Right after that comes the list of encryption algorithms both sides support (also sent in plaintext, since they need to agree on this before they can encrypt anything) — this is normal negotiation data, not credentials. Everything after that point is genuinely unreadable binary data. **The username and password (`marshel` / `049977`) never appear anywhere in this capture.** This is a good contrast to keep in mind for the plaintext credential capture scenario later in this project, where credentials sent over unencrypted HTTP will be fully readable.

### 4.5 Session count and timing

`Statistics → Conversations → TCP` shows each password attempt as a separate TCP stream:

![Conversations view: 9 separate TCP sessions to port 22](assets/11-conversations-tcp-view.png)

The capture shows **9 separate TCP sessions** to port 22, each around 23–27 packets and roughly 5KB — close to Hydra's 8 password attempts (the small difference is likely due to how Hydra manages parallel/final connections). This lines up well with the 8-attempt threshold used in the earlier Splunk-based detection for this same attack.

---

## 5. Summary Table

| Item | Value |
|------|-------|
| Attempts made | 8 (per Hydra output) |
| TCP sessions observed | 9 |
| Attack duration | ~5 seconds (Hydra) |
| Credentials visible in capture? | No — encrypted after key exchange |
| What *is* visible | SSH version banners, encryption algorithm list, connection timing pattern |
| Password found | `049977` (user: `marshel`) |

---

## 6. Key Takeaways

- SSH encrypts credentials before they're ever sent, so a packet capture alone can't recover a username or password from a brute-force attempt — this is very different from what we'll see with a plaintext HTTP login later in this project.
- Even without visible credentials, the attack is still detectable purely from behavior: many short connections to the same port, from the same source, in rapid succession.
- The SSH version banner and algorithm negotiation are the only genuinely readable parts of the exchange — useful for fingerprinting client/server software, but not sensitive on their own.
- This network-level evidence (9 sessions in a few seconds) lines up closely with the log-based detection built earlier in the Splunk project (8 attempts in 24 seconds), showing the same attack from two different vantage points.

---

## 7. MITRE ATT&CK Mapping

| Technique | ID |
|-----------|-----|
| Brute Force: Password Guessing | T1110.001 |

---

## 8. Files

- `ssh-bruteforce-hydra.pcapng` — full packet capture for this scenario