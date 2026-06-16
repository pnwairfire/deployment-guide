# Deployment Strategy: 20-Service Web Application
*A practical guide for project managers and developers*

---

## Overview

This document describes a deployment strategy for a suite of 20 interdependent dockerized web services running on AWS. The goals are straightforward: reliable uptime, easy updates every few days, visibility into system health, and a setup that a small team can actually understand and maintain.

The strategy uses four tools in combination, each handling a distinct layer of the system:

| Tool | Role |
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

---

### Docker Compose — Service Definition and Local Orchestration

Each EC2 instance runs one or more services defined in a `docker-compose.yml` file. Docker Compose describes which Docker images to run, what environment variables they receive, which ports they expose, and how they depend on each other.

**Key capabilities for this deployment:**

- **Restart policies** — `restart: always` means a crashed container automatically restarts without human intervention
- **Dependency ordering** — Compose knows that Service B requires Service A to be healthy before starting
- **Environment configuration** — each service receives its config (API keys, database URLs, feature flags) cleanly via environment variables
- **Pooled services** — services that need multiple instances can be scaled with a single setting

**Why this matters to the team:** The `docker-compose.yml` file is a readable, single-file description of everything running on a given instance. Any developer can look at it and understand what's deployed. Updates are applied by pulling a new Docker image and restarting the relevant service — no complex orchestration required.

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

---

## How the Tools Work Together

```
Developer pushes code
        ↓
Docker image built and pushed to registry (CI/CD)
        ↓
Fabric deploys new image to EC2 instances (rolling, one at a time)
        ↓
Docker Compose restarts affected service with new image
        ↓
Prefect detects the change; monitors health; alerts if problems arise
        ↓
Terraform (unchanged unless infrastructure itself needs updating)
```

Terraform operates on a different timescale from the rest — it's only run when the underlying infrastructure changes (adding an instance, changing a security rule, etc.), which happens rarely compared to application updates.

---

## Addressing Key Requirements

### Updates every few days
Fabric makes deployments repeatable and scriptable. After the initial setup, deploying a new version of any service is a single command. Services in pools are updated one at a time automatically.

### 20 services, some in pools
Each service has its own `docker-compose.yml` configuration. Pooled services run on multiple instances; the Fabric deploy script handles rolling updates across the pool. Terraform provisions the right number of instances for each pool.

### 99%+ uptime
Three mechanisms work together:
- Docker Compose `restart: always` recovers from crashes automatically
- Rolling deploys via Fabric ensure at least one instance in each pool is serving traffic during updates
- Prefect alerts catch problems quickly so the team can respond before users are significantly impacted

This approach does not provide *automatic* failover if an EC2 instance dies entirely — that would require Kubernetes or a similar platform. However, Prefect alerts mean the team is notified quickly, and restarting or replacing an instance is straightforward.

### Dashboard and alerts
Prefect Cloud provides both out of the box. The dashboard shows service status and dependency health; alerts route to whatever notification channel the team prefers.

### Complicated dependency graph
Prefect models service dependencies explicitly and reflects them in the dashboard. Docker Compose handles startup ordering within each instance. Security groups (managed by Terraform) enforce which services are permitted to communicate at the network level.

---

## What This Stack Does Not Provide

Being clear about limitations avoids surprises later:

- **Automatic instance failover** — if an EC2 instance becomes unresponsive, a human (or a script triggered by a Prefect alert) needs to intervene. Kubernetes handles this automatically but at significantly higher operational complexity.
- **Automatic capacity scaling** — adding instances in response to traffic spikes requires running Terraform and Fabric manually or building additional automation.
- **Sub-second deployment granularity** — this stack is well-suited to updates every few days or hours, not continuous deployment pipelines with dozens of releases per day.

For a small team without dedicated DevOps, these are reasonable trade-offs. The stack is understandable, debuggable, and maintainable by generalist developers.

---

## Suggested Implementation Sequence

1. **Terraform** — stand up VPC, security groups, and EC2 instances; verify SSH access
2. **Docker Compose** — define all 20 services; deploy and verify manually on each instance
3. **Prefect** — connect services to Prefect Cloud; configure the dashboard and initial alerts
4. **Fabric** — write and test the deploy script against one service, then expand to all 20
5. **First real deploy** — use Fabric to push an update; confirm Prefect reflects it correctly
6. **Runbook** — document the deploy procedure and alert-response steps while the setup is fresh

Total setup time for a developer unfamiliar with these tools: approximately 3–5 weeks to production-ready, depending on the complexity of the service dependency graph.
