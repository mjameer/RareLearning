# AWS SysDE Interview Prep — Networking
> Principal Software Engineer level answers + deep dives from study session

---

## Table of Contents
1. [DNS Flow & How DNS Works](#1-dns-flow--how-dns-works)
2. [What Happens When You Type amazon.com](#2-what-happens-when-you-type-amazoncom)
3. [TCP/IP Connection Process](#3-tcpip-connection-process)
4. [TCP vs UDP](#4-tcp-vs-udp)
5. [HTTPS / TLS / Certificate Trust](#5-https--tls--certificate-trust)
6. [DHCP & the DORA Process](#6-dhcp--the-dora-process)
7. [NAT — Network Address Translation](#7-nat--network-address-translation)
8. [Subnets](#8-subnets)
9. [IP Classes](#9-ip-classes)
10. [Network Gateway — Public & Private](#10-network-gateway--public--private)
11. [Switch vs Hub vs Router](#11-switch-vs-hub-vs-router)
12. [MAC vs IP / Troubleshooting](#12-mac-vs-ip--troubleshooting)
13. [Wireless Router — How It Works at Home](#13-wireless-router--how-it-works-at-home)
14. [Website Unavailable — Troubleshooting Steps](#14-website-unavailable--troubleshooting-steps)
15. [HTTP vs HTTPS / HTTP-to-HTTPS Routing](#15-http-vs-https--http-to-https-routing)
16. [Full Forms & Memory Tricks](#16-full-forms--memory-tricks)

---

## 1. DNS Flow & How DNS Works

### Principal-Level Answer
> "DNS is a distributed lookup system — think of it as a hierarchical phone book. When I type amazon.com, my machine first checks its local cache, then the OS resolver. If it's not there, it goes to a recursive resolver — usually my ISP's or something like 8.8.8.8. That resolver works its way up the hierarchy: asks a root nameserver who's responsible for .com, gets pointed to the .com TLD server, which then points it to Amazon's authoritative nameserver, which finally hands back the actual IP. The resolver caches that result per the TTL and returns it to me. The whole thing usually takes milliseconds."

### Full Flow

```
You type amazon.com
        │
        ▼
Browser Cache       ← "Did I look this up recently?"
        │ No
        ▼
OS Resolver         ← Checks local DNS cache + /etc/hosts
        │ No
        ▼
Recursive Resolver  ← Your ISP or 8.8.8.8 (does the hard work)
        │
        ├──→ Root Nameserver        "Who handles .com?"
        │         ↓
        ├──→ TLD Nameserver (.com)  "Who handles amazon.com?"
        │         ↓
        └──→ Authoritative NS       "amazon.com = 205.251.242.103"
                  ↓
        Returns IP → Browser opens TCP connection
```

### What Is the OS Resolver?

Your Operating System has a built-in DNS resolver. When your browser asks "what's the IP for amazon.com?", it doesn't go to the internet directly — it asks the OS first.

The OS checks two things in order:

**A) OS DNS Cache**
- Every DNS lookup is cached at the OS level
- Multiple apps (Chrome, Slack, Spotify) share this cache
- If one app already resolved `amazon.com`, all others benefit

**B) The hosts file**
- A plain text file that manually maps names to IPs
- Checked BEFORE any network query goes out
- Windows: `C:\Windows\System32\drivers\etc\hosts`
- Linux/Mac: `/etc/hosts`

```
# hosts file example
127.0.0.1    localhost
192.168.1.10 my-dev-server
# If you add: 1.2.3.4 amazon.com
# Browser goes to 1.2.3.4 — no DNS query leaves your machine
```

> **Security note:** Malware often edits `/etc/hosts` to redirect legitimate domains to fake servers.

### OS Resolver on Windows

On Windows, the OS resolver is a background Windows Service called **DNS Client** (internally: `dnscache`).

```cmd
# See everything currently cached
ipconfig /displaydns

# Output:
amazon.com
  Record Name:  amazon.com
  Record Type:  1 (A record — IPv4)
  Time To Live: 289 seconds remaining
  A (Host) Record: 205.251.242.103

# Flush the cache (use when DNS is behaving strangely after changes)
ipconfig /flushdns

# See your configured DNS servers
ipconfig /all
# DNS Servers: 192.168.1.1 (your router)

# Manually query DNS
nslookup amazon.com
# Server: router.home (192.168.1.1) — which server answered
# Name:   amazon.com
# Address: 205.251.242.103

# PowerShell version (more detail)
Resolve-DnsName amazon.com
```

### DNS Hierarchy Explained

| Layer | Who It Is | What It Knows |
|---|---|---|
| Root Nameserver | 13 sets worldwide (NASA, ICANN, Verisign) | Which server handles each TLD (.com, .org, .in) |
| TLD Nameserver | Verisign for .com | Which server is authoritative for each domain in .com |
| Authoritative NS | Amazon's own DNS server | The actual IP for amazon.com |

### Key Terms

| Term | What It Is |
|---|---|
| OS Resolver | Built-in OS component — checks local cache + hosts file |
| Recursive Resolver | ISP/Google/Cloudflare server that does all the lookups for you |
| Root Nameserver | Top of DNS tree — knows who owns each TLD |
| TLD Nameserver | Knows who owns each domain within that TLD |
| Authoritative NS | Domain owner's server — has the actual IP |
| TTL | Time To Live — how long to cache the answer |
| /etc/hosts | Local override file — checked before any DNS query |

---

## 2. What Happens When You Type amazon.com

### Principal-Level Answer
> "There are a few layers here. First, DNS resolution — I just described that. Once I have the IP, my browser initiates a TCP connection with a three-way handshake: SYN, SYN-ACK, ACK. Since it's HTTPS, on top of that we do a TLS handshake — the server presents its certificate, my browser validates it against a trusted root CA, they negotiate a session key, and now we have an encrypted channel. Then the actual HTTP GET goes out, the server responds with HTML, and the browser starts fetching sub-resources — CSS, JS, images — often in parallel. Modern browsers also use HTTP/2 or HTTP/3 to multiplex those requests over a single connection, which reduces latency significantly."

### Full Step-by-Step

```
1. DNS Resolution
   → Look up IP for amazon.com (full flow in section 1)

2. TCP 3-way Handshake
   Client → Server: SYN
   Server → Client: SYN-ACK
   Client → Server: ACK
   → Connection established

3. TLS Handshake (because it's HTTPS)
   → Server presents certificate
   → Browser validates certificate chain
   → Session key negotiated
   → Encrypted channel established

4. HTTP GET Request
   GET / HTTP/1.1
   Host: amazon.com

5. Server Response
   → HTML sent back
   → Browser parses HTML

6. Sub-resource Fetching
   → CSS, JavaScript, images fetched (often in parallel)
   → HTTP/2 multiplexes all over one connection

7. Browser Renders the Page
```

---

## 3. TCP/IP Connection Process

### Principal-Level Answer
> "The handshake is three steps. Client sends SYN — 'I want to connect, here's my sequence number.' Server responds SYN-ACK — 'Got it, here's mine.' Client sends ACK — connection is established. Both sides now have synchronized sequence numbers, which is how TCP guarantees ordered, reliable delivery. Teardown is four steps because each side closes independently — FIN from one side, ACK, then FIN from the other, ACK. The reason I bring this up is that in high-throughput systems, TIME_WAIT states from teardown can actually exhaust ports if you're not careful — something I've dealt with in practice."

### The 3-Way Handshake

```
You                         Amazon
 │                             │
 │——— SYN (seq=100) ————————→  │   "I want to connect"
 │                             │
 │  ←— SYN-ACK (seq=500,  ————│   "OK, I'm ready too"
 │         ack=101)            │
 │                             │
 │——— ACK (ack=501) ————————→  │   "Great, let's go"
 │                             │
 |======= DATA FLOWS ===========|
```

### Why Sequence Numbers?

Imagine Amazon sends you a 1000-page book split into 1000 packets. They might arrive out of order — packet 47 before packet 3. Sequence numbers let you:
- Reassemble packets in the correct order
- Detect missing packets and request retransmission
- That's how TCP guarantees **reliable, ordered delivery**

### Teardown (4-way)

```
Client → Server: FIN   (I'm done sending)
Server → Client: ACK   (Got it)
Server → Client: FIN   (I'm done too)
Client → Server: ACK   (Got it — connection closed)
```

Each side closes independently. After the final ACK, the connection enters `TIME_WAIT` for 2×MSL (Maximum Segment Lifetime) before fully releasing the port.

> **Production note:** In high-throughput systems, `TIME_WAIT` states can exhaust the ephemeral port range (~60,000 ports). Mitigations: `SO_REUSEADDR`, `tcp_tw_reuse` kernel param, or connection pooling.

---

## 4. TCP vs UDP

### Principal-Level Answer
> "TCP gives you reliability — guaranteed delivery, ordering, retransmission. UDP is fire-and-forget, no handshake, no ordering guarantees. The tradeoff is latency and overhead. For something like HTTP or SSH, you need TCP — you can't have packets arriving out of order. But for video streaming, VoIP, or gaming, you'd rather drop a frame than wait for a retransmit and introduce jitter. DNS is interesting — it uses UDP by default because it's a small, fast query-response, but falls back to TCP when the response is too large."

### Comparison Table

| | TCP | UDP |
|---|---|---|
| Connection | Connection-oriented (handshake required) | Connectionless |
| Reliability | Guaranteed delivery, ordering, retransmit | Best-effort, no guarantee |
| Speed | Slower (overhead) | Faster |
| Header size | 20 bytes | 8 bytes |
| Use cases | HTTP/S, SSH, FTP, email | DNS, video streaming, VoIP, gaming |

### DNS Uses Both

- **UDP by default** — fast, small query-response payload
- **Falls back to TCP** when response exceeds 512 bytes (e.g. large zone transfers)

---

## 5. HTTPS / TLS / Certificate Trust

### Principal-Level Answer
> "HTTPS is just HTTP over TLS. The TLS handshake is what makes it secure. The server presents a certificate that contains its public key and is signed by a certificate authority. My browser has a trust store — a list of root CAs pre-installed by the OS — and it walks the certificate chain from the leaf cert up to a root CA it trusts. If that chain validates and the domain matches, we're good. Then we do a key exchange — typically something like ECDHE — to derive a shared session key, and from that point everything is encrypted symmetrically, which is fast. The asymmetric crypto is only used to establish that key."

### What Is a Certificate (X.509)?

A certificate is a digitally signed data structure containing:

```
- Subject:        CN=amazon.com
- Issuer:         CN=DigiCert SHA2 Secure Server CA
- Public Key:     RSA 2048-bit or EC P-256
- Serial Number:  unique identifier
- Validity:       NotBefore / NotAfter timestamps
- SAN:            Subject Alternative Names (*.amazon.com, amazon.com)
- Signature:      Issuer's digital signature over all the above
```

### How Digital Signing Works

```
DigiCert signs Amazon's cert:
1. Takes all cert fields → hashes them (SHA-256) → digest
2. Encrypts digest with DigiCert's PRIVATE key → that's the signature
3. Attaches signature to cert

Your browser verifies:
1. Takes same cert fields → hashes them → digest A
2. Decrypts signature using DigiCert's PUBLIC key → digest B
3. If A == B → DigiCert really signed this ✓
```

### The Three Layers (Chain of Trust)

```
Root CA
├── Self-signed (signs itself)
├── Pre-installed in OS/browser trust store
├── Private key kept OFFLINE in physical vaults
└── Example: "DigiCert Global Root CA"

Intermediate CA
├── Signed by Root CA
├── Does day-to-day cert signing
├── Isolates Root key from online exposure
└── Example: "DigiCert SHA2 Secure Server CA"

Leaf Certificate
├── Signed by Intermediate CA
├── Issued directly to the domain (amazon.com)
├── Contains the server's public key
└── Presented to your browser on every connection
```

### Walking the Chain — Step by Step

```
Browser receives leaf cert from amazon.com
        │
Step 1: Parse leaf cert
        Subject:  amazon.com
        Issuer:   DigiCert Intermediate CA
        │
Step 2: Find issuer's cert (DigiCert Intermediate)
        Server sends it / browser fetches via AIA URL
        │
Step 3: Verify leaf cert's signature
        Hash(leaf cert fields) == Decrypt(sig, DigiCert Intermediate pubkey)?
        ✅ Yes
        │
Step 4: Validate Intermediate cert
        Issuer: DigiCert Root CA
        Is Root CA in my trust store? ✅ Yes
        │
Step 5: Verify Intermediate's signature using Root CA pubkey
        ✅ Yes
        │
Step 6: Root CA is self-signed and in trust store → CHAIN COMPLETE ✓
```

### Domain Validation (SAN Check)

```
Cert contains Subject Alternative Names (SAN):
  DNS: amazon.com
  DNS: *.amazon.com
  DNS: www.amazon.com

You typed: amazon.com
SAN contains: amazon.com ✅ match → proceed

You typed: amazon.com
SAN contains: evil.com   ❌ mismatch → ERR_CERT_COMMON_NAME_INVALID
```

Wildcard `*.amazon.com` matches ONE level deep only.

### Certificate Revocation

| Method | How It Works | Drawback |
|---|---|---|
| CRL | CA publishes list of revoked serial numbers | Can be large, stale |
| OCSP | Browser queries CA's OCSP responder in real-time | Extra network call |
| OCSP Stapling | Server pre-fetches OCSP response and attaches it to TLS handshake | Best — fast + private |

### What Breaks the Chain

| Failure | Browser Error |
|---|---|
| Signature verification fails | `ERR_CERT_AUTHORITY_INVALID` |
| Root CA not in trust store | `ERR_CERT_AUTHORITY_INVALID` |
| Domain doesn't match SAN | `ERR_CERT_COMMON_NAME_INVALID` |
| Cert expired | `ERR_CERT_DATE_INVALID` |
| Cert revoked | `ERR_CERT_REVOKED` |
| Intermediate cert missing | `ERR_CERT_INCOMPLETE` |

### TLS 1.3 Handshake (Full Context)

```
Client → Server:  ClientHello (TLS version, cipher suites, random)
Server → Client:  ServerHello + Certificate + CertificateVerify
Client:           [validates certificate chain as above]
Client → Server:  Finished (key derivation complete)
Server → Client:  Finished
══════════════════════════════════════════
Symmetric encryption begins (AES-GCM)
```

Key exchange uses **ECDHE (Elliptic Curve Diffie-Hellman Ephemeral)** — provides forward secrecy (even if private key is later compromised, past sessions remain safe).

> **Why Intermediate CAs exist:** Root CA private keys are the most sensitive in internet security. If compromised, millions of certs become untrusted. So Root CAs are kept air-gapped in physical vaults, only turned on a few times a year to sign Intermediate CA certs. Intermediates do day-to-day work. If an Intermediate is compromised, it can be revoked without touching the Root.

---

## 6. DHCP & the DORA Process

### What Is DHCP?

**DHCP = Dynamic Host Configuration Protocol**

DHCP is the **system/service** that automatically assigns IP addresses and network configuration to devices. Without it, a network admin would have to manually configure every device — impossible at scale.

**DHCP has two parts:**

```
DHCP Server                        DHCP Client
─────────────────────              ─────────────────────
Runs on: your router               Runs on: your laptop/phone
                                   
Maintains:                         Triggers DORA automatically when:
  - IP pool                          - You open your laptop
  - Lease table (who has what)       - WiFi connects
  - Lease timers                     - Lease expires
  
Responds to DORA requests          Applies received configuration
```

**DORA is the conversation between them** — 4 messages, under 1 second, completely invisible to you.

### Where DHCP Is Used

| Environment | DHCP Server |
|---|---|
| Home WiFi | Built into your router firmware |
| Corporate office | Windows Server DHCP role / Linux ISC-DHCP |
| AWS VPC | Built-in per-VPC DHCP server (configured via DHCP Option Sets) |
| ISP network | Assigns your router its public IP |
| Kubernetes | CNI plugin handles pod IP assignment (DHCP-like) |

### DORA — Full Technical Flow

**D = Discover**
```
UDP Packet:
  Source IP:        0.0.0.0          ← client has no IP yet
  Destination IP:   255.255.255.255  ← broadcast to entire network
  Source Port:      68               ← DHCP client port
  Destination Port: 67               ← DHCP server port

Payload:
  - Client MAC address
  - Transaction ID (random, to match request/response)
  - "I want an IP"

Why broadcast? Client has no IP → can't unicast. Only DHCP servers respond.
```

**O = Offer**
```
UDP Packet:
  Source IP:        192.168.1.1      ← DHCP server IP
  Destination IP:   255.255.255.255  ← still broadcast (client still has no IP)

Payload:
  - Offered IP:      192.168.1.101
  - Subnet Mask:     255.255.255.0
  - Default Gateway: 192.168.1.1
  - DNS Server:      8.8.8.8
  - Lease Duration:  86400 seconds (24h)
  - Server ID:       192.168.1.1
  - Same Transaction ID

Note: Server doesn't assign yet — just reserves temporarily.
      Multiple DHCP servers could offer; client picks one.
```

**R = Request**
```
UDP Packet:
  Source IP:        0.0.0.0          ← STILL no IP
  Destination IP:   255.255.255.255  ← still broadcast

Why broadcast again?
  1. Client still has no IP — can't unicast
  2. Informs ALL DHCP servers — others see this and release their reserved IPs

Payload:
  - Requested IP:   192.168.1.101
  - Server ID:      192.168.1.1 (which offer it's accepting)
```

**A = Acknowledge**
```
UDP Packet:
  Source IP:  192.168.1.1
  Dst IP:     255.255.255.255

Payload:
  - Confirmed IP:   192.168.1.101
  - Lease time:     86400s
  - T1 (renewal):   43200s (50% — start trying to renew)
  - T2 (rebind):    75600s (87.5% — last chance)

Client now configures its interface:
  ip addr: 192.168.1.101/24
  gateway: 192.168.1.1
  dns:     8.8.8.8
```

### DHCP Lease Lifecycle

```
T=0s          Client gets IP, lease = 86400s
              │
T=43200s (T1) Client sends unicast RENEW to same server
(50%)         "Can I keep 192.168.1.101?"
              Server ACKs → lease reset to 86400s
              │
T=75600s (T2) No response to T1?
(87.5%)       Client broadcasts REBIND to ANY server
              │
T=86400s      No response → lease expires
(100%)        Client loses IP, starts DORA from scratch
```

### Why UDP and Not TCP?

```
TCP requires a connection → connection requires an IP
But you're using DHCP because you don't have an IP yet
→ Chicken and egg problem
→ DHCP uses UDP (connectionless — no IP needed)
```

### DHCP Ports

```
DHCP Server listens: UDP port 67
DHCP Client listens: UDP port 68
```

### Static vs Dynamic IP

| | Static IP | Dynamic IP (DHCP) |
|---|---|---|
| Set by | Admin manually | DHCP server automatically |
| Changes? | Never | Can change each connection |
| Used for | Servers, printers, routers | Laptops, phones, IoT |
| Tracking | Manual | DHCP lease table |

### Useful Commands

```cmd
ipconfig /all               # See DHCP server, lease obtained/expires
ipconfig /release           # Give up current IP
ipconfig /renew             # Trigger full DORA again
```

---

## 7. NAT — Network Address Translation

### Principal-Level Answer
> "NAT — Network Address Translation — is how multiple devices behind a router share a single public IP. The router maintains a mapping table of private IP:port to public IP:port, and rewrites the headers on the way out and back in. It's what makes home networking practical, and also why the internet didn't run out of IPv4 addresses as fast as people feared."

### Public IP vs Private IP

| | Public IP | Private IP |
|---|---|---|
| Who assigns it | ISP assigns to your router | Your router assigns to devices |
| Example | 103.21.58.147 | 192.168.1.101 |
| Visible on internet? | YES | NO |
| Unique globally? | YES | NO — every home has 192.168.1.x |

### Private IP Ranges (RFC 1918)

These are never routable on the internet:
```
10.0.0.0    – 10.255.255.255     (10.x.x.x)
172.16.0.0  – 172.31.255.255     (172.16.x.x to 172.31.x.x)
192.168.0.0 – 192.168.255.255    (192.168.x.x)  ← your home
```

Every home router in the world uses these same ranges — that's fine because they never leave the private network.

### How NAT Works

```
Your Laptop              Router                    Internet
192.168.1.101         103.21.58.147             google.com
      │                    │                         │
      │ "GET google.com"    │                         │
      │ Src: 192.168.1.101  │                         │
      │ ─────────────────→  │                         │
      │                     │ NAT rewrites Src IP     │
      │                     │ Src: 103.21.58.147      │
      │                     │ ──────────────────────→ │
      │                     │                         │
      │                     │ Response: Dst 103.21... │
      │                     │ ←────────────────────── │
      │                     │ NAT translates back     │
      │ Response arrives    │ Dst: 192.168.1.101      │
      │ ←─────────────────  │                         │
```

Router maintains a **NAT translation table**:
```
Private IP:Port       →   Public IP:Port
192.168.1.101:45230  →   103.21.58.147:45230
192.168.1.102:52100  →   103.21.58.147:52100
```

### Two Levels of DHCP at Home

```
LEVEL 1 — ISP to Router (Public IP)
  Airtel/Jio DHCP → assigns 103.21.58.147 to router's WAN port
  This IP is globally unique
  Websites see this IP when you visit them

LEVEL 2 — Router to Your Devices (Private IP)
  Router DHCP → assigns 192.168.1.101 to your laptop
  This IP exists only inside your home
  Internet has no idea this IP exists

NAT bridges the two worlds
```

### Check It on Your Laptop

```cmd
# Your private IP (what router gave you)
ipconfig
→ 192.168.1.101

# Your public IP (what internet sees)
# Open browser → whatismyip.com
→ 103.21.58.147   (completely different — that's NAT working)
```

---

## 8. Subnets

### Principal-Level Answer
> "Subnets are logical partitions of an IP network. A /24 gives you 256 addresses, 254 usable. You use subnets to isolate traffic, enforce security boundaries, and reduce broadcast domains. In a cloud context like AWS, the distinction between public and private subnets is architectural — public subnets have a route to an Internet Gateway, private ones route outbound through a NAT Gateway but aren't directly reachable from the internet."

### How Subnet Masks Work

```
IP:          192.168.1.50
Subnet mask: 255.255.255.0  (or /24)

/24 means: 24 bits = network part | 8 bits = host part
Network:   192.168.1.x  (fixed — same for everyone on this subnet)
Hosts:     192.168.1.1 to 192.168.1.254  (254 usable)
```

### Common Subnet Sizes

| CIDR | Subnet Mask | Usable Hosts | Use Case |
|---|---|---|---|
| /8 | 255.0.0.0 | 16,777,214 | Large enterprise |
| /16 | 255.255.0.0 | 65,534 | Medium network |
| /24 | 255.255.255.0 | 254 | Home / small office |
| /28 | 255.255.255.240 | 14 | Small AWS subnet |

### Why Subnets?

- **Isolate traffic** — broadcast stays within subnet
- **Security boundaries** — firewall rules between subnets
- **Reduce broadcast domains** — less noise on each segment
- **AWS architecture** — public subnets have IGW route; private subnets use NAT Gateway

---

## 9. IP Classes

### Principal-Level Answer
> "The classful addressing model divided the IPv4 space into ranges — Class A is 1 to 126, giving you a massive host space per network; Class B is 128 to 191; Class C is 192 to 223, which is what most home networks use — that's your 192.168.x.x. In practice, classful addressing is mostly historical — CIDR replaced it — but the private ranges from RFC 1918 still matter: 10.x.x.x, 172.16 through 172.31, and 192.168.x.x."

### IP Classes Table

| Class | Range | Default Mask | Hosts Per Network | Use |
|---|---|---|---|---|
| A | 1–126 | /8 | 16 million | Large orgs (10.x.x.x private) |
| B | 128–191 | /16 | 65,534 | Medium networks |
| C | 192–223 | /24 | 254 | Home/small office (192.168.x.x) |
| D | 224–239 | — | — | Multicast |
| E | 240–255 | — | — | Reserved/experimental |

> **Memory trick:** C = Cozy Home = 192.168.x.x — the one you'll always see.

### Private Ranges (RFC 1918)

```
Class A private: 10.0.0.0 – 10.255.255.255
Class B private: 172.16.0.0 – 172.31.255.255
Class C private: 192.168.0.0 – 192.168.255.255  ← your home
```

Not routable on the public internet.

---

## 10. Network Gateway — Public & Private

### Principal-Level Answer
> "A default gateway is just the router your device sends all non-local traffic to — it's the exit point for your network. NAT runs on it to translate private IPs to the public IP. In AWS, the Internet Gateway is the public gateway, and a NAT Gateway lets private subnets reach the internet without being directly reachable from it."

### Types of Gateways

| Type | What It Is |
|---|---|
| Default Gateway | Router your device sends all non-local traffic to (e.g. 192.168.1.1) |
| Private Gateway | Gateway inside your private network (LAN) |
| Public Gateway | Your ISP's edge router — first hop toward internet |
| AWS Internet Gateway (IGW) | Public — allows internet access for public subnets |
| AWS NAT Gateway | Private subnets can reach internet, not reachable from internet |

---

## 11. Switch vs Hub vs Router

### Principal-Level Answer
> "A hub is L1 — it just broadcasts everything to every port. Dumb, creates a lot of unnecessary traffic, basically obsolete. A switch is L2 — it learns MAC addresses and builds a table, so it only forwards frames to the correct port. Much more efficient. A router is L3 — it routes between different networks using IP addresses. The practical distinction: a switch connects devices within the same subnet; a router connects different networks together, like your LAN to the internet."

### Comparison Table

| Device | OSI Layer | How It Works | Uses |
|---|---|---|---|
| Hub | L1 (Physical) | Broadcasts to ALL ports | Obsolete |
| Switch | L2 (Data Link) | Forwards using MAC address table | Connect devices within same network |
| Router | L3 (Network) | Routes using IP addresses | Connect different networks (LAN ↔ Internet) |

### Switch vs Router — Key Distinction

```
Switch: connects devices WITHIN the same subnet
  Your laptop ↔ your printer ↔ your desktop (all 192.168.1.x)

Router: connects DIFFERENT networks
  Your LAN (192.168.1.x) ↔ Internet (public IPs)
```

---

## 12. MAC vs IP / Troubleshooting

### Principal-Level Answer
> "MAC is the hardware address — L2, used within a local segment. IP is L3, logical, assigned by DHCP or configured statically. ARP is the bridge — it maps an IP to a MAC so the switch knows where to send the frame on the local network."

### MAC vs IP

| | MAC Address | IP Address |
|---|---|---|
| Layer | L2 (Data Link) | L3 (Network) |
| Scope | Local network segment only | Global (public) or local (private) |
| Assigned by | Hardware manufacturer | DHCP server or static config |
| Changes? | No (permanent, can be spoofed) | Yes (DHCP, network changes) |
| Format | AA:BB:CC:DD:EE:FF | 192.168.1.101 |
| Analogy | Your name — never changes | Your address — changes when you move |

### ARP — The Bridge Between Them

```
ARP (Address Resolution Protocol):
"Who has IP 192.168.1.1? Tell me your MAC."
→ builds ARP table: IP ↔ MAC mapping
→ switch uses MAC to deliver the frame on local network
```

### Troubleshooting: Website Is Unavailable

> "I work bottom-up through the stack. First: can I reach anything at all? ping 8.8.8.8 — if that fails, it's a connectivity or routing issue. If that works, ping amazon.com — if that fails, it's DNS. Then traceroute to see where the path breaks. If I can reach the IP but the site's still down, curl -v to check HTTP/TLS errors. Then I look at the application layer — is the service down? Load balancer health check failures? Certificate expiry? Also check local: firewall rules, /etc/hosts hijacks, proxy config."

```
Step 1: ping 8.8.8.8
  ✓ works → internet connectivity is fine
  ✗ fails → local network or ISP issue

Step 2: ping amazon.com
  ✓ works → DNS is fine
  ✗ fails → DNS issue
        → try: nslookup amazon.com 8.8.8.8
        → try: ipconfig /flushdns

Step 3: traceroute amazon.com
  → shows every hop — find where it drops

Step 4: curl -v https://amazon.com
  → check HTTP status codes
  → check TLS errors
  → check certificate validity

Step 5: Check application layer
  → Is the service itself down? (check status.amazon.com)
  → Health check failures?
  → Certificate expired?

Step 6: Check local factors
  → Firewall blocking?
  → /etc/hosts hijack?
  → Proxy misconfiguration?
  → Try different network / device
```

**Key principle:** Always be systematic — eliminate one layer at a time, bottom-up. Never guess.

---

## 13. Wireless Router — How It Works at Home

### Principal-Level Answer
> "A home router is actually three devices in one box — a router, a switch, and a wireless access point. The router component handles L3 — it sits between your ISP and your LAN, runs NAT, and routes traffic. The switch component handles the wired ports on the back. The WAP handles WiFi and bridges wireless clients onto the LAN. Your ISP assigns a public IP to the router's WAN interface, and the router runs DHCP internally to hand out private addresses to your devices. All outbound traffic gets NATed to that public IP."

### Three Devices in One

```
Your Home Router = Router + Switch + Wireless Access Point

Router (L3)
  - Connects LAN to internet
  - Runs NAT
  - Routes traffic between networks

Switch (L2)
  - The 4 ethernet ports on the back
  - Connects wired devices on your LAN

Wireless Access Point
  - Broadcasts WiFi
  - Bridges wireless devices onto the LAN

Flow:
  Device → (WiFi/Ethernet) → Switch → Router → NAT → ISP → Internet
```

---

## 14. Website Unavailable — Troubleshooting Steps

*(Also covered in section 12 — reproduced here for completeness)*

### Principal-Level Answer
> "I structure it bottom-up through the OSI stack. Physical first — is the connection even up? Then IP connectivity, then DNS, then application layer. I don't guess — I eliminate."

```
1. Physical / Connectivity
   ping 8.8.8.8
   → Fails = no internet (check cable, WiFi, ISP)

2. DNS Resolution
   ping amazon.com
   nslookup amazon.com
   ipconfig /flushdns  (if cached stale record)

3. Routing
   traceroute amazon.com
   → Identify which hop drops packets

4. Application / TLS
   curl -v https://amazon.com
   → Check HTTP status (503? 404? 301?)
   → Check TLS errors (cert expired? chain issue?)

5. Server-Side
   → Status page
   → Health checks
   → Load balancer

6. Local
   → Firewall rules (Windows Defender, iptables)
   → /etc/hosts or C:\Windows\System32\drivers\etc\hosts
   → Proxy settings
   → Try different device / browser / network
```

---

## 15. HTTP vs HTTPS / HTTP-to-HTTPS Routing

### Principal-Level Answer
> "HTTP is plaintext — like sending a postcard anyone can read. HTTPS is HTTP over TLS — the data is encrypted in transit. For HTTP to HTTPS routing, the server listens on port 80 and responds with a 301 redirect to the HTTPS URL. But the smarter approach is HSTS — the server tells the browser 'never connect to me over HTTP again,' and the browser enforces that locally. This prevents downgrade attacks."

### HTTP vs HTTPS

| | HTTP | HTTPS |
|---|---|---|
| Port | 80 | 443 |
| Encrypted | No | Yes (TLS) |
| Certificate | Not required | Required |
| Analogy | Postcard — anyone can read | Sealed envelope — only sender/receiver |

### HTTP → HTTPS Redirect

```
1. Browser sends:
   GET http://amazon.com HTTP/1.1

2. Server responds:
   301 Moved Permanently
   Location: https://amazon.com

3. Browser follows redirect to HTTPS
```

### HSTS (HTTP Strict Transport Security)

```
Server sends header:
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload

Browser stores this:
"Never try HTTP for amazon.com — always use HTTPS directly"
"Enforce this for 1 year"

Result:
- No HTTP request ever leaves the browser
- Prevents SSL stripping / downgrade attacks
- Even if attacker intercepts, browser won't send HTTP
```

---

## 16. Full Forms & Memory Tricks

| Acronym | Full Form | Memory Trick |
|---|---|---|
| DHCP | Dynamic Host Configuration Protocol | **D**ad **H**elps **C**onfigure **P**hone |
| DORA | Discover, Offer, Request, Acknowledge | Dora the Explorer / Job interview analogy |
| DNS | Domain Name System | **D**ial by **N**ame **S**ystem |
| TCP | Transmission Control Protocol | **T**rust **C**omplete **P**ackets |
| UDP | User Datagram Protocol | **U** **D**on't **P**romise |
| NAT | Network Address Translation | **N**ame **A**ddress **T**wist |
| HTTP | HyperText Transfer Protocol | **H**ow **T**ext **T**ravels **P**ublicly |
| HTTPS | HyperText Transfer Protocol Secure | Same but **S**ealed |
| TLS | Transport Layer Security | **T**amper-proof **L**ocked **S**afe |
| SSL | Secure Sockets Layer | **S**ealed **S**ecret **L**etter (older, now TLS) |
| ARP | Address Resolution Protocol | Translates IP to MAC |
| SYN | Synchronize | Saying "hi" first |
| ACK | Acknowledge | Saying "got it" back |
| TTL | Time To Live | Expiry on cached DNS answers |
| HSTS | HTTP Strict Transport Security | Browser-enforced HTTPS-only |
| SAN | Subject Alternative Names | Domain list inside a certificate |
| OCSP | Online Certificate Status Protocol | Checks if cert is revoked |
| ECDHE | Elliptic Curve Diffie-Hellman Ephemeral | Key exchange in TLS — provides forward secrecy |

### DORA vs TCP — Don't Confuse Them

```
DORA          → getting an IP address (before you can talk to anyone)
               D → O → R → A  (4 steps)

TCP Handshake → making a connection (after you have an IP)
               SYN → SYN-ACK → ACK  (3 steps)

Order: DORA comes first. You need an IP before you can make a TCP connection.
```

---

## Quick Reference — One-Liners for Each Topic

| Topic | One-Liner |
|---|---|
| DNS | Hierarchical phone book — browser → OS cache → recursive resolver → Root → TLD → Authoritative → IP |
| TCP Handshake | SYN (client) → SYN-ACK (server) → ACK (client) → data flows |
| TCP vs UDP | TCP = reliable+ordered+slow; UDP = fast+unreliable+no handshake |
| TLS Chain | Walk from leaf cert → intermediate CA → root CA in trust store; verify signature at each step + domain match |
| DHCP | System that auto-assigns IP config to devices joining a network |
| DORA | 4-step conversation between DHCP client and server to get an IP |
| NAT | Router translates private IPs to one public IP — maintains port mapping table |
| Subnet | Logical partition of IP space — /24 = 254 hosts, /16 = 65K hosts |
| IP Classes | A=10.x (huge), B=172.16.x (medium), C=192.168.x (home) |
| Gateway | Exit door for your network — all non-local traffic goes here |
| Switch | L2 — forwards using MAC table (within same network) |
| Router | L3 — routes using IP (between different networks) |
| Hub | L1 — broadcasts to everyone — obsolete |
| MAC vs IP | MAC = hardware identity (permanent); IP = logical address (can change) |
| HTTPS | HTTP over TLS — encrypted; 301 redirect from HTTP + HSTS for enforcement |

---

*Document generated from AWS SysDE interview prep session — Aug 2026*
*Interview date: Aug 28, 2026*
