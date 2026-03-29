
This document serves as a comprehensive reference guide for troubleshooting distributed systems, backend infrastructure, and API reliability. 

---

## 1. Observability & Distributed Systems (The LGTM Stack)

Modern microservice architectures require strict observability to trace requests as they travel from the frontend through various backend services.

### Core Components
* **Grafana:** The central visualization dashboard for infrastructure metrics (CPU, Memory, networking) and log aggregation.
* **Tempo (Distributed Tracing):** Tracks the lifecycle of a single request across multiple microservices. 
    * **Trace ID:** A unique alphanumeric "passport" assigned to a request (via OpenTelemetry).
    * **Spans & Waterfall Charts:** Visual representations of how long a request spent in each service (e.g., 0.5s in Django, 1.5s in the Flask AI worker).
* **Loki (Log Aggregation):** A highly efficient log searching system using **LogQL**.
    * **The Funnel Method:** Always start with a Stream Selector (the bucket) and pipe it into a Line Filter (the sieve). 
    * *Example:* `{service="flask-ai-microservice"} |= "c9e3f55e12deca6890fc22116ca64f79"`

### The SRE Troubleshooting Workflow
1.  **User Report:** Identify the user and the timestamp of the failure.
2.  **Loki Search:** Query the logs using the user ID to locate the exact error and extract the `Trace ID`.
3.  **Tempo Visualization:** Input the `Trace ID` to view the waterfall chart and pinpoint exactly *which* microservice failed.
4.  **Action:** Escalate to developers with exact logs, or apply infrastructure fixes.

---

## 2. API Reliability & Architecture

### Idempotency (Preventing Race Conditions)
A core REST API concept where making multiple identical requests has the exact same effect as making a single request. This prevents duplicate data (e.g., a user double-clicking a "Submit" button on slow Wi-Fi).

* **The Mechanism:** The frontend application generates a unique `UUID` *when the form loads* (not when the button is clicked) and sends it in an `Idempotency-Key` HTTP header.
* **The Backend Logic:** 1. Check an in-memory cache (like Redis) for the key.
    2. If the key exists, return the cached success response immediately.
    3. If the key does not exist, process the transaction, save to the primary database, and cache the response.

```python
# Example: FastAPI Idempotency Check
@app.post("/api/v1/notes")
async def create_note(note_data: dict, idempotency_key: str = Header(...)):
    if idempotency_key in redis_cache:
        return redis_cache[idempotency_key] # Block duplicate
    
    saved_record = save_to_database(note_data)
    redis_cache[idempotency_key] = saved_record
    return saved_record
```

### Pagination (Preventing 502 Bad Gateway Timeouts)
Attempting to fetch massive datasets (e.g., 50,000 rows) in a single API call overloads server memory and causes reverse proxies (like Cloudflare or Nginx) to drop the connection due to timeouts.
* **The Fix:** APIs must enforce pagination. The client script must request data in batches using query parameters such as `?limit=100&offset=0`.

---

## 3. Infrastructure & Containerization

### Docker & Version Pinning
* **The `:latest` Trap:** Never use the `latest` tag in `docker-compose.yml` or production deployments. It pulls the bleeding-edge version, which can introduce breaking configuration changes.
* **Version Pinning:** Always lock infrastructure to a specific, stable release (e.g., `image: grafana/tempo:2.3.0`).

### Memory Leaks (The Grafana Staircase)
A backend bug where an application allocates RAM to process a task but fails to release it back to the system (bypassing Garbage Collection).
* **The Symptoms:** In Grafana, the container's memory utilization graph looks like a staircase. It steadily climbs to 100%, triggers an **OOM (Out of Memory)** crash, restarts, drops to 0%, and repeats the cycle.

### Top-Down Troubleshooting Method
When facing a sudden spike in errors or latency during peak traffic:
1.  **Check Resource Saturation:** Look at CPU and Memory for the entry-point server (e.g., Django). Is it simply overloaded?
2.  **Check Database Health:** Are there locked tables, blocked queries, or exhausted connection pools?
3.  **Check Worker Queues:** Are background workers (e.g., Flask/Celery) starving or timing out?

---

## 4. HTTP Status Codes Playbook

* **400 Bad Request:** The client (frontend) sent a malformed payload or is missing required fields. *Action: Check frontend code or API parameters.*
* **403 Forbidden:** The request was blocked by the server or a Web Application Firewall (WAF) due to rate limiting or invalid credentials.
* **500 Internal Server Error:** The backend code crashed. Common causes include unhandled exceptions, null pointer references, or character encoding failures (e.g., failing to process UTF-8 special characters like apostrophes in names).
* **502 Bad Gateway:** The proxy server (Nginx/Cloudflare) received an invalid response from the upstream server, or the upstream server took too long to respond (timeout).

---

## 5. Bug Escalation & Jira Reporting

A Senior Support Engineer provides developers with actionable, complete tickets that require zero follow-up questions.

**The Golden Ticket Structure:**
* **Environment:** (e.g., Staging, Production - iOS app v2.1)
* **Description:** Clear, concise summary of the failure and the resulting error code.
* **Steps to Reproduce:** Numbered, sequential instructions.
    1. *Navigate to X.*
    2. *Input Y.*
    3. *Observe Z.*
* **Evidence:** Attached Trace IDs, Grafana log snippets, or network tab screenshots.
* **Root Cause / Suggestion:** Your technical hypothesis (e.g., *"The database is rejecting the payload due to a lack of UTF-8 encoding support."*)

---

## 6. Advanced Concepts (To Expand Your SRE Knowledge)

As you transition deeper into backend architecture, these concepts are critical for ensuring system reliability:

### Dead Letter Queues (DLQ)
When using message brokers like Kafka or RabbitMQ, sometimes a message cannot be processed (e.g., the audio file is completely corrupted). Instead of retrying forever and blocking the queue, the system routes the failed message to a DLQ. Engineers can then inspect the DLQ, fix the bug, and manually re-process the failed events.

### Retry Mechanisms & Exponential Backoff
If a microservice tries to call an external API and it fails, it should retry. However, immediate retries can cause a "Thundering Herd" that accidentally DDoS-attacks the struggling server. 
* **Exponential Backoff:** The system waits 1 second, then 2 seconds, then 4, then 8, before retrying, giving the failing server time to recover.

### Circuit Breakers
A design pattern used in microservices. If Service A calls Service B, and Service B fails 5 times in a row, the Circuit Breaker "trips" (opens). Service A immediately stops sending traffic to Service B and returns an error right away. This prevents cascading failures across the entire infrastructure while Service B recovers.