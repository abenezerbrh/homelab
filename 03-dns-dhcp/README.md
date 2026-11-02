# Project 3 - Homelab DNS + DHCP Server

## Overview

This project focused on running my own name resolution and address assignment.

In the first part, I set up Unbound as a recursive DNS resolver behind Pi-hole, so homelab01 looks up DNS answers itself instead of relying on a public DNS provider. In the second part, I built an isolated lab to run a DHCP server and a DHCP relay, and tested what happens when DNS and DHCP fail.

---

## Scope

homelab01 is on a shared network that belongs to my landlord, so the project is split in two:

- **DNS (Part A)** runs on my real network but only affects my own devices, which are manually set to use Pi-hole.
- **DHCP (Part B)** runs only inside an isolated Containerlab lab. Running a second DHCP server on a shared network would hand out wrong settings to other people's devices (a "rogue DHCP server"), so it never touches the real network.

---

## Environment

| Item | Value |
|---|---|
| Host | `homelab01`, Dell OptiPlex 5060, Ubuntu Linux |
| DNS filter | Pi-hole |
| Recursive resolver | Unbound (`127.0.0.1#5335`) |
| DNS clients | My Windows laptop (DNS set to `192.168.1.78`) |
| DHCP lab | Containerlab (Part B) |

---

## Architecture

**Screenshot 1**

![Screenshot 1](screenshots/01.png)

---

# Part A: DNS

## 1. Current Pi-hole Setup

### Commands

```bash
pihole status; pihole -v
```

```bash
sudo pihole-FTL --config dns.upstreams
```

### Observations

- Pi-hole was running, on Core 6.5, Web 6.6.1, and FTL 6.8, all up to date.
- The upstream DNS servers were Cloudflare's `1.1.1.1` and `1.0.0.1`. I noted these as a rollback in case Unbound didn't work.

**Screenshot 2**

![Screenshot 2](screenshots/02.png)

---

## 2. Installing Unbound

### Command

```bash
sudo apt update && sudo apt install unbound
```

### Observations

- Unbound installed along with three libraries: libevent (handles many network connections efficiently), libhiredis (optional Redis cache support), and libprotobuf-c (optional dnstap logging).
- The installer enabled Unbound to start at boot, but failed to start it right away ("Could not execute systemctl"). Unbound's default config tries to use port 53, which Pi-hole already uses.
- The installer also enabled `unbound-resolvconf`, a helper that would make Unbound homelab01's own DNS server. I disabled it in section 4 to keep the router as homelab01's DNS server.

---

## 3. Configuring Unbound

Created `/etc/unbound/unbound.conf.d/pi-hole.conf`:

```
server:
    verbosity: 0
    interface: 127.0.0.1
    port: 5335
    do-ip4: yes
    do-udp: yes
    do-tcp: yes
    do-ip6: no
    prefer-ip6: no
    harden-glue: yes
    harden-dnssec-stripped: yes
    use-caps-for-id: no
    edns-buffer-size: 1232
    prefetch: yes
    num-threads: 1
    private-address: 192.168.0.0/16
    private-address: 169.254.0.0/16
    private-address: 172.16.0.0/12
    private-address: 10.0.0.0/8
    private-address: fd00::/8
    private-address: fe80::/10
```

### Key settings

- **`interface: 127.0.0.1` and `port: 5335`:** Unbound only listens on homelab01 itself, on port 5335. It doesn't compete with Pi-hole for port 53, and only Pi-hole can reach it.
- **`harden-dnssec-stripped`:** enforces DNSSEC validation, so tampered answers are rejected.
- **`prefetch`:** refreshes popular records before they expire, keeping lookups fast.
- **`private-address`:** blocks outside DNS servers from returning private IP addresses, which protects against DNS rebinding attacks.

---

## 4. Keeping homelab01's Own DNS Separate

### Commands

```bash
systemctl is-enabled unbound-resolvconf
sudo systemctl disable --now unbound-resolvconf
```

### Observations

Installing Unbound also enabled `unbound-resolvconf`, a helper service that makes Unbound the DNS server for homelab01 itself. `systemctl is-enabled` confirmed it was `enabled`, so I disabled it.

### What was learned

Pi-hole and Unbound answer DNS for my laptop, but homelab01 uses the router for its own lookups. If homelab01 depended on Unbound and Unbound broke, it couldn't resolve names at all, so I couldn't run `apt` or keep Tailscale working to fix it. Keeping the server I'm experimenting on independent from the experiment means it can always be repaired.

---

## 5. Testing Unbound

### Commands

```bash
dig pi-hole.net @127.0.0.1 -p 5335
```

```bash
dig fail01.dnssec.works @127.0.0.1 -p 5335
dig dnssec.works @127.0.0.1 -p 5335
```

### Observations (recursion and caching)

- The first query for `pi-hole.net` took **385 ms**. Unbound had to look it up from scratch, starting at the root DNS servers.
- The same query a few seconds later took **0 ms**, because Unbound answered from its cache.
- The TTL dropped from `3600` to `3580` between the two queries. A cached answer is only kept for its TTL (here, one hour), and the number shows how much time is left.
- `pi-hole.net` had no `ad` flag because the domain isn't DNSSEC-signed. Unsigned domains still resolve normally, just without being verified.

**Screenshot 3**

![Screenshot 3](screenshots/03.png)

**Screenshot 4**

![Screenshot 4](screenshots/04.png)

---

## 6. Pointing Pi-hole at Unbound

Replaced Pi-hole's upstream DNS servers with Unbound (`127.0.0.1#5335`) under **Settings → DNS** in the Pi-hole admin page.

### Observations

- Unticked all the preset upstream providers, including Cloudflare, and added `127.0.0.1#5335` as the only custom DNS server.
- Pi-hole's own hint text uses this exact format as its example for a local Unbound instance (`IP#port`).
- `pihole-FTL --config dns.upstreams` confirmed `127.0.0.1#5335` was the only upstream.
- Pi-hole's built-in DHCP server stays disabled, since enabling it would put a second DHCP server on the shared network.

**Screenshot 5**

![Screenshot 5](screenshots/05.png)

---

## 7. Testing from the Laptop

### Command

```powershell
nslookup github.com 192.168.1.78
```

### Observations

- The lookup from my laptop returned an IP for github.com, with the server listed as `pi.hole` (192.168.1.78). The laptop's DNS path is now laptop → Pi-hole → Unbound.
- "Non-authoritative answer" means the answer came from a resolver, not from GitHub's own DNS servers.
- The same lookup from homelab01 earlier returned a different IP (140.82.114.4 vs 140.82.113.3). GitHub hands out different addresses to spread traffic across its servers.

**Screenshot 6**

![Screenshot 6](screenshots/06.png)

---

## 8. Watching Recursive Resolution

_To be added_

---

## 9. Local DNS Records

_To be added_

---

## 10. Query Logging and Blocking

_To be added_

---

# Part B: DHCP

_To be added_

---

## Problems Encountered

_To be added_

---

## Break and Fix Tests

_To be added_

---

## Key Concepts Learned

_To be added_

---

## Commands Reference

_To be added_

---

## Project Outcome

_To be added_