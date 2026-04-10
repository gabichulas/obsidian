# The Chaos Lab: SRE & L3 Incident Response Simulator

## 📌 Overview
The Chaos Lab is a localized Chaos Engineering environment designed to simulate real-world production outages. It was built to practice systematic L3 incident response, container troubleshooting, and bash-based automation. Instead of manually breaking components, an automated "Chaos Monkey" script introduces random, silent failures into a Dockerized microservices stack, requiring active debugging using standard Linux and Docker observability tools.

## 🏗️ Architecture

The environment simulates a standard three-tier architecture running entirely within Docker.

    [ Client (curl) ] ---> [ Nginx (Reverse Proxy) ]
                                   |
                                   |---> [ FastAPI Backend A ] ---> [ PostgreSQL DB ]
                                   |---> [ FastAPI Backend B ] ---> [ PostgreSQL DB ]

* **Reverse Proxy:** Nginx, load-balancing traffic across the backend APIs.
* **Backend Services:** Two identical Python/FastAPI containers.
* **Database:** A single PostgreSQL container.

## 🛠️ Technology Stack
* **Infrastructure:** Docker, Docker Compose
* **Networking & Proxy:** Nginx, iptables
* **Backend & DB:** Python (FastAPI), PostgreSQL
* **Automation:** Advanced Bash Scripting

## 🐒 The Chaos Monkey Script (`chaos_monkey.sh`)
This Bash utility is the core of the project. It uses `docker exec` and native Linux utilities to inject faults into the running containers without taking down the entire environment.

```bash
#!/bin/bash

# Define arrays and variables for the target containers
APP_CONTAINERS=("api_backend_a" "api_backend_b")
DB_CONTAINER="postgres_primary"
NGINX_CONTAINER="nginx_proxy"

# Scenario 0: Simulate a network partition on the database
function trigger_db_blackhole() {
    echo "Initiating DB network drop..." > /dev/null
    # Execute iptables inside the database container to drop incoming connections
    docker exec -u 0 "$DB_CONTAINER" iptables -A INPUT -p tcp --dport 5432 -j DROP
}

# Scenario 1: Simulate a memory leak leading to a hard crash
function trigger_app_crash() {
    # Randomly select one of the backend containers to kill
    RANDOM_INDEX=$((RANDOM % 2))
    TARGET_APP="${APP_CONTAINERS[$RANDOM_INDEX]}"
    
    echo "Killing app container: $TARGET_APP..." > /dev/null
    docker kill "$TARGET_APP"
}

# Scenario 2: Simulate a corrupted reverse proxy configuration
function trigger_nginx_502() {
    echo "Corrupting Nginx configuration..." > /dev/null
    # Overwrite the nginx.conf inside the container with an invalid upstream port
    docker exec "$NGINX_CONTAINER" sed -i 's/proxy_pass http:\/\/backend_pool;/proxy_pass http:\/\/127.0.0.1:9999;/g' /etc/nginx/nginx.conf
    docker exec "$NGINX_CONTAINER" nginx -s reload
}

# Execute a random disaster scenario
SCENARIO=$((RANDOM % 3))

case $SCENARIO in
    0) trigger_db_blackhole ;;
    1) trigger_app_crash ;;
    2) trigger_nginx_502 ;;
esac

echo "Chaos Monkey has struck. The system is compromised. Good luck."
```

## 🚨 Attack Vectors & Troubleshooting Runbooks

### 1. The Database Blackhole (Scenario 0)
* **Symptom:** API endpoints return `500 Internal Server Error` or hang indefinitely.
* **Investigation:** Check API container logs (`docker logs api_backend_a`). Observe `Connection Refused` errors targeting the DB hostname. Ping the DB container from the API container to verify network reachability. 
* **Resolution:** Access the Postgres container as root (`docker exec -it -u 0 postgres_primary bash`), verify dropped packets using `iptables -L`, and flush the rules using `iptables -F`.

### 2. The Application Crash (Scenario 1)
* **Symptom:** Occasional missing responses or slightly degraded performance if load balancer health checks are slow to catch the dead node.
* **Investigation:** Run `docker ps` to notice one API container has exited. Inspect the exit code using `docker inspect <container_id>`.
* **Resolution:** Restart the dead container using `docker start api_backend_<x>`. Investigate Docker daemon logs for OOM (Out of Memory) kills if this were a real memory leak.

### 3. The Proxy Corruption (Scenario 2)
* **Symptom:** All API requests immediately return `502 Bad Gateway`.
* **Investigation:** Check Nginx logs (`docker logs nginx_proxy`). Observe errors stating `connect() failed (111: Connection refused) while connecting to upstream`. Exec into the Nginx container and review `/etc/nginx/nginx.conf`.
* **Resolution:** Correct the `proxy_pass` directive in the Nginx configuration and perform a zero-downtime reload using `nginx -s reload`.

---

## 📄 Sample Post-Mortem Incident Report

**Incident ID:** INC-20260410-01
**Date:** April 10, 2026
**Severity:** High (Complete API Outage)

**1. Executive Summary:**
At approximately 14:00 UTC, the primary API gateway began returning `502 Bad Gateway` errors for 100% of incoming traffic. The issue was identified as a corrupted Nginx configuration file that routed traffic to a dead port. Service was restored at 14:15 UTC.

**2. Timeline (UTC):**
* **14:00** - Uptime monitoring alerted on 5xx errors from the main endpoint.
* **14:02** - L3 Engineer began investigation. Initial `curl` verified total failure.
* **14:05** - Checked `docker ps`; all containers were running (Nginx, API A/B, Postgres).
* **14:07** - Checked `docker logs nginx_proxy`. Found recurring `Connection refused` to upstream `127.0.0.1:9999`.
* **14:10** - Exec'd into the proxy container and reviewed `/etc/nginx/nginx.conf`. Found corrupted `proxy_pass` directive.
* **14:13** - Applied hotfix using `sed` to restore the correct upstream pool.
* **14:15** - Ran `nginx -s reload`. Health checks turned green. Incident resolved.

**3. Root Cause:**
The `chaos_monkey.sh` script (Scenario 2) was executed, simulating a bad configuration deployment. It overwrote the upstream block and performed an automated reload.

**4. Action Items (Preventative Measures):**
* Implement syntax checking (`nginx -t`) before any automated reload script is allowed to execute.
* Lock down file permissions on `/etc/nginx/nginx.conf` to prevent unauthorized modification by non-root users.