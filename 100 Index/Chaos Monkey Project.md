## 🏗️ Architecture (Asynchronous Microservices)
The environment simulates a robust, decoupled architecture running entirely within Docker.

```text
[ Client (curl) ] ---> [ Nginx (Reverse Proxy/Load Balancer) ]
                               |
                               |---> [ FastAPI Backend A ] --(Read/Write)--> [ Redis Cache/Broker ]
                               |---> [ FastAPI Backend B ] --(Read/Write)--> [ Redis Cache/Broker ]
                                                                                   |
                                                                                   |--(Tasks)
                                                                                   v
                                                           [ Celery Worker A ] ---> [ PostgreSQL DB ]
                                                           [ Celery Worker B ] ---> [ PostgreSQL DB ]
```

## 🛠️ Technology Stack
* **Infrastructure:** Docker, Docker Compose (with strict Memory Cgroups)
* **Networking & Proxy:** Nginx, `iptables`
* **Application Layer:** Python 3.11+, FastAPI
* **Asynchronous Processing:** Celery
* **In-Memory Store / Message Broker:** Redis
* **Relational Database:** PostgreSQL
* **Automation & Chaos Injection:** Advanced Bash Scripting

## 🐒 The Chaos Monkey Script (`chaos_monkey.sh`)
This utility injects targeted faults into the running containers without completely destroying the Docker network, forcing the engineer to isolate the specific failing component.

```bash
#!/bin/bash

# Define arrays and variables for the target containers
APP_CONTAINERS=("api_backend_a" "api_backend_b")
WORKER_CONTAINERS=("celery_worker_a" "celery_worker_b")
DB_CONTAINER="postgres_primary"
CACHE_CONTAINER="redis_cache"
NGINX_CONTAINER="nginx_proxy"

# Scenario 0: Simulate a network partition on the database
function trigger_db_blackhole() {
    echo "Initiating DB network drop..." > /dev/null
    docker exec -u 0 "$DB_CONTAINER" iptables -A INPUT -p tcp --dport 5432 -j DROP
}

# Scenario 1: Simulate a Cache Stampede / Eviction Storm
function trigger_cache_stampede() {
    echo "Flushing Redis cache to trigger database stampede..." > /dev/null
    docker exec "$CACHE_CONTAINER" redis-cli FLUSHALL
}

# Scenario 2: Simulate a Poison Pill / Worker Crash
function trigger_worker_crash() {
    echo "Injecting poison pill into worker queue..." > /dev/null
    RANDOM_INDEX=$((RANDOM % 2))
    TARGET_WORKER="${WORKER_CONTAINERS[$RANDOM_INDEX]}"
    docker kill "$TARGET_WORKER"
}

# Scenario 3: Simulate a corrupted reverse proxy configuration
function trigger_nginx_502() {
    echo "Corrupting Nginx configuration..." > /dev/null
    docker exec "$NGINX_CONTAINER" sed -i 's/proxy_pass http:\/\/backend_pool;/proxy_pass http:\/\/127.0.0.1:9999;/g' /etc/nginx/nginx.conf
    docker exec "$NGINX_CONTAINER" nginx -s reload
}

# Execute a random disaster scenario
SCENARIO=$((RANDOM % 4))

case $SCENARIO in
    0) trigger_db_blackhole ;;
    1) trigger_cache_stampede ;;
    2) trigger_worker_crash ;;
    3) trigger_nginx_502 ;;
esac

echo "Chaos Monkey has struck. The system is compromised. Good luck."
```

## 🚨 Attack Vectors & Troubleshooting Runbooks

### 1. The Database Blackhole (Scenario 0)
* **Symptom:** API endpoints relying on synchronous DB reads hang, or background workers fail to persist data.
* **Investigation:** Check worker logs (`docker logs celery_worker_a`). Observe `Timeout` or `Connection Refused` errors targeting the DB hostname.
* **Resolution:** Access the Postgres container as root (`docker exec -it -u 0 postgres_primary bash`), verify dropped packets using `iptables -L`, and flush the rules using `iptables -F`.

### 2. The Cache Stampede (Scenario 1)
* **Symptom:** Sudden, extreme CPU spike on the PostgreSQL container. API latency increases drastically (from milliseconds to seconds).
* **Investigation:** Run `docker stats` to observe resource exhaustion on `postgres_primary`. Inspect Redis metrics (`docker exec redis_cache redis-cli INFO memory`) to see an empty dataset.
* **Resolution:** Implement rate limiting at the Nginx level temporarily to allow the cache to rebuild, or restart the API containers to reset connection pools. 

### 3. The Worker Crash / Queue Buildup (Scenario 2)
* **Symptom:** APIs return 200 OK (tasks accepted), but background processing halts.
* **Investigation:** Check the queue length in Redis (`docker exec redis_cache redis-cli LLEN celery`). Observe the queue size growing indefinitely. Run `docker ps` to find a worker container in an `Exited` state.
* **Resolution:** Restart the dead worker (`docker start celery_worker_<x>`). Check Docker daemon logs to verify if the container was killed by the kernel due to an Out of Memory (OOM) event.

### 4. The Proxy Corruption (Scenario 3)
* **Symptom:** Immediate `502 Bad Gateway` across all endpoints.
* **Investigation:** Check Nginx logs (`docker logs nginx_proxy`). Observe `connect() failed (111: Connection refused)`.
* **Resolution:** Exec into the proxy, correct the `proxy_pass` directive in `/etc/nginx/nginx.conf`, and execute `nginx -t` followed by `nginx -s reload`.

## 📄 Sample Post-Mortem Incident Report (v2.0)

**Incident ID:** INC-20260422-02
**Date:** April 22, 2026
**Severity:** High (Asynchronous Processing Outage / Data Delay)

**1. Executive Summary:**
At approximately 16:30 UTC, background processing of telemetry data halted completely, though the main API continued to accept requests. The issue was identified as a crashed Celery worker node caused by an injected poison pill, leading to a massive queue buildup in Redis. Service was restored at 16:45 UTC.

**2. Timeline (UTC):**
* **16:30** - Custom Prometheus metrics alerted on `celery_queue_length` exceeding 500 pending tasks.
* **16:32** - L3 Engineer initiated investigation. Verified API was returning 200 OK via `curl`.
* **16:35** - Executed `docker ps` and noted `celery_worker_b` had exited with code 137 (SIGKILL).
* **16:38** - Checked Redis queue size (`redis-cli LLEN celery`). Confirmed 1,200 pending tasks.
* **16:40** - Reviewed `dmesg` logs on the host to confirm the container was killed due to memory limits (simulated poison pill).
* **16:42** - Restarted the worker node (`docker start celery_worker_b`).
* **16:45** - Monitored the queue draining successfully. Incident resolved.

**3. Root Cause:**
The `chaos_monkey.sh` script (Scenario 2) was executed, simulating a critical worker crash. The remaining worker (`worker_a`) could not handle the total throughput alone, causing the Redis broker queue to back up.

**4. Action Items (Preventative Measures):**
* Implement a Dead Letter Queue (DLQ) in Celery to route failing/poisonous tasks away from the main processing pipeline.
* Configure auto-scaling rules (or Docker restart policies like `restart: unless-stopped`) to automatically recover exited worker containers without manual intervention.

## ⚙️ Deployment Strategy (Homelab / Bare-Metal)
This stack is optimized to run alongside other tools on constrained hardware.

1.  **Resource Constraints:** The `docker-compose.yml` explicitly defines `deploy.resources.limits` to prevent Redis or PostgreSQL from consuming the host's entire memory pool.
2.  **Environment Isolation:** To switch contexts between monitoring (`f1-dataops` observability) and simulation, bring down the entire stack cleanly:
    ```bash
    docker compose -f docker-compose.chaos.yml down
    docker compose -f docker-compose.observability.yml up -d
    ```