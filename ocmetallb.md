# OpenShift Routes - HTTP Only (Two VIPs) — Fixed Lab Guide

> **Fix Applied:** MetalLB controller fails to start when the `memberlist` secret and webhook TLS secret are missing. Secrets must be created **before** installing MetalLB so pods can start cleanly on first boot.

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

Now that secrets are in place, install MetalLB. The controller and speaker pods will find the secrets immediately on startup:

```bash
oc apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml
```

Wait for CRDs to be established:

```bash
oc wait --namespace metallb-system --for=condition=established crd/ipaddresspools.metallb.io --timeout=60s
```

Wait for all pods to be ready:

```bash
oc wait --namespace metallb-system --for=condition=ready pod --selector=app=metallb --timeout=120s
```

Verify all pods are running:

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

> **Troubleshooting:** If the controller shows `CrashLoopBackOff`, secrets were likely applied after install. Delete the controller pod to force a restart: `oc delete pod -n metallb-system -l component=controller`

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
VIP 1 = 192.168.130.100  -->  erp.ow.com
VIP 2 = 192.168.130.101  -->  hr.ow.com
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

```bash
curl http://erp.ow.com
curl http://hr.ow.com
```

---

## Troubleshooting

### Controller CrashLoopBackOff (secrets created after install)

Force a pod restart to pick up the secrets:

```bash
oc delete pod -n metallb-system -l component=controller
oc logs -n metallb-system deployment/controller
```

### Webhook service has no endpoints

The `webhook-service` showing `<none>` means the controller pod is not running. Confirm secrets exist and restart the controller:

```bash
oc get secrets -n metallb-system
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
