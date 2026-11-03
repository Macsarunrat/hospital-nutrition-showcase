# Security Architecture: Hardening the Docker Daemon with Docker Socket Proxy

When using modern edge reverse proxies like **Traefik**, **Træfik**, or **Nginx Proxy Manager**, the proxy needs real-time awareness of running containers. To achieve this, standard documentation frequently suggests mounting the host's Docker socket:

```yaml
# ⚠️ DANGEROUS PATTERN FREQUENTLY FOUND IN TUTORIALS
services:
  traefik:
    image: traefik:v2.11
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro  # CRITICAL SECURITY RISK
```

In this production deployment, we deliberately rejected this pattern and implemented **`tecnativa/docker-socket-proxy`**. Here is the engineering rationale and threat model analysis.

---

## 1. The Threat Model: Why Mounting `docker.sock` is Root Access

The Docker socket (`/var/run/docker.sock`) is the UNIX domain socket that the Docker CLI uses to communicate with the Docker Daemon.

Even if you mount it read-only (`:ro`), the socket allows arbitrary HTTP requests to the Docker Engine API. 

If an attacker finds a remote code execution (RCE) vulnerability in Traefik or an edge gateway:
1. **Container Escape via Volume Mount:** The attacker can instruct the daemon to create a new container mounting the host's root filesystem (`-v /:/host`).
2. **Arbitrary Container Termination:** The attacker can issue API calls to stop or kill security monitoring containers.
3. **Privilege Escalation:** By controlling the daemon, the attacker effectively possesses root capabilities across the entire physical host machine.

```text
[Internet] --> [Traefik (Compromised)] --> /var/run/docker.sock --> [Docker Daemon (Root on Host)]
```

---

## 2. The Solution: Sandboxed Socket Proxy

We deployed `tecnativa/docker-socket-proxy` as an intermediary security gateway on the private `traefik-internal` network.

```text
[Internet] 
    │
    ▼
[Traefik] ──(TCP: 2375)──> [docker-socket-proxy] ──(UNIX Socket)──> [Docker Daemon]
                             ▲
                             │ (Enforces Strict Whitelist)
                             ├── ALLOW: GET /containers, GET /events, GET /version, GET /_ping
                             └── DENY:  ALL POST, PUT, DELETE, EXEC, VOLUMES, NETWORKS
```

### Production Configuration:
In our `docker-compose.prod.yml`:

```yaml
  socket-proxy:
    image: tecnativa/docker-socket-proxy
    container_name: socket-proxy
    restart: unless-stopped
    environment:
      # Explicit read-only allowances for Traefik container discovery
      - CONTAINERS=1
      - EVENTS=1
      - PING=1
      - VERSION=1
      # Explicitly keep all mutative and execution endpoints disabled
      - POST=0
      - DELETE=0
      - EXEC=0
      - VOLUMES=0
      - NETWORKS=0
      - SECRETS=0
      - SWARM=0
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - traefik-internal
```

And instructed Traefik to connect exclusively to the proxy endpoint over TCP:

```yaml
  traefik:
    image: traefik:v2.11
    command:
      - "--providers.docker=true"
      - "--providers.docker.endpoint=tcp://socket-proxy:2375"
      - "--providers.docker.exposedbydefault=false"
```

---

## 3. Defense-in-Depth Comparison

| Security Capability | Standard Tutorial Setup (`/var/run/docker.sock:ro`) | Our Hardened Setup (`docker-socket-proxy`) |
| :--- | :--- | :--- |
| **API Endpoints Accessible** | All Docker API endpoints (100% of daemon surface) | Only 4 read endpoints (`/containers`, `/events`, `/_ping`, `/version`) |
| **Can Container Spawn New Containers?** | Yes (`POST /containers/create`) | **Blocked** (Returns 403 Forbidden) |
| **Can Container Execute Host Commands?** | Yes (`POST /containers/{id}/exec`) | **Blocked** (Returns 403 Forbidden) |
| **Can Container Alter Docker Networks?** | Yes (`POST /networks/create`) | **Blocked** (Returns 403 Forbidden) |
| **Direct Socket Exposure** | Exposed into the public-facing Traefik container | Isolated in an unprivileged proxy container |

---

## 4. Verification

To verify that the proxy restricts destructive actions while allowing discovery:

```bash
# 1. Test ping (Allowed)
curl http://socket-proxy:2375/_ping
# Output: OK

# 2. Test listing containers (Allowed)
curl http://socket-proxy:2375/containers/json
# Output: 200 OK with container list

# 3. Test creating a rogue container (Blocked)
curl -X POST http://socket-proxy:2375/containers/create
# Output: 403 Forbidden
```

By decoupling Traefik from the physical UNIX socket, we follow the **Principle of Least Privilege (PoLP)** and protect the clinical host server from lateral movement attacks.
