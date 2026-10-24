# TaskFlow — Engineering Log — October 7, 2026

## Focus

Infrastructure automation, application deployment, observability, troubleshooting, and Git workflow.

## Work Completed

### Infrastructure

* Fully managed the TaskFlow AWS infrastructure with Terraform.
* Continued operating the Application EC2 as infrastructure managed through code.
* Created an automated provisioning script for the Application EC2.

### Server Provisioning

The provisioning script automated the preparation of the Application EC2, including:

* System dependencies
* Node.js
* PostgreSQL
* Application user
* Permissions
* Node Exporter
* systemd
* Application server prerequisites

### Application Deployment

* Deployed the NestJS application to `/opt/taskflow-api`.
* Configured the dedicated `taskflow` user.
* Corrected the systemd entry point to `dist/src/main.js`.
* Rebuilt the application.
* Generated Prisma Client.
* Applied Prisma migrations.

### Troubleshooting

Resolved dependency installation problems involving:

```text
TAR_ENTRY_ERROR
EACCES
```

The underlying causes were stale dependency files and npm/user permission issues.

### Health Checks

Implemented:

```text
/health/live
/health/ready
```

The readiness endpoint verifies PostgreSQL connectivity.

### Application Metrics

Implemented:

```text
/metrics
```

using `prom-client`.

The endpoint successfully exposed:

* Node.js runtime metrics
* Garbage collection metrics
* HTTP request counters
* HTTP request duration histograms

### Metrics Verification

Generated real HTTP traffic and verified that the corresponding metrics increased.

### Networking

Detected an Application EC2 IP address change and traced the resulting configuration issue to the changed IP.

The affected configuration was corrected and verified.

### Git

A GitHub push initially failed with a remote server error.

The repository was synchronized using:

```bash
git pull --rebase origin main
```

The subsequent push succeeded.

## Result

The day moved TaskFlow from basic infrastructure and application deployment toward a reproducible and observable operating environment.

The major operational chain established was:

```text
Terraform
    ↓
AWS Infrastructure
    ↓
Automated Server Provisioning
    ↓
Application Deployment
    ↓
systemd
    ↓
Health Checks
    ↓
Prometheus Metrics
    ↓
Monitoring Infrastructure
```

The day also provided practical troubleshooting experience across:

* Linux
* npm
* Node.js
* NestJS
* PostgreSQL
* systemd
* AWS networking
* Terraform
* Git
* Prometheus instrumentation
