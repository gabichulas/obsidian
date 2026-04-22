# The Chaos Lab: SRE & L3 Incident Response Simulator (k3s Edition)

## 📌 Overview
The Chaos Lab is an advanced, localized Chaos Engineering environment designed to simulate complex, real-world production outages on Kubernetes. It runs on a lightweight `k3s` cluster, making it suitable for constrained homelab hardware (4GB RAM).

This version introduces asynchronous processing queues within a Kubernetes orchestrator, simulating critical pain points: cache stampedes, DNS resolution failures, pod eviction storms, and corrupted deployments. It requires active debugging using `kubectl` and standard Kubernetes observability practices.

## 🏗️ Architecture (Kubernetes Microservices)
The environment simulates a decoupled architecture where all components are managed as Kubernetes Deployments and Services.

```text
[ Client (curl) ] ---> [ Nginx Ingress Controller ]
                               |
                               |---> [ Service: api-backend ] --(Read/Write)--> [ Service: redis-broker ]
                                                                                       |
                                                                                       |--(Tasks)
                                                                                       v
                                                               [ Service: celery-worker ] ---> [ Service: postgres-db ]
```

## 🛠️ Technology Stack
* **Orchestrator:** Kubernetes (`k3s` lightweight distribution)
* **Ingress & Networking:** Traefik / Nginx Ingress, Kubernetes NetworkPolicies
* **Application Layer:** Python 3.11+, FastAPI
* **Asynchronous Processing:** Celery
* **State & Data:** Redis (Message Broker), PostgreSQL (Relational DB)
* **Automation & Chaos Injection:** Advanced Bash Scripting with `kubectl`

## 🐒 The Chaos Monkey Script (`kube_chaos_monkey.sh`)
This utility interacts directly with the Kubernetes API to inject cluster-level faults, forcing the engineer to troubleshoot Deployments, Services, and CoreDNS.

```bash
#!/bin/bash
# Require KUBECONFIG to be set
NAMESPACE="chaos-lab"

# Scenario 0: Simulate a DNS Blackout (Break CoreDNS)
function trigger_dns_blackout() {
    echo "Initiating Cluster DNS Blackout..." > /dev/null
    # Scale down the CoreDNS deployment to 0, breaking internal service discovery
    kubectl scale deployment coredns -n kube-system --replicas=0
}

# Scenario 1: Simulate a Network Partition (DB Isolation)
function trigger_db_partition() {
    echo "Applying NetworkPolicy to drop DB traffic..." > /dev/null
    # Apply a deny-all ingress policy to the postgres pods
    cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: isolate-postgres
  namespace: $NAMESPACE
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
  - Ingress
EOF
}

# Scenario 2: Simulate a CrashLoopBackOff (Worker Poison Pill)
function trigger_worker_crashloop() {
    echo "Injecting bad command into worker deployment..." > /dev/null
    # Patch the worker deployment to run a failing command, causing a CrashLoopBackOff
    kubectl patch deployment celery-worker -n $NAMESPACE -p \
    '{"spec":{"template":{"spec":{"containers":[{"name":"worker","command":["/bin/sh","-c","exit 1"]}]}}}}'
}

# Scenario 3: Eviction Storm / Resource Exhaustion
function trigger_eviction_storm() {
    echo "Deploying Memory Stress pod to force evictions..." > /dev/null
    # Deploy a pod that rapidly consumes memory without limits, triggering kubelet evictions
    cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: memory-stress
  namespace: $NAMESPACE
spec:
  containers:
  - name: stress
    image: vish/stress
    args: ["-mem-total", "2000M", "-mem-alloc-size", "10M", "-mem-alloc-sleep", "1s"]
EOF
}

# Execute a random disaster scenario
SCENARIO=$((RANDOM % 4))

case $SCENARIO in
    0) trigger_dns_blackout ;;
    1) trigger_db_partition ;;
    2) trigger_worker_crashloop ;;
    3) trigger_eviction_storm ;;
esac

echo "Chaos Monkey has struck the Kubernetes cluster. Good luck."
```

## 🚨 Attack Vectors & Troubleshooting Runbooks

### 1. The DNS Blackout (Scenario 0)
* **Symptom:** APIs return 500 errors. Application logs show `Name or service not known` when trying to reach `redis-broker` or `postgres-db`.
* **Investigation:** Exec into a pod (`kubectl exec -it <api-pod> -- sh`) and try `nslookup postgres-db`. Check the status of CoreDNS: `kubectl get pods -n kube-system -l k8s-app=kube-dns`.
* **Resolution:** Restore the CoreDNS deployment: `kubectl scale deployment coredns -n kube-system --replicas=1`.

### 2. The Database Partition (Scenario 1)
* **Symptom:** API requests time out. Celery workers throw connection errors. All pods are `Running` and DNS resolves correctly.
* **Investigation:** Check pod events (`kubectl describe pod <worker-pod>`). Run a network debug pod. Check for active NetworkPolicies blocking traffic: `kubectl get networkpolicies -n chaos-lab`.
* **Resolution:** Delete the rogue network policy: `kubectl delete networkpolicy isolate-postgres -n chaos-lab`.

### 3. The Worker CrashLoopBackOff (Scenario 2)
* **Symptom:** Redis queue fills up indefinitely.
* **Investigation:** Check pod status: `kubectl get pods -n chaos-lab`. Notice the worker pods are in a `CrashLoopBackOff` state. Check logs to see why they are failing: `kubectl logs deployment/celery-worker --previous`. Check deployment history: `kubectl rollout history deployment/celery-worker`.
* **Resolution:** Roll back the deployment to the previous healthy revision: `kubectl rollout undo deployment/celery-worker -n chaos-lab`.

### 4. The Eviction Storm (Scenario 3)
* **Symptom:** APIs randomly return 502/503. Pods keep restarting or disappearing. 
* **Investigation:** Check cluster events: `kubectl get events --sort-by='.metadata.creationTimestamp'`. Look for `Evicted` status due to `NodeHasNoDiskPressure` or `NodeHasMemoryPressure`. Find the rogue pod consuming resources: `kubectl top pods -n chaos-lab`.
* **Resolution:** Delete the `memory-stress` pod. To prevent this in the future, implement strict `ResourceQuotas` at the namespace level and `limits`/`requests` on all application deployments.

## ⚙️ Deployment Strategy (k3s on 4GB Homelab)

1. **Install k3s without unnecessary components:**
   To save memory on the constrained hardware, install `k3s` without the Traefik ingress (port-forward or deploy Nginx Ingress manually) and without the servicelb.
   ```bash
   curl -sfL [https://get.k3s.io](https://get.k3s.io) | INSTALL_K3S_EXEC="server --disable traefik --disable servicelb" sh -
   ```
2. **Configure `kubectl` access:**
   ```bash
   mkdir -p ~/.kube
   sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
   sudo chown $USER ~/.kube/config
   ```
3. **Environment Isolation:** Use a dedicated namespace (`chaos-lab`) so you can easily wipe the entire environment without reinstalling k3s.
   ```bash
   kubectl create namespace chaos-lab
   # To reset the environment:
   kubectl delete namespace chaos-lab
   ```