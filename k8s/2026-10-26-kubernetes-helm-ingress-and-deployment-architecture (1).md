# Kubernetes Learning Journal — October 26, 2026

**Topics:** kOps cluster access, kubeconfig, WordPress and MySQL with Helm, persistent storage, Ingress, Deployments, ReplicaSets, Services, and Pods.

## 1. Session overview

Today I continued working with Kubernetes on AWS. I practised managing my kOps cluster from Windows, deploying a WordPress–MySQL application with Helm, troubleshooting a Helm upgrade, and reviewing how an NGINX Ingress Controller relates to my application's Deployment, Service, and Pods.

My biggest conceptual improvement was separating **how Kubernetes manages application instances** from **how network requests reach those instances**.

## 2. Managing the cluster from Windows

My Kubernetes cluster is named `cluster.k8s.local`, with kOps configuration stored in `s3://tinoscript-kops-cluster-state` in AWS `us-east-1`.

I learned that `kubectl` can manage a remote cluster from my Windows laptop; it does not need to run on a cluster node. It needs a kubeconfig containing the API server address, certificate authority information, authentication credentials, and selected context.

The usual Windows location is `~/.kube/config`. During setup, I encountered this error:

```text
error loading config file: yaml: line 21: could not find expected ':'
```

I inspected the file with `nl -ba ~/.kube/config` and found that I had accidentally pasted **two kubeconfig outputs**, including a Linux shell prompt and the redacted `DATA+OMITTED` values from `kubectl config view`. The file needed to contain only the actual kubeconfig YAML, without shell prompts or redacted placeholders.

Useful verification commands:

```bash
kubectl config current-context
kubectl get nodes
```

**Security lesson:** kubeconfig can contain an administrative client private key. I should not paste its raw contents into chats, repositories, or documentation. Since I previously exposed credential material, I should replace or revoke those credentials where feasible; merely copying the file elsewhere does not rotate them.

## 3. Deploying WordPress and MySQL with Helm

I worked with a Helm chart defining a WordPress frontend and a MySQL database. The chart included:

- WordPress Deployment and ClusterIP Service, with HTTP exposed on port 80.
- MySQL Deployment and an internal headless Service on port 3306.
- PersistentVolumeClaims for WordPress files and MySQL data.
- A Secret supplying database configuration to the applications.
- An NGINX Ingress resource routing `/` to WordPress.

I installed the chart using:

```bash
helm install wp ./wordpress-chart -n wp-ns --create-namespace
```

The installation reported `STATUS: deployed`. My checks showed one WordPress Pod and one MySQL Pod, both `1/1 Running`, with their Deployments available.

Attempting to run the same `helm install` command again produced:

```text
cannot reuse a name that is still in use
```

I learned that a **Helm release** has a name within a namespace. To update an existing installation, I use `helm upgrade` rather than repeating `helm install`:

```bash
helm upgrade wp ./wordpress-chart -n wp-ns
```

Helm charts package reusable Kubernetes templates; `values.yaml` supplies configuration; a release is an installed instance of a chart. Helm is useful for managing related Kubernetes resources together, but Kubernetes is still responsible for running them.

## 4. Understanding PVC immutability

While adapting the chart, I changed the persistence configuration from a StorageClass named `default` to `null`, intending to use the cluster's default StorageClass.

The Helm upgrade failed because the existing **bound PersistentVolumeClaims** already had `storageClassName: default`. Kubernetes does not allow the StorageClass of a bound PVC to be changed in place.

The affected claims were `wp-wordpress` and `wp-wordpress-mysql`.

**Resolution for this existing installation:** restore the original `storageClass: "default"` value in the Helm configuration, preserving the existing claims and data. A new PVC can omit `storageClassName` to use whichever StorageClass the cluster marks as default, but that is not the same as changing an existing bound claim.

**General lesson:** not every field in a Kubernetes resource is mutable. Before running Helm upgrades, I should inspect the rendered manifests and consider how changes will affect existing stateful resources. I should not delete PVCs casually because they may contain application data.

## 5. Adapting Ingress to an AWS load-balancer DNS address

The original tutorial chart expected a domain I did not own. I adapted the Ingress to use **no host restriction**, with an NGINX IngressClass and the root path `/`.

Conceptual example:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wordpress-ingress
  namespace: wp-ns
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: wp-wordpress
                port:
                  number: 80
```

The installed chart used a Helm-generated Ingress name, but the routing relationship was the same: requests for `/` went to the WordPress Service on **Service port 80**.

`kubectl describe ingress` showed:

- Ingress class `nginx`.
- Host `*`, meaning the rule had no hostname requirement.
- Path `/` pointing to `wp-wordpress:80`.
- A ready WordPress backend endpoint.
- An AWS ELB DNS address and controller synchronization events.

This confirms that the ingress rule and an application endpoint were visible to Kubernetes; **it does not by itself prove the website loaded successfully in a browser**.

## 6. What an Ingress Controller actually is

I reviewed the difference between an **Ingress resource** and an **Ingress Controller**:

- **Ingress resource:** a Kubernetes API object containing HTTP/HTTPS routing rules.
- **Ingress Controller:** running software, such as NGINX, that watches relevant Ingress objects and implements their routing rules.

Ingress controller software is generally installed using maintained Kubernetes manifests or a Helm chart. A typical installation creates a controller Deployment, controller Pods, a Service, an IngressClass, and RBAC resources. I normally do not need to write these controller resources from scratch.

The NGINX controller's Deployment is **separate from my WordPress Deployment**. The controller Service exposes the NGINX controller Pods; the WordPress Service provides access to WordPress Pods.

A representative external traffic path in this lab is:

```text
Browser
  → AWS Load Balancer
  → LoadBalancer Service for the NGINX Ingress Controller
  → NGINX Ingress Controller Pod
  → WordPress backend selected by the Ingress rule
  → Ready WordPress Pod
```

The NGINX controller can proxy directly to application Pod endpoints discovered from Kubernetes, so the WordPress Service is a **logical backend reference**, not necessarily an extra physical network hop.

The controller's Service `targetPort` refers to the **controller Pod's** listening port (typically 80 or 443), not to WordPress's application port. The WordPress Service defines its own `port` and `targetPort`.

## 7. Application management versus traffic routing

I reinforced two distinct relationships:

**Management:**

```text
WordPress Deployment → ReplicaSet → WordPress Pods → Containers
```

**Networking:**

```text
NGINX Ingress Controller → WordPress Service → Eligible WordPress Pods
```

The Deployment specifies and maintains desired application state. A ReplicaSet maintains the desired number of matching Pods. Pods run containers. Separately, a Service uses **label selectors** to discover eligible Pods and provide stable access to them. A Service does not point to a ReplicaSet or manage its replicas.

For example, if multiple WordPress Pods carry the label selected by the Service, that Service can send traffic to all ready matching Pods. Scaling the Deployment does not ordinarily require changing the Service or Ingress rule.

I also clarified that the NGINX controller and WordPress run in **different Deployment hierarchies**. Deleting the WordPress Deployment would not shut down the controller, but it would normally remove WordPress's managed Pods, leaving no healthy WordPress backend until replacement application Pods existed.

## 8. Service types revisited

- **ClusterIP:** exposes an internal, stable service endpoint within the Kubernetes cluster. This was the application-facing Service type for WordPress.
- **NodePort:** makes a Service reachable through a configured port on eligible cluster nodes, subject to networking and firewall access.
- **LoadBalancer:** requests an external load balancer when supported by the environment. In this lab, the NGINX controller was exposed through an AWS load balancer.

I connected these to the difference between a Service's `port`, the application's `targetPort`, and a node's `nodePort`.

## 9. Key troubleshooting practices

Commands practised or reviewed:

```bash
kubectl get all -n wp-ns
kubectl get pvc -n wp-ns
kubectl get ingress -n wp-ns
kubectl describe ingress <ingress-name> -n wp-ns
kubectl get storageclass
helm template wp ./wordpress-chart -n wp-ns
helm upgrade wp ./wordpress-chart -n wp-ns
```

I should inspect rendered Helm output before applying it, confirm the backend Service name and port, and check PVCs separately because `kubectl get all` does not display every resource type.

## 10. What remains to verify or learn

- Confirm that WordPress can be loaded and configured successfully through the external endpoint.
- Review **Deployments and ReplicaSets** in more depth, especially rollout behavior and the relationship between selectors and Pod templates.
- Explore probes, failures, endpoint readiness, and how Ingress responds when application Pods are unavailable.
- Practise further Helm upgrades while preserving stateful resources.
- If I delete this temporary kOps cluster, verify afterward that AWS worker instances, load balancers, and related resources are cleaned up. The deletion command was discussed, but successful teardown was not confirmed in the conversation.

## 11. Reflection

Today I became more comfortable connecting separate Kubernetes building blocks into one architectural model. I now understand that the NGINX Ingress Controller is itself a deployed application, that it follows routing instructions defined in Ingress resources, and that my own WordPress application has a separate Deployment and Service.

The hands-on Helm and PVC errors reinforced an important engineering lesson: **successful resource creation is different from successful operation, and declarative tools must still respect the rules of the underlying platform**.

The next step is to keep practising and revisit Deployments until their management relationship with ReplicaSets and Pods feels as intuitive as Service and Ingress routing.
