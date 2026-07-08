---
name: airfire-deployment
description: Guidelines and deployment strategy details comparing the legacy EC2 + Docker Compose footprint and the modern AWS ECS/Fargate architecture.
---

# Deployment Strategy: 20-Service Web Application
*A practical guide for project managers and developers*

---

## Overview

This document describes a deployment strategy for a suite of 20 interdependent dockerized web services running on AWS. The goals are straightforward: reliable uptime, easy updates every few days, visibility into system health, and a setup that a small team can actually understand and maintain.

The strategy outlines two distinct footprints based on project lifecycle:
1. **Existing/Legacy Footprint:** A practical **EC2 + Docker Compose + Fabric** stack managed via automated pipelines.
2. **New Projects Footprint:** A modern, serverless **AWS ECS + Fargate** stack targeting automatic failover, auto-scaling, and zero static credentials.

For the legacy footprint, four primary tools are used in combination:

| Tool | Role (Legacy Footprint) |
|---|---|
| **Terraform** | Provisions cloud infrastructure (servers, networking, security) |
| **Docker Compose** | Defines and runs the 20 services on each server |
| **Fabric** | Deploys updates by scripting SSH commands across servers |
| **Prefect** | Monitors workflows, provides a dashboard, and sends alerts |

These tools are complementary — each does one job well, and together they cover the full lifecycle from infrastructure to monitoring.

---

## The Four Tools

### Terraform — Infrastructure Provisioning

Terraform is responsible for creating and managing the AWS resources that everything else runs on. You write a description of what infrastructure you want (servers, networking rules, security settings), and Terraform makes it so. Run it again and it only changes what's different — it won't rebuild things that already exist.

**What Terraform manages in this deployment:**

- EC2 instances (one per service, plus pool instances where needed)
- VPC networking — the private network your instances communicate over
- Security groups — firewall rules controlling which services can talk to which
- IAM roles — AWS permissions assigned to each instance
- Any shared resources: S3 buckets, databases, load balancers

**Why this matters to the team:** Infrastructure is defined in text files that live in version control alongside your code. Changes are proposed via `terraform plan` (which shows exactly what will change before anything happens) and applied deliberately. There are no undocumented manual changes, and any environment can be rebuilt from scratch if needed.

> [!NOTE]
> **New Projects Roadmap:** For new services, Terraform remains our provisioning tool but is configured to build AWS ECS clusters, Fargate Task Definitions, Application Load Balancers (ALBs), and IAM roles rather than individual EC2 instances.

---

### Docker Compose — Service Definition and Local Orchestration

Each EC2 instance runs one or more services defined in a `docker-compose.yml` file. Docker Compose describes which Docker images to run, what environment variables they receive, which ports they expose, and how they depend on each other.

**Key capabilities for this deployment:**

- **Restart policies** — `restart: always` means a crashed container automatically restarts without human intervention
- **Dependency ordering** — Compose knows that Service B requires Service A to be healthy before starting
- **Environment configuration** — each service receives its config (API keys, database URLs, feature flags) cleanly via environment variables
- **Pooled services** — services that need multiple instances can be scaled with a single setting

**Why this matters to the team:** The `docker-compose.yml` file is a readable, single-file description of everything running on a given instance. Any developer can look at it and understand what's deployed. Updates are applied by pulling a new Docker image and restarting the relevant service — no complex orchestration required.

> [!NOTE]
> **New Projects Roadmap:** Docker Compose is bypassed in production for new services. Instead, container configurations, resource limits, environment variables, and dependencies are defined in ECS Task Definitions.

---

### Fabric — Deployment Automation

Fabric is a Python-based tool for running commands on remote servers over SSH. In this deployment, it serves as the deployment engine: when it's time to push an update, a Fabric script connects to the relevant instances, pulls the new Docker image, and restarts the affected service — all without manual SSH sessions.

**A typical deployment sequence (automated via Fabric):**

1. New Docker image is built and pushed to a container registry (e.g., AWS ECR)
2. Fabric connects to the target instance(s) over SSH
3. Pulls the new image: `docker compose pull <service>`
4. Restarts the service with zero-config: `docker compose up -d <service>`
5. Moves to the next instance in the pool

**Deploying one instance at a time** — for pooled services, Fabric deploys to one instance, waits for it to come back healthy, then moves to the next. This achieves rolling updates (no downtime) without requiring a complex orchestration platform.

**Why this matters to the team:** Deployments that happen every few days become a routine, low-stress operation. The deploy script is just Python — any developer can read it, modify it, and run it. There's no proprietary deployment platform to learn.

> [!NOTE]
> **New Projects Roadmap:** Fabric is phased out entirely for new projects. Deployments are managed natively by CI/CD updating AWS ECS services directly using the AWS API/CLI. This eliminates SSH access and keys.

---

### Prefect — Monitoring, Dashboard, and Alerts

Prefect is a workflow orchestration platform with a built-in dashboard and alerting system. In this deployment it serves as the operational nerve center: the place where the team goes to see whether everything is running correctly, and the system that pages someone when it isn't.

**Capabilities relevant to this deployment:**

- **Dashboard** — a live UI showing the status of all services and workflow runs
- **Dependency graph visibility** — Prefect models the relationships between your services explicitly; the dashboard reflects which services are healthy and which are blocked by an upstream failure
- **Alerts** — configurable notifications (Slack, email, PagerDuty) triggered by failures, delays, or unhealthy states
- **Retry logic** — failed tasks automatically retry according to a configurable policy before an alert is raised
- **Centralized logging** — logs from all services are queryable in one place, organized by service and time

**Prefect Cloud** offers a free tier that covers the needs of most small deployments, meaning no self-hosted monitoring infrastructure to maintain.

**Why this matters to the team:** Instead of SSHing into instances to check logs when something seems wrong, the team has a single URL that shows system-wide status. Alerts mean problems surface proactively rather than when a user reports them.

> [!NOTE]
> **New Projects Roadmap:** Prefect continues to orchestrate and monitor logical data workflows. For container-level health, logs, and system metrics, new projects also leverage native AWS CloudWatch Container Insights and ECS service health monitoring.

---

## How the Tools Work Together

We use automated CI/CD runners to execute deployments, shifting the resource-heavy Docker build off production servers and keeping developer credentials off dev laptops. The deployment flows differ between existing and new projects:

### Legacy/Existing Pipeline (EC2 + Fabric)
```
Developer pushes code to version control (GitHub/GitLab)
        ↓
CI/CD Runner builds the Docker image in the cloud
        ↓
CI/CD Runner pushes the pre-built image to AWS ECR
        ↓
CI/CD Runner executes the Fabric deploy script
        ↓
Fabric connects to EC2 over SSH to trigger 'docker compose pull & up'
        ↓
Prefect / CloudWatch monitors service health & triggers alerts
        ↓
Terraform (unchanged unless EC2 instances/security rules need updating)
```

### New Projects Pipeline (AWS ECS / Fargate)
```
Developer pushes code to version control (GitHub/GitLab)
        ↓
CI/CD Runner builds the Docker image in the cloud
        ↓
CI/CD Runner pushes the pre-built image to AWS ECR
        ↓
CI/CD Runner registers new ECS Task Definition & updates ECS Service
        ↓
AWS ECS orchestrates native rolling deployment (pulls image, shifts traffic, terminates old tasks)
        ↓
CloudWatch / Prefect monitors ECS service health & triggers alerts
        ↓
Terraform (defines ECS clusters, tasks, ALBs; updated via CI/CD when needed)
```

By shifting the Docker build off production infrastructure, we keep production CPU resources dedicated to serving traffic, eliminate local credential management, and maintain absolute deployment consistency across both footprints.

Terraform operates on a different timescale from the rest — it's only run when the underlying infrastructure changes (adding an instance, changing a security rule, etc.), which happens rarely compared to application updates.

---

## Addressing Key Requirements

### Updates every few days
*   **Legacy (EC2 + Fabric):** The CI/CD runner executes Fabric, making deployments repeatable and scriptable. Services in pools are updated one at a time sequentially.
*   **New Projects (ECS/Fargate):** Fully automated via native rolling updates in ECS. Deploying is a standard step in the GitHub Actions/Jenkins pipeline, requiring no SSH script maintenance.

### 20 services, some in pools
*   **Legacy (EC2 + Docker Compose):** Each service has its own `docker-compose.yml` configuration. Pooled services run on multiple instances; the Fabric deploy script handles rolling updates across the pool. Terraform provisions the right number of EC2 instances.
*   **New Projects (ECS/Fargate):** Services are defined as ECS task definitions. Scaling is handled natively by setting the desired task count in the ECS Service. AWS Application Load Balancers (ALBs) automatically handle traffic routing to new/existing tasks.

### 99%+ uptime & Failover Capability
*   **Legacy (EC2 + Docker Compose):** Docker Compose `restart: always` recovers from container-level crashes. Rolling deploys via Fabric ensure at least one instance in a pool is serving traffic during updates. However, this approach **does not provide automatic VM-level failover** if an EC2 instance dies entirely. Prefect alerts notify the team, but a developer must manually replace or reboot the VM.
*   **New Projects (ECS/Fargate):** ECS natively provides **high availability and automatic failover**. ECS tasks are deployed across multiple Availability Zones (AZs). If an instance of a task or the underlying AWS infrastructure fails, ECS automatically provisions a replacement task on healthy hardware, registers it with the ALB, and de-registers the unhealthy container without any downtime or human intervention.

### Dashboard and alerts
*   **Legacy (EC2 + Docker Compose):** Prefect Cloud provides a dashboard showing service status and dependency health, routing alerts to Slack/PagerDuty.
*   **New Projects (ECS/Fargate):** Prefect Cloud is used for workflow-level dashboards, augmented by AWS CloudWatch Container Insights and Route 53 health check alerts for infrastructure-level visibility.

### Complicated dependency graph
*   **Legacy (EC2 + Docker Compose):** Prefect models service dependencies explicitly. Docker Compose handles startup ordering within each EC2 instance. Security groups enforce network-level isolation.
*   **New Projects (ECS/Fargate):** Task dependencies within the same service are managed via ECS container definitions (`dependsOn`). Microservice-level dependencies are resolved using AWS App Mesh or ALB routing rules, with IAM policies controlling service-to-service communication.

---

## What This Stack Does Not Provide (Legacy Footprint)

Being clear about the limitations of the legacy EC2 + Docker Compose footprint avoids surprises:

- **Automatic instance failover** — if an EC2 instance becomes unresponsive, a human (or a script triggered by a Prefect alert) needs to intervene.
- **Automatic capacity scaling** — adding instances in response to traffic spikes requires running Terraform and Fabric manually or building additional automation.
- **Sub-second deployment granularity** — this footprint is well-suited to updates every few days or hours, not continuous deployment pipelines with dozens of releases per day.

For a small team managing legacy services, these are reasonable trade-offs. The stack is understandable, debuggable, and maintainable by generalist developers. 

> [!TIP]
> **Resolution in New Projects:** All three limitations are resolved natively in the New Projects footprint (AWS ECS/Fargate), which provides automatic failover, auto-scaling policies, and native zero-downtime rolling deployments via ALBs.

---

## Strategy Lifecycle: Legacy vs. New Projects

This deployment strategy serves as an attainable middle target (Phase 1) rather than the ultimate long-term destination. While the EC2 + Fabric stack is highly practical for getting started, a mature production environment must transition toward automated delivery pipelines and phase out direct SSH access and static keys to meet modern security compliance.

To manage this transition smoothly, we draw a clear line between existing/legacy applications and new projects:

### 1. Existing/Legacy Services (The 20-Service Suite)
We keep these services on the **EC2 + Docker Compose** footprint to avoid a complex migration, but transition the deployment execution away from local developer machines:
*   **Centralized CI/CD Runner:** Rather than running Fabric locally, we automate it inside a centralized pipeline (like GitHub Actions or Jenkins). Merging code to `main` triggers the pipeline to build the Docker image in the cloud, push the pre-built image to ECR, and then execute the Fabric deploy script to pull and run it on the host.
*   **Resource & Security Benefits:** This shifts the resource-heavy Docker builds off our host EC2 servers (keeping production CPU dedicated to serving traffic), removes local credential management from dev laptops, and guarantees deployment consistency.

### 2. New Projects
Any new services must natively target newer, dynamic platforms to phase out direct VM access and manual server management entirely.
*   **Modern Target Footprint:** New workloads must deploy to managed or serverless container runtimes (like **AWS ECS/Fargate**).
*   **CI/CD Native:** Deployments for new projects must be fully managed via automated CI/CD pipelines targeting the modern runtime natively, with no direct SSH/Fabric access required.
*   **Zero-Access Security:** Dropping direct SSH access and static credentials entirely. Operational tasks (like database migrations or debug sessions) are run as one-off Fargate tasks or via AWS ECS Exec under IAM role-based temporary permissions.

---

### Comparison: Legacy vs. New Projects Architecture

| Feature / Dimension | Legacy Footprint (EC2 + Fabric) | New Projects Footprint (AWS ECS / Fargate) |
| :--- | :--- | :--- |
| **Compute Type** | Dedicated virtual machines (EC2) | Serverless container instances (Fargate) |
| **Container Runtime** | Docker Compose | AWS Elastic Container Service (ECS) Tasks |
| **Orchestration / Deploy** | Python Fabric scripts executing over SSH | Native AWS ECS API rolling updates via CI/CD |
| **Credentials & Access** | SSH keys & AWS IAM (automated in runner) | AWS IAM Roles only (no static keys / SSH) |
| **Failover Capability** | Manual VM replacement (triggered by Prefect alerts) | Automated multi-AZ container replacement and health checks |
| **Scaling** | Manual via Terraform & pool deployment | Automated auto-scaling policies based on metrics |
| **OS Maintenance** | Team must patch and maintain EC2 instances | Handled entirely by AWS (Serverless) |
| **Traffic Routing** | Direct host port mapping | Application Load Balancer (ALB) dynamic target groups |

---

## Suggested Implementation Sequence

1. **Terraform** — stand up VPC, security groups, and EC2 instances; verify SSH access
2. **Docker Compose** — define all 20 services; deploy and verify manually on each instance
3. **Prefect** — connect services to Prefect Cloud; configure the dashboard and initial alerts
4. **Fabric** — write and test the deploy script against one service, then expand to all 20
5. **First real deploy** — use Fabric to push an update; confirm Prefect reflects it correctly
6. **Runbook** — document the deploy procedure and alert-response steps while the setup is fresh

Total setup time for a developer unfamiliar with these tools: approximately 3–5 weeks to production-ready, depending on the complexity of the service dependency graph.
