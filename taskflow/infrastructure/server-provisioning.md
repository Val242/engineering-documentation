# TaskFlow — Application Server Provisioning

## Overview

The Application EC2 server is configured using an automated provisioning script rather than relying entirely on manual server configuration.

The goal is to make a newly created Application EC2 reproducible.

## Provisioning Responsibilities

The provisioning process prepares the server for TaskFlow by configuring:

* Operating-system dependencies
* Node.js
* PostgreSQL
* Application user
* Application directories
* File permissions
* Node Exporter
* systemd
* Required runtime configuration

## Server Identity

The application server is currently identified as:

```text
web01
```

The application is installed under:

```text
/opt/taskflow-api
```

A dedicated Linux user is used:

```text
taskflow
```

This avoids running the application as `root`.

## Provisioning Flow

```text
Fresh EC2
    │
    ▼
System Preparation
    │
    ▼
Runtime Installation
    │
    ├── Node.js
    └── PostgreSQL
    │
    ▼
Application User
    │
    ▼
Application Directories
    │
    ▼
Node Exporter
    │
    ▼
systemd
    │
    ▼
Application-Ready Server
```

## Node Exporter

Node Exporter runs as a dedicated service user and exposes host metrics on:

```text
:9100
```

Prometheus later scrapes this endpoint to collect system-level metrics.

## systemd

The TaskFlow API is registered as a systemd service:

```text
taskflow-api.service
```

The service provides:

* Automatic startup
* Process supervision
* Automatic restart after failure
* Controlled application lifecycle

## Reproducibility

The provisioning script reduces the amount of manual knowledge required to prepare a server.

This is important because an EC2 instance should ideally be replaceable without losing undocumented configuration knowledge.
