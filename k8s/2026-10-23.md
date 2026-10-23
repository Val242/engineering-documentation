# Kubernetes Learning Journal — October 23, 2026

## 1. Session Overview

**Main focus:** Kubernetes ConfigMaps, Secrets, environment-variable injection, and practical resource management.

Today, I moved beyond learning Kubernetes concepts theoretically and practised configuring an existing NGINX container using Kubernetes resources. I created a ConfigMap and a Secret, referenced both from a Pod, and used Kubernetes inspection commands to verify the configuration.

## 2. Concepts Learned

### ConfigMaps

A ConfigMap stores non-sensitive configuration separately from an application container image.

Examples include:
- Application environment, such as development or production.
- Logging levels.
- Database hostnames and other non-sensitive configuration.

I created a ConfigMap named `nginx-config` using `configmap.yaml`.

The ConfigMap contains the `app_environment` key, which my Pod references to populate the `APP_ENV` environment variable.

**Key lesson:** Configuration can be changed independently of the container image, making it easier to deploy the same image across different environments.

### Secrets

A Kubernetes Secret is intended to store sensitive configuration, such as passwords, API tokens, and credentials.

I created a Secret named `nginx-secret` using `secret.yaml`. It contains the `app_password` key, which the Pod references to populate `APP_PASSWORD`.

My first attempt failed because the Secret's `data` field contained invalid Base64. I corrected this by encoding the value:

```bash
echo -n "practice-only" | base64
```

The command returned:

```text
cHJhY3RpY2Utb25seQ==
```

I updated the manifest and successfully created the Secret.

**Key lessons:**
- Kubernetes expects valid Base64-encoded values when using the Secret `data` field.
- The `stringData` field can accept plain-text values in a manifest.
- Base64 is encoding, not encryption. Secrets still require appropriate access controls and security configuration.

### Environment-variable injection

I configured the NGINX Pod to obtain its environment variables from the ConfigMap and Secret rather than hardcoding the values directly into the Pod specification.

The `kubectl describe pod nginx` output confirmed the references:

- `APP_ENV` references `nginx-config`, key `app_environment`.
- `APP_PASSWORD` references `nginx-secret`, key `app_password`.

This helped me understand that Kubernetes can supply configuration to an existing container without requiring me to rebuild its image.

One distinction is important: NGINX does not automatically use arbitrary environment variables to configure its web server. In this exercise, the variables demonstrate Kubernetes configuration injection.

## 3. Practical Work Completed

### Resources created

| Resource | Name | Purpose |
|---|---|---|
| ConfigMap | `nginx-config` | Stores non-sensitive configuration |
| Secret | `nginx-secret` | Stores sensitive configuration |
| Pod | `nginx` | Runs the NGINX container with injected configuration |

The Pod uses the existing image `nginx:1.14.2`.

I verified that the Pod was `Running` and `Ready`, and that Kubernetes had successfully resolved both configuration references.

### Commands practised

Inspect Pods:

```bash
kubectl get pods
kubectl describe pod nginx
```

Create resources from YAML manifests:

```bash
kubectl create -f configmap.yaml
kubectl create -f secret.yaml
kubectl create -f pod.yaml
```

Inspect ConfigMaps and Secrets:

```bash
kubectl get cm
kubectl get secrets
```

Delete a Pod:

```bash
kubectl delete pod nginx
```

Encode a value for a Secret's `data` field:

```bash
echo -n "practice-only" | base64
```

I also encountered a command typo while attempting to describe Pods. Correcting the command reinforced the importance of accurate Kubernetes CLI syntax.

## 4. Important Operational Lesson: Configuration Updates

I learned that changing a ConfigMap or Secret does not automatically update environment variables inside an already-running container.

When environment variables are injected at container startup, the running container continues using its existing values after the source resource changes.

To apply updated environment-variable values, the affected Pods generally need to be recreated.

Mounted ConfigMap and Secret files behave differently: Kubernetes can update their contents eventually, although the application must reload the configuration to use it. Mounts using `subPath` do not receive those automatic updates.

For applications that need reliable updates, a Deployment can manage Pod replacement through a rollout restart:

```bash
kubectl rollout restart deployment/<deployment-name>
```

This is one reason Deployments are generally preferable to standalone Pods for applications that need replicas, recovery, and controlled updates.

**Status:** I have learned the principle, but I have not yet completed the practical ConfigMap-update experiment or verified the restart behavior in this session.

## 5. Troubleshooting and Verification

The main issue encountered was the invalid Base64 value in `secret.yaml`.

My troubleshooting process was:

1. Attempt to create the Secret.
2. Read the Kubernetes API error.
3. Encode the intended value using `base64`.
4. Update the manifest.
5. Reapply the manifest successfully.
6. Inspect the Pod to verify that both configuration references were resolved.

This reinforced the value of reading error messages and using `kubectl describe` to inspect the actual state of a resource rather than relying only on the manifest.

## 6. Mental Model

The relationship between the resources can be understood as follows:

- **Container image:** Provides the application and its executable.
- **ConfigMap:** Supplies non-sensitive configuration.
- **Secret:** Supplies sensitive configuration.
- **Pod:** References those resources and runs the container with the resulting configuration.
- **Deployment:** Manages replicated Pods and supports controlled replacement and rollout operations.

Separating application packaging from configuration makes deployments more reusable and easier to manage across environments.

## 7. Next Learning Objectives

When I resume, I will:

1. Inspect the actual environment variables inside the running NGINX container using `kubectl exec`.
2. Update the ConfigMap and observe whether the running container's environment changes.
3. Recreate the Pod and verify the updated environment value.
4. Move from a standalone Pod to a Deployment with two replicas.
5. Practise updating configuration and restarting a Deployment's Pods.
6. Review how Kubernetes Secret access controls and encryption at rest contribute to security.

## 8. Reflection

Today's session connected Kubernetes configuration concepts with practical implementation. I created resources, corrected a real manifest error, and verified the resulting Pod configuration.

The most important takeaway is that Kubernetes resources have distinct responsibilities: ConfigMaps and Secrets provide configuration, Pods consume it, and Deployments manage application replicas and their lifecycle.

I have not mastered every aspect of configuration management yet, but I now have a practical foundation to build on through continued experimentation and troubleshooting.