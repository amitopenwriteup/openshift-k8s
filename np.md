# OpenShift Network Policy Workshop
## Hands-On Lab: Ingress & Egress Controls with SCC and UID on CRC

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Workshop Overview](#workshop-overview)
3. [Environment Setup — OpenShift CRC](#environment-setup--openshift-crc)
4. [Security Context Constraints (SCC) & UID](#security-context-constraints-scc--uid)
5. [Deploy the Two Test Applications](#deploy-the-two-test-applications)
6. [Understanding Network Policies](#understanding-network-policies)
7. [Lab 1 — Default Deny All](#lab-1--default-deny-all)
8. [Lab 2 — Allow Ingress Between Apps](#lab-2--allow-ingress-between-apps)
9. [Lab 3 — Egress Control](#lab-3--egress-control)
10. [Testing Methodology](#testing-methodology)
11. [Cleanup](#cleanup)
12. [Reference: Policy Cheat Sheet](#reference-policy-cheat-sheet)

---

## Prerequisites

| Tool | Version | Purpose |
|------|---------|---------|
| OpenShift CRC (CodeReady Containers) | ≥ 2.x | Local OpenShift cluster |
| `oc` CLI | Matches CRC version | Cluster interaction |
| `kubectl` | ≥ 1.25 | Alternative CLI |
| `curl` / `wget` | Any | HTTP testing inside pods |

```bash
# Verify CRC is running
crc status

# Log in as kubeadmin
oc login -u kubeadmin -p $(crc console --credentials | grep kubeadmin | awk '{print $NF}') https://api.crc.testing:6443
```

---

## Workshop Overview

In this workshop you will:

- Deploy **two applications** (`app-frontend` and `app-backend`) in dedicated namespaces.
- Configure **Security Context Constraints (SCC)** with specific `runAsUser` UIDs so pods run with controlled identities.
- Write and apply **NetworkPolicy** objects that control:
  - **Ingress** — who is allowed to talk *to* a pod.
  - **Egress** — where a pod is allowed to talk *to*.
- **Verify** each policy by exec-ing into pods and probing connectivity.

```
┌────────────────────────────────────────────────────────────┐
│  Namespace: workshop-frontend     Namespace: workshop-backend │
│                                                            │
│  ┌──────────────────┐   Ingress   ┌─────────────────────┐  │
│  │  app-frontend    │ ──────────► │   app-backend       │  │
│  │  UID: 1001       │             │   UID: 1002         │  │
│  └──────────────────┘             └─────────────────────┘  │
│         │                                  │               │
│         │ Egress (DNS only)                │ Egress (blocked) │
│         ▼                                  ▼               │
│      Internet / DNS                   [Denied]             │
└────────────────────────────────────────────────────────────┘
```

---

## Environment Setup — OpenShift CRC

### 1. Start CRC and Set Up Namespaces

```bash
crc start

oc new-project workshop-frontend
oc new-project workshop-backend

oc get namespaces | grep workshop
```

### 2. Label the Namespaces

Labels are used by NetworkPolicy selectors to identify namespaces.

```bash
oc label namespace workshop-frontend environment=workshop role=frontend
oc label namespace workshop-backend environment=workshop role=backend

oc get namespace workshop-frontend -o jsonpath='{.metadata.labels}' | python3 -m json.tool
oc get namespace workshop-backend -o jsonpath='{.metadata.labels}' | python3 -m json.tool
```

---

## Security Context Constraints (SCC) & UID

OpenShift uses SCCs to control what a pod is *allowed to do* at the OS level. For this workshop we use a custom SCC that pins each application to a specific UID, preventing privilege escalation.

### Why UID Matters for Network Policy

When combined with NetworkPolicy:
- A known UID makes it easier to audit which process generated network traffic.
- `anyuid` SCC is avoided in production; we pin UIDs instead.
- CRC ships with `restricted` SCC by default — we must grant access explicitly.

### Create a Custom SCC (`workshop-scc`)

```yaml
# workshop-scc.yaml
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: workshop-scc
allowPrivilegedContainer: false
allowPrivilegeEscalation: false
allowedCapabilities: []
defaultAddCapabilities: []
requiredDropCapabilities:
  - ALL
runAsUser:
  type: MustRunAs
  uid: 1001
fsGroup:
  type: MustRunAs
  ranges:
    - min: 1001
      max: 1002
seLinuxContext:
  type: MustRunAs
supplementalGroups:
  type: RunAsAny
volumes:
  - configMap
  - downwardAPI
  - emptyDir
  - secret
```

```bash
oc apply -f workshop-scc.yaml

oc adm policy add-scc-to-user workshop-scc system:serviceaccount:workshop-frontend:default
oc adm policy add-scc-to-user workshop-scc system:serviceaccount:workshop-backend:default

oc describe scc workshop-scc
```

---

## Deploy the Two Test Applications

### App 1 — Frontend (UID 1001)

```yaml
# frontend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-frontend
  namespace: workshop-frontend
  labels:
    app: app-frontend
    tier: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app-frontend
  template:
    metadata:
      labels:
        app: app-frontend
        tier: frontend
    spec:
      securityContext:
        runAsUser: 1001
        runAsGroup: 1001
        fsGroup: 1001
      containers:
        - name: frontend
          image: registry.access.redhat.com/ubi9/ubi-minimal:latest
          command: ["/bin/sh", "-c"]
          args:
            - |
              microdnf install -y curl ncat && while true; do echo "frontend running"; sleep 30; done
          ports:
            - containerPort: 8080
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
---
apiVersion: v1
kind: Service
metadata:
  name: app-frontend-svc
  namespace: workshop-frontend
spec:
  selector:
    app: app-frontend
  ports:
    - port: 8080
      targetPort: 8080
```

### App 2 — Backend (UID 1002)

```yaml
# backend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-backend
  namespace: workshop-backend
  labels:
    app: app-backend
    tier: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app-backend
  template:
    metadata:
      labels:
        app: app-backend
        tier: backend
    spec:
      securityContext:
        runAsUser: 1002
        runAsGroup: 1002
        fsGroup: 1002
      containers:
        - name: backend
          image: registry.access.redhat.com/ubi9/ubi-minimal:latest
          command: ["/bin/sh", "-c"]
          args:
            - |
              microdnf install -y ncat curl && while true; do echo "HTTP/1.1 200 OK\r\nContent-Length: 13\r\n\r\nHello Backend" | nc -l -p 8080 -q 1; done
          ports:
            - containerPort: 8080
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
---
apiVersion: v1
kind: Service
metadata:
  name: app-backend-svc
  namespace: workshop-backend
spec:
  selector:
    app: app-backend
  ports:
    - port: 8080
      targetPort: 8080
```

```bash
oc apply -f frontend-deployment.yaml
oc apply -f backend-deployment.yaml

oc rollout status deployment/app-frontend -n workshop-frontend
oc rollout status deployment/app-backend -n workshop-backend

# Verify UIDs (must show 1001 and 1002 respectively)
oc exec -n workshop-frontend deploy/app-frontend -- id
oc exec -n workshop-backend deploy/app-backend -- id
```

---

## Understanding Network Policies

A `NetworkPolicy` in Kubernetes/OpenShift is a namespace-scoped object that selects pods and defines rules.

```
NetworkPolicy
  ├── podSelector       →  which pods THIS policy applies to
  ├── policyTypes       →  [Ingress, Egress, or both]
  ├── ingress[]         →  allowed INBOUND sources
  │     ├── from[].namespaceSelector
  │     ├── from[].podSelector
  │     └── ports[]
  └── egress[]          →  allowed OUTBOUND destinations
        ├── to[].namespaceSelector
        ├── to[].podSelector
        ├── to[].ipBlock
        └── ports[]
```

**Key rules:**
- If **no** NetworkPolicy selects a pod → all traffic is allowed (default-open).
- If **any** NetworkPolicy selects a pod → only explicitly allowed traffic is permitted.
- `policyTypes: [Ingress]` alone does not restrict egress, and vice-versa.

---

## Lab 1 — Default Deny All

This is the foundation of a zero-trust network posture. Apply a blanket deny to each namespace, then add allow rules selectively.

### 1a. Deny All Ingress in Backend Namespace

```yaml
# deny-all-backend.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: workshop-backend
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  # No ingress rules = deny all inbound
```

### 1b. Deny All Egress in Backend Namespace

```yaml
# deny-all-egress-backend.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-egress
  namespace: workshop-backend
spec:
  podSelector: {}
  policyTypes:
    - Egress
  # No egress rules = deny all outbound
```

```bash
oc apply -f deny-all-backend.yaml
oc apply -f deny-all-egress-backend.yaml

oc get networkpolicy -n workshop-backend
```

### Test: Connectivity Should Now Fail

```bash
FRONTEND_POD=$(oc get pod -n workshop-frontend -l app=app-frontend -o jsonpath='{.items[0].metadata.name}')
BACKEND_SVC_IP=$(oc get svc app-backend-svc -n workshop-backend -o jsonpath='{.spec.clusterIP}')

oc exec -n workshop-frontend $FRONTEND_POD -- curl -v --connect-timeout 5 http://$BACKEND_SVC_IP:8080

# Expected: Connection timed out
```

---

## Lab 2 — Allow Ingress Between Apps

Now allow `app-frontend` (in `workshop-frontend`) to reach `app-backend` (in `workshop-backend`) on port 8080 only.

### 2a. Allow Ingress to Backend from Frontend Namespace

```yaml
# allow-ingress-from-frontend.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-from-frontend
  namespace: workshop-backend
spec:
  podSelector:
    matchLabels:
      app: app-backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              role: frontend
          podSelector:
            matchLabels:
              app: app-frontend
      ports:
        - protocol: TCP
          port: 8080
```

> **Note:** When `namespaceSelector` and `podSelector` are on the **same** `from` entry (no extra `-` dash), they are ANDed — both conditions must be true. Separated by a `-`, they are ORed.

```bash
oc apply -f allow-ingress-from-frontend.yaml

oc get networkpolicy -n workshop-backend
```

### 2b. Test: Connectivity Should Succeed on Port 8080

```bash
FRONTEND_POD=$(oc get pod -n workshop-frontend -l app=app-frontend -o jsonpath='{.items[0].metadata.name}')
BACKEND_SVC_IP=$(oc get svc app-backend-svc -n workshop-backend -o jsonpath='{.spec.clusterIP}')

# Should succeed
oc exec -n workshop-frontend $FRONTEND_POD -- curl -v --connect-timeout 5 http://$BACKEND_SVC_IP:8080

# Port 9999 should still fail
oc exec -n workshop-frontend $FRONTEND_POD -- curl -v --connect-timeout 5 http://$BACKEND_SVC_IP:9999
```

### 2c. Verify a Rogue Pod Cannot Access Backend

```bash
oc new-project workshop-rogue
oc run rogue --image=registry.access.redhat.com/ubi9/ubi-minimal:latest --command -- sleep 9999 -n workshop-rogue

oc exec -n workshop-rogue rogue -- curl -v --connect-timeout 5 http://$BACKEND_SVC_IP:8080

# Expected: Connection timed out — policy blocks it
```

---

## Lab 3 — Egress Control

Control where pods are allowed to **send** traffic. This prevents data exfiltration and limits blast radius.

### 3a. Allow Backend to Reach DNS Only

DNS resolution (port 53) is required for service discovery. All other egress is blocked.

```yaml
# allow-egress-dns-backend.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-dns
  namespace: workshop-backend
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

```bash
oc apply -f allow-egress-dns-backend.yaml
```

### 3b. Allow Frontend Egress to Backend Only

```yaml
# allow-egress-frontend-to-backend.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-to-backend
  namespace: workshop-frontend
spec:
  podSelector:
    matchLabels:
      app: app-frontend
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              role: backend
          podSelector:
            matchLabels:
              app: app-backend
      ports:
        - protocol: TCP
          port: 8080
    - ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

```bash
oc apply -f allow-egress-frontend-to-backend.yaml

# Frontend can reach backend — should succeed
oc exec -n workshop-frontend $FRONTEND_POD -- curl -v --connect-timeout 5 http://$BACKEND_SVC_IP:8080

# Frontend cannot reach external internet — should fail
oc exec -n workshop-frontend $FRONTEND_POD -- curl -v --connect-timeout 5 https://www.google.com
```

### 3c. Allow Egress to a Specific External IP Block

If the backend legitimately needs to call an external API at `203.0.113.0/24`:

```yaml
# allow-egress-external-cidr.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-external-api
  namespace: workshop-backend
spec:
  podSelector:
    matchLabels:
      app: app-backend
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 203.0.113.0/24
            except:
              - 203.0.113.1/32
      ports:
        - protocol: TCP
          port: 443
    - ports:
        - protocol: UDP
          port: 53
```

```bash
oc apply -f allow-egress-external-cidr.yaml
```

---

## Testing Methodology

### Connectivity Matrix

Set helper variables first, then run each test individually:

```bash
# Set variables
FRONTEND_POD=$(oc get pod -n workshop-frontend -l app=app-frontend -o jsonpath='{.items[0].metadata.name}')
BACKEND_POD=$(oc get pod -n workshop-backend -l app=app-backend -o jsonpath='{.items[0].metadata.name}')
BACKEND_IP=$(oc get svc app-backend-svc -n workshop-backend -o jsonpath='{.spec.clusterIP}')
FRONTEND_IP=$(oc get svc app-frontend-svc -n workshop-frontend -o jsonpath='{.spec.clusterIP}')

# [1] Frontend → Backend:8080 — SHOULD SUCCEED
oc exec -n workshop-frontend $FRONTEND_POD -- curl -s -o /dev/null -w "%{http_code}" --connect-timeout 5 http://$BACKEND_IP:8080

# [2] Frontend → Backend:9999 — SHOULD FAIL (port not allowed)
oc exec -n workshop-frontend $FRONTEND_POD -- curl -s -o /dev/null -w "%{http_code}" --connect-timeout 5 http://$BACKEND_IP:9999

# [3] Backend → Frontend — SHOULD FAIL (egress blocked)
oc exec -n workshop-backend $BACKEND_POD -- curl -s -o /dev/null -w "%{http_code}" --connect-timeout 5 http://$FRONTEND_IP:8080

# [4] Frontend → External Internet — SHOULD FAIL (egress restricted)
oc exec -n workshop-frontend $FRONTEND_POD -- curl -s -o /dev/null -w "%{http_code}" --connect-timeout 5 https://1.1.1.1

# [5] DNS from Frontend — SHOULD SUCCEED
oc exec -n workshop-frontend $FRONTEND_POD -- nslookup app-backend-svc.workshop-backend.svc.cluster.local

# UID Verification
oc exec -n workshop-frontend $FRONTEND_POD -- id
oc exec -n workshop-backend $BACKEND_POD -- id
```

### Inspect Active Policies

```bash
oc get networkpolicy -n workshop-frontend
oc get networkpolicy -n workshop-backend

oc describe networkpolicy allow-ingress-from-frontend -n workshop-backend

oc get networkpolicy allow-ingress-from-frontend -n workshop-backend -o yaml
```

### Check OVN/SDN Logs (CRC)

```bash
oc logs -n openshift-ovn-kubernetes $(oc get pod -n openshift-ovn-kubernetes -l app=ovnkube-node -o jsonpath='{.items[0].metadata.name}') -c ovnkube-controller --tail=50

oc get clusteroperator network
```

---

## Cleanup

```bash
oc delete networkpolicy --all -n workshop-frontend
oc delete networkpolicy --all -n workshop-backend

oc delete -f frontend-deployment.yaml
oc delete -f backend-deployment.yaml

oc adm policy remove-scc-from-user workshop-scc system:serviceaccount:workshop-frontend:default
oc adm policy remove-scc-from-user workshop-scc system:serviceaccount:workshop-backend:default

oc delete scc workshop-scc

oc delete project workshop-frontend workshop-backend workshop-rogue
```

---

## Reference: Policy Cheat Sheet

### Ingress Rules

| Goal | Key Field |
|------|-----------|
| Allow from specific namespace | `from[].namespaceSelector.matchLabels` |
| Allow from specific pod | `from[].podSelector.matchLabels` |
| Allow from specific pod IN specific namespace | Both selectors on same `from` entry |
| Allow from any pod in cluster | `from[].namespaceSelector: {}` |
| Deny all inbound | `policyTypes: [Ingress]` with no `ingress` block |

### Egress Rules

| Goal | Key Field |
|------|-----------|
| Allow to specific namespace | `to[].namespaceSelector.matchLabels` |
| Allow to specific pod | `to[].podSelector.matchLabels` |
| Allow to external CIDR | `to[].ipBlock.cidr` |
| Exclude IPs from CIDR | `to[].ipBlock.except[]` |
| Allow DNS only | Port 53 UDP + TCP, no `to` selector |
| Deny all outbound | `policyTypes: [Egress]` with no `egress` block |

### Common Port References

| Service | Protocol | Port |
|---------|----------|------|
| DNS | UDP/TCP | 53 |
| HTTP | TCP | 80 |
| HTTPS | TCP | 443 |
| App (workshop) | TCP | 8080 |
| OpenShift API | TCP | 6443 |

### SCC Quick Reference

| SCC | UID Behaviour | Use Case |
|-----|--------------|----------|
| `restricted` | UID allocated from namespace range | Default, most pods |
| `restricted-v2` | Same as restricted, stricter caps | OCP 4.11+ default |
| `anyuid` | Any UID including root | Legacy apps — avoid |
| `workshop-scc` | MustRunAs specific UID | This workshop |

---

## Summary of Policies Applied

```
workshop-backend namespace
├── deny-all-ingress              (Lab 1a) — Blocks all inbound
├── deny-all-egress               (Lab 1b) — Blocks all outbound
├── allow-ingress-from-frontend   (Lab 2a) — Opens port 8080 from frontend only
├── allow-egress-dns              (Lab 3a) — Permits DNS resolution
└── allow-egress-external-api     (Lab 3c) — Permits specific external CIDR

workshop-frontend namespace
└── allow-egress-to-backend       (Lab 3b) — Permits outbound to backend:8080 + DNS
```

---

*Workshop version 1.1 — OpenShift CRC · OVN-Kubernetes CNI · NetworkPolicy v1*
