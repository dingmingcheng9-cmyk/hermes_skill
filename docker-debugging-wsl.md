---
name: docker-debugging-wsl
description: Debug Docker containers in WSL2 — find hidden containers via iptables DNAT, fix .pyc cache staleness with hot-restart, and isolate cloud LB routing issues.
tags: [docker, wsl, debugging, networking, iptables]
---

# Docker Debugging on WSL2

Common debugging scenarios when running Docker inside WSL2 Ubuntu.

## Find Hidden Containers (not shown by `docker ps`)

When Docker containers are proxied through host networking or port forwarding, `docker ps` may not show all active containers. Use iptables to discover them:

```bash
sudo iptables -t nat -L DOCKER
```

The output shows DNAT rules with container IPs:

```
DNAT       tcp  --  anywhere  anywhere  tcp dpt:8080 to:172.17.0.2:80
DNAT       tcp  --  anywhere  anywhere  tcp dpt:8443 to:172.17.0.3:443
```

The `to:` address after the port mapping reveals the container's internal IP.

### Verify a hidden container

```bash
# Directly query container by IP
docker inspect <container-name> | grep -i ipaddress

# Check all containers (including stopped)
docker ps -a

# Network-level check
docker network inspect bridge
```

## Fix Stale .pyc Cache After Code Changes

When you modify Python code in a running container but changes don't take effect, stale `.pyc` cache is the most common cause.

### Hotfix sequence

```bash
# 1. Copy the updated file into the container
docker cp /path/to/new/file.py <container>:/app/file.py

# 2. Clear Python bytecode cache inside container
docker exec <container> find /app -name '__pycache__' -type d -exec rm -rf {} + 2>/dev/null
# OR more targeted:
docker exec <container> rm -rf /app/__pycache__

# 3. Restart the container (not just the process)
docker restart <container>
```

### Why this happens
- Docker layer caching means `docker cp` overwrites the file but Python's `.pyc` cache still has the old bytecode
- `docker restart` triggers a fresh Python process that recompiles from the updated `.py` files
- Simply touching the file or sending SIGHUP is NOT enough — the `.pyc` files must be cleared

## Isolate Cloud Load Balancer (CLB) Routing Issues

When a service works inside the container but fails from outside, cloud LBs can intercept and route to different backends.

### Diagnosis approach

| Test Location | What it reveals |
|---|---|
| Inside container (`curl localhost:PORT`) | App process health |
| On WSL host (`curl localhost:MAPPED_PORT`) | Docker port mapping + container networking |
| On Windows host (`curl localhost:EXTERNAL_PORT`) | WSL port forwarding + Windows firewall |
| External (public IP/domain) | Cloud LB routing + security group |

### Testing commands

```bash
# Inside container
docker exec <container> curl -sI http://localhost:8080/health

# On WSL host
curl -sI http://localhost:8080/health

# On Windows (from cmd)
curl -sI http://localhost:8080/health

# External (from WSL or anywhere)
curl -sI https://your-domain.com/health
```

### Common CLB issues

1. **Sticky sessions**: LB may route to a different container replica than expected
2. **Health check failing**: LB takes the backend out of rotation even if the app works locally
3. **TLS termination**: LB decrypts at edge, so internal traffic is HTTP while external is HTTPS
4. **Port remapping**: External port 443 may map to internal port 8080

## Pitfalls

- `iptables -t nat -L DOCKER` requires `sudo` — don't forget
- After `docker cp`, always clear `__pycache__` before restart, or Python will use stale bytecode
- Cloud LBs (ALB/CLB/NLB) have their own health check paths and intervals — a 502 from outside doesn't mean the app is down
