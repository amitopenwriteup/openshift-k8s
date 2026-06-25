# OpenShift Routes - HTTP Only (Two VIPs) — Fixed Lab Guide

> **Fixes Applied:**
> 1. The `memberlist` secret and webhook TLS secret must be created **before** installing MetalLB so pods can start cleanly on first boot.
> 2. On OpenShift, the `controller` and `speaker` service accounts in `metallb-system` have **no SCC granted by default**, so the ReplicaSet/DaemonSet cannot create pods at all (`FailedCreate`, `0/1 replicas`). SCCs must be granted to these service accounts **immediately after install**, before any pods can be scheduled.

---

## Architecture

```
http://erp.ow.com  -->  VIP 1 (MetalLB 192.168.130.100)  -->  HAProxy Router  -->  erp-svc  -->  erp pods
http://hr.ow.com   -->  VIP 2 (MetalLB 192.168.130.101)  -->  HAProxy Router  -->  hr-svc   -->  hr pods
```

---

## Step 1 — Create the `metallb-system` Namespace First

The namespace must exist before secrets can be created in it:

```bash
oc create namespace metallb-system
```

---

## Step 2 — Create Required Secrets (Before Installing MetalLB)

MetalLB needs two secrets present at pod startup:

1. `memberlist` — used by speaker pods for leader election
2. `webhook-server-cert` — TLS certificate for the validating webhook server

### 2a — Create the `memberlist` Secret

```bash
oc create secret generic memberlist --namespace metallb-system --from-literal=secretkey="$(openssl rand -base64 128)"
```

Verify:

```bash
oc get secret memberlist -n metallb-system
```

Expected output:

```
NAME         TYPE     DATA   AGE
memberlist   Opaque   1      5s
```

### 2b — Generate TLS Certificates for the Webhook

```bash
mkdir -p /tmp/metallb-certs && cd /tmp/metallb-certs
```

```bash
openssl genrsa -out ca.key 2048
```

```bash
openssl req -x509 -new -nodes -key ca.key -subj "/CN=metallb-webhook-ca" -days 3650 -out ca.crt
```

```bash
openssl genrsa -out tls.key 2048
```

```bash
openssl req -new -key tls.key -subj "/CN=webhook-service.metallb-system.svc" -out tls.csr
```

```bash
openssl x509 -req -in tls.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out tls.crt -days 3650 -extensions v3_req -extfile <(printf "[v3_req]\nsubjectAltName=DNS:webhook-service.metallb-system.svc,DNS:webhook-service.metallb-system.svc.cluster.local")
```

### 2c — Create the `webhook-server-cert` Secret

```bash
oc create secret tls webhook-server-cert --namespace metallb-system --cert=/tmp/metallb-certs/tls.crt --key=/tmp/metallb-certs/tls.key
```

Verify:

```bash
oc get secret webhook-server-cert -n metallb-system
```

Expected output:

```
NAME                  TYPE                DATA   AGE
webhook-server-cert   kubernetes.io/tls   2      5s
```

### 2d — Confirm Both Secrets Exist

```bash
oc get secrets -n metallb-system
```

Expected output:

```
NAME                  TYPE                DATA   AGE
memberlist            Opaque              1      1m
webhook-server-cert   kubernetes.io/tls   2      1m
```

---

## Step 3 — Install MetalLB

Now that secrets are in place, install MetalLB:

```bash
oc apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml
```

Wait for CRDs to be established:

```bash
oc wait --namespace metallb-system --for=condition=established crd/ipaddresspools.metallb.io --timeout=60s
```

> **Do not wait on pod readiness yet.** On OpenShift, the `controller` and `speaker` pods cannot be created at all until the SCC fix in Step 3a is applied — they will sit at `0/1 replicas` with `FailedCreate` events. Apply Step 3a immediately, then proceed to the wait/verify commands.

---

## Step 3a — Grant SCCs to MetalLB Service Accounts (Required on OpenShift)

The upstream `metallb-native.yaml` manifest is written for vanilla Kubernetes and assumes the cluster will permit `hostNetwork`, host ports, and the `NET_RAW` capability by default. OpenShift's SCC admission model requires an **explicit binding** between a service account and an SCC before any pod using these privileged features can be created. Without this, the ReplicaSet (`controller`) and DaemonSet (`speaker`) will fail to create pods entirely — not crash, **never even get created**.

Grant the `privileged` SCC to both service accounts:

```bash
oc adm policy add-scc-to-user privileged -z controller -n metallb-system
oc adm policy add-scc-to-user privileged -z speaker -n metallb-system
```

Verify the bindings:

```bash
oc get scc privileged -o yaml | grep -A5 users
```

You should see both service accounts listed:

```
users:
- system:serviceaccount:metallb-system:controller
- system:serviceaccount:metallb-system:speaker
```

> **Note:** `speaker` genuinely requires `privileged` (or `hostnetwork`) because it uses `hostNetwork: true` and host ports `7472`/`7946`. `controller` may work with a less permissive SCC in hardened environments, but `privileged` is used here to unblock the lab quickly; tighten later if required by policy.

Once the SCC is bound, the existing ReplicaSet/DaemonSet controllers will automatically retry pod creation within a few seconds — **no pod deletion is needed** if pods were never created. If pods exist in a bad state, force a refresh:

```bash
oc delete pod -n metallb-system -l component=controller
oc delete pod -n metallb-system -l component=speaker
```

---

## Step 3b — Wait for and Verify Pods

```bash
oc wait --namespace metallb-system --for=condition=ready pod --selector=app=metallb --timeout=120s
```

```bash
oc get pods -n metallb-system
```

Expected output:

```
NAME                                  READY   STATUS    RESTARTS   AGE
controller-xxxx                       1/1     Running   0          2m
metallb-operator-controller-manager   1/1     Running   0          2m
speaker-xxxx                          1/1     Running   0          2m
speaker-yyyy                          1/1     Running   0          2m
```

> **Troubleshooting — CrashLoopBackOff:** If the controller pod is created but shows `CrashLoopBackOff`, the secrets were likely applied after install. Delete the controller pod to force a restart: `oc delete pod -n metallb-system -l component=controller`
>
> **Troubleshooting — 0/1 replicas, FailedCreate, never reaches CrashLoopBackOff:** This is the SCC issue, not the secrets issue. Confirm with `oc describe rs -n metallb-system` or `oc get events -n metallb-system --sort-by=.lastTimestamp`. If you see `unable to validate against any security context constraint` with every provider ending in `Forbidden: not usable by user or serviceaccount`, return to Step 3a.

---

## Step 4 — Configure IP Address Pools

Create `ipaddresspool.yaml`:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: workshop-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.130.100/32
  - 192.168.130.101/32
  - 192.168.126.100/32
  - 192.168.126.101/32
```

```bash
oc apply -f ipaddresspool.yaml
```

Create `l2advertisement.yaml`:

```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: workshop-l2
  namespace: metallb-system
spec:
  ipAddressPools:
  - workshop-pool
```

```bash
oc apply -f l2advertisement.yaml
```

VIP assignment:

```
VIP 1 = 192.168.130.100 / 192.168.126.100  -->  erp.ow.com
VIP 2 = 192.168.130.101 / 192.168.126.101  -->  hr.ow.com
```

---

## Step 5 — Create Namespace

```bash
oc new-project workshop
```

---

## Step 6 — Deploy ERP

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

`erp-svc.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: erp-svc
  namespace: workshop
  annotations:
    metallb.universe.tf/loadBalancerIPs: 192.168.130.100
spec:
  selector:
    app: erp
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer
```

```bash
oc apply -f erp-svc.yaml
```

---

## Step 7 — Deploy HR

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
  annotations:
    metallb.universe.tf/loadBalancerIPs: 192.168.130.101
spec:
  selector:
    app: hr
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer
```

```bash
oc apply -f hr-svc.yaml
```

---

## Step 8 — Create Routes

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

## Step 9 — Verify

```bash
oc get secrets -n metallb-system
oc get scc privileged -o yaml | grep -A5 users
oc get ipaddresspool -n metallb-system
oc get svc -n workshop
oc get routes -n workshop
oc get pods -n workshop
```

Expected `svc` output:

```
NAME      TYPE           CLUSTER-IP   EXTERNAL-IP       PORT(S)
erp-svc   LoadBalancer   10.x.x.x     192.168.130.100   80/TCP
hr-svc    LoadBalancer   10.x.x.x     192.168.130.101   80/TCP
```

Expected `routes` output:

```
NAME        HOST/PORT     SERVICES   PORT
route-erp   erp.ow.com    erp-svc    80
route-hr    hr.ow.com     hr-svc     80
```

---

## Step 10 — DNS

If DNS is not configured, add entries to `/etc/hosts`:

```bash
echo "192.168.130.100 erp.ow.com" | sudo tee -a /etc/hosts
echo "192.168.130.101 hr.ow.com" | sudo tee -a /etc/hosts
```

---

## Step 11 — Test

Since MetalLB L2 VIPs are reachable only from within the cluster network, run tests using a temporary pod:

```bash
oc run curltest --image=registry.access.redhat.com/ubi9/ubi --restart=Never --rm -it -n workshop -- curl -s http://192.168.126.100
```

```bash
oc run curltest --image=registry.access.redhat.com/ubi9/ubi --restart=Never --rm -it -n workshop -- curl -s http://192.168.126.101
```

---

## Troubleshooting

### Pods stuck at 0/1 replicas, FailedCreate events, never reach Pending/CrashLoopBackOff (SCC issue)

This means the ReplicaSet (`controller`) or DaemonSet (`speaker`) was rejected by SCC admission before a Pod object was ever created — this is **not** a secrets problem.

```bash
oc get rs -n metallb-system
oc describe rs <controller-replicaset-name> -n metallb-system
oc get events -n metallb-system --sort-by=.lastTimestamp | tail -20
```

Look for an event like:

```
Error creating: pods "controller-xxxx" is forbidden: unable to validate against any
security context constraint: [provider "anyuid": Forbidden: not usable by user or
serviceaccount, ... provider "privileged": Forbidden: not usable by user or serviceaccount]
```

If every provider in the list ends in `Forbidden: not usable by user or serviceaccount`, the service account has no SCC bound at all. Fix:

```bash
oc adm policy add-scc-to-user privileged -z controller -n metallb-system
oc adm policy add-scc-to-user privileged -z speaker -n metallb-system
```

No pod deletion is required — pending ReplicaSet/DaemonSet controllers retry automatically once the SCC is bound. See **Step 3a** for full details.

### Controller CrashLoopBackOff (secrets created after install)

This is a different failure mode: the pod *was* created and is starting, but crashing. This happens when secrets (`memberlist`, `webhook-server-cert`) were applied after MetalLB install.

Force a pod restart to pick up the secrets:

```bash
oc delete pod -n metallb-system -l component=controller
oc logs -n metallb-system deployment/controller
```

### Webhook service has no endpoints

The `webhook-service` showing `<none>` means the controller pod is not running. Confirm secrets and SCC bindings, then check pod status:

```bash
oc get secrets -n metallb-system
oc get scc privileged -o yaml | grep -A5 users
oc get endpoints -n metallb-system
oc get pods -n metallb-system
```

### Service stuck in `<pending>` (no IP assigned)

```bash
oc describe svc erp-svc -n workshop
oc get ipaddresspool -n metallb-system
oc get pods -n metallb-system
```

### 503 error — check pods and endpoints

```bash
oc get pods -n workshop
oc get endpoints erp-svc -n workshop
oc get endpoints hr-svc -n workshop
```

### EndpointSlice deprecation warning

The warning `v1 Endpoints is deprecated in v1.33+` is informational only and does not affect functionality:

```bash
oc get endpointslices -n metallb-system
```
