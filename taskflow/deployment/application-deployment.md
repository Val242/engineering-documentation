# TaskFlow — Application Deployment

## Overview

The TaskFlow NestJS API is deployed directly onto the Application EC2 instance.

Docker and Kubernetes are intentionally not used in this deployment phase.

## Application Location

```text
/opt/taskflow-api
```

The application runs under:

```text
taskflow
```

rather than the root user.

## Installation

Application dependencies are installed using:

```bash
npm ci
```

The lockfile is retained to ensure deterministic dependency installation.

## Prisma

After installing dependencies:

```bash
npx prisma generate
```

generates the Prisma Client.

Database migrations are applied using:

```bash
npx prisma migrate deploy
```

## Build

The NestJS application is compiled with:

```bash
npm run build
```

The resulting entry point is:

```text
dist/src/main.js
```

## systemd

The application is managed by:

```text
taskflow-api.service
```

The service executes:

```text
/usr/bin/node /opt/taskflow-api/dist/src/main.js
```

The service is configured to restart the application after failure.

## Deployment Verification

After deployment, the following endpoints are verified:

```text
GET /health/live
GET /health/ready
GET /metrics
```

These provide basic verification that:

* The process is running
* The database is accessible
* Metrics are available

## Deployment Troubleshooting

During redeployment, stale or incomplete `node_modules` contents caused npm extraction errors.

The dependency directory was removed and rebuilt using:

```bash
npm ci
```

A permissions issue also occurred because the `taskflow` user did not initially have a writable home/npm cache directory.

The server environment was corrected before repeating the installation.

This demonstrated the importance of running application installation and execution under the same dedicated application identity used by the service.
