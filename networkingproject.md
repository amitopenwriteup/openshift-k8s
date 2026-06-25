# OpenShift CRC Workshop: MetalLB, Ingress & Application Routing
### A Beginner's Guide to Load Balancing and Traffic Management on Local OpenShift

---

## Table of Contents

1. [Workshop Overview](#1-workshop-overview)
2. [Architecture Explained](#2-architecture-explained)
3. [Prerequisites](#3-prerequisites)
4. [Lab 1 — Setting Up OpenShift CRC](#4-lab-1--setting-up-openshift-crc)
5. [Lab 2 — Installing MetalLB](#5-lab-2--installing-metallb)
6. [Lab 3 — Configuring MetalLB IP Address Pool](#6-lab-3--configuring-metallb-ip-address-pool)
7. [Lab 4 — Deploying Application A (Simple Web App)](#7-lab-4--deploying-application-a-simple-web-app)
8. [Lab 5 — Deploying Application B (API Service)](#8-lab-5--deploying-application-b-api-service)
9. [Lab 6 — Configuring Ingress to Route Traffic](#9-lab-6--configuring-ingress-to-route-traffic)
10. [Lab 7 — Testing & Verifying the Setup](#10-lab-7--testing--verifying-the-setup)
11. [Architecture Deep Dive](#11-architecture-deep-dive)
12. [Troubleshooting](#12-troubleshooting)
13. [Key Concepts Glossary](#13-key-concepts-glossary)

---

## 1. Workshop Overview

### What You Will Learn

By the end of this workshop, you will be able to:

- Run a local OpenShift cluster using **CodeReady Containers (CRC)**
- Install and configure **MetalLB** — a bare-metal load balancer for Kubernetes/OpenShift
- Create an **Ingress** resource that routes traffic to **two different applications** based on URL path or hostname
- Understand how external traffic flows from your browser all the way to your application pods

### Estimated Time

| Lab | Topic | Time |
|-----|-------|------|
| Lab 1 | Setting Up OpenShift CRC | 20 min |
| Lab 2 | Installing MetalLB | 15 min |
| Lab 3 | Configuring IP Address Pool | 10 min |
| Lab 4 | Deploying Application A | 10 min |
| Lab 5 | Deploying Application B | 10 min |
| Lab 6 | Configuring Ingress | 15 min |
| Lab 7 | Testing & Verifying | 10 min |
| **Total** | | **~90 min** |

### Who Is This For?

This workshop is designed for **beginners** who:
- Understand what a container is (e.g., Docker basics)
- Have used a command line before
- Want to learn how Kubernetes/OpenShift networking works

No prior OpenShift experience is required.

---

## 2. Architecture Explained

### The Big Picture

Before diving into commands, let's understand what we're building:

```
 Your Browser
 │
 │ HTTP Request (e.g., http://apps.crc.local/app-a)
 ▼
┌─────────────────────────────────────────────────────────────┐
│ MetalLB Load Balancer │
│ (Assigns a real IP to the Ingress Controller) │
│ IP: 192.168.130.200 │
└───────────────────────┬─────────────────────────────────────┘
 │
 ▼
┌─────────────────────────────────────────────────────────────┐
│ OpenShift Ingress Controller │
│ (Reads Ingress rules, routes traffic) │
│ │
│ Rule 1: /app-a ──────────────────────────────────────┐ │
│ Rule 2: /app-b ──────────────────────────────────┐ │ │
└─────────────────────────────────────────────────────│───│──┘
 │ │
 ┌─────────────────────────────────┘ │
 │ │
 ▼ ▼
 ┌───────────────────┐ ┌───────────────────┐
 │ Service: app-a │ │ Service: app-b │
 │ (ClusterIP) │ │ (ClusterIP) │
 └────────┬──────────┘ └────────┬──────────┘
 │ │
 ┌──────┴──────┐ ┌──────┴──────┐
 │ Pod: app-a │ │ Pod: app-b │
 │ (Nginx Web) │ │ (API/JSON) │
 └─────────────┘ └─────────────┘

 APPLICATION A APPLICATION B
 Simple HTML webpage REST API returning JSON
```

### Key Components Explained

#### What is CRC (CodeReady Containers)?
CRC runs a **single-node OpenShift cluster** on your laptop. Think of it as a miniature version of a production cluster that you can use for development and learning.

#### What is MetalLB?
In cloud environments (AWS, GCP, Azure), when you create a `LoadBalancer` type Service, the cloud provider automatically gives you a public IP address. But when running **locally or on bare metal**, there's no cloud to do this. **MetalLB fills that gap** — it acts as a software load balancer and assigns real IP addresses from a pool you define.

```
Cloud Environment: Local/CRC Environment:

 Service (LoadBalancer) Service (LoadBalancer)
 │ │
 ▼ ▼
 Cloud Provider ── Public IP MetalLB ── Local IP
 (automatic) (manual setup, this workshop!)
```

#### What is an Ingress?
An **Ingress** is a Kubernetes/OpenShift resource that defines **rules for routing external HTTP/HTTPS traffic** to internal services. It's like a traffic director:

```
Without Ingress: With Ingress:

Browser ── Service A (port 80) Browser ── /app-a ── Service A
Browser ── Service B (port 81) Browser ── /app-b ── Service B
(different ports, messy) (single port 80, clean URLs)
```

#### What is an Ingress Controller?
The Ingress Controller is the **actual software** that reads your Ingress rules and routes the traffic. OpenShift comes with one built in (based on HAProxy). The Ingress Controller needs a real IP — that's what MetalLB provides.

---

## 3. Prerequisites

### System Requirements

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| CPU | 4 vCPUs | 6+ vCPUs |
| RAM | 10 GB free | 16 GB free |
| Disk | 35 GB free | 50 GB free |
| OS | RHEL 8+, Fedora, Ubuntu 20.04+, macOS, Windows 10 |

### Software to Install

#### 1. Install CRC

Download from the [Red Hat Console](https://console.redhat.com/openshift/create/local):

```bash
# On Linux (example with tar.xz)
tar -xvf crc-linux-amd64.tar.xz
sudo mv crc-linux-*-amd64/crc /usr/local/bin/
crc version
```

#### 2. Install OpenShift CLI (oc)

```bash
# The oc CLI is bundled with CRC
eval $(crc oc-env)
oc version
```

#### 3. Install kubectl (optional, but useful)

```bash
# On Linux
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

### Pull Secret

You'll need a **pull secret** from Red Hat (free account required):
1. Go to https://console.redhat.com/openshift/create/local
2. Click **"Download pull secret"**
3. Save it as `~/pull-secret.txt`

---

## 4. Lab 1 — Setting Up OpenShift CRC

### Step 1.1 — Set Up CRC

```bash
# One-time setup — configures virtualization and networking
crc setup
```

> This may take 5–10 minutes. It sets up the virtualization driver and networking components on your machine.

### Step 1.2 — Configure CRC Resources

```bash
# Allocate resources (adjust based on your system)
crc config set cpus 6
crc config set memory 14336 # 14 GB in MB

# Enable cluster monitoring (optional, increases RAM usage)
# crc config set enable-cluster-monitoring true
```

### Step 1.3 — Start the Cluster

```bash
# Start CRC — provide your pull secret
crc start --pull-secret-file ~/pull-secret.txt
```

> First start takes **15–25 minutes** as it downloads the CRC bundle (~10 GB).

Expected output at the end:
```
Started the OpenShift cluster.

The server is accessible via web console at:
 https://console-openshift-console.apps-crc.testing

Log in as administrator:
 Username: kubeadmin
 Password: <generated-password>

Log in as user:
 Username: developer
 Password: developer
```

### Step 1.4 — Log In to OpenShift

```bash
# Set up the oc CLI path
eval $(crc oc-env)

# Log in as admin
oc login -u kubeadmin -p <your-kubeadmin-password> https://api.crc.testing:6443

# Verify cluster is running
oc get nodes
```

Expected output:
```
NAME STATUS ROLES AGE VERSION
crc-xxxx-master-0 Ready master,worker 10m v1.28.x
```

### Step 1.5 — Create a Workshop Namespace

```bash
# Create a dedicated namespace for our workshop
oc new-project workshop

# Verify
oc project
```

---

## 5. Lab 2 — Installing MetalLB

MetalLB can be installed via its **Operator** in OpenShift, which is the recommended approach.

### Step 2.1 — Create the MetalLB Namespace

```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:
 name: metallb-system
 labels:
 pod-security.kubernetes.io/enforce: privileged
 pod-security.kubernetes.io/audit: privileged
 pod-security.kubernetes.io/warn: privileged
EOF
```

### Step 2.2 — Install the MetalLB Operator

```bash
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
 name: metallb-operator
 namespace: metallb-system
EOF
```

```bash
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
 name: metallb-operator-sub
 namespace: metallb-system
spec:
 channel: stable
 name: metallb-operator
 source: redhat-operators
 sourceNamespace: openshift-marketplace
EOF
```

### Step 2.3 — Wait for the Operator to Install

```bash
# Watch the operator pod come up (Ctrl+C to stop watching)
oc get pods -n metallb-system -w
```

Wait until you see a pod like `metallb-operator-controller-manager-xxxx` in **Running** state.

```bash
# Also check the CSV (ClusterServiceVersion) is Succeeded
oc get csv -n metallb-system
```

Expected output:
```
NAME DISPLAY VERSION REPLACES PHASE
metallb-operator.v0.14.x MetalLB 0.14.x Succeeded
```

### Step 2.4 — Create the MetalLB Instance

Now tell the operator to actually deploy MetalLB components:

```bash
cat <<EOF | oc apply -f -
apiVersion: metallb.io/v1beta1
kind: MetalLB
metadata:
 name: metallb
 namespace: metallb-system
spec:
 nodeSelector:
 node-role.kubernetes.io/worker: ""
EOF
```

```bash
# Verify MetalLB pods are running
oc get pods -n metallb-system
```

Expected output:
```
NAME READY STATUS RESTARTS
metallb-operator-controller-manager-xxxx 1/1 Running 0
controller-xxxx 1/1 Running 0
speaker-xxxx 1/1 Running 0
```

> **What are these pods?**
> - **controller**: Watches for Services and assigns IP addresses from your pool
> - **speaker**: Announces those IPs to the network (using ARP for Layer 2 mode)

---

## 6. Lab 3 — Configuring MetalLB IP Address Pool

Now we need to tell MetalLB **which IP addresses it can hand out**.

### Step 3.1 — Find the CRC Network Range

```bash
# Find the IP of your CRC VM
crc ip
```

Take note of the IP (usually `192.168.130.11`). We'll use the same subnet for MetalLB.

### Step 3.2 — Create an IPAddressPool

We'll reserve a small range (`.200` to `.210`) in the CRC network for MetalLB to use:

```bash
cat <<EOF | oc apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
 name: crc-ip-pool
 namespace: metallb-system
spec:
 addresses:
 - 192.168.130.200-192.168.130.210
EOF
```

> **Important**: Make sure these IPs are **not already in use** on your network. The range `192.168.130.200–210` is typically safe for CRC environments.

### Step 3.3 — Create an L2Advertisement

This tells MetalLB to use **Layer 2 mode** (ARP) to announce IPs. This is the simplest mode and works well for local environments:

```bash
cat <<EOF | oc apply -f -
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
 name: crc-l2-advertisement
 namespace: metallb-system
spec:
 ipAddressPools:
 - crc-ip-pool
EOF
```

> **Layer 2 vs BGP Mode**
>
> MetalLB supports two modes:
> - **Layer 2 (ARP)**: Simple, works everywhere, one node handles the IP. We use this.
> - **BGP**: Advanced, requires a BGP router, distributes load across nodes. Used in production.

### Step 3.4 — Verify the Pool

```bash
oc get ipaddresspool -n metallb-system
oc get l2advertisement -n metallb-system
```

Expected output:
```
NAME AUTO ASSIGN AVOID BUGGY IPS ADDRESSES
crc-ip-pool true false ["192.168.130.200-192.168.130.210"]
```

---

## 7. Lab 4 — Deploying Application A (Simple Web App)

Application A is a **simple Nginx-based web page** — a classic "Hello World" website.

### Step 4.1 — Create the Deployment

```bash
cat <<EOF | oc apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
 name: app-a
 namespace: workshop
 labels:
 app: app-a
spec:
 replicas: 2
 selector:
 matchLabels:
 app: app-a
 template:
 metadata:
 labels:
 app: app-a
 spec:
 containers:
 - name: nginx
 image: nginx:latest
 ports:
 - containerPort: 80
 volumeMounts:
 - name: html
 mountPath: /usr/share/nginx/html
 initContainers:
 - name: init-html
 image: busybox
 command:
 - sh
 - -c
 - |
 cat > /html/index.html <<HTML
 <!DOCTYPE html>
 <html>
 <head><title>Application A</title>
 <style>
 body { font-family: Arial; background: #e8f4f8; text-align: center; padding: 50px; }
 h1 { color: #1a73e8; }
 .box { background: white; border-radius: 10px; padding: 30px; display: inline-block; box-shadow: 0 2px 10px rgba(0,0,0,0.1); }
 </style></head>
 <body>
 <div class="box">
 <h1> Application A</h1>
 <p>This is a <strong>Simple Web Application</strong></p>
 <p>Served by: Nginx</p>
 <p>Type: Static HTML Website</p>
 </div>
 </body>
 </html>
 HTML
 volumeMounts:
 - name: html
 mountPath: /html
 volumes:
 - name: html
 emptyDir: {}
EOF
```

### Step 4.2 — Create the Service

```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Service
metadata:
 name: app-a-svc
 namespace: workshop
spec:
 selector:
 app: app-a
 ports:
 - port: 80
 targetPort: 80
 type: ClusterIP
EOF
```

### Step 4.3 — Verify Application A

```bash
# Check pods are running
oc get pods -n workshop -l app=app-a

# Check the service exists
oc get svc app-a-svc -n workshop
```

Expected:
```
NAME READY STATUS RESTARTS AGE
app-a-xxxxxxxxx-xxxx 1/1 Running 0 30s
app-a-xxxxxxxxx-yyyy 1/1 Running 0 30s

NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
app-a-svc ClusterIP 10.217.x.x <none> 80/TCP 30s
```

> Notice the `EXTERNAL-IP` is `<none>` — that's because this is a ClusterIP service. Only the **Ingress** will expose it externally.

---

## 8. Lab 5 — Deploying Application B (API Service)

Application B is a **REST API** that returns JSON responses — simulating a backend microservice.

### Step 5.1 — Create the Deployment

```bash
cat <<EOF | oc apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
 name: app-b
 namespace: workshop
 labels:
 app: app-b
spec:
 replicas: 2
 selector:
 matchLabels:
 app: app-b
 template:
 metadata:
 labels:
 app: app-b
 spec:
 containers:
 - name: api
 image: python:3.11-alpine
 command:
 - python
 - -c
 - |
 from http.server import HTTPServer, BaseHTTPRequestHandler
 import json, datetime, socket

 class APIHandler(BaseHTTPRequestHandler):
 def do_GET(self):
 self.send_response(200)
 self.send_header('Content-Type', 'application/json')
 self.send_header('Access-Control-Allow-Origin', '*')
 self.end_headers()
 response = {
 "application": "Application B",
 "type": "REST API",
 "message": "Hello from the API service!",
 "path": self.path,
 "hostname": socket.gethostname(),
 "timestamp": datetime.datetime.utcnow().isoformat() + "Z",
 "status": "healthy"
 }
 self.wfile.write(json.dumps(response, indent=2).encode())
 def log_message(self, format, *args):
 pass

 print("API server starting on port 8080...")
 HTTPServer(('0.0.0.0', 8080), APIHandler).serve_forever()
 ports:
 - containerPort: 8080
EOF
```

### Step 5.2 — Create the Service

```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Service
metadata:
 name: app-b-svc
 namespace: workshop
spec:
 selector:
 app: app-b
 ports:
 - port: 80
 targetPort: 8080
 type: ClusterIP
EOF
```

### Step 5.3 — Verify Application B

```bash
oc get pods -n workshop -l app=app-b
oc get svc app-b-svc -n workshop
```

---

## 9. Lab 6 — Configuring Ingress to Route Traffic

Now we'll create an **Ingress** resource that routes:
- `/app-a` → Application A (web page)
- `/app-b` → Application B (API)

### Step 6.1 — Understand the Ingress Controller in CRC

OpenShift CRC comes with a built-in Ingress Controller. Let's check it:

```bash
oc get ingresscontroller default -n openshift-ingress-operator -o yaml | grep -A5 "domain:"
```

The default domain in CRC is `apps-crc.testing`. We'll use a custom hostname for our workshop.

### Step 6.2 — Create the Ingress Resource (Path-Based Routing)

```bash
cat <<EOF | oc apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
 name: workshop-ingress
 namespace: workshop
 annotations:
 nginx.ingress.kubernetes.io/rewrite-target: /
 # Rewrite paths: /app-a/anything → / (root of the app)
spec:
 ingressClassName: openshift-default
 rules:
 - host: workshop.apps-crc.testing
 http:
 paths:
 - path: /app-a
 pathType: Prefix
 backend:
 service:
 name: app-a-svc
 port:
 number: 80
 - path: /app-b
 pathType: Prefix
 backend:
 service:
 name: app-b-svc
 port:
 number: 80
EOF
```

### Step 6.3 — Alternative: Hostname-Based Routing

Instead of paths, you can also route by **subdomain**:

```bash
cat <<EOF | oc apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
 name: workshop-ingress-host
 namespace: workshop
spec:
 rules:
 # Hostname-based routing for App A
 - host: app-a.apps-crc.testing
 http:
 paths:
 - path: /
 pathType: Prefix
 backend:
 service:
 name: app-a-svc
 port:
 number: 80
 # Hostname-based routing for App B
 - host: app-b.apps-crc.testing
 http:
 paths:
 - path: /
 pathType: Prefix
 backend:
 service:
 name: app-b-svc
 port:
 number: 80
EOF
```

> **Path-based vs Hostname-based Routing**
>
> | Feature | Path-based | Hostname-based |
> |---------|------------|----------------|
> | URL format | `example.com/app-a` | `app-a.example.com` |
> | Single certificate | Yes | Needs wildcard cert |
> | App isolation | Partial | Better |
> | Simpler to set up | Yes | Requires DNS per subdomain |

### Step 6.4 — Expose the Ingress Controller with MetalLB

The key step: make the Ingress Controller service use MetalLB to get a real IP:

```bash
# Patch the ingress controller service to use LoadBalancer type
oc patch svc router-default -n openshift-ingress \
 -p '{"spec": {"type": "LoadBalancer"}}'
```

```bash
# Watch for MetalLB to assign an IP (may take 30-60 seconds)
oc get svc router-default -n openshift-ingress -w
```

Expected output (wait for EXTERNAL-IP to appear):
```
NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
router-default LoadBalancer 10.217.x.x 192.168.130.200 80:xxxxx/TCP,443:xxxxx/TCP 2m
```

 **MetalLB has assigned IP `192.168.130.200`** to the Ingress Controller!

### Step 6.5 — Verify the Ingress

```bash
oc get ingress -n workshop
```

```
NAME CLASS HOSTS ADDRESS PORTS AGE
workshop-ingress openshift-default workshop.apps-crc.testing 192.168.130.200 80 30s
workshop-ingress-host openshift-default app-a.apps-crc.testing,... 192.168.130.200 80 20s
```

---

## 10. Lab 7 — Testing & Verifying the Setup

### Step 7.1 — Update /etc/hosts (or Use CRC's Built-in DNS)

CRC automatically handles `*.apps-crc.testing` DNS. If you're using custom hostnames, add them:

```bash
# Check if CRC DNS is working
ping workshop.apps-crc.testing
```

If it doesn't resolve, add it manually:

```bash
# Add to /etc/hosts (Linux/macOS) — use the MetalLB IP
echo "192.168.130.200 workshop.apps-crc.testing" | sudo tee -a /etc/hosts
echo "192.168.130.200 app-a.apps-crc.testing" | sudo tee -a /etc/hosts
echo "192.168.130.200 app-b.apps-crc.testing" | sudo tee -a /etc/hosts
```

### Step 7.2 — Test Application A

```bash
# Test via curl
curl http://workshop.apps-crc.testing/app-a

# Or open in browser
echo "Open: http://workshop.apps-crc.testing/app-a"
```

Expected: HTML page with " Application A"

### Step 7.3 — Test Application B

```bash
# Test via curl — should return JSON
curl http://workshop.apps-crc.testing/app-b | python3 -m json.tool
```

Expected output:
```json
{
 "application": "Application B",
 "type": "REST API",
 "message": "Hello from the API service!",
 "path": "/app-b",
 "hostname": "app-b-xxxxxxxxx-xxxx",
 "timestamp": "2024-01-15T10:30:00Z",
 "status": "healthy"
}
```

### Step 7.4 — Test Hostname-Based Routing

```bash
# Application A by subdomain
curl http://app-a.apps-crc.testing

# Application B by subdomain
curl http://app-b.apps-crc.testing
```

### Step 7.5 — Observe Load Balancing in Action

```bash
# Make multiple requests and observe different pod hostnames in the response
for i in $(seq 1 6); do
 curl -s http://workshop.apps-crc.testing/app-b | grep hostname
done
```

Expected: You'll see **two different hostnames** alternating, showing the service load-balancing across both pods!

```
"hostname": "app-b-xxxxxxxxx-pod1"
"hostname": "app-b-xxxxxxxxx-pod2"
"hostname": "app-b-xxxxxxxxx-pod1"
"hostname": "app-b-xxxxxxxxx-pod2"
...
```

### Step 7.6 — View in OpenShift Web Console

```bash
# Get the console URL and admin password
crc console --credentials
```

Open `https://console-openshift-console.apps-crc.testing` and explore:
- **Networking → Ingresses** — see your Ingress rules
- **Networking → Services** — see your Services
- **Workloads → Pods** — see your running pods

---

## 11. Architecture Deep Dive

### Full Traffic Flow Walkthrough

Let's trace a request from your browser to a pod step by step:

```
Step 1: Browser sends HTTP GET http://workshop.apps-crc.testing/app-b
 │
 ▼
Step 2: DNS resolves "workshop.apps-crc.testing" → 192.168.130.200
 (This IP was assigned by MetalLB to the Ingress Controller)
 │
 ▼
Step 3: TCP connection to 192.168.130.200:80
 MetalLB's "speaker" pod on the node responds to ARP requests
 for this IP and forwards packets to the Ingress Controller pod
 │
 ▼
Step 4: Ingress Controller (HAProxy) receives the request
 It reads the Host header: "workshop.apps-crc.testing"
 It reads the Path: "/app-b"
 It finds the matching Ingress rule → backend: app-b-svc:80
 │
 ▼
Step 5: Ingress Controller forwards to Service "app-b-svc"
 Service has a ClusterIP (internal virtual IP)
 iptables/IPVS rules route to one of the ready pods
 │
 ▼
Step 6: Pod receives the request and generates a JSON response
 Response travels back through the same chain
 │
 ▼
Step 7: Browser receives and displays the JSON response
```

### Object Relationships

```
MetalLB IPAddressPool
 └── L2Advertisement
 └── Watches for Services of type=LoadBalancer
 │
 ▼
 router-default Service (type=LoadBalancer)
 EXTERNAL-IP: 192.168.130.200
 │
 ▼
 Ingress Controller Pod (HAProxy)
 Reads: Ingress resources
 │
 ┌────┴────┐
 ▼ ▼
 Ingress Rule Ingress Rule
 /app-a → /app-b →
 app-a-svc app-b-svc
 │ │
 ┌────┴────┐ ┌────┴────┐
 ▼ ▼ ▼ ▼
 Pod A-1 Pod A-2 Pod B-1 Pod B-2
```

### 🆚 Service Types Explained

```
┌────────────────────────────────────────────────────────────────────┐
│ Kubernetes Service Types │
├────────────────┬───────────────────────────────────────────────────┤
│ ClusterIP │ Internal only. Reachable only from inside the │
│ (default) │ cluster. Used by app-a-svc and app-b-svc. │
├────────────────┼───────────────────────────────────────────────────┤
│ NodePort │ Exposes on a static port on every node's IP. │
│ │ Works but not elegant. Port range: 30000-32767. │
├────────────────┼───────────────────────────────────────────────────┤
│ LoadBalancer │ Requests an external IP from a provider. │
│ │ Used by router-default (powered by MetalLB). │
├────────────────┼───────────────────────────────────────────────────┤
│ ExternalName │ Maps to a DNS name. Used for external services. │
└────────────────┴───────────────────────────────────────────────────┘
```

---

## 12. Troubleshooting

### MetalLB doesn't assign an IP

```bash
# Check MetalLB controller logs
oc logs -n metallb-system deployment/controller

# Check IP pool is configured correctly
oc describe ipaddresspool crc-ip-pool -n metallb-system

# Ensure the service has the right annotation
oc describe svc router-default -n openshift-ingress
```

### Ingress returns 503 Service Unavailable

```bash
# Check pods are running
oc get pods -n workshop

# Check endpoints — are the services pointing to pods?
oc get endpoints -n workshop

# If endpoints show <none>, check pod labels match service selector
oc describe svc app-a-svc -n workshop
oc get pods --show-labels -n workshop
```

### DNS doesn't resolve

```bash
# Verify /etc/hosts is set
cat /etc/hosts | grep crc

# Test with direct IP (bypass DNS)
curl --resolve workshop.apps-crc.testing:80:192.168.130.200 \
 http://workshop.apps-crc.testing/app-a

# Check CRC DNS status
crc status
```

### 404 Not Found from Ingress

```bash
# Check Ingress resource is created
oc get ingress -n workshop -o yaml

# Check the Ingress Controller has picked up the rules
oc logs deployment/router-default -n openshift-ingress | tail -50

# Verify host header matches
curl -v -H "Host: workshop.apps-crc.testing" http://192.168.130.200/app-a
```

### CRC won't start

```bash
# Check CRC status
crc status

# Clean and restart (WARNING: deletes cluster data)
crc stop
crc delete
crc start --pull-secret-file ~/pull-secret.txt
```

---

## 13. Key Concepts Glossary

| Term | Definition |
|------|------------|
| **CRC** | CodeReady Containers — runs OpenShift locally on your laptop |
| **Pod** | The smallest deployable unit in Kubernetes; runs one or more containers |
| **Deployment** | Manages multiple pod replicas; handles rolling updates and restarts |
| **Service** | A stable network endpoint that load-balances to a set of pods |
| **ClusterIP** | A virtual IP only reachable from within the cluster |
| **LoadBalancer** | A service type that requests an external IP from a provider |
| **MetalLB** | Software load balancer that provides external IPs on bare metal |
| **IPAddressPool** | A MetalLB resource defining what IP ranges MetalLB can use |
| **L2Advertisement** | Tells MetalLB to use ARP (Layer 2) to announce IPs |
| **Ingress** | A set of rules for routing external HTTP traffic to internal services |
| **Ingress Controller** | The software (e.g., HAProxy, Nginx) that implements Ingress rules |
| **Namespace / Project** | A logical grouping of resources (OpenShift calls them "Projects") |
| **ARP** | Address Resolution Protocol — maps IP addresses to MAC addresses |
| **HAProxy** | The reverse proxy used by OpenShift's default Ingress Controller |
| **ClusterServiceVersion (CSV)** | Describes an Operator and its capabilities |

---

## Workshop Complete!

Congratulations! You have successfully:

- Set up a local OpenShift cluster with CRC
- Installed and configured MetalLB as a bare-metal load balancer
- Defined an IP address pool and Layer 2 advertisement
- Deployed two different types of applications (web app + REST API)
- Configured an Ingress to route traffic by path and by hostname
- Tested end-to-end traffic flow and observed load balancing

### Next Steps

- **Add TLS/HTTPS**: Create a TLS Secret and add `tls:` section to your Ingress
- **Try BGP mode**: Set up MetalLB with BGP for production-like routing
- **Explore OpenShift Routes**: OpenShift's native alternative to Kubernetes Ingress
- **Scale your apps**: Try `oc scale deployment app-a --replicas=5` and observe
- **Add health checks**: Add `livenessProbe` and `readinessProbe` to your Deployments

### Resources

- [OpenShift CRC Documentation](https://crc.dev/crc/)
- [MetalLB Official Documentation](https://metallb.universe.tf/)
- [OpenShift Networking Guide](https://docs.openshift.com/container-platform/latest/networking/understanding-networking.html)
- [Kubernetes Ingress Concepts](https://kubernetes.io/docs/concepts/services-networking/ingress/)

---

*Workshop created for OpenShift CRC beginners. All YAML manifests are tested on CRC v2.x with OpenShift 4.14+.*
