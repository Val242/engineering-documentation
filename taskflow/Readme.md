# TaskFlow — Engineering Documentation

This repository contains the engineering documentation for **TaskFlow**, a self-service AWS deployment platform designed to demonstrate production-oriented software engineering, infrastructure automation, deployment, observability, and operational practices.

The implementation source code is maintained separately from this repository.

## Documentation Scope

This repository documents:

* System architecture
* AWS infrastructure
* Terraform infrastructure as code
* Server provisioning
* Application deployment
* Health checks
* Prometheus metrics
* Monitoring architecture
* Operational troubleshooting
* Engineering decisions
* Development and operational history

## Repository Structure

```text
architecture/
├── system-architecture.md
├── infrastructure-architecture.md
└── observability-architecture.md

infrastructure/
├── terraform.md
└── server-provisioning.md

deployment/
└── application-deployment.md

observability/
└── application-metrics.md

operations/
└── troubleshooting.md

engineering-log/
└── 2026-10-07.md
```

## Documentation Philosophy

The documentation is organized around the lifecycle of the system:

```text
Architecture
     ↓
Infrastructure
     ↓
Server Provisioning
     ↓
Application Deployment
     ↓
Observability
     ↓
Operations
```

The engineering logs preserve the chronological development process, while the technical documentation presents the final engineering decisions and procedures in a structured form.

## Project Infrastructure

TaskFlow uses AWS infrastructure managed through Terraform.

The current architecture includes:

* Application EC2
* Prometheus EC2
* Grafana EC2
* Private networking
* Security groups
* AWS Systems Manager for private administrative access

The application server runs:

* Nginx
* NestJS
* PostgreSQL
* systemd
* Node Exporter

The monitoring infrastructure runs:

* Prometheus
* Alertmanager
* Grafana

The project intentionally does not use Docker or Kubernetes in this phase.

## Documentation Status

Documentation is maintained alongside the evolution of the project and is progressively refined from engineering notes into stable technical documentation.
