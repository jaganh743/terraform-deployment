# Cloud Assessment – Production-Like AWS Architecture

> **Assessment submission for the Cloud Engineer position at Siddhan Intelligence.**

---

## Architecture Overview

```
Internet
    │
    ▼
[Application Load Balancer]  ← public subnets (AZ-1, AZ-2)
    │
    ▼
[ECS Fargate Service]        ← private subnets (AZ-1, AZ-2)
 └─ Nginx container (Docker)
    │
    ▼
[NAT Gateway]  →  Internet (for ECR image pull & CloudWatch logs)
```

All infrastructure is defined as code using **Terraform** and deployed on **AWS**.

---

## Stack

| Layer | Technology |
|---|---|
| Containerisation | Docker + Nginx (Alpine) |
| Infrastructure-as-Code | Terraform ≥ 1.3 |
| Compute | AWS ECS Fargate |
| Container Registry | AWS ECR |
| Networking | AWS VPC – public + private subnets across 2 AZs |
| Load Balancing | AWS Application Load Balancer (ALB) |
| Auto-scaling | AWS Application Auto Scaling (CPU & Memory based) |
| CI/CD | GitHub Actions |
| Monitoring | AWS CloudWatch Logs + CloudWatch Alarms |

---

## Repository Structure

```
.
├── app/
│   ├── Dockerfile          # Multi-stage Nginx container
│   ├── nginx.conf          # Custom Nginx config with /health endpoint
│   └── index.html          # Static web page
├── terraform/
│   ├── main.tf             # All AWS resources
│   ├── variables.tf        # Input variables
│   └── outputs.tf          # Useful output values
├── .github/
│   └── workflows/
│       └── deploy.yml      # CI/CD pipeline (GitHub Actions)
└── README.md
```

---

## Design Decisions

### 1. ECS Fargate over EC2
Fargate eliminates the need to manage EC2 instances, OS patching, and cluster capacity. For a stateless web app, it provides the right balance of simplicity and production-readiness. The trade-off is slightly higher per-unit cost vs. EC2 with Reserved Instances at scale.

### 2. Public/Private Subnet Split
ECS tasks run in **private subnets** — they are never directly reachable from the internet. Only the ALB lives in public subnets. This follows the principle of least exposure and is a standard production pattern.

### 3. Application Load Balancer
The ALB provides:
- HTTP routing and health-check-based traffic management
- A single stable DNS endpoint for the application
- TLS termination point (HTTPS can be added by attaching an ACM certificate to the listener)

### 4. Auto-scaling on CPU and Memory
Two independent target-tracking policies scale the ECS service:
- Scale out when CPU > 60% (60 s cooldown)
- Scale out when Memory > 70% (60 s cooldown)
- Scale in after 300 s to avoid thrashing

Min = 2 tasks ensures high availability across AZs even at idle.

### 5. Single NAT Gateway
A single NAT Gateway in AZ-1 is used for cost reasons (NAT Gateways are ~$32/month each). For a highly available production system, one NAT Gateway per AZ is recommended — the variable `aws_nat_gateway` can be expanded to a `count = 2` pattern easily.

### 6. CloudWatch Container Insights
ECS Container Insights is enabled on the cluster, giving CPU, memory, disk, and network metrics per service and per task without any additional instrumentation.

---

## Trade-offs Considered

| Decision | Chosen | Alternative | Reason |
|---|---|---|---|
| Compute | ECS Fargate | EKS / EC2 | Lower operational overhead for this scope |
| State storage | Local (default) | S3 backend | S3 backend commented in for easy activation |
| NAT redundancy | 1 NAT GW | 1 per AZ | Cost; can be expanded trivially |
| TLS | HTTP only | HTTPS (ACM) | Requires a domain; listener upgrade is 2 lines |
| Logging | CloudWatch | ELK / Datadog | Native AWS, zero extra cost |

---

## Cost Awareness & Optimisation

### Estimated Monthly Cost (us-east-1, 2 tasks running 24/7)

| Resource | Approx. Cost |
|---|---|
| ECS Fargate (2 × 0.25 vCPU, 512 MiB) | ~$7 |
| Application Load Balancer | ~$16 |
| NAT Gateway (1) | ~$32 + data |
| ECR storage | ~$0.10/GB |
| CloudWatch Logs | ~$0.50/GB ingested |
| **Estimated Total** | **~$56–65/month** |

### Optimisation Levers
- **Spot/Fargate Spot** – enable `capacity_provider_strategy` with `FARGATE_SPOT` for up to 70% compute savings on non-critical tasks.
- **Scale to 1 task at night** – use scheduled scaling actions for dev/staging environments.
- **Reserved NAT usage** – move to VPC Endpoints for ECR and CloudWatch to eliminate most NAT data charges in production.
- **Log retention** – set to 7 days (already configured); adjust per compliance requirements.

---

## How to Deploy

### Prerequisites
- AWS CLI configured (`aws configure`)
- Terraform ≥ 1.3 installed
- Docker installed

### Step 1 – Provision Infrastructure

```bash
cd terraform
terraform init
terraform plan
terraform apply
```

Note the `ecr_repository_url` and `alb_dns_name` from the outputs.

### Step 2 – Build & Push Docker Image

```bash
# Login to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin <ecr_repository_url>

# Build and push
docker build -t cloud-assessment ./app
docker tag cloud-assessment:latest <ecr_repository_url>:latest
docker push <ecr_repository_url>:latest
```

### Step 3 – Force ECS to pull new image

```bash
aws ecs update-service \
  --cluster cloud-assessment-cluster \
  --service cloud-assessment-service \
  --force-new-deployment
```

### Step 4 – Access the app

Open the `alb_dns_name` output URL in your browser.

---

## CI/CD Pipeline

The GitHub Actions workflow (`.github/workflows/deploy.yml`) automates the full pipeline on every push to `main`:

1. **Build** – builds the Docker image and runs a container health check
2. **Push** – tags with `$GITHUB_SHA` and pushes to ECR
3. **Deploy** – updates the ECS task definition and triggers a rolling deployment with zero downtime

### Required GitHub Secrets

| Secret | Description |
|---|---|
| `AWS_ACCESS_KEY_ID` | IAM user access key |
| `AWS_SECRET_ACCESS_KEY` | IAM user secret key |

---

## Monitoring

| What | Where |
|---|---|
| Container logs (stdout/stderr) | CloudWatch Log Group `/ecs/cloud-assessment` |
| CPU/Memory metrics | CloudWatch → ECS Container Insights |
| ALB 5xx errors alarm | CloudWatch Alarm `cloud-assessment-alb-5xx` |
| High CPU alarm | CloudWatch Alarm `cloud-assessment-high-cpu` |
| Health check | `/health` endpoint on every container |

---

## Clean Up

```bash
cd terraform
terraform destroy
```

> ⚠️ This will remove all AWS resources created by this project.
