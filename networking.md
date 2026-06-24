# Workshop: OpenShift CRC — Services, Ingress & Routes

**Goal:** Understand the five ways to expose workloads in OpenShift — ClusterIP, NodePort, LoadBalancer, Ingress, and Route — when to use each, and how to prove they work on a local CRC cluster.

**Audience:** Developers and platform engineers learning OpenShift networking from the ground up.

**Time:** ~60 minutes

---

## 0. Background: The Exposure Ladder

Every pod in Kubernetes/OpenShift gets an IP, but that IP is ephemeral and internal. Services and Routes solve different layers of the exposure problem:

```
External User / Browser
        │
        ▼
 [ Route / Ingress ]      ← Layer 7: hostname + path-based routing (HTTP/HTTPS)
        │
        ▼
 [ LoadBalancer Service ] ← Layer 4: external IP, TCP/UDP (cloud or MetalLB)
        │
        ▼
 [ NodePort Service ]     ← Layer 4: static port on every node (30000–32767)
        │
        ▼
 [ ClusterIP Service ]    ← Internal only: stable virtual IP inside the cluster
        │
        ▼
     [ Pods ]
```

| Type | Reachable From | Protocol | Typical Use |
|---|---|---|---|
| ClusterIP | Inside cluster only | TCP/UDP | Service-to-service communication |
| NodePort | Node IP + static port | TCP/UDP | Dev/test, no cloud LB available |
| LoadBalancer | External IP (cloud/MetalLB) | TCP/UDP | Production external access |
| Ingress | Hostname/path | HTTP/HTTPS | Multi-service HTTP routing |
| Route | Hostname/path | HTTP/HTTPS | OpenShift-native, built-in TLS termination |

---

## 1. Prerequisites

- CRC running (`crc start` completed)
- `oc` CLI in `$PATH`
- Logged in as `kubeadmin`

```bash
eval $(crc oc-env)
oc login -u kubeadmin -p <kubeadmin-password> https://api.crc.testing:6443
oc whoami
```

---

## 2. Create the Project and a Sample App

All service types in this workshop expose the same app — a simple NGINX web server — so you can focus on the networking layer, not the application.

```bash
oc new-project networking-workshop
```

```yaml
# sample-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
  namespace: networking-workshop
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
        - name: nginx
          image: registry.access.redhat.com/ubi9/nginx-120
          ports:
            - containerPort: 8080
          command: ["/bin/bash", "-c"]
          args:
            - |
              echo "<h1>Hello from $(hostname)</h1>" > /tmp/index.html
              nginx -g 'daemon off;' -c /etc/nginx/nginx.conf &
              sleep infinity
```

> **Note:** The UBI NGINX image runs on port `8080` (not `80`) to avoid needing elevated privileges. All service manifests below target port `8080`.

```bash
oc apply -f sample-app.yaml
oc rollout status deployment/sample-app -n networking-workshop
oc get pods -n networking-workshop -o wide
```

Note the pod IPs — they change whenever pods restart, which is exactly why Services exist.

---

## 3. ClusterIP Service

### What it is

A **ClusterIP** service assigns a stable virtual IP address inside the cluster. Traffic to that IP is load-balanced across all matching pods. It is the default service type and the foundation all other types build on.

It is **not** reachable from outside the cluster.

### When to use it

- Service-to-service communication (e.g., WordPress → MySQL)
- Any backend that should never be directly exposed externally
- As the backing service for an Ingress or Route

### Manifest

```yaml
# clusterip-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: sample-app-clusterip
  namespace: networking-workshop
spec:
  type: ClusterIP          # Default; can be omitted
  selector:
    app: sample-app        # Matches pods with this label
  ports:
    - name: http
      port: 80             # Port the Service listens on (inside the cluster)
      targetPort: 8080     # Port the container listens on
```

```bash
oc apply -f clusterip-service.yaml
oc get svc sample-app-clusterip -n networking-workshop
```

**Expected output:**

```
NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
sample-app-clusterip    ClusterIP   172.30.x.x      <none>        80/TCP    5s
```

`EXTERNAL-IP` is `<none>` — this is by design.

### Verify

Spin up a temporary debug pod inside the cluster and curl the ClusterIP:

```bash
# Get the ClusterIP
CLUSTER_IP=$(oc get svc sample-app-clusterip -n networking-workshop -o jsonpath='{.spec.clusterIP}')
echo "ClusterIP: $CLUSTER_IP"

# Curl from inside the cluster
oc run curl-test --image=curlimages/curl:latest --rm -it --restart=Never \
  -n networking-workshop -- curl -s http://$CLUSTER_IP/
```

You should see the NGINX welcome response. The ClusterIP also resolves by DNS inside the cluster:

```bash
oc run curl-test --image=curlimages/curl:latest --rm -it --restart=Never \
  -n networking-workshop -- curl -s http://sample-app-clusterip.networking-workshop.svc.cluster.local/
```

DNS format: `<service-name>.<namespace>.svc.cluster.local`

### Key Points

- Stable IP survives pod restarts and reschedules
- kube-proxy / OVN handles the IP translation to pod IPs
- Deleting and recreating a Service gives a new ClusterIP — don't hardcode IPs; always use DNS

---

## 4. NodePort Service

### What it is

A **NodePort** service opens a static port in the range `30000–32767` on **every node** in the cluster. External traffic hitting `<NodeIP>:<NodePort>` is forwarded to the backing pods.

### When to use it

- Quick external access without a cloud load balancer
- Development and testing on bare metal or local clusters like CRC
- Exposing non-HTTP TCP traffic without an Ingress

### Manifest

```yaml
# nodeport-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: sample-app-nodeport
  namespace: networking-workshop
spec:
  type: NodePort
  selector:
    app: sample-app
  ports:
    - name: http
      port: 80             # ClusterIP port (internal)
      targetPort: 8080     # Container port
      nodePort: 30080      # Static port on every node (omit to auto-assign)
```

```bash
oc apply -f nodeport-service.yaml
oc get svc sample-app-nodeport -n networking-workshop
```

**Expected output:**

```
NAME                   TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
sample-app-nodeport    NodePort   172.30.x.x     <none>        80:30080/TCP   5s
```

### Verify

Get the CRC VM's IP and curl the NodePort directly:

```bash
CRC_IP=$(crc ip)
echo "CRC node IP: $CRC_IP"

curl http://$CRC_IP:30080/
```

You should get the NGINX response directly from your host machine — no `oc exec` needed.

### Key Points

- All nodes expose the same NodePort, even nodes not running the pod
- Port range `30000–32767` is not customisable without cluster configuration changes
- Not recommended for production — no TLS, no hostname routing, port numbers are opaque
- Underlying ClusterIP is still created automatically

---

## 5. LoadBalancer Service

### What it is

A **LoadBalancer** service requests an external IP from the underlying infrastructure (a cloud provider or MetalLB on bare metal). On CRC, there is no real cloud provider, so the service stays in `<pending>` unless MetalLB is installed.

This section covers both the manifest and what to do on CRC where a real LB isn't available.

### When to use it

- Production clusters on AWS, GCP, Azure, or bare metal with MetalLB
- Exposing non-HTTP TCP/UDP services that can't go through an Ingress/Route
- When you want a dedicated, stable external IP per service

### Manifest

```yaml
# lb-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: sample-app-lb
  namespace: networking-workshop
spec:
  type: LoadBalancer
  selector:
    app: sample-app
  ports:
    - name: http
      port: 80
      targetPort: 8080
```

```bash
oc apply -f lb-service.yaml
oc get svc sample-app-lb -n networking-workshop
```

**On CRC without MetalLB:**

```
NAME             TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
sample-app-lb    LoadBalancer   172.30.x.x     <pending>     80:3xxxx/TCP   10s
```

`EXTERNAL-IP` stays `<pending>` indefinitely — CRC has no cloud controller to fulfil the request.

### CRC Workaround: Simulate with NodePort

For this workshop, use the auto-assigned NodePort that OpenShift created alongside the LB service:

```bash
NODE_PORT=$(oc get svc sample-app-lb -n networking-workshop -o jsonpath='{.spec.ports[0].nodePort}')
CRC_IP=$(crc ip)
curl http://$CRC_IP:$NODE_PORT/
```

### Installing MetalLB on CRC (optional)

If you want a working LoadBalancer on CRC, install the MetalLB Operator:

```bash
# Install MetalLB Operator via OLM
oc apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: metallb-system
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: metallb-operator-group
  namespace: metallb-system
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: metallb-operator
  namespace: metallb-system
spec:
  channel: stable
  name: metallb-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

# After the operator is ready, create a MetalLB instance and IPAddressPool
# using the CRC VM's IP range (run `crc ip` to find it)
```

Full MetalLB configuration is outside this workshop's scope — see the [MetalLB Operator docs](https://docs.openshift.com/container-platform/latest/networking/metallb/about-metallb.html).

### Key Points

- LoadBalancer is a superset of NodePort — it creates both
- In cloud environments, each LoadBalancer service provisions a new cloud LB (and its cost)
- On bare metal / CRC, MetalLB fills the gap by assigning IPs from a local address pool

---

## 6. Ingress

### What it is

A Kubernetes **Ingress** resource defines Layer 7 routing rules: map `hostname/path` combinations to backend Services. It requires an **Ingress Controller** to be running in the cluster to act on those rules.

On OpenShift, the built-in **HAProxy-based Router** also serves as the Ingress Controller, so standard `networking.k8s.io/v1` Ingress objects work natively.

### When to use it

- Routing multiple HTTP/HTTPS services through a single external IP
- Path-based routing (`/api` → service A, `/web` → service B)
- When you need to stay Kubernetes-portable (Ingress is not OpenShift-specific)

### Manifest

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sample-app-ingress
  namespace: networking-workshop
  annotations:
    # Force HTTP → HTTPS redirect (handled by the OpenShift router)
    route.openshift.io/termination: "edge"
spec:
  rules:
    - host: sample-app.apps-crc.testing     # CRC wildcard domain
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: sample-app-clusterip  # Points to the ClusterIP service
                port:
                  number: 80
  # Optional: TLS with a secret containing cert + key
  # tls:
  #   - hosts:
  #       - sample-app.apps-crc.testing
  #     secretName: sample-app-tls
```

```bash
oc apply -f ingress.yaml
oc get ingress sample-app-ingress -n networking-workshop
```

**Expected output:**

```
NAME                  CLASS    HOSTS                         ADDRESS                    PORTS   AGE
sample-app-ingress    <none>   sample-app.apps-crc.testing   router-default.apps-crc…   80      5s
```

### Verify

```bash
curl http://sample-app.apps-crc.testing/
```

CRC's wildcard DNS (`*.apps-crc.testing`) resolves to the CRC VM automatically on your host — no `/etc/hosts` editing needed.

### Key Points

- Ingress is Kubernetes-native; works on any cluster with an Ingress Controller
- OpenShift's Router (HAProxy) implements Ingress natively — no separate controller install needed
- Annotations vary by Ingress Controller (nginx, HAProxy, Traefik); OpenShift uses `route.openshift.io/` annotations
- For advanced TLS, wildcard certs, or sticky sessions, the OpenShift Route (next section) gives more control

---

## 7. OpenShift Route

### What it is

An OpenShift **Route** is OpenShift's own Layer 7 exposure resource, predating Kubernetes Ingress. It is processed directly by the OpenShift Router (HAProxy) without needing an Ingress object. Routes give you more declarative control over TLS termination modes than Ingress annotations.

### TLS Termination Modes

| Mode | TLS terminated at | Backend traffic | Use case |
|---|---|---|---|
| `edge` | Router | Plain HTTP | Most common; cert managed at the router |
| `passthrough` | Pod (app handles TLS) | Encrypted | End-to-end encryption; no cert at router |
| `reencrypt` | Router → re-encrypts to pod | Encrypted | Router has its own cert; pod has a different cert |

### When to use it

- OpenShift-specific deployments where portability isn't a concern
- When you need edge TLS termination with a Let's Encrypt or custom cert
- When you need `passthrough` or `reencrypt` TLS modes (not supported by standard Ingress)
- Simpler syntax for common cases compared to annotated Ingress objects

---

### 7.1 Edge Route (TLS terminated at the Router)

```yaml
# route-edge.yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: sample-app-edge
  namespace: networking-workshop
spec:
  host: sample-app-edge.apps-crc.testing
  to:
    kind: Service
    name: sample-app-clusterip
  port:
    targetPort: http                  # Must match the port name in the Service
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect   # HTTP → HTTPS redirect
```

```bash
oc apply -f route-edge.yaml
oc get route sample-app-edge -n networking-workshop
```

**Expected output:**

```
NAME               HOST/PORT                           PATH   SERVICES               PORT   TERMINATION     WILDCARD
sample-app-edge    sample-app-edge.apps-crc.testing           sample-app-clusterip   http   edge/Redirect   None
```

Verify:

```bash
# HTTPS works (CRC uses a self-signed cert — use -k to skip verification)
curl -k https://sample-app-edge.apps-crc.testing/

# HTTP redirects to HTTPS
curl -v http://sample-app-edge.apps-crc.testing/
# Look for: < HTTP/1.1 301 Moved Permanently
```

---

### 7.2 Passthrough Route (TLS terminated in the Pod)

Passthrough routes send encrypted traffic directly to the pod — the router never sees the plaintext. The application must handle TLS itself.

```yaml
# route-passthrough.yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: sample-app-passthrough
  namespace: networking-workshop
spec:
  host: sample-app-passthrough.apps-crc.testing
  to:
    kind: Service
    name: sample-app-clusterip
  port:
    targetPort: http
  tls:
    termination: passthrough
```

> **Note:** For this to work in practice, the pod must serve HTTPS. The sample NGINX app in this workshop serves plain HTTP, so this route will fail to connect — the manifest is here to demonstrate the syntax. In production, configure the NGINX container with TLS certs and update the `targetPort` to the HTTPS port.

```bash
oc apply -f route-passthrough.yaml
oc get route sample-app-passthrough -n networking-workshop
```

---

### 7.3 Reencrypt Route

The router terminates TLS from the client, then opens a new TLS connection to the pod using a separate certificate.

```yaml
# route-reencrypt.yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: sample-app-reencrypt
  namespace: networking-workshop
spec:
  host: sample-app-reencrypt.apps-crc.testing
  to:
    kind: Service
    name: sample-app-clusterip
  port:
    targetPort: http
  tls:
    termination: reencrypt
    # Optional: provide the CA cert the router uses to verify the pod's cert
    # destinationCACertificate: |-
    #   -----BEGIN CERTIFICATE-----
    #   ...
    #   -----END CERTIFICATE-----
```

```bash
oc apply -f route-reencrypt.yaml
oc get route sample-app-reencrypt -n networking-workshop
```

> **Note:** In a real scenario, you would populate `destinationCACertificate` with the CA that signed the pod's serving certificate so the router can validate the backend connection.

---

### 7.4 HTTP Route (no TLS)

The simplest route — plain HTTP, no TLS, useful for internal tooling or development.

```yaml
# route-http.yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: sample-app-http
  namespace: networking-workshop
spec:
  host: sample-app-http.apps-crc.testing
  to:
    kind: Service
    name: sample-app-clusterip
  port:
    targetPort: http
```

```bash
oc apply -f route-http.yaml
oc get route sample-app-http -n networking-workshop
curl http://sample-app-http.apps-crc.testing/
```

---

## 8. Side-by-Side Comparison

After applying all manifests, list everything at once:

```bash
oc get svc,ingress,route -n networking-workshop
```

**Expected output (condensed):**

```
NAME                           TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)
service/sample-app-clusterip   ClusterIP      172.30.x.x     <none>        80/TCP
service/sample-app-nodeport    NodePort       172.30.x.x     <none>        80:30080/TCP
service/sample-app-lb          LoadBalancer   172.30.x.x     <pending>     80:3xxxx/TCP

NAME                                    CLASS    HOSTS
ingress/sample-app-ingress              <none>   sample-app.apps-crc.testing

NAME                                          HOST/PORT                                  TERMINATION
route.route.openshift.io/sample-app-edge      sample-app-edge.apps-crc.testing          edge/Redirect
route.route.openshift.io/sample-app-http      sample-app-http.apps-crc.testing
route.route.openshift.io/sample-app-passthrough sample-app-passthrough.apps-crc.testing passthrough
route.route.openshift.io/sample-app-reencrypt sample-app-reencrypt.apps-crc.testing     reencrypt
```

---

## 9. Decision Guide

```
Do you need to expose this workload outside the cluster?
│
├── No → ClusterIP
│         Use for: databases, internal APIs, anything pod-to-pod
│
└── Yes
    │
    ├── Is it HTTP or HTTPS?
    │   │
    │   ├── Yes
    │   │   ├── Need OpenShift-specific TLS control (passthrough, reencrypt)?
    │   │   │   └── Yes → Route
    │   │   │
    │   │   ├── Need Kubernetes-portable configuration?
    │   │   │   └── Yes → Ingress
    │   │   │
    │   │   └── Simple HTTP/HTTPS on OpenShift?
    │   │       └── Route (edge) — simpler syntax than annotated Ingress
    │   │
    │   └── No (raw TCP/UDP, databases, MQTT, gRPC without HTTP routing)
    │       │
    │       ├── Cloud cluster or MetalLB available?
    │       │   └── Yes → LoadBalancer
    │       │
    │       └── Local/CRC/bare metal, no MetalLB?
    │           └── NodePort
```

---

## 10. Cleanup

```bash
oc delete project networking-workshop
```

All Services, Ingress, and Routes are namespace-scoped and are removed with the project.

---

## Summary Table

| Concept | Type | Scope | Reachable From | TLS |
|---|---|---|---|---|
| ClusterIP | Service | Namespaced | Inside cluster only | No |
| NodePort | Service | Namespaced | Node IP + port | No |
| LoadBalancer | Service | Namespaced | External IP (needs LB provider) | No |
| Ingress | Ingress | Namespaced | External via hostname/path | Via annotation |
| Route (HTTP) | Route | Namespaced | External via hostname | No |
| Route (edge) | Route | Namespaced | External via hostname | At router |
| Route (passthrough) | Route | Namespaced | External via hostname | At pod |
| Route (reencrypt) | Route | Namespaced | External via hostname | At router + pod |

---

## Troubleshooting

**ClusterIP not reachable from inside the cluster:**
Check that the `selector` in the Service matches the pod labels exactly — a single typo means the Endpoints list is empty. Run `oc get endpoints <service-name>` and confirm pod IPs are listed.

**NodePort connection refused from host:**
Confirm the CRC VM firewall allows the port. Run `crc ip` to get the node IP and `nc -zv <crc-ip> <nodeport>` to test TCP connectivity before curl.

**LoadBalancer stuck in `<pending>`:**
Expected on CRC without MetalLB. The service still works via its auto-assigned NodePort — use that for testing.

**Ingress returns 503:**
The backend Service has no ready endpoints. Run `oc get endpoints` and `oc get pods` to check pod health. Also confirm the Ingress `host` matches the CRC wildcard domain (`*.apps-crc.testing`).

**Route shows `HostAlreadyClaimed`:**
Two Routes in different namespaces are using the same hostname. Each Route hostname must be unique cluster-wide. Rename the `host` field.

**Edge Route returns SSL cert error in browser:**
CRC uses a self-signed wildcard cert for `*.apps-crc.testing`. Either trust the CRC CA (`crc ca-bundle-path` shows the file) or use `-k` with curl for testing.

**`targetPort` name mismatch on Route:**
The Route's `targetPort` must match the `name` field on the Service's port entry, not the number. If the Service defines `name: http`, the Route must say `targetPort: http`.
