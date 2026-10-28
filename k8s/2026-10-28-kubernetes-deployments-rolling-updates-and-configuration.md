# Kubernetes Learning Journal — October 28, 2026

**Main focus:** Deployment rolling-update strategies and integrating Kubernetes resources in a practical NGINX exercise.

## 1. Session Overview

Today I revised how Kubernetes Deployments coordinate rolling updates and explored the roles of `maxSurge` and `maxUnavailable`. I then started an exercise combining Deployments, ReplicaSets, Pods, Services, Ingress, ConfigMaps, and Secrets.

I created the five Kubernetes manifest files for the exercise. The horizontal scaling, rolling update, and rollback portions are planned for the next session. I have not yet reported successful deployment or runtime verification of these manifests.

## 2. Deployments and ReplicaSets

A Deployment manages the desired application state by controlling ReplicaSets, which maintain the required number of Pods. Each Pod runs one or more containers.

**Management hierarchy:** Deployment → ReplicaSet → Pods → Containers.

When a Deployment's Pod template changes, such as when the container image or a template annotation is updated, the Deployment normally creates a new ReplicaSet and gradually transitions from the previous ReplicaSet to the new one. During a rollout, both old and new ReplicaSets may temporarily have active Pods. After a successful rollout, the new ReplicaSet normally holds the desired replica count, while earlier ReplicaSets may remain at zero replicas for rollback history.

Changing only the number of replicas usually scales the current ReplicaSet rather than creating a new revision.

## 3. Understanding `maxSurge` and `maxUnavailable`

These settings apply to a Deployment using the `RollingUpdate` strategy:

- **`maxSurge`** limits how many additional non-terminating Pods may be created above the desired replica count during a rollout.
- **`maxUnavailable`** limits how many of the desired replicas may be unavailable during the rollout.

For the exercise's initial configuration:

| Setting | Value | Implication |
|---|---:|---|
| Desired replicas | 3 | Normal Pod count |
| `maxSurge` | 1 | Up to 4 non-terminating Pods during a rollout |
| `maxUnavailable` | 0 | At least 3 available Pods should be maintained during the planned rollout |

When the Deployment is later scaled to five replicas, the same strategy would allow up to **6 non-terminating Pods** while maintaining **at least 5 available Pods** during a planned rollout. Pods that are terminating can temporarily cause the total visible Pod count to exceed the surge-based limit.

These settings manage planned rollout behavior; they do not guarantee availability during failures outside the rollout.

## 4. Integrating Kubernetes Resources

The practice exercise uses the public `nginx:1.28` container image and brings several Kubernetes concepts together.

**Application management:** Deployment → ReplicaSet → NGINX Pods.

**Traffic routing (conceptual):** Browser → external load balancer → NGINX Ingress Controller → application backend selected by the Ingress rule → NGINX Pods.

The application Service uses a label selector to find ready Pods. The Ingress rule references the Service by name and Service port, not the Deployment or ReplicaSet. The Ingress Controller can proxy to eligible Pod endpoints discovered through Kubernetes networking.

The ConfigMap and Secret provide configuration to containers through environment variables. NGINX does not automatically use these arbitrary exercise variables to change its server behavior; they are included to practise configuration injection.

## 5. Manifest Files Created

I prepared five separate manifest files for the exercise:

| File | Kubernetes resource | Intended configuration |
|---|---|---|
| `configmap.yaml` | ConfigMap | `webapp-config`: `APP_ENV=staging`, `LOG_LEVEL=info` |
| `secret.yaml` | Secret | `webapp-secret`: dummy `API_TOKEN` |
| `deployment.yaml` | Deployment | `webapp-deployment`, image `nginx:1.28`, 3 replicas, rolling-update settings |
| `service.yaml` | Service | `web-service`, ClusterIP, port 80 to target port 80, selector `app=webapp` |
| `ingress.yaml` | Ingress | `webapp-ingress`, class `nginx`, hostless `/` route to `web-service:80` |

The Deployment is expected to inject the ConfigMap and Secret values into its containers, and its Pod template should carry the `app: webapp` label so the Service can find them.

**Status:** The files have been created. Their application to the cluster and the expected runtime behavior have not yet been confirmed in today's report.

## 6. Connection to Earlier Lessons

This exercise consolidates several concepts already studied:

- **Pod:** Runs the application container.
- **ReplicaSet:** Maintains the desired number of Pods.
- **Deployment:** Manages ReplicaSets, scaling, rolling updates, and rollout history.
- **ClusterIP Service:** Provides stable internal access to eligible Pods through label selection.
- **Ingress and Ingress Controller:** Define and implement HTTP routing to application backends.
- **ConfigMap:** Stores non-sensitive configuration.
- **Secret:** Stores sensitive values; real credentials still require secure handling and access controls.

The important architectural distinction is that Deployments manage the **lifecycle of application Pods**, while Services and Ingress provide **network access to those Pods**.

## 7. Next Session: Horizontal Scaling and Rollouts

The next session begins with the **horizontal scaling section** of the exercise:

1. Confirm that the manifests have been applied successfully and that the initial three Pods are ready.
2. Scale the Deployment from three to five replicas and inspect the resulting ReplicaSet and Pods.
3. Confirm that the Service discovers the additional ready endpoints.
4. Delete one Pod and observe how the ReplicaSet restores the desired count.
5. Change a harmless Pod-template annotation to trigger a rolling update and inspect the old and new ReplicaSets.
6. Observe the behavior of `maxSurge: 1` and `maxUnavailable: 0` with five desired replicas.
7. Practise rolling back to the previous Deployment revision, and verify that the Service and Ingress remain unchanged.

## 8. Reflection

Today's progress was in connecting individual Kubernetes concepts into a single deployment exercise. I reviewed the rollout controls and created the resources needed for an application that combines configuration injection, replica management, internal networking, and external routing.

The next step is to test the actual cluster behavior, rather than assuming the manifests work because they have been written.

**Key takeaway:** Deployments manage how application Pods are created, scaled, and updated; Services and Ingress let clients continue reaching eligible Pods through stable routes as those Pods change.
