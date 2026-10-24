# Tech Challenge 1 — Containerized React/Express App on AWS ECS Fargate

A React frontend and Express backend, containerized with Docker, deployed to AWS ECS on Fargate via Terraform, with CI/CD automated through a self-hosted Jenkins pipeline. The app autoscales under load and is reachable through a single public Application Load Balancer (ALB) URL.

**Live app:** http://tech-challenge-1-alb-62754989.us-east-1.elb.amazonaws.com
**Jenkins:** http://18.235.167.253:8080

---

## 1. Architecture Overview

![Architecture Diagram](docs/architecture.png)

```
GitHub (main branch)
      |
      | webhook (push)
      v
Jenkins (EC2, Docker-in-Docker)
      |
      | build images -> push to ECR -> force new ECS deployment
      v
Amazon ECR (frontend + backend repos)
      |
      v
Amazon ECS Cluster (Fargate)
  |-- Frontend Service (min 1 / desired 1 / max 4 tasks)
  |-- Backend  Service (min 1 / desired 1 / max 4 tasks)
      |
      v
Application Load Balancer (public, port 80)
  |-- default action        -> Frontend target group
  |-- path pattern /api,/api/* -> Backend target group
```

- **ECS Fargate** runs both services — no EC2 instances to patch or manage for the app itself, AWS handles the underlying compute.
- **Each service**: 0.5 vCPU / 1 GB RAM per task, runs in private subnets (not directly internet-reachable — only the ALB is public).
- **Networking**: custom VPC, 2 public subnets (ALB + NAT Gateway) + 2 private subnets (ECS tasks), Internet Gateway for public traffic, NAT Gateway so private-subnet tasks can still reach the internet (e.g. to pull images).
- **Autoscaling**: AWS Application Auto Scaling, target-tracking policy on ECS Service Average CPU Utilization, target = 50%. AWS automatically raises/lowers `DesiredCount` between 1 and 4 tasks to hold CPU near that target — no manual step scaling rules to maintain.
- **CI/CD**: Jenkins (self-managed, running as a Docker container on its own EC2 instance) — not built with Terraform, since a CI server is treated as long-lived infrastructure that's configured once by hand rather than reprovisioned on every apply. Jenkins builds both Docker images, pushes them to ECR, then forces a new ECS deployment so the services pick up the new images.
- **CORS**: the backend's `config.js` allow-list is set to the ALB's public DNS name, since the frontend and backend are served from different paths on the same load balancer and the browser enforces CORS on cross-origin API calls.

---

## 2. Repository Layout

```
.
├── backend/            # Express API (Node 16), Dockerfile, config.js
├── frontend/           # React app (Node 18), Dockerfile, config.js
├── terraform/          # All AWS infrastructure as code
│   ├── provider.tf
│   ├── networking.tf
│   ├── ecs.tf
│   ├── ecr.tf
│   ├── security.tf
│   ├── alb.tf
│   ├── autoscaling.tf
│   ├── jenkins.tf
│   └── variables.tf
├── Jenkinsfile          # CI/CD pipeline definition (main branch)
└── README.md
```

---

## 3. Running It Locally

```bash
# Backend
cd backend
npm ci
npm start                # runs on localhost:8080

# Frontend (separate terminal)
cd frontend
npm ci
npm start                # runs on localhost:3000, calls localhost:8080
```

Or with Docker:

```bash
docker build -t backend-app ./backend
docker build -t frontend-app ./frontend
docker run -d --name backend  -p 8080:8080 backend-app
docker run -d --name frontend -p 3000:3000 frontend-app
```

Both should return a page showing `SUCCESS` and a GUID when working correctly.

---

## 4. Deploying the Infrastructure (Terraform)

```bash
cd terraform
terraform init
terraform plan  -var="my_ip=<your-ip>/32"
terraform apply -var="my_ip=<your-ip>/32"
```

This provisions, in order: the VPC and subnets, ECR repositories, IAM roles, security groups, the ECS cluster/services/task definitions, the ALB and its listener rules, the autoscaling policies, and the Jenkins EC2 instance (with an attached Elastic IP so its public address survives a stop/start).

**Note:** `.terraform/` and `*.tfstate` are git-ignored — state is never committed.

---

## 5. Jenkins & CI/CD Pipeline

Jenkins runs as a Docker container on a dedicated EC2 instance (Amazon Linux 2023, `t3.small`), provisioned once by hand rather than through Terraform recreations, since a CI server is meant to persist across infrastructure changes to the app itself.

**Pipeline (`Jenkinsfile`), triggered by a GitHub webhook on every push to `main`:**
1. Checkout the repo
2. Build the backend and frontend Docker images
3. Authenticate to Amazon ECR
4. Tag and push both images
5. Force a new deployment on both ECS services, so they pull the freshly pushed images

This means a plain `git push` to `main` is the entire deploy trigger — no manual AWS Console steps required after the initial setup.

---

## 6. Load Testing & Autoscaling Results

Load was generated from a local machine using [siege](https://github.com/JoeDog/siege), which drives raw concurrent HTTP requests against the public ALB URL:

```bash
siege -c 250 -t 5M http://tech-challenge-1-alb-62754989.us-east-1.elb.amazonaws.com
```

**Results:**
- 27,755 hits, 99.81% availability, ~92.4 transactions/sec sustained over the run
- The **frontend** ECS service scaled from 1 task up to its configured **max of 4 tasks**, all healthy in the ALB target group, as CPU utilization stayed above the 50% target
- The **backend** service correctly stayed at 1 task throughout — `siege` performs raw HTTP GETs and does not execute the frontend's client-side JavaScript, so it never triggers the frontend's `fetch('/api')` call that would put load on the backend. This is an expected property of the test tool, not a defect in the autoscaling configuration.
- After load stopped, the frontend service scaled back down to 1 task automatically once average CPU dropped back below target — no manual intervention needed in either direction.

---

## 7. GitOps Branch (Bonus)

A separate `gitops` branch replaces the Jenkins-triggered pipeline with GitHub Actions, so the app can be deployed without Jenkins involved at all — proving the deploy logic isn't accidentally tied to one specific CI server.

**Workflow file:** `.github/workflows/deploy.yml`, triggered on every push to the `gitops` branch. It performs the same steps as the Jenkins pipeline (build both Docker images, push to ECR, force a new ECS deployment on both services) but through GitHub's own runners instead of a self-hosted EC2 box.

**Authentication — OIDC instead of stored AWS keys:**
Rather than storing long-lived AWS access keys as GitHub secrets, this branch uses OpenID Connect (OIDC) federation:
- An **IAM OIDC identity provider** (`token.actions.githubusercontent.com`) tells AWS to trust short-lived, signed identity tokens issued by GitHub.
- An **IAM role** (`github-actions-deploy-role`) has a trust policy that only allows that identity provider to assume it, and only when the token's `sub` (subject) claim matches this exact repo and branch — so no other GitHub repo, and no other branch of this repo, can assume the role.
- Each workflow run requests a fresh, minutes-long token from GitHub, trades it for temporary AWS credentials via `sts:AssumeRoleWithWebIdentity`, and those credentials expire on their own. Nothing long-lived is ever stored in GitHub.

**A real debugging note worth keeping:** GitHub's OIDC tokens embed immutable numeric IDs alongside the human-readable org/repo names in the `sub` claim (e.g. `repo:Drezzym6@160346145/devops-code-challenge1@1375181663:ref:refs/heads/gitops`), while AWS's own IAM console wizard still generates trust policies using the plain-name format (`repo:Drezzym6/devops-code-challenge1:ref:refs/heads/gitops`). That mismatch causes a generic `AccessDenied: Not authorized to perform sts:AssumeRoleWithWebIdentity` error even when everything looks correct. It was root-caused using AWS CloudTrail's Event history (searching for the `AssumeRoleWithWebIdentity` event) to see the exact identity string AWS actually received, then fixing the trust policy's `sub` condition to match it exactly.

---

## 8. Cleanup

To avoid ongoing AWS charges after grading: `terraform destroy` from the `terraform/` directory, plus manual removal of the Jenkins EC2 instance's EBS volume and Elastic IP if not fully covered by destroy, and manual removal of any leftover ECR images.
