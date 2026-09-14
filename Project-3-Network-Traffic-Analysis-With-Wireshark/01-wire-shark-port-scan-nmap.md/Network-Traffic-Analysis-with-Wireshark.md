# Project 3: Network Traffic Analysis with Wireshark
## Scenario 1 — Detecting a Port Scan (Nmap SYN Scan)

**Date:** September 14, 2026
**Environment:** Kali Linux (attacker, 192.168.208.30) → apache-web-01 (target, 192.168.208.10), host-only lab network 192.168.208.0/24
**Tools used:** Wireshark, Nmap

---

## 1. Goal

The goal of this scenario is to run a port scan against apache-web-01 from Kali and capture the traffic in Wireshark, so I can see exactly what a port scan looks like at the packet level — not just in a tool's output, but on the wire.

---

## 2. Setting Up the Capture

Before running the scan, I had to make sure Wireshark was capturing on the correct network interface.

Kali has two interfaces:
- **eth0** — NAT adapter, IP `10.0.2.15`, used for internet access
- **eth1** — host-only adapter, IP `192.168.208.30`, used to reach the lab machines (apache-web-01, splunk-siem-01, wazuh-manager)

My first capture was accidentally on eth0. It only showed IPv6 router/multicast chatter, with no lab traffic at all, because eth0 isn't connected to the lab network.

![Wrong interface - eth0 shows no lab traffic](assets/01-wrong-interface-eth0.png)

I confirmed the correct interface with `ip a`, which showed eth1 holding the `192.168.208.30` address. Switching the Wireshark capture to eth1 immediately showed real lab traffic — ARP broadcasts from Kali trying to find the MAC addresses of splunk-siem-01 (`.40`) and wazuh-manager (`.20`).

![Correct interface - eth1 shows ARP traffic on the lab network](assets/02-correct-interface-eth1-arp.png)

**Lesson:** always confirm you're capturing on the right interface before running an attack — otherwise you can capture for several minutes and get nothing useful.

### Sanity check with ping

Before scanning, I pinged apache-web-01 from Kali to confirm connectivity and confirm the capture was picking up real traffic:

```
$ ping -c 4 192.168.208.10
PING 192.168.208.10 (192.168.208.10) 56(84) bytes of data.
64 bytes from 192.168.208.10: icmp_seq=1 ttl=64 time=2.08 ms
64 bytes from 192.168.208.10: icmp_seq=2 ttl=64 time=1.66 ms
64 bytes from 192.168.208.10: icmp_seq=3 ttl=64 time=1.18 ms
64 bytes from 192.168.208.10: icmp_seq=4 ttl=64 time=1.27 ms
4 packets transmitted, 4 received, 0% packet loss
```

All 4 pings succeeded with 0% packet loss, confirming the network path was working before running the actual scan.

---

## 3. Running the Scan

With Wireshark capturing on eth1, I ran an Nmap SYN scan against apache-web-01:

```
$ sudo nmap -sS 192.168.208.10

Starting Nmap 7.99 at 2026-09-14
Nmap scan report for 192.168.208.10
Host is up (0.00076s latency).
Not shown: 997 filtered tcp ports (no-response)
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
443/tcp open  https
MAC Address: 08:00:27:34:99:6B (Oracle VirtualBox virtual NIC)
```

**What `-sS` means:** this is a "SYN scan," also called a half-open scan. Instead of completing a full TCP connection to each port, Nmap sends a SYN packet and just watches how the target responds — then it never actually finishes the handshake. This makes the scan faster and quieter than a full connection scan.

**Result:** 3 ports open (22/SSH, 80/HTTP, 443/HTTPS), 997 ports filtered (no response at all).

---

## 4. What the Traffic Looks Like

I filtered the capture down to just the conversation between Kali and apache-web-01:

```
ip.addr == 192.168.208.30 and ip.addr == 192.168.208.10 and tcp
```

### 4.1 Open port — clean example (port 80)

For an open port, the pattern is always the same 3 packets, all within about 1 millisecond of each other:

1. Kali sends **SYN** to the port
2. apache-web-01 replies **SYN, ACK** — this means the port is open and listening
3. Kali immediately replies with **RST** instead of finishing the handshake with an ACK

That last step (RST instead of ACK) is what makes this a half-open scan — the connection is never actually completed.

![Port 80 - clean SYN, SYN-ACK, RST sequence](assets/06-port-80-clean-triplet.png)

Port 22 (SSH) showed the exact same 3-packet pattern.

![Port 22 (SSH) - open port triplet, interleaved with port 443 traffic](assets/03-ssh-open-port-triplet.png)

### 4.2 Filtered port — no response (port 993)

For a filtered port, there is no reply at all — just a SYN going out into silence:

![Port 993 - SYN sent, no reply at all](assets/04-filtered-port-993.png)

Wireshark even labels this for you. If you expand the TCP details on that packet, it shows **"Conversation completeness: Incomplete, SYN_SENT"** — Wireshark's own way of saying the handshake never got past the first step because nothing ever answered.

![Port 993 - Wireshark flags the connection as incomplete](assets/08-filtered-port-completeness-tag.png)

Nmap actually retried this port once (a second SYN, a few seconds later, from a new source port) before giving up and marking it "filtered / no-response" in its results.

### 4.3 An interesting anomaly — port 443

Port 443 (HTTPS) did something unexpected. Instead of one clean triplet like ports 22 and 80, it showed **three separate SYN → SYN-ACK → RST sequences**, spaced about 1.3 seconds apart.

![Port 443 - three separate SYN/SYN-ACK/RST sequences instead of one](assets/05-port-443-anomaly.png)

This isn't what you'd normally expect — once Nmap gets a SYN-ACK back, it should mark the port open and move on. A likely explanation is that Nmap was still calibrating its round-trip-time estimate early in the scan (the lab network is extremely fast, under 1ms), which can cause it to send an extra probe to a port before it's finished processing the first reply. It's a good reminder that real captures don't always look perfectly clean, and it's worth explaining *why* rather than ignoring it.

### 4.4 Conversation summary view

`Statistics → Conversations → TCP` gives a clean table of every TCP session in the capture, which is useful for a quick overview of how many ports were touched and how.

![Conversations view](assets/07-conversations-view.png)

---

## 5. Summary Table

| Port | Service | Result | Packets seen |
|------|---------|--------|---------------|
| 22 | SSH | Open | SYN → SYN-ACK → RST |
| 80 | HTTP | Open | SYN → SYN-ACK → RST |
| 443 | HTTPS | Open | SYN → SYN-ACK → RST (x3, anomaly — see 4.3) |
| 993 (example) | IMAPS | Filtered | SYN only, no reply, one retry |
| 997 other ports | various | Filtered | SYN only, no reply |

---

## 6. Key Takeaways

- Always double-check which network interface you're capturing on before starting — the wrong interface gives you no useful data.
- A SYN scan is identifiable in Wireshark by its RST instead of ACK after a SYN-ACK — a real application connection would send ACK and continue talking.
- Filtered ports show up as silence (no reply), not as an explicit rejection. A closed port instead gets an instant RST back with no SYN-ACK first.
- Wireshark's "Conversation completeness" field is a fast way to spot an incomplete handshake without counting packets by hand.
- Not every capture is perfectly clean — the port 443 anomaly here is a good example of why it helps to actually look at the packets instead of trusting the tool's summary output alone.

---

## 7. MITRE ATT&CK Mapping

| Technique | ID |
|-----------|-----|
| Network Service Discovery | T1046 |
