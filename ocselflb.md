# OpenShift Routes - HTTP Only (Client-Managed External LB / VIPs) — Lab Guide

> **Scenario:** The client already owns and operates an external load balancer (F5, NetScaler, NGINX, HAProxy appliance, cloud LB, etc.) that provides the VIPs. MetalLB is **not used** in this design — VIP assignment is the client's LB's job, not the cluster's. OpenShift only needs to expose its Router (HAProxy) so the client's LB has something to forward to.

---

## What Changed vs. the MetalLB Version

| Removed | Added / Changed |
|---|---|
| `metallb-system` namespace | Step to discover Router (HAProxy) pod/node IPs |
| `memberlist` secret | Client LB backend-pool configuration step |
| `webhook-server-cert` secret + OpenSSL cert generation | DNS/VIP step now points at **client LB VIPs**, not MetalLB IPs |
| MetalLB install manifest | Services use `ClusterIP` instead of `LoadBalancer` |
| SCC grants for `controller` / `speaker` service accounts | Troubleshooting now covers client-LB / backend-pool issues |
| `IPAddressPool` CR | — |
| `L2Advertisement` CR | — |
| `metallb.universe.tf/loadBalancerIPs` Service annotation | — |
| "VIP only reachable inside cluster" caveat (L2 mode limitation) | Test step now assumes VIPs are externally reachable, since the client LB is a real external device |

Everything else (Deployments, Routes, namespace, DNS, `oc` commands for Routes) stays conceptually the same.

---

## Architecture

```
http://erp.ow.com  -->  Client External LB (VIP 1: 192.168.130.100)  -->  OpenShift Router (HAProxy, infra/worker node port 80)  -->  erp-svc (ClusterIP)  -->  erp pods
http://hr.ow.com   -->  Client External LB (VIP 2: 192.168.130.101)  -->  OpenShift Router (HAProxy, infra/worker node port 80)  -->  hr-svc  (ClusterIP)  -->  hr pods
```

The OpenShift Router does the host-based dispatch (it reads the `Host:` header and matches it to the `erp.ow.com` / `hr.ow.com` Route). Because of this, **both VIPs on the client LB can point at the same backend pool** (the set of nodes running router pods) — the VIP doesn't need to map 1:1 to an app, only the Route's `host:` field matters for dispatch.

---

## Step 1 — Identify the OpenShift Router's Node IPs

The client's LB needs a backend pool consisting of the node IP(s) where the default `IngressController` (Router/HAProxy) pods run.

```bash
oc get pods -n openshift-ingress -o wide
```

Note the `NODE` column for each `router-default-xxxx` pod, then get that node's IP:

```bash
oc get nodes -o wide
```

By default the Router listens on the node's host network on ports **80** and **443**. If router pods are spread across multiple nodes (HA), include **all** of those node IPs in the LB's backend pool.

> **Tip:** If the cluster has dedicated infra nodes for routers, confirm the router is actually scheduled there:
> ```bash
> oc get pods -n openshift-ingress -o wide | grep router-default
> ```

---

## Step 2 — Hand Backend Pool Info to the Client / Configure the Client LB

Give the client (or configure, if you manage their LB) the following:

- **Backend pool members:** the node IP(s) from Step 1, port `80` (HTTP only, per this guide's scope)
- **Health check:** TCP check on port 80, or HTTP check against any existing Route host (e.g. `GET / Host: erp.ow.com` expecting a 200/3xx/404 — anything but connection refused/5xx confirms the router is alive)
- **VIP 1** (`192.168.130.100`) → forwards to the backend pool → used for `erp.ow.com`
- **VIP 2** (`192.168.130.101`) → forwards to the backend pool → used for `hr.ow.com`

Example generic backend config (vendor-agnostic, e.g. HAProxy-style, for illustration only — adapt to the client's actual LB):

```
frontend vip_erp
    bind 192.168.130.100:80
    default_backend ocp_routers

frontend vip_hr
    bind 192.168.130.101:80
    default_backend ocp_routers

backend ocp_routers
    balance roundrobin
    option httpchk GET /
    server router-node1 <NODE1_IP>:80 check
    server router-node2 <NODE2_IP>:80 check
```

> **No `oc` commands are needed for this step** — it happens entirely on the client's LB. This is the main operational difference from the MetalLB version: the cluster does nothing to "claim" an IP; the client's infrastructure decides where traffic for that VIP lands.

---

## Step 3 — Create Namespace

```bash
oc new-project workshop
```

---

## Step 4 — Deploy ERP

`erp-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: erp
  namespace: workshop
spec:
  replicas: 2
  selector:
    matchLabels:
      app: erp
  template:
    metadata:
      labels:
        app: erp
    spec:
      containers:
      - name: erp
        image: nginxinc/nginx-unprivileged
        ports:
        - containerPort: 8080
```

```bash
oc apply -f erp-deployment.yaml
```

`erp-svc.yaml` — **plain `ClusterIP`, no LoadBalancer annotation needed**:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: erp-svc
  namespace: workshop
spec:
  selector:
    app: erp
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
```

```bash
oc apply -f erp-svc.yaml
```

---

## Step 5 — Deploy HR

`hr-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hr
  namespace: workshop
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hr
  template:
    metadata:
      labels:
        app: hr
    spec:
      containers:
      - name: hr
        image: nginxinc/nginx-unprivileged
        ports:
        - containerPort: 8080
```

```bash
oc apply -f hr-deployment.yaml
```

`hr-svc.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hr-svc
  namespace: workshop
spec:
  selector:
    app: hr
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
```

```bash
oc apply -f hr-svc.yaml
```

---

## Step 6 — Create Routes

`route-erp.yaml`:

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: route-erp
  namespace: workshop
spec:
  host: erp.ow.com
  to:
    kind: Service
    name: erp-svc
    weight: 100
  port:
    targetPort: 80
```

```bash
oc apply -f route-erp.yaml
```

`route-hr.yaml`:

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: route-hr
  namespace: workshop
spec:
  host: hr.ow.com
  to:
    kind: Service
    name: hr-svc
    weight: 100
  port:
    targetPort: 80
```

```bash
oc apply -f route-hr.yaml
```

---

## Step 7 — Verify

```bash
oc get svc -n workshop
oc get routes -n workshop
oc get pods -n workshop
oc get pods -n openshift-ingress -o wide
```

Expected `svc` output (note: **no external IP**, since the client LB — not the cluster — owns the VIP):

```
NAME      TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)
erp-svc   ClusterIP   10.x.x.x     <none>        80/TCP
hr-svc    ClusterIP   10.x.x.x     <none>        80/TCP
```

Expected `routes` output:

```
NAME        HOST/PORT     SERVICES   PORT
route-erp   erp.ow.com    erp-svc    80
route-hr    hr.ow.com     hr-svc     80
```

---

## Step 8 — DNS

Point DNS (or `/etc/hosts` for lab/testing) at the **client LB's VIPs**, not at any cluster-assigned IP:

```bash
echo "192.168.130.100 erp.ow.com" | sudo tee -a /etc/hosts
echo "192.168.130.101 hr.ow.com" | sudo tee -a /etc/hosts
```

---

## Step 9 — Test

Because the client's LB is a real external device (not an in-cluster L2 VIP), it should be reachable from outside the cluster network — test directly from a workstation/laptop, not just from inside the cluster:

```bash
curl -s http://erp.ow.com
curl -s http://hr.ow.com
```

If the client LB isn't reachable from your test host yet (e.g. still being provisioned), you can still validate the Router/Service/Pod chain from inside the cluster against a router node IP directly:

```bash
oc run curltest --image=registry.access.redhat.com/ubi9/ubi --restart=Never --rm -it -n workshop -- curl -s -H "Host: erp.ow.com" http://<ROUTER_NODE_IP>
```

```bash
oc run curltest --image=registry.access.redhat.com/ubi9/ubi --restart=Never --rm -it -n workshop -- curl -s -H "Host: hr.ow.com" http://<ROUTER_NODE_IP>
```

---

## Troubleshooting

### Client LB shows backend pool members "down" / health check failing

```bash
oc get pods -n openshift-ingress -o wide
```

Confirm router pods are `Running` and `1/1` (or `2/2` etc.) on the node IP(s) the client LB is checking. If the router pods moved to different nodes (e.g. after a node drain), the client LB's backend pool needs to be updated to match — this is a manual sync point since nothing in the cluster auto-updates the client's LB config.

### Client LB forwards traffic, but request times out or connection refused

- Confirm the node IP in the LB's pool is correct and port `80` is open (security group / firewall / `NetworkPolicy` on that node).
- Confirm router pods are actually bound to host port 80:
  ```bash
  oc get pods -n openshift-ingress -o wide
  oc describe pod -n openshift-ingress -l ingresscontroller.operator.openshift.io/owning-ingresscontroller=default
  ```

### 503 from the Router itself (request reaches OpenShift, but app errors)

```bash
oc get pods -n workshop
oc get endpoints erp-svc -n workshop
oc get endpoints hr-svc -n workshop
```

A `503` from HAProxy with no app-level response usually means the Route resolved correctly but the Service has no healthy endpoints — check pod readiness in `workshop`.

### Wrong app served for a given hostname

This means the client LB is forwarding correctly, but the Route's `host:` doesn't match what's expected, or DNS/`/etc/hosts` points the hostname at the wrong VIP. Check:

```bash
oc get routes -n workshop
```

and confirm `erp.ow.com` / `hr.ow.com` map to the intended Routes, and that DNS resolves each hostname to the intended client-LB VIP.

### Service stuck or behaving oddly because it still says `type: LoadBalancer`

If a Service from the old MetalLB version is still applied with `type: LoadBalancer` and a `metallb.universe.tf/loadBalancerIPs` annotation, it will sit in `<pending>` forever (no MetalLB controller present to satisfy it). Re-apply the Service as `ClusterIP` as shown in Steps 4–5.
