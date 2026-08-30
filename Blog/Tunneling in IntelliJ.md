# Local Port Forwarding — `tunnel.sh`

`tunnel.sh` is a helper script that establishes an **SSH tunnel** from WSL to a remote host.  
It forwards multiple remote services (databases, web UIs, brokers, etc.) to `localhost` so that tools like **IntelliJ** (running on Windows) can connect as if the services were local.

<img width="1280" height="800" alt="Gemini_Generated_Image_b8rdeub8rdeub8rd" src="https://github.com/user-attachments/assets/0a9baa15-2211-4306-8505-803045bc9b33" />

---

## 🧠 How SSH Tunneling Works

### The Core Idea

When Postgres (or any service) runs on a remote machine, it only listens on `localhost` of that machine — it is **not reachable from outside**. SSH tunneling solves this by creating an encrypted pipe between your local port and the remote port.

```
Your WSL (192.168.1.10)          Remote Linux (192.168.1.100)
┌────────────────────────┐        ┌──────────────────────────┐
│                        │        │                          │
│  IntelliJ / psql       │        │  Postgres                │
│         ↓              │        │  listening on            │
│   localhost:8432       │═══════▶│  localhost:5432          │
│                        │  SSH   │                          │
│  port 8432 opens HERE  │  pipe  │  never exposed outside   │
│  only while tunnel     │        │  this machine            │
│  is alive              │        │                          │
└────────────────────────┘        └──────────────────────────┘
```

### What `-L 8432:localhost:5432` Means

Read it left to right — three parts separated by colons:

```
-L   8432      :    localhost    :    5432
     ↑               ↑               ↑
  LOCAL PORT     WHERE ON REMOTE   REMOTE PORT
  Opens on       (remote talks      Postgres on
  YOUR WSL       to itself)         remote machine
```

> **Rule:** Traffic you send to `localhost:8432` on YOUR machine travels over SSH and arrives at `localhost:5432` on the REMOTE machine.

### What `-fN` Means

| Flag | What It Does | Without It |
|------|-------------|------------|
| `-f` | Fork to background — terminal stays free | Terminal locks, Ctrl+C kills tunnel |
| `-N` | No remote shell — just hold the pipe open | SSH opens a bash shell on remote, then dies taking the tunnel with it |

```
ssh -f  (no -N):   connects → opens remote shell → backgrounds it
                   → shell has nothing to do → exits in seconds
                   → tunnel DIES ❌

ssh -fN:           connects → -N says "no shell, just forward the port"
                   → process stays alive, only job is forwarding
                   → tunnel LIVES ✅
```

### Real Console Demo

```bash
# STEP 1 — Without tunnel, connection fails
mj@wsl:~$ psql -h 192.168.1.100 -p 5432 -U postgres
psql: error: Connection refused   ← Postgres not exposed to outside ❌

# STEP 2 — Start the tunnel
mj@wsl:~$ ssh -fN -L 8432:localhost:5432 root@192.168.1.100
mj@wsl:~$   ← prompt returns immediately, tunnel alive in background

# STEP 3 — Verify tunnel is alive
mj@wsl:~$ ss -tlnp | grep 8432
LISTEN  127.0.0.1:8432   ← YOUR machine now has this port ✅

# STEP 4 — Connect through tunnel
mj@wsl:~$ psql -h localhost -p 8432 -U postgres
psql (14.5)
postgres=# select inet_server_addr();
 127.0.0.1    ← this is REMOTE's localhost, not yours ✅

# STEP 5 — Kill tunnel
mj@wsl:~$ kill $(pgrep -f "8432:localhost:5432")
mj@wsl:~$ psql -h localhost -p 8432 -U postgres
psql: error: Connection refused   ← tunnel gone, port gone ❌
```

---

## Code

```sh
#!/bin/bash
# Kill any existing tunnel on port 12350
pid=$(pgrep -f "12350:localhost:12350")
if [[ -n $pid ]]; then
    kill -9 "$pid"
    echo "Stopped existing tunnel (pid: $pid)"
fi
# Handle disable flag
if [[ "$1" == "-d" ]]; then
    echo "Tunneling is disabled"
    exit 0
fi
if [[ -z "$1" ]]; then
    echo "Usage: $0 [hostname] | -d"
    exit 1
fi
# SSH tunnel setup
ssh -fN \
  -o ExitOnForwardFailure=yes \
  -o ServerAliveInterval=30 \
  -o ServerAliveCountMax=3 \
  \
  # --- App / Dev ports ---
  -L 12350:localhost:12350 -L 9998:localhost:9998 -L 8006:localhost:8006 \
  -L 8010:localhost:8010 -L 8012:localhost:8012 -L 8013:localhost:8013 \
  -L 8014:localhost:8014 -L 8016:localhost:8016 -L 8020:localhost:8020 \
  \
  # --- Middleware / DB / Search ---
  -L 8050:localhost:8050 -L 8161:localhost:8161 -L 8432:localhost:5432 \
  -L 8500:localhost:8500 -L 9200:localhost:9200 -L 61616:localhost:61616 \
  \
  # --- HTTPS / Monitoring ---
  -L 8443:localhost:8443 -L 9090:localhost:9090 -L 9162:localhost:162 \
  root@"$1"
# Check result of ssh command
if [[ $? -eq 0 ]]; then
    echo "Tunnel successfully established to $1"
else
    echo "Failed to establish tunnel to $1"
    exit 1
fi
```

### Script Options Explained

| Option | What It Does |
|--------|-------------|
| `ExitOnForwardFailure=yes` | Fail immediately if any port is already bound — no silent half-tunnels |
| `ServerAliveInterval=30` | Send keepalive ping every 30 seconds to detect dead connections |
| `ServerAliveCountMax=3` | After 3 missed pings (90s), declare connection dead and exit |
| `pgrep -f / kill -9` | Kills any stale tunnel before starting a new one |

---

## 🚀 Usage

```bash
# 1. Make the script executable (first time only)
chmod +x tunnel.sh

# 2. Start the tunnel (replace with your host/IP)
./tunnel.sh my.remote.host
# Tunnel successfully established to my.remote.host

# 3. Stop the tunnel when done
./tunnel.sh -d
# Tunneling is disabled
```

---

## 🔧 Ports Forwarded

| Local Port | Remote Port | Service / Purpose |
|------------|-------------|-------------------|
| 12350 | 12350 | Custom service (marker / tunnel health check) |
| 8432 | 5432 | PostgreSQL (local 8432 → remote 5432) |
| 9200 | 9200 | Elasticsearch |
| 61616 | 61616 | ActiveMQ broker |
| 8161 | 8161 | ActiveMQ web UI |
| 8443 | 8443 | HTTPS service |
| 9090 | 9090 | Prometheus / Monitoring |
| 8500 | 8500 | Consul UI |
| 9162 | 162 | SNMP trap testing |
| 8006–8020 | same | App / dev microservice ports |

> 👉 All forwards bind to **localhost only** — safe, not exposed to your local network.

---

## 🎯 Amazon SysDE Interview — Will This Be Asked?

**Yes — directly and indirectly.** Here is how it maps to their topics:

### Concept Map

| tunnel.sh concept | Interview topic |
|-------------------|----------------|
| `-L local:remote` | Network proxying, secure service exposure |
| 18 ports over one SSH session | Multiplexing — one TCP connection carries many streams |
| `ServerAliveInterval` keepalive | Heartbeating, connection health in distributed systems |
| `ExitOnForwardFailure` | Fail-fast design principle |
| `pgrep/kill` before restart | Process lifecycle, idempotent operations |

### Questions You Should Expect

**"How would you securely expose an internal service without opening a firewall port?"**
> SSH tunnel for dev access. For production, AWS recommends SSM Session Manager — same concept but no port 22 needed.

**"What is the difference between forward and reverse SSH tunneling?"**
> `-L` (forward): you pull remote service to local. `-R` (reverse): remote machine pushes a port to your machine — used for NAT traversal when remote can't accept inbound connections.

**"What is the AWS-native equivalent of this script?"**
> AWS SSM port forwarding — no open SSH port required:
> ```bash
> aws ssm start-session \
>   --target i-0abc1234 \
>   --document-name AWS-StartPortForwardingSession \
>   --parameters '{"portNumber":["5432"],"localPortNumber":["8432"]}'
> ```

**"How does multiplexing work here?"**
> All 18 `-L` tunnels share a single TCP connection to the remote. SSH multiplexes them internally — same concept as HTTP/2 streams or gRPC channels over one connection.

### Key Points to Memorize

```
-L  LOCAL_PORT : remote_host : REMOTE_PORT
    "open port on MY machine → forward to REMOTE's port"

-f  = background (non-blocking)
-N  = no shell (tunnel only, stays alive)

Without -N → SSH opens remote shell → no command → exits → tunnel dies
With    -N → SSH holds connection for port forwarding only → tunnel lives
```
