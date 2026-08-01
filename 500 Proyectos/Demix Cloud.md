# Demix Cloud — Implementation & Tracking Checklist

This document tracks the technical milestones required to build, provision, bootstrap, and validate the **Demix Cloud** platform end-to-end.

---

## 🎯 Phase 1: Local Development & Application Code

### 1.1 FastAPI Gateway Service (`/gateway`)
- [ ] Initialize Python environment and project structure for FastAPI.
- [ ] Implement `POST /api/v1/demix/separate` endpoint accepting `.mp3` audio files and model parameter (`unet` or `vit`).
- [ ] Integrate AWS SDK (`boto3`) for Amazon S3 file upload logic.
- [ ] Implement AWS SQS payload publisher returning a `202 Accepted` response with a UUID `job_id`.
- [ ] Containerize the application (`Dockerfile`) and test locally via Docker.

### 1.2 TensorFlow Worker Service (`/worker`)
- [ ] Structure the consumer loop listening for AWS SQS messages.
- [ ] Integrate STFT (Short-Time Fourier Transform) audio preprocessing and inverse STFT postprocessing (`dsp.py`).
- [ ] Load pre-trained TensorFlow models (U-Net and Vision Transformer) for inference execution.
- [ ] Implement S3 download logic for raw input `.mp3` files and S3 upload logic for output stems (`vocals`, `drums`, `bass`, `other`).
- [ ] Add SQS message deletion / acknowledgment upon successful execution.
- [ ] Containerize worker environment (`Dockerfile` with TensorFlow dependencies).

---

## 🏗️ Phase 2: Infrastructure as Code (Terraform)

### 2.1 Networking & Security (`/terraform`)
- [ ] Declare AWS Provider ($\ge 5.0$).
- [ ] Implement Multi-AZ `aws_vpc` (`10.0.0.0/16`) across 3 Availability Zones (`us-east-1a`, `us-east-1b`, `us-east-1c`).
- [ ] Create 3x Public Subnets and 3x Private Subnets with corresponding Route Tables and Internet/NAT Gateways.
- [ ] Draft Security Groups enforcing tight internal VPC communication (`10.0.0.0/16`) and NLB health check access.
- [ ] Configure IAM Roles with `AmazonSSMManagedInstanceCore` attached for passwordless/SSH-less management.

### 2.2 Storage, Messaging & Compute
- [ ] Create private `aws_s3_bucket` resources for raw audio inputs and processed stems.
- [ ] Create `aws_sqs_queue` with a 300-second visibility timeout to handle heavy inference execution.
- [ ] Declare AWS Network Load Balancer (NLB) in Public Subnets with Target Groups for Ports 80/443.
- [ ] Provision 3x K3s Master EC2 instances (`t3.small`) across the 3 Private Subnets.
- [ ] Provision Worker EC2 instances (`t3.medium`) in Private Subnets.
- [ ] Export critical metrics via Terraform `outputs.tf` (NLB DNS, SQS Queue URL, S3 Bucket names).

---

## ⚙️ Phase 3: Configuration Management & Cluster Bootstrap (Ansible)

### 3.1 Orchestration Setup (`/ansible`)
- [ ] Configure Ansible `inventory.aws_ec2.yml` using dynamic tags and `community.aws.aws_ssm` connection plugin.
- [ ] Test passwordless SSM reachability across all private nodes (`ansible all -m ping`).

### 3.2 K3s High Availability Bootstrap
- [ ] Execute primary Master node initialization using `--cluster-init` (embedded `etcd` quorum setup).
- [ ] Retrieve cluster token securely from Master 1.
- [ ] Join Master 2 and Master 3 to complete the 3-node Control Plane quorum.
- [ ] Join Worker nodes to the cluster targeting the NLB endpoint.
- [ ] Verify cluster state (`kubectl get nodes -o wide`) showing all nodes `Ready` across 3 AZs.

---

## 🚀 Phase 4: Kubernetes Deployments & Event-Driven Autoscaling

### 4.1 Cluster Workloads (`/kubernetes`)
- [ ] Create `demix` Namespace and S3/SQS IAM secrets.
- [ ] Deploy Traefik Ingress configuration mapping incoming traffic to the FastAPI Gateway.
- [ ] Deploy Gateway Deployment and ClusterIP Service.
- [ ] Deploy TensorFlow Worker Deployment with zero initial replicas (`replicas: 0`).

### 4.2 KEDA (Kubernetes Event-driven Autoscaling)
- [ ] Install KEDA operator in the cluster via Helm (`kedacore/keda`).
- [ ] Apply `ScaledObject` pointing to the AWS SQS queue URL.
- [ ] Configure trigger metric (`queueLength: 1`) to dynamically scale Worker pods from `0` up to `N`.

---

## 🔬 Phase 5: Verification & Portfolio Production

- [ ] Execute end-to-end integration test: Upload a track, verify S3 raw storage, observe SQS message consumption, and inspect output stems in S3.
- [ ] Run load test script (e.g., 10 concurrent requests) and record terminal footage of KEDA scaling Worker pods from 0.
- [ ] Perform Chaos Engineering test: Terminate 1 Master node or Worker node during inference and verify recovery.
- [ ] Record a 60-second Loom/Asciinema split-screen demo.
- [ ] Publish repository, attach verification screenshots/diagrams to `README.md`, and share on LinkedIn.