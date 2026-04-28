### Phase 0: The Layer 3 Foundation (IP, CIDR, & Subnetting)

**Goal:** Be able to look at a CIDR block like `10.42.0.0/16` and instantly know how many IP addresses it holds, what its boundaries are, and if it overlaps with another network.

- **Resource 1: "Subnetting Mastery" by Practical Networking (YouTube)**
    
    - **Why it's the best:** Ed Harmoush is the undisputed king of teaching subnetting. He strips away all the legacy academic fluff and teaches you a foolproof method for doing CIDR math.
        
    - **Action:** Watch episodes 1 through 4. He teaches a specific "cheat sheet" method that you will end up writing down in every DevOps technical interview you ever take.
        
- **Resource 2: CIDR.xyz (Interactive Tool)**
    
    - **Why it's the best:** It is a minimal, interactive visualizer for IP ranges.
        
    - **Action:** Whenever you are confused by a subnet mask while watching the videos, plug the CIDR notation in here to see exactly how the network is being sliced in real-time.

### Phase 1: Core Protocols (The Rules of the Highway)

**Goal:** Deeply understand TCP/UDP, DNS, and HTTP/TLS. If you don't know these, you cannot troubleshoot a misconfigured load balancer or a failing API call.

- **Resource 1: "High Performance Browser Networking" by Ilya Grigorik**
    
    - **Why it's the best:** This book is legendary in the SRE/DevOps space. It is completely free to read online. You don't need to read the whole thing, but you **must** read the primer chapters on TCP, UDP, and TLS. It will teach you exactly how packets are established and why latency happens.
        
    - **Action:** Read chapters 1-4.
        
- **Resource 2: Hussein Nasser (YouTube)**
    
    - **Why it's the best:** Hussein is a backend and architecture master. He doesn't just draw diagrams; he explains the _why_ behind networking from an engineer's perspective.
        
    - **Action:** Search his channel for his specific deep-dives on: **TCP vs UDP**, **How DNS Works**, and **Layer 4 vs Layer 7 Load Balancing**.
        
- **Resource 3: Julia Evans' Zines (Mess With DNS)**
    
    - **Why it's the best:** Julia creates incredible, intuitive guides. Her tool, `messwithdns.net`, lets you experiment with DNS records safely to see exactly how they propagate and resolve in real-time.
        

### Phase 2: Linux & Container Networking (The Magic Trick)

**Goal:** Understand how a Linux server routes traffic and how containers trick the system into giving them their own isolated networks. This is the foundation of Docker and Kubernetes.

- **Resource 1: Iximiuz (Ivan Velichko's Blog)**
    
    - **Why it's the best:** This is the undisputed holy grail for understanding how containers actually work under the hood. He breaks down the complex Linux kernel features into digestible, visual articles.
        
    - **Action:** Read his articles on **Linux Network Namespaces**, **veth pairs**, and **How Docker Networking Works**. This will make you realize that containers aren't magic; they are just isolated Linux processes.
        
- **Resource 2: SadServers.com**
    
    - **Why it's the best:** You mentioned this earlier, and your intuition was 100% correct. SadServers is the absolute best way to practice Linux troubleshooting. They spin up ephemeral, broken Linux VMs and tell you to fix them.
        
    - **Action:** Do the "Network Troubleshooting" challenges. You will be forced to use commands like `ss`, `netstat`, `ip route`, and `curl` to figure out why services can't talk to each other.
        

### Phase 3: Kubernetes Networking (The Final Boss)

**Goal:** Understand how Kubernetes manages thousands of containers across multiple nodes using virtual networks.

- **Resource 1: TechWorld with Nana (YouTube)**
    
    - **Why it's the best:** Nana has the best free Kubernetes crash courses on the internet. She explains the abstract K8s concepts clearly.
        
    - **Action:** Watch her specific videos on **Kubernetes Networking Explained**. You need to understand the difference between Pod IP, ClusterIP (internal services), NodePort, and LoadBalancer.
        
- **Resource 2: Kubernetes Ingress Controllers (NGINX)**
    
    - **Why it's the best:** Once your cluster is running, you need a way to route `yourdomain.com/api` to one pod and `yourdomain.com/web` to another. This is done via Ingress.
        
    - **Action:** Read the official Kubernetes documentation on **Ingress**. Then, deploy the NGINX Ingress Controller on your homelab.


---

### Module 1: The Docker Networking Trap

**The Concept:** Containers must communicate over isolated bridge networks. Never publish ports directly to the host on a production server. **The "Why":** Docker alters `iptables` directly. If you configure a strict Linux firewall (UFW) to block everything, but you run `docker run -p 8000:8000`, Docker bypasses your firewall and exposes the container to the public internet.

**Your Action Plan:**

1. **Create Custom Bridge Networks:** Build a `docker-compose.yml` where you explicitly define a custom network.
    
2. **Remove Port Bindings:** Do not use the `ports: ["8000:8000"]` array for your backend or database. Only use the `expose` directive, or rely on the bridge network.
    
3. **Use an API Gateway/Proxy:** The _only_ container that should have a `ports` mapping to the host is your Reverse Proxy (like NGINX). NGINX sits on the host ports (80/443) and routes traffic internally to your isolated containers using their Docker DNS names (e.g., `http://backend_api:8000`).
    

---

### Module 2: Cloud Architecture (Public vs. Private)

**The Concept:** Host infrastructure in private networks, exposing only what is necessary via a Load Balancer in the public network. **The Translation:**

- **AWS:** VPC with Public Subnet (contains Internet Gateway) and Private Subnet (contains NAT Gateway). Traffic enters via **ELB (Elastic Load Balancer)**.
    
- **GCP:** VPC with public/private subnets. Traffic enters via **Cloud Load Balancing**.
    

**Your Action Plan (On GCP):**

1. Create a custom VPC in GCP. Create two subnets: `subnet-public` and `subnet-private`.
    
2. Deploy a Compute Engine VM in the `subnet-private` (do not give it an External IP). This will hold your application.
    
3. Deploy a Google Cloud Load Balancer (ELB equivalent). Attach its frontend to a public IP, and point its backend to your private VM. This ensures external users can only talk to the Load Balancer, never directly to your server.
    

---

### Module 3: Micro-Segmentation & Secure Access

**The Concept:** Configure Security Groups properly and use a Bastion SSH tunnel proxy to access private instances safely. **The Translation:**

- **AWS:** Security Groups (Stateful virtual firewalls).
    
- **GCP:** VPC Firewall Rules.
    

**Your Action Plan (On GCP):**

1. **Security Groups (Firewalls):** Configure GCP Firewall rules with a "Default Deny" mindset. Create a rule that allows your Load Balancer to talk to your private VM on port 8000. Create another rule that explicitly denies all other incoming traffic to that VM.
    
2. **The Bastion Host:** Since your VM has no public IP, you cannot SSH into it from your laptop.
    
    - Deploy a tiny, cheap VM in your `subnet-public`. This is your Bastion (Jump Box).
        
    - Configure firewall rules so you can _only_ SSH into the Bastion from your home IP address.
        
    - Configure rules so the Bastion is the _only_ machine allowed to SSH into your private VM.
        
    - _Workflow:_ You SSH into the Bastion, and from inside the Bastion, you SSH into the private server.
        

---

### Module 4: Application Security & Database Protection

**The Concept:** Review OWASP Top 10, never expose the database to the internet, never use default passwords, and use bcrypt instead of MD5. **The "Why":** MD5 is a fast hashing algorithm originally designed for file checksums; hackers use "rainbow tables" to crack MD5 passwords in milliseconds. `bcrypt` has deliberate mathematical slow-downs (salting and key stretching) making brute-force attacks economically impossible.

**Your Action Plan:**

1. **Database Isolation:** Ensure your PostgreSQL database lives strictly in the private network. Change the default `postgres` user password immediately upon creation.
    
2. **Auth Implementation:** In your backend code, implement authentication securely.
    
    Python
    
    ```
    import bcrypt
    
    # Hash a password before storing it in the database
    def hash_password(plain_text_password: str) -> bytes:
        # Generate a salt and hash the password
        salt = bcrypt.gensalt()
        hashed = bcrypt.hashpw(plain_text_password.encode('utf-8'), salt)
        return hashed
    
    # Verify a user login attempt
    def check_password(plain_text_password: str, hashed_password: bytes) -> bool:
        return bcrypt.checkpw(plain_text_password.encode('utf-8'), hashed_password)
    ```
    
3. **OWASP Review:** Read the OWASP Top 10 list, focusing heavily on Injection, Broken Authentication, and Security Misconfigurations.
    

---

### Module 5: The Edge & Traffic Control

**The Concept:** Use a WAF (Web Application Firewall), configure Rate Limits at both the WAF and App level, and use Cloudflare. **The Translation:**

- **AWS:** AWS WAF.
    
- **GCP:** Google Cloud Armor.
    

**Your Action Plan (On your Homelab/Juana Manso):**

1. **Cloudflare Tunnel:** Instead of opening ports on your home router, install `cloudflared` on your Juana Manso. It creates a secure, outbound-only tunnel to Cloudflare.
    
2. **WAF & Rate Limits (Edge Level):** Go to the Cloudflare dashboard. Enable the free WAF to block malicious payloads automatically. Set up a Rate Limiting rule (e.g., block any IP making more than 100 requests in 1 minute to your login endpoint).
    
3. **Rate Limits (App Level):** Implement an emergency backup rate limiter directly in your code (e.g., using `slowapi` or custom middleware) in case the WAF is bypassed. This protects your database from being overwhelmed.
    

---

### Module 6: Kubernetes Networking Stuff

**The Concept:** Master how traffic moves inside an orchestrator. **The "Why":** Kubernetes networking builds upon everything above (Docker bridges, Load Balancers, Security Groups) but automates it at scale.

**Your Action Plan (On your Homelab):**

1. **ClusterIP:** Understand that this is the K8s equivalent of a private subnet. Pods can only talk to each other.
    
2. **Ingress:** Deploy an NGINX Ingress Controller. This is the Kubernetes equivalent of an ELB/Cloud Load Balancer. It takes external traffic and routes it to internal ClusterIP services based on the URL path.
    
3. **Network Policies:** This is the Kubernetes equivalent of Security Groups. Write YAML files that dictate exactly which pods are allowed to communicate with the database pod.