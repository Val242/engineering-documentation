# Kubernetes Learning Journal — October 27, 2026

**Main focus:** Deployments, ReplicaSets, rolling updates, rollback, and how application management connects to Kubernetes networking.

## 1. Session Overview

Today I continued studying Kubernetes and clarified several important concepts, especially the relationship between Deployments and ReplicaSets.

Today's progress was primarily **conceptual understanding and discussion**, not a completed hands-on Deployment exercise.

## 2. Deployments, ReplicaSets, Pods, and Containers

I now understand the management hierarchy:

**Deployment → ReplicaSet → Pods → Containers**

- **Deployment:** Describes the desired application state and manages the ReplicaSets used to achieve it. It supports rolling updates, rollout history, and rollback.
- **ReplicaSet:** Maintains the assigned number of Pods. If a managed Pod is deleted, the ReplicaSet creates a replacement.
- **Pod:** Runs one or more containers and is scheduled onto a worker node.
- **Container:** Runs the actual application process, such as NGINX or a NestJS API.

A ReplicaSet replaces **Pods, not worker nodes**. Worker-node recovery and scaling belong to the cluster infrastructure and node-management mechanisms.

## 3. Why Use a Deployment Instead of Creating a ReplicaSet Directly?

A ReplicaSet is sufficient for maintaining a number of Pods, but it does not orchestrate application-version updates by itself.

A Deployment builds on ReplicaSets by coordinating:

- Desired application replica counts.
- Controlled rolling updates when the Pod template changes.
- Rollout status and revision history.
- Rollback to an earlier application revision.

Deployments are useful in development and testing as well as production; they are not simply “production ReplicaSets.”

## 4. Multiple ReplicaSets Under One Deployment

One Deployment can manage multiple ReplicaSets, generally because the application has gone through multiple revisions.

For example, if the desired replica count is **5** and an update to version 4 has completed:

| ReplicaSet | Application revision | Desired Pods after rollout |
|---|---|---:|
| RS-1 | v1 | 0 |
| RS-2 | v2 | 0 |
| RS-3 | v3 | 0 |
| RS-4 | v4 (current) | 5 |

The Deployment's `replicas: 5` means **five desired application Pods overall**, not five per ReplicaSet.

Older ReplicaSets are commonly kept with zero replicas for rollback history, subject to retention settings. During an active rolling update, **old and new ReplicaSets may both have running Pods**.

A change to the **Pod template**, such as updating an image, normally produces a new ReplicaSet. Changing only the replica count scales the existing active ReplicaSet without creating a new application revision.

## 5. How Rolling Updates Work

Suppose five NGINX Pods run version v1, and the Deployment's Pod template is updated to use version v2.

1. The Deployment Controller creates or reuses the ReplicaSet corresponding to the new Pod template.
2. New Pods start running the updated application version.
3. As new Pods become available, the Deployment gradually scales down the old ReplicaSet.
4. When the rollout succeeds, the new ReplicaSet normally maintains all five desired Pods.
5. Previous ReplicaSets may remain at zero replicas for rollback.

During the transition, two versions can be serving requests at the same time. Kubernetes does **not** modify the existing containers in place to perform a standard Deployment rolling update.

If new Pods fail their readiness checks, the rollout may stall. Kubernetes ordinarily does **not** automatically roll back solely because the rollout failed; a human or external automation must initiate rollback.

## 6. Understanding `maxUnavailable` and `maxSurge`

These Deployment rolling-update settings control availability and temporary additional capacity.

- **`maxUnavailable`:** How many of the desired replicas may be unavailable during the rollout.
- **`maxSurge`:** How many additional Pods may be created beyond the desired replica count during the rollout.

For **5 replicas**, `maxUnavailable: 0` and `maxSurge: 2` mean:

- At least **5 Pods should remain available** throughout the planned rollout.
- The rollout can temporarily have up to **7 non-terminating Pods**.

For **10 replicas**, `maxUnavailable: 2` and `maxSurge: 3` mean:

- Minimum available Pods: **10 − 2 = 8**.
- Maximum non-terminating Pods: **10 + 3 = 13**.

I initially reversed what the two numbers represented, then understood the distinction: *unavailable* concerns minimum availability; *surge* concerns temporary extra capacity. Terminating Pods may briefly cause the total visible Pod count to exceed the surge-based limit.

## 7. How Deployments Connect to Services and Ingress

Kubernetes separates application lifecycle management from traffic routing.

**Management path:**

`Deployment → ReplicaSet → Pods`

**Traffic path (conceptual):**

`External client → Load Balancer → NGINX Ingress Controller → Application Service → Eligible Application Pods`

The NGINX Ingress Controller itself normally runs as Pods managed by **its own Deployment**, separate from my application's Deployment.

The application Service does not point to the ReplicaSet or Deployment. It finds eligible Pods using **label selectors**. This means a Service can route traffic to healthy Pods belonging to both old and new ReplicaSets during a rolling update if the Pods have matching labels.

Ingress resources define HTTP/HTTPS routing rules, such as routing `/` to WordPress or `/api` to a backend Service. The installed Ingress Controller implements those rules.

The exact network forwarding can vary; an NGINX controller may proxy directly to backend Pod endpoints discovered through Kubernetes.

## 8. Concepts Revised From Earlier Sessions

I revisited how these Kubernetes resources relate to the Deployment model:

| Resource | Current understanding |
|---|---|
| **ClusterIP Service** | Provides a stable internal endpoint for eligible Pods. |
| **NodePort Service** | Exposes a Service using a port on cluster nodes, subject to network access. |
| **LoadBalancer Service** | Uses supported cloud or infrastructure integration to expose a Service externally. |
| **Ingress** | Specifies HTTP/HTTPS routes to application Services; a controller is needed to implement them. |
| **ConfigMap** | Holds non-sensitive application configuration. |
| **Secret** | Holds sensitive configuration, requiring appropriate access controls. |
| **Pod** | Runs one or more application containers on a worker node. |

These were **revisions of concepts previously studied**, not new practical implementations completed today.

## 9. Current Understanding and Gaps

I now have a clearer conceptual understanding of:

- The Deployment → ReplicaSet → Pod hierarchy.
- Why Deployments provide more functionality than managing ReplicaSets directly.
- How application revisions produce ReplicaSets.
- How old and new ReplicaSets coexist during a rolling update.
- How Services can continue routing traffic during updates without being tied to any one ReplicaSet.
- The basic purpose of rollout availability settings.

I still need more practical repetition with Deployment YAML, scaling, readiness probes, rollout inspection, and rollback behavior to make these concepts automatic.

## 10. Next Practical Session

When I return to this topic, I can deploy an existing NGINX image using a Deployment, then:

1. Create multiple replicas and inspect the Pods and ReplicaSet.
2. Add a Service that selects the Pods through labels.
3. Change the image tag and observe the new ReplicaSet during rollout.
4. Inspect rollout status and available Pods.
5. Practise rollback and compare the ReplicaSets afterward.

## 11. Personal Reflection

I am beginning to recognize that understanding Kubernetes comes from repeatedly revisiting the relationships between its components, rather than trying to master everything in one sitting. I have a conceptual foundation in Services, Ingress, ConfigMaps, Secrets, Pods, and now Deployments.

The next step is to strengthen that foundation with more practice. Consistent documentation means I can return to today's understanding whenever I need to revise it.

**Session takeaway:** A ReplicaSet maintains Pods; a Deployment manages the application lifecycle and coordinates ReplicaSets across updates. Services independently provide stable network access to the resulting Pods.