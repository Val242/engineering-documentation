# Kubernetes Learning Journal — October 29, 2026

**Focus:** Ingress routing, Deployment labels, horizontal scaling, ReplicaSets, self-healing, and internal Kubernetes networking.

## 1. Overview

Today I continued the Kubernetes practical exercise I started previously. I worked with an NGINX application managed by a Deployment and experimented with its Ingress, Pod replicas, and recovery behavior. I also clarified how application components communicate through Services in a multi-tier Kubernetes architecture.

The major achievement was **observing Kubernetes reconciliation in practice**: after increasing the desired replica count, Kubernetes created more Pods; after deleting Pods, Kubernetes created replacements to restore the desired state.

## 2. Understanding Labels in a Deployment

I clarified the three different locations where labels appear in a Deployment manifest:

| Field | Applies to | Purpose |
| --- | --- | --- |
| `metadata.labels` | Deployment object | Organizes and identifies the Deployment resource itself |
| `spec.selector.matchLabels` | Deployment's Pod selection | Defines which Pods belong to the workload managed through its ReplicaSets |
| `spec.template.metadata.labels` | Pods created from the template | Assigns labels to newly created Pods |

The Pod template's labels must satisfy the Deployment selector, but the Deployment object's own metadata labels are independent. A Service also has its own selector to discover matching Pods for network traffic.

## 3. Ingress: Path Matching Versus Path Rewriting

I deployed an Ingress named `webapp-ingress` with:

- Ingress class: `nginx`
- No custom hostname (hostless rule)
- Backend Service: `web-service`, port `80`
- Initially, path `/` with `Prefix` matching
- An externally accessible AWS load balancer address shown by Kubernetes

I changed the path from `/` to `/test` and received a **404 Not Found** response when visiting `/test`.

This illustrated an important distinction:

- **Ingress path matching** decides which backend Service receives an HTTP request.
- **Path rewriting** changes the URL path passed to the backend application.

By default, the NGINX Ingress Controller forwards the requested path. The stock NGINX image serves its welcome page at `/` but does not automatically provide a resource at `/test`. A rewrite annotation can direct `/test` to `/` where appropriate. A 404 can also come from the Ingress Controller if no rule matches, so the response source must be checked during troubleshooting.

I also learned that after changing an existing manifest, the correct command is `kubectl apply -f ingress.yaml`, not `kubectl update`, which is not a valid kubectl subcommand. The first apply warned about a missing last-applied-configuration annotation and automatically patched it.

## 4. Internal Traffic in a Multi-Tier Application

While studying the VProfile Kubernetes architecture, I clarified why an application may have **only one public Ingress route but several internal Services**.

Typical external request path:

`Browser → AWS Load Balancer → NGINX Ingress Controller → Tomcat Service → Tomcat Pod`

From there, the Tomcat application independently connects to internal dependencies using Kubernetes Services and DNS:

- **MySQL Service:** database access, typically on port `3306`
- **Memcached Service:** caching, typically on port `11211`
- **RabbitMQ Service:** messaging, typically on port `5672`

Those internal connections do not require their own public Ingress paths. The Ingress Controller runs as Pods; a Service of type `LoadBalancer` can expose the controller externally, while application/backend Services commonly use `ClusterIP`.

I also revisited the storage-related resources for databases:

- **Secrets** provide credentials to containers.
- **PersistentVolumeClaims (PVCs)** request storage for database files.
- **StorageClasses** determine how compatible persistent storage is provisioned.
- **AWS EBS** may back the PersistentVolume that is mounted into a MySQL Pod.

## 5. Horizontal Scaling: Three Replicas to Five

I verified that my NGINX Deployment initially reported:

```text
webapp-deployment   3/3   3   3
```

It was managed by a ReplicaSet named:

```text
webapp-deployment-5cb94f845c
```

I scaled the Deployment:

```bash
kubectl scale deployment webapp-deployment --replicas=5
```

The Deployment then reported:

```text
Replicas: 5 desired | 5 updated | 5 total | 5 available | 0 unavailable
```

The event history explicitly stated:

```text
Scaled up replica set webapp-deployment-5cb94f845c from 3 to 5
```

### What this demonstrated

- The Deployment's desired replica count changed from three to five.
- The **existing ReplicaSet** created two additional Pods.
- The original Pods remained running while the new Pods were added.
- **No new ReplicaSet was created** because the Pod template did not change.

The Deployment's rolling-update strategy remained `maxSurge: 1` and `maxUnavailable: 0`, but scaling replica count alone is not a rolling update.

## 6. Self-Healing: Deleting One Pod

I deleted a Pod:

```bash
kubectl delete pod webapp-deployment-5cb94f845c-z9jp9
```

Kubernetes created a replacement with a different Pod name:

```text
webapp-deployment-5cb94f845c-glrw6   1/1   Running
```

This demonstrated how the **ReplicaSet controller reconciles actual state with desired state**:

1. The ReplicaSet expects five active Pods.
2. I delete one of the Pods.
3. The controller detects that the active count is below five.
4. It creates a replacement Pod.
5. The replacement can receive traffic after becoming ready and being included among the Service endpoints.

The ReplicaSet creates a new Pod; it does not recover the exact deleted Pod instance.

## 7. Self-Healing: Deleting All Pods

I then ran:

```bash
kubectl delete pods --all
```

Kubernetes deleted the application's five Pods, but the Deployment and ReplicaSet still existed. Subsequent output reported:

```text
webapp-deployment                 5/5   5   5
webapp-deployment-5cb94f845c     5     5   5
```

The newly listed Pods had different names from the original Pods, confirming replacement. The last pasted Pod listing showed four Pods, so I should recheck the full list when I resume rather than assume that listing was complete.

### Important caveat

`kubectl delete pods --all` deletes every Pod in the selected namespace, not just the NGINX application's Pods. A label-scoped deletion is safer for targeted experiments.

Self-healing also does not guarantee zero downtime: a single-replica service or deleting all replicas may cause temporary interruption while replacement Pods start and become ready.

## 8. Deployment and ReplicaSet Responsibilities

My mental model is now clearer:

```text
Deployment (declares desired workload and manages revisions)
    └── ReplicaSet (maintains the desired number of Pods)
          ├── Pod
          ├── Pod
          ├── Pod
          ├── Pod
          └── Pod
```

**Deployment:** Coordinates the desired replica count and rolling updates by managing ReplicaSets.

**ReplicaSet:** Ensures the specified number of matching Pods exists, replacing deleted Pods when needed.

**Pod:** Runs the application container and can be replaced without preserving its identity.

This is an example of Kubernetes' **reconciliation model**, not simply a one-time script that creates containers.

## 9. Exercise Status

### Completed and observed

- Created the five manifest files for ConfigMap, Secret, Deployment, Service, and Ingress.
- Ran an NGINX Deployment with three ready replicas.
- Inspected the existing ReplicaSet and Pods.
- Tested hostless Ingress configuration and investigated a `/test` 404.
- Scaled from three to five replicas.
- Observed the same ReplicaSet scale from three to five.
- Deleted one Pod and observed a replacement.
- Deleted all Pods in the namespace and observed the workload controller restoring capacity.

### Still to verify or complete

- **ConfigMap and Secret injection:** `kubectl describe deployment` showed `Environment: <none>`, so the required environment variables were not yet confirmed or configured in that inspected Deployment.
- **Service endpoint discovery:** Confirm that `web-service` targets all five ready Pods.
- **Five-Pod final state:** Obtain a complete fresh Pod listing after replacement.
- **Rolling update:** Change a harmless Pod-template annotation (e.g., `training/revision: v1` to `v2`) and observe the new ReplicaSet.
- **Rollback:** Restore the previous Deployment revision and inspect the ReplicaSets.

## 10. Key Takeaways

1. **Labels and selectors have different roles.** Deployment labels label the Deployment, Pod-template labels label Pods, and selectors identify which Pods belong to the workload or receive Service traffic.
2. **Ingress routing does not necessarily rewrite application URLs.** Matching `/test` and forwarding `/test` are different from rewriting it to `/`.
3. **One public application route can support several internal dependencies.** Applications communicate with database, cache, and message-broker Services through cluster networking and DNS.
4. **Scaling a Deployment does not inherently create a new ReplicaSet.** A Pod-template change does.
5. **ReplicaSets enable Pod self-healing.** They continuously recreate Pods to reach the declared desired replica count.
6. **Self-healing is not the same as guaranteed zero downtime.** Readiness and healthy replicas still matter.

## 11. Next Session

Complete the remaining configuration and Service verification, then move to **rolling updates and rollback**. The main next observation is how Kubernetes transitions between old and new ReplicaSets when the Pod template changes, compared with simply scaling an existing ReplicaSet.

**Session conclusion:** Today I moved from understanding Kubernetes controllers in theory to observing their real behavior during scaling and Pod deletion in my AWS-hosted cluster.
