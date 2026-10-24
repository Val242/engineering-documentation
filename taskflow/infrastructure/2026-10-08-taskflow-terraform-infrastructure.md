# TaskFlow — Terraform Infrastructure

## Overview

TaskFlow's AWS infrastructure is managed entirely through Terraform.

Terraform is used as the infrastructure-as-code layer responsible for defining and managing cloud resources.

The objective is to make infrastructure:

* Reproducible
* Version controlled
* Reviewable
* Consistent
* Easier to recreate

## Infrastructure Responsibility

Terraform manages the cloud infrastructure layer.

Conceptually:

```text
Terraform
    │
    ▼
AWS
    │
    ├── EC2
    ├── Networking
    ├── Security Groups
    └── Supporting Resources
```

Terraform does not replace the application deployment process.

The responsibilities are separated:

```text
Terraform
    → Infrastructure

Provisioning Script
    → Server configuration

Deployment Process
    → Application deployment

systemd
    → Application lifecycle

Prometheus/Grafana
    → Observability
```

## Infrastructure as Code Workflow

```text
Terraform Configuration
        │
        ▼
terraform plan
        │
        ▼
Review Infrastructure Changes
        │
        ▼
terraform apply
        │
        ▼
AWS Resources
```

## State Management

Terraform state is treated as infrastructure management data and is not committed to source control.

The `.terraform` directory is also excluded from version control.

Sensitive Terraform variable files are excluded from the repository where appropriate.

## Benefits

Using Terraform provides:

* Reproducible environments
* Declarative infrastructure
* Change tracking through Git
* Easier recovery
* Reduced manual configuration
* Foundation for self-service environment creation

Terraform also provides the foundation for the later TaskFlow requirement where users can request isolated environments programmatically.
