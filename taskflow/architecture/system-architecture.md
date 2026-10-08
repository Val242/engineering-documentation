# TaskFlow — System Architecture

## Overview

TaskFlow is designed as an AWS-hosted application platform with a dedicated application server and separate monitoring infrastructure.

The application is deployed directly onto an EC2 instance rather than using containers or Kubernetes.

## High-Level Architecture

```text
                         Internet
                            │
                          HTTPS
                            │
                            ▼
                    ┌─────────────────┐
                    │ Application EC2 │
                    │                 │
                    │     Nginx       │
                    │       │         │
                    │       ▼         │
                    │    NestJS       │
                    │      :3000      │
                    │       │         │
                    │       ▼         │
                    │   PostgreSQL    │
                    └────────┬────────┘
                             │
                       Private Network
                             │
              ┌──────────────┴──────────────┐
              │                             │
              ▼                             ▼
      ┌───────────────┐             ┌───────────────┐
      │ Prometheus EC2│             │  Grafana EC2   │
      │               │             │               │
      │ Prometheus    │────────────▶│ Grafana        │
      │ Alertmanager  │   PromQL    │               │
      └───────────────┘             └───────────────┘
```

## Application Request Flow

```text
Client
  │
  │ HTTPS :443
  ▼
Nginx
  │
  │ HTTP :3000
  ▼
NestJS
  │
  │ Prisma
  ▼
PostgreSQL
```

Nginx acts as the public-facing reverse proxy.

NestJS handles application requests.

PostgreSQL provides persistent application storage.

## Monitoring Flow

The application exposes two different categories of metrics.

### Application Metrics

```text
NestJS :3000/metrics
        │
        ▼
    Prometheus
```

These include HTTP request metrics and Node.js runtime metrics.

### Host Metrics

```text
Node Exporter :9100
        │
        ▼
    Prometheus
```

These provide system-level metrics such as CPU, memory, and filesystem statistics.

## Application Server

The Application EC2 hosts:

* Nginx
* NestJS
* PostgreSQL
* Node Exporter
* systemd

The NestJS application listens privately on port `3000`.

Node Exporter listens on port `9100`.

PostgreSQL is local to the application server and is not exposed publicly.

## Monitoring Servers

Prometheus and Alertmanager run on the Prometheus EC2.

Grafana runs on a separate Grafana EC2.

Grafana queries Prometheus using PromQL.

Prometheus sends alert notifications to Alertmanager.

## Administrative Access

Private infrastructure is administered through AWS Systems Manager rather than exposing unnecessary SSH access to the public internet.

Grafana and Prometheus remain private and can be accessed through controlled administrative access when required.

## Design Principles

The architecture emphasizes:

* Infrastructure as code
* Least-privilege networking
* Private internal services
* Reproducible server configuration
* Process supervision
* Application observability
* Operational troubleshooting
* Separation of application and monitoring responsibilities
