# TaskFlow — Operational Troubleshooting

## Purpose

This document records operational issues encountered while deploying and operating TaskFlow and the methods used to diagnose them.

The goal is not only to record fixes, but to preserve the reasoning used to identify the underlying causes.

---

## 1. Incorrect NestJS Build Path

### Symptom

systemd failed because it attempted to execute:

```text
dist/main.js
```

The actual NestJS build output was:

```text
dist/src/main.js
```

### Diagnosis

The compiled files were inspected after running the build.

### Resolution

The systemd service was updated to:

```text
ExecStart=/usr/bin/node /opt/taskflow-api/dist/src/main.js
```

Then systemd was reloaded and the service restarted.

### Lesson

Never assume the framework's build output path.

Verify the generated artifacts before configuring process managers.

---

## 2. npm Dependency Installation Failure

### Symptoms

npm produced:

```text
TAR_ENTRY_ERROR ENOENT
```

and:

```text
EACCES: permission denied
```

### Diagnosis

The existing `node_modules` directory was incomplete/stale, and the `taskflow` user lacked a writable home/npm cache location.

### Resolution

The stale dependency directory was removed.

A writable home and npm cache were created for the application user.

Dependencies were then reinstalled with:

```bash
npm ci
```

### Lesson

Dependency installation failures can originate from the server environment rather than the dependency definitions themselves.

---

## 3. IP Address Change

### Symptom

A previously working infrastructure/monitoring configuration stopped behaving as expected.

### Diagnosis

The Application EC2 IP address had changed.

The affected configuration was still referencing the previous address.

### Resolution

The changed IP was identified and the dependent configuration was corrected.

The system was then tested again.

### Lesson

Infrastructure components that directly reference ephemeral IP addresses can become invalid when underlying resources change.

This also identified a future requirement for dynamic monitoring target registration or service discovery.

---

## 4. GitHub Push Failure

### Symptom

A push to GitHub returned:

```text
remote: Internal Server Error
```

### Diagnosis

The issue was not caused by committing from a subdirectory.

The local branch needed to synchronize with the remote branch.

### Resolution

The repository was synchronized using:

```bash
git pull --rebase origin main
```

The changes were then pushed:

```bash
git push origin main
```

### Lesson

A rebase can synchronize local commits with remote changes while maintaining a clean linear history.

---

## Operational Troubleshooting Method

The incidents encountered during the project reinforced a consistent troubleshooting process:

```text
Observe
  ↓
Collect evidence
  ↓
Identify the changed component
  ↓
Trace dependencies
  ↓
Determine root cause
  ↓
Apply targeted fix
  ↓
Verify recovery
```

This approach is preferred over repeatedly changing configuration without understanding the failure.
