# CloudRecover — Daily Engineering Log

**Date:** October 22, 2026
**Project:** CloudRecover — Multi-Region AWS Disaster Recovery Platform
**Focus:** Ubuntu 26.04 provisioning, automated deployment, PostgreSQL TLS, Prisma migrations, and systemd

## 1. Objectives

Today's work focused on preparing the CloudRecover API server on Ubuntu 26.04 LTS and improving the provisioning script so that a new server can be configured with minimal manual intervention.

The main objectives were to:

- Automate application server provisioning.
- Install Node.js and application dependencies.
- Configure secure connectivity between the NestJS API and Amazon RDS PostgreSQL.
- Apply Prisma database migrations.
- Run the API as a systemd-managed service.
- Diagnose and fix an error in the provisioning script.

## 2. Provisioning the Application Server

I worked on the `setup-cloudrecover.sh` script to automate the setup of a new Ubuntu 26.04 server.

The script is responsible for configuring the application environment, preparing the application directory, installing dependencies, building the NestJS application, configuring database connectivity, and setting up the necessary system services.

### Key components

| Component               | Configuration              |
| ----------------------- | -------------------------- |
| Operating system        | Ubuntu 26.04 LTS           |
| Runtime                 | Node.js 24.x LTS           |
| Application framework   | NestJS                     |
| Application directory   | `/opt/cloudrecover-api`    |
| Application system user | `cloudrecover`             |
| Application service     | `cloudrecover-api.service` |
| Application port        | `3000`                     |
| Node Exporter port      | `9100`                     |
| Database                | Amazon RDS PostgreSQL      |
| ORM                     | Prisma 7.x                 |

The application repository is cloned from GitHub using sparse checkout, targeting the `cloudrecover-api` directory rather than checking out unrelated repository content.

The script installs dependencies using `npm ci`, generates the Prisma client, and builds the application before starting the service.

## 3. Configuring Secure RDS Connectivity

A major part of today's work was understanding how the application establishes a secure connection to Amazon RDS PostgreSQL.

### The original problem

The Prisma application initially encountered database connection errors.

Two important errors appeared during troubleshooting:

- `P1010` — the connection attempt was rejected because PostgreSQL required an encrypted connection.
- `P1011` — the TLS connection failed because the certificate chain could not be verified successfully.

These errors represented different stages of the connection problem: first, encryption was required; then, certificate validation needed to be configured correctly.

### Resolution

I downloaded the official Amazon RDS CA bundle to:

`/etc/ssl/certs/rds/global-bundle.pem`

The Prisma PostgreSQL adapter was configured with the database connection string and explicit TLS options:

- The RDS CA bundle is loaded from the filesystem.
- `rejectUnauthorized: true` enables certificate verification.
- The persistent `DATABASE_URL` does not contain `sslmode=require`, avoiding a conflict with the explicit TLS configuration supplied to the adapter.

The runtime database connection subsequently worked, and the API returned successful health-check and product responses.

### An important distinction

The Prisma application runtime and Prisma CLI migrations did not use exactly the same connection configuration.

For the runtime application, TLS was configured explicitly through `PrismaPg`.

For the migration command, `sslmode=require` was temporarily added to the environment of the migration process. This allowed the migration to execute without permanently modifying the application's `.env` file.

This distinction matters because configuration that works for a database migration command does not necessarily behave identically when used by the application's database adapter.

## 4. Environment Variables and Secrets

The provisioning script collects the database connection details when a `DATABASE_URL` has not already been configured.

The connection string includes the database hostname, database name, username, and password.

The `.env` file is stored at:

`/opt/cloudrecover-api/.env`

Its ownership and permissions were configured as follows:

- Owner: `cloudrecover`
- Group: `cloudrecover`
- Permissions: `600`

These permissions restrict access to the file to its owner.

The script also preserves an existing `.env` file when synchronizing application files, rather than replacing it with repository contents.

**Security lesson:** database credentials should remain in protected environment configuration and must never be printed in logs, committed to Git, or included in documentation.

## 5. Applying Prisma Database Migrations

After configuring database connectivity, I applied the Prisma migration to the RDS PostgreSQL database.

The migration that was successfully applied was:

`20261013164200_inventory`

The migration created the database structures required by the inventory API, including the tables used for products and stock movements, along with Prisma's migration-tracking table.

The command completed with the confirmation:

> All migrations have been successfully applied.

This demonstrated that the application could reach the database, authenticate, establish the required connection, and execute the pending schema migration.

It also reinforced the distinction between generating a Prisma client and applying migrations:

- **Client generation** creates the application-side code needed to interact with the database schema.
- **Migration deployment** applies version-controlled schema changes to the actual database.

Generating the client alone does not create the corresponding tables in RDS.

## 6. Running the API with systemd

The NestJS API was configured to run as a systemd service rather than as a manually launched Node.js process.

The service uses:

- A dedicated, non-root `cloudrecover` user.
- `/opt/cloudrecover-api` as its working directory.
- The `.env` file as its environment configuration.
- The compiled NestJS entry point, `dist/src/main.js`.
- Automatic restart on failure.
- Systemd security settings, including `NoNewPrivileges`, `PrivateTmp`, and `ProtectSystem`.

The service is named:

`cloudrecover-api.service`

The API's readiness endpoint was checked at:

`http://127.0.0.1:3000/health/ready`

The service passed the health check.

### Why this matters

Running the API under systemd provides lifecycle management. The operating system can start the service during boot, monitor its process, and restart it following certain failures.

Using a dedicated system user also reduces the privileges available to the application compared with running it as root.

## 7. The Provisioning Script Failure

The main automation issue encountered today was an error during Section 5 of `setup-cloudrecover.sh`.

The error was:

`Error: Cannot read package config /root/package.json: permission denied.`

### What caused it?

The script launched Node.js using `runuser` to switch to the `cloudrecover` user.

However, switching users does not automatically guarantee that the process starts in the application directory. The process inherited the `/root` working directory.

Node.js 24 attempted to resolve package configuration relative to that directory and encountered a permissions error.

The problem was therefore not evidence that the application's dependencies were missing or that the `.env` file was unreadable.

I verified that:

- `dotenv` could be loaded when executing Node.js from `/opt/cloudrecover-api`.
- The existing `DATABASE_URL` could be parsed.
- The URL contained the expected PostgreSQL hostname, port, and database name.
- The persistent connection string did not contain `sslmode`.
- The Prisma CLI was available.

The migration was then executed successfully after explicitly changing into the application directory.

### The fix

The provisioning script needs to specify the working directory when launching the Node.js validation and migration blocks.

Instead of relying on the inherited working directory, the corrected command uses:

`runuser --chdir="${APP_DIR}" -u "${APP_USER}" -- ...`

This ensures the process starts in `/opt/cloudrecover-api` before Node.js executes.

Both Node.js blocks in Section 5 need this correction:

1. The block that validates the `DATABASE_URL`.
2. The block that executes the Prisma migration.

The JavaScript logic inside those blocks does not need to change for this particular error.

### Engineering lesson

A process's user identity and its working directory are separate properties.

Changing the user determines whose permissions the process uses. Changing the working directory determines where relative path and package resolution begin.

Automation scripts must configure both explicitly when correctness depends on them.

## 8. Health Checks and Validation

The following checks were completed during troubleshooting and deployment:

| Check                                   | Result     |
| --------------------------------------- | ---------- |
| Node.js installed                       | Successful |
| Application dependencies available      | Successful |
| `.env` readable by the application user | Successful |
| Database URL parsed correctly           | Successful |
| RDS CA bundle downloaded                | Successful |
| Prisma migration deployed               | Successful |
| systemd API service started             | Successful |
| API readiness health check              | Passed     |

The API also has endpoints for liveness, product retrieval, stock movement retrieval, and metrics exposure.

These checks validate different layers of the deployment:

- **Process and runtime:** Node.js and the compiled application can execute.
- **Database connectivity:** the readiness endpoint verifies database availability.
- **Application functionality:** product endpoints test application-level operations.
- **Observability:** the metrics endpoint exposes application metrics, while Node Exporter exposes host-level metrics.

## 9. What I Learned

### 9.1 Troubleshooting should isolate the failing layer

A database connection error, a filesystem permissions error, and a process startup error require different investigations.

Rather than changing the database configuration immediately, I tested dependency resolution, environment-variable parsing, file permissions, and migration execution separately.

This helped identify the actual cause of the provisioning failure.

### 9.2 TLS encryption and certificate verification are different concerns

Requiring an encrypted connection does not, by itself, establish that the server's certificate chain is trusted.

A secure TLS connection must both encrypt traffic and validate the server certificate appropriately. The RDS CA bundle allows the client to verify the certificate chain.

### 9.3 Runtime configuration and CLI configuration can differ

The application uses `PrismaPg` with explicit TLS options, while the migration command temporarily adds `sslmode=require` to its own environment.

This taught me to distinguish between the configuration of the running application and the configuration of a separate operational command.

### 9.4 Database migrations are part of deployment

Deploying a backend is not just a matter of copying code and starting Node.js. The target database must also be compatible with the application's expected schema.

Migration execution is therefore an important deployment step and should be handled deliberately in the provisioning or release process.

### 9.5 Idempotent provisioning requires careful state handling

A provisioning script may run on an existing server, not only on a brand-new machine. It must therefore handle existing environment files, service definitions, dependencies, and database state carefully.

Preserving `.env` and using Prisma's migration deployment command are examples of approaches that help avoid unnecessary changes or repeated schema creation.

However, the full script still needs fresh-instance testing before I can consider the provisioning process completely validated.

## 10. Remaining Work

- [x] Update both `runuser` commands in Section 5 to set the application working directory.
- [ ] Test the corrected script from the beginning on a fresh Ubuntu 26.04 instance.
- [ ] Verify that the script completes without manual intervention.
- [ ] Confirm the application starts after a reboot.
- [ ] Verify the liveness, readiness, product, and metrics endpoints.
- [ ] Confirm Node Exporter is running and accessible only from permitted network sources.
- [ ] Confirm Prometheus can scrape the application and host metrics from the intended monitoring server.
- [ ] Review AWS security group rules and resource costs before leaving the environment running.

## Summary

Today's work advanced CloudRecover's application deployment on Ubuntu 26.04. I configured the Node.js application environment, resolved RDS TLS configuration issues, successfully applied the Prisma inventory migration, and confirmed that the systemd-managed API passed its readiness check.

The most important debugging lesson was that switching to a service user does not necessarily change the working directory. The provisioning failure came from Node.js starting in `/root` rather than `/opt/cloudrecover-api`.

The next step is to incorporate that working-directory correction and validate the complete provisioning process on a fresh instance.
