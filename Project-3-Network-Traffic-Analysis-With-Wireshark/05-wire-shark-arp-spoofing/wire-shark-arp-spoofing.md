# Wireshark Scenario 5: ARP Spoofing / Man-in-the-Middle

## Overview
This scenario captures and analyzes an active ARP spoofing (cache poisoning) attack against two live hosts on the lab network — apache-web-01 and wazuh-manager — with Kali inserted as a man-in-the-middle. Unlike the earlier scenarios, which targeted a single victim from the outside, this one demonstrates interception of traffic between two hosts that weren't communicating with the attacker at all.

## Environment
- **Attacker:** Kali Linux VM — `192.168.208.30` (`08:00:27:34:99:6b`)
- **Victim 1:** apache-web-01 — `192.168.208.10` (`08:00:27:74:0f:50`)
- **Victim 2:** wazuh-manager — `192.168.208.20` (`08:00:27:80:03:53`)
- **Capture interface:** `eth1` (host-only segment, 192.168.208.0/24)
- **Capture tool:** `tcpdump`, analyzed in Wireshark

Before targeting this pair, `ip route` was checked on apache-web-01, wazuh-manager, and splunk-siem-01. All three showed their default gateway (`10.0.2.2`) sitting on a separate NAT interface (`enp0s3`), with `192.168.208.0/24` reachable directly with no gateway on `enp0s8`. Since Kali only sits on the `192.168.208.0/24` segment, the standard victim-to-gateway MITM setup wasn't possible here — Kali can only spoof ARP between hosts sharing its own broadcast domain. apache-web-01 and wazuh-manager were selected as the target pair since they have a real, meaningful relationship (Wazuh agent-to-manager communication over TCP/1514).

## Attack Execution
Capture started on Kali:
```bash
sudo tcpdump -i eth1 -w arp-spoof.pcap
```

IP forwarding enabled on Kali so intercepted traffic continues to its real destination instead of dying at the attacker:
```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

Both ARP spoofing directions run simultaneously on Kali, in separate terminals:
```bash
sudo arpspoof -i eth1 -t 192.168.208.10 192.168.208.20   # tell apache-web-01 Kali is wazuh-manager
sudo arpspoof -i eth1 -t 192.168.208.20 192.168.208.10   # tell wazuh-manager Kali is apache-web-01
```

With both spoofing processes active, the Wazuh agent on apache-web-01 was restarted to generate fresh, meaningful traffic to the manager:
```bash
sudo systemctl restart wazuh-agent
```

## Observations

### 1. ARP cache poisoning confirmed
Filtering on `arp` (327 of 3,129 packets, 10.5% of the capture) showed repeated spoofed replies claiming `192.168.208.10 is at 08:00:27:34:99:6b` and `192.168.208.20 is at 08:00:27:34:99:6b` — both pointing to **Kali's MAC address**, not the real MAC of either host. The volume and repetition of these replies (far exceeding normal ARP chatter) is itself a detection signature; legitimate hosts don't re-announce their own MAC that aggressively.

### 2. Wazuh agent-manager traffic successfully intercepted
Filtering on `tcp.port == 1514` isolated 2,798 packets — **89.4% of the entire capture**. This is the key evidence: this traffic is between apache-web-01 and wazuh-manager, captured on Kali's own interface. Under normal conditions Kali is neither source nor destination and would never see this traffic at all. Its presence in Kali's capture is only possible because the ARP poisoning successfully rerouted it through Kali first.

### 3. Layer 2 / Layer 3 address mismatch — direct proof of interception
Inspecting an individual TCP packet within the port-1514 stream (frame 28) showed:
- **Ethernet II:** `Src: 08:00:27:34:99:6b` (Kali)
- **Internet Protocol:** `Src: 192.168.208.10, Dst: 192.168.208.20` (apache-web-01 → wazuh-manager)

The IP layer still addresses the packet between the two real hosts, while the Ethernet layer shows it physically passing through Kali. This IP-says-one-thing / MAC-says-another mismatch is the clearest single proof that Kali sat in the middle of a conversation it was never part of.

### 4. Side effects of the interception
- **ICMP Redirects** from apache-web-01 (frames 6, 16, 29), each flagging that traffic to wazuh-manager was taking an unexpected path via Kali — a network-layer defense mechanism reacting to the spoof in real time, even though it didn't fully undo it.
- **TCP retransmissions and duplicate ACKs** scattered throughout the port-1514 traffic — a natural side effect of routing legitimate traffic through an extra, unplanned hop, adding latency and occasional loss. Secondary evidence that the interception was actively disrupting normal traffic flow, not passively observing it.

## MITRE ATT&CK Mapping
| Technique | ID | Notes |
|---|---|---|
| Adversary-in-the-Middle: ARP Cache Poisoning | T1557.002 | Kali poisoned the ARP caches of two internal hosts to intercept traffic between them |

## Detection Takeaways
- Repeated ARP replies claiming ownership of an IP already known to belong to a different MAC is the primary detection signature — a host's real MAC doesn't change, so any conflicting reply is inherently suspicious
- A single host physically touching (Ethernet layer) traffic it has no business handling (IP layer addressed to two other hosts) is definitive proof of interception, not just poisoning attempts
- ICMP Redirects generated by the victim host can be an organic, built-in signal that something is wrong with the routing path, even without dedicated ARP-spoofing detection tooling
- Latency/retransmission anomalies on a normally low-latency internal link are a soft but useful secondary indicator

