# local port forwarding

`tunnel.sh` is a helper script that establishes an **SSH tunnel** from WSL to a remote host.  
It forwards multiple remote services (databases, web UIs, brokers, etc.) to `localhost` so that tools like **IntelliJ** (running on Windows) can connect as if the services were local.  


<img width="1035" height="634" alt="image" src="https://github.com/user-attachments/assets/e5ee20af-2aab-4279-8897-fcfa2a2e38cd" />


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
---

## 🚀 Usage

Run the following steps in order:

```bash
# 1. Make the script executable (first time only)
chmod +x tunnel.sh

# 2. Start the tunnel (replace with your host/IP)
./tunnel.sh my.remote.host

# You’ll see:
# Tunnel successfully established to my.remote.host

# 3. Stop the tunnel when done
./tunnel.sh -d
```
---

## 🔧 Ports Forwarded

| Local Port | Remote Port | Service / Purpose |
|------------|-------------|-------------------|
| 12350      | 12350       | Custom service (marker) |
| 5432       | 5432        | PostgreSQL |
| 9200       | 9200        | Elasticsearch |
| 61616      | 61616       | ActiveMQ |
| 8443       | 8443        | HTTPS service |
| 9090       | 9090        | Monitoring |
| 8006–8500  | same        | App/dev ports, Service UIs / tools |
| 9162       | 9162         | SNMP (trap testing) |

👉 All forwards bind to **localhost only**, so they’re safe and only accessible to your machine.  

---

