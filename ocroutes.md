# OpenShift CRC Workshop: Routing Traffic with OpenShift Routes
### A Beginner's Guide to OpenShift-Native Application Routing on Local OpenShift

---

## Table of Contents

1. [Workshop Overview](#1-workshop-overview)
2. [Architecture Explained](#2-architecture-explained)
3. [Routes vs Ingress — Key Differences](#3-routes-vs-ingress--key-differences)
4. [Prerequisites](#4-prerequisites)
5. [Lab 1 — Setting Up OpenShift CRC](#5-lab-1--setting-up-openshift-crc)
6. [Lab 2 — Deploying Application A (Simple Web App)](#6-lab-2--deploying-application-a-simple-web-app)
7. [Lab 3 — Deploying Application B (API Service)](#7-lab-3--deploying-application-b-api-service)
8. [Lab 4 — Creating a Route for Application A](#8-lab-4--creating-a-route-for-application-a)
9. [Lab 5 — Creating a Route for Application B](#9-lab-5--creating-a-route-for-application-b)
10. [Lab 6 — Advanced Routing (Path-Based & TLS)](#10-lab-6--advanced-routing-path-based--tls)
11. [Lab 7 — Testing & Verifying the Setup](#11-lab-7--testing--verifying-the-setup)
12. [Architecture Deep Dive](#12-architecture-deep-dive)
13. [Troubleshooting](#13-troubleshooting)
14. [Key Concepts Glossary](#14-key-concepts-glossary)

---

## 1. Workshop Overview

### What You Will Learn

By the end of this workshop, you will be able to:

- Run a local OpenShift cluster using **CodeReady Containers (CRC)**
- Understand what an OpenShift **Route** is and how it differs from a Kubernetes Ingress
- Create Routes to expose **two different applications** to external traffic
- Use **hostname-based** and **path-based** routing with Routes
- Secure a Route with **Edge TLS termination**
- Understand how the **HAProxy Router** works under the hood

### Why Routes Instead of Ingress?

In the previous workshop we used Kubernetes Ingress with MetalLB. This workshop uses **OpenShift Routes** — the native OpenShift way to expose applications. Routes:

- Require **no MetalLB** — the Router already has an external IP in CRC out of the box
- Are **simpler to set up** — one resource instead of Ingress + IngressClass + MetalLB
- Support **TLS termination**, passthrough, and re-encryption natively
- Are **OpenShift-specific** — not available in plain Kubernetes

### Estimated Time

| Lab | Topic | Time |
|-----|-------|------|
| Lab 1 | Setting Up OpenShift CRC | 20 min |
| Lab 2 | Deploying Application A | 10 min |
| Lab 3 | Deploying Application B | 10 min |
| Lab 4 | Route for Application A | 10 min |
| Lab 5 | Route for Application B | 10 min |
| Lab 6 | Advanced Routing & TLS | 15 min |
| Lab 7 | Testing & Verifying | 10 min |
| **Total** | | **~85 min** |

### Who Is This For?

This workshop is designed for **beginners** who:
- Have completed the MetalLB + Ingress workshop, or understand basic OpenShift concepts
- Want to learn the OpenShift-native way to expose applications
- Are interested in TLS and secure routing

---

## 2. Architecture Explained

### The Big Picture

```
Your Browser
      |
      |  HTTP/HTTPS Request (e.g., http://app-a.apps-crc.testing)
      v
+-------------------------------------------------------------+
|               HAProxy Router (OpenShift Router)             |
|    Already has IP from CRC — no MetalLB needed              |
|    IP: 192.168.130.11 (CRC node IP, port 80/443)            |
|                                                             |
|   Route rule: app-a.apps-crc.testing --> app-a-svc         |
|   Route rule: app-b.apps-crc.testing --> app-b-svc         |
+----------------------------+--------------------------------+
                             |
              +--------------+--------------+
              |                             |
              v                             v
  +---------------------+       +---------------------+
  |  Service: app-a-svc |       |  Service: app-b-svc |
  |  (ClusterIP)        |       |  (ClusterIP)        |
  +----------+----------+       +----------+----------+
             |                             |
       +-----+-----+                 +-----+-----+
       v           v                 v           v
   Pod: app-a   Pod: app-a       Pod: app-b   Pod: app-b
   (Nginx Web)  (Nginx Web)      (API/JSON)   (API/JSON)

   APPLICATION A                  APPLICATION B
   Simple HTML webpage            REST API returning JSON
```

### How Routes Work Differently from Ingress

With **Kubernetes Ingress + MetalLB** (previous workshop):

```
Browser --> MetalLB IP --> Ingress Controller --> Service --> Pod
           (you set up)    (reads Ingress rules)
```

With **OpenShift Routes** (this workshop):

```
Browser --> HAProxy Router --> Service --> Pod
           (built into CRC,   (reads Route rules)
            no extra setup)
```

The Router is already running in CRC. Every Route you create is automatically picked up by the Router. No LoadBalancer service, no MetalLB, no IPAddressPool needed.

---

## 3. Routes vs Ingress -- Key Differences

| Feature | OpenShift Route | Kubernetes Ingress |
|---------|----------------|-------------------|
| Origin | OpenShift-specific | Kubernetes standard |
| Requires controller setup | No (built into OpenShift) | Yes (need IngressClass) |
| Requires MetalLB on bare metal | No | Yes |
| TLS Edge termination | Built in, one field | Needs annotation or cert-manager |
| TLS Passthrough | Built in | Limited support |
| TLS Re-encryption | Built in | Not standard |
| Wildcard routes | Supported | Limited |
| Path-based routing | Supported | Supported |
| Hostname-based routing | Supported | Supported |
| Works in plain Kubernetes | No | Yes |
| Auto-generates hostname | Yes (`.apps-crc.testing`) | No |

### What is a Route, Exactly?

A Route is an OpenShift resource (kind: `Route`) that tells the HAProxy Router:

- Which **hostname** to listen for
- Which **Service** to forward traffic to
- Which **port** on that Service to use
- Whether to use **TLS** and what kind

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: my-route
spec:
  host: myapp.apps-crc.testing     # hostname to listen for
  to:
    kind: Service
    name: my-service               # service to forward to
  port:
    targetPort: 8080               # port on the service/pod
  tls:
    termination: edge              # TLS type (optional)
```

### Three Types of TLS Termination

```
Edge termination:
  Browser --HTTPS--> Router (decrypts here) --HTTP--> Pod
  Router holds the TLS certificate. Pod sees plain HTTP.

Passthrough termination:
  Browser --HTTPS--> Router (passes through) --HTTPS--> Pod
  Pod holds the TLS certificate. Router never sees plain text.

Re-encryption termination:
  Browser --HTTPS--> Router (decrypts, re-encrypts) --HTTPS--> Pod
  Router holds one cert, Pod holds another. Double TLS.
```

---

## 4. Prerequisites

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
# On Linux
tar -xvf crc-linux-amd64.tar.xz
sudo mv crc-linux-*-amd64/crc /usr/local/bin/
crc version
```

#### 2. Install OpenShift CLI (oc)

```bash
# Bundled with CRC
eval $(crc oc-env)
oc version
```

### Pull Secret

You will need a pull secret from Red Hat (free account):
1. Go to https://console.redhat.com/openshift/create/local
2. Click "Download pull secret"
3. Save as `~/pull-secret.txt`

---

## 5. Lab 1 -- Setting Up OpenShift CRC

> Note: If you already have CRC running from the previous workshop, skip to Step 1.4 and just log in.

### Step 1.1 -- Set Up CRC

```bash
crc setup
```

### Step 1.2 -- Configure Resources

```bash
crc config set cpus 6
crc config set memory 14336
```

### Step 1.3 -- Start the Cluster

```bash
crc start --pull-secret-file ~/pull-secret.txt
```

> First start takes 15-25 minutes.

### Step 1.4 -- Log In

```bash
eval $(crc oc-env)

oc login -u kubeadmin -p <your-kubeadmin-password> https://api.crc.testing:6443
```

### Step 1.5 -- Verify the Router is Already Running

This is the key difference from the MetalLB workshop. The router is already there:

```bash
oc get pods -n openshift-ingress
```

Expected output:

```
NAME                              READY   STATUS    RESTARTS   AGE
router-default-xxxxxxxxxx-xxxx    1/1     Running   0          10m
```

```bash
# Check the router service -- note it already has an EXTERNAL-IP
oc get svc router-default -n openshift-ingress
```

Expected output:

```
NAME             TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)
router-default   LoadBalancer   10.217.x.x    192.168.130.11   80:xxxxx/TCP,443:xxxxx/TCP
```

> In CRC, the Router service already has an external IP assigned automatically. This is why you do not need MetalLB for Routes. CRC pre-configures this for you.

### Step 1.6 -- Check the Default Domain

```bash
oc get ingresscontroller default -n openshift-ingress-operator \
  -o jsonpath='{.status.domain}'
```

Expected output:

```
apps-crc.testing
```

This is the wildcard domain. Any Route you create will automatically get a hostname ending in `.apps-crc.testing`. This DNS wildcard is pre-configured in CRC and resolves to the router IP.

### Step 1.7 -- Create a Workshop Namespace

```bash
oc new-project workshop
oc project
```

---

## 6. Lab 2 -- Deploying Application A (Simple Web App)

Application A is a simple **Nginx-based web page**.

### Step 2.1 -- Create the Deployment

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
            .box { background: white; border-radius: 10px; padding: 30px;
                   display: inline-block; box-shadow: 0 2px 10px rgba(0,0,0,0.1); }
          </style></head>
          <body>
            <div class="box">
              <h1>Application A</h1>
              <p>This is a <strong>Simple Web Application</strong></p>
              <p>Served by: Nginx</p>
              <p>Type: Static HTML Website</p>
              <p>Routed via: OpenShift Route</p>
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

### Step 2.2 -- Create the Service

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
    name: http
  type: ClusterIP
EOF
```

### Step 2.3 -- Verify Application A

```bash
oc get pods -n workshop -l app=app-a
oc get svc app-a-svc -n workshop
```

Expected:

```
NAME                   READY   STATUS    RESTARTS   AGE
app-a-xxxxxxxxx-xxxx   1/1     Running   0          30s
app-a-xxxxxxxxx-yyyy   1/1     Running   0          30s

NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
app-a-svc    ClusterIP   10.217.x.x    <none>        80/TCP    30s
```

---

## 7. Lab 3 -- Deploying Application B (API Service)

Application B is a **Python REST API** returning JSON.

### Step 3.1 -- Create the Deployment

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
                      "routing": "OpenShift Route",
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

### Step 3.2 -- Create the Service

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
    name: http
  type: ClusterIP
EOF
```

### Step 3.3 -- Verify Application B

```bash
oc get pods -n workshop -l app=app-b
oc get svc app-b-svc -n workshop
```

---

## 8. Lab 4 -- Creating a Route for Application A

This is where Routes shine. One command and your app is accessible externally.

### Step 4.1 -- Create the Route (Quick Way)

OpenShift lets you expose a service with a single command:

```bash
oc expose svc app-a-svc \
  --name=route-app-a \
  --hostname=app-a.apps-crc.testing \
  -n workshop
```

That is all that is needed. No MetalLB, no IngressClass, no LoadBalancer service.

### Step 4.2 -- Create the Route (YAML Way)

The same thing as a YAML file, so you can see exactly what was created:

```bash
cat <<EOF | oc apply -f -
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: route-app-a
  namespace: workshop
spec:
  host: app-a.apps-crc.testing
  to:
    kind: Service
    name: app-a-svc
    weight: 100
  port:
    targetPort: http
  wildcardPolicy: None
EOF
```

> Use one of the two methods above, not both.

### Step 4.3 -- Verify the Route

```bash
oc get route route-app-a -n workshop
```

Expected output:

```
NAME          HOST/PORT                  PATH   SERVICES    PORT   TERMINATION   WILDCARD
route-app-a   app-a.apps-crc.testing           app-a-svc   http                 None
```

```bash
# Describe gives more detail
oc describe route route-app-a -n workshop
```

Key fields to notice:
- `Host`: the hostname the Router listens for
- `Service`: where the Router sends traffic
- `TLS Termination`: empty means plain HTTP (we add TLS in Lab 6)

---

## 9. Lab 5 -- Creating a Route for Application B

### Step 5.1 -- Create the Route

```bash
cat <<EOF | oc apply -f -
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: route-app-b
  namespace: workshop
spec:
  host: app-b.apps-crc.testing
  to:
    kind: Service
    name: app-b-svc
    weight: 100
  port:
    targetPort: http
  wildcardPolicy: None
EOF
```

### Step 5.2 -- Verify Both Routes

```bash
oc get routes -n workshop
```

Expected output:

```
NAME          HOST/PORT                  PATH   SERVICES    PORT   TERMINATION   WILDCARD
route-app-a   app-a.apps-crc.testing           app-a-svc   http                 None
route-app-b   app-b.apps-crc.testing           app-b-svc   http                 None
```

Notice each app gets its own hostname. The Router now has two rules:

```
app-a.apps-crc.testing  -->  app-a-svc  -->  Nginx pods
app-b.apps-crc.testing  -->  app-b-svc  -->  Python API pods
```

---

## 10. Lab 6 -- Advanced Routing (Path-Based & TLS)

### Part A: Path-Based Routing with a Single Hostname

Instead of two subdomains, route both apps under one hostname using paths:

```bash
cat <<EOF | oc apply -f -
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: route-app-a-path
  namespace: workshop
spec:
  host: workshop.apps-crc.testing
  path: /app-a
  to:
    kind: Service
    name: app-a-svc
    weight: 100
  port:
    targetPort: http
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: route-app-b-path
  namespace: workshop
spec:
  host: workshop.apps-crc.testing
  path: /app-b
  to:
    kind: Service
    name: app-b-svc
    weight: 100
  port:
    targetPort: http
EOF
```

Now traffic is split by path:

```
workshop.apps-crc.testing/app-a  -->  app-a-svc
workshop.apps-crc.testing/app-b  -->  app-b-svc
```

### Part B: Securing a Route with Edge TLS

Edge TLS means the Router terminates HTTPS and forwards plain HTTP to the pod. This is the most common setup.

#### Generate a Self-Signed Certificate (for testing)

```bash
# Generate a private key
openssl genrsa -out tls.key 2048

# Generate a self-signed certificate
openssl req -new -x509 \
  -key tls.key \
  -out tls.crt \
  -days 365 \
  -subj "/CN=app-a.apps-crc.testing/O=Workshop"
```

#### Create the TLS Route

```bash
oc create route edge route-app-a-tls \
  --service=app-a-svc \
  --hostname=app-a-secure.apps-crc.testing \
  --key=tls.key \
  --cert=tls.crt \
  --insecure-policy=Redirect \
  -n workshop
```

The `--insecure-policy=Redirect` means HTTP requests are automatically redirected to HTTPS.

#### Verify the TLS Route

```bash
oc get route route-app-a-tls -n workshop
```

Expected output:

```
NAME              HOST/PORT                         PATH   SERVICES    PORT   TERMINATION     WILDCARD
route-app-a-tls   app-a-secure.apps-crc.testing            app-a-svc   http   edge/Redirect   None
```

Notice `TERMINATION` now shows `edge/Redirect`.

#### TLS Route as YAML

The same route in YAML form, for reference:

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: route-app-a-tls
  namespace: workshop
spec:
  host: app-a-secure.apps-crc.testing
  to:
    kind: Service
    name: app-a-svc
    weight: 100
  port:
    targetPort: http
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
    key: |
      -----BEGIN RSA PRIVATE KEY-----
      <contents of tls.key>
      -----END RSA PRIVATE KEY-----
    certificate: |
      -----BEGIN CERTIFICATE-----
      <contents of tls.crt>
      -----END CERTIFICATE-----
```

### Part C: Passthrough TLS Route

For passthrough, the pod itself handles TLS. The Router passes the encrypted traffic straight through without decrypting it.

```bash
cat <<EOF | oc apply -f -
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: route-app-b-passthrough
  namespace: workshop
spec:
  host: app-b-secure.apps-crc.testing
  to:
    kind: Service
    name: app-b-svc
    weight: 100
  port:
    targetPort: 8443
  tls:
    termination: passthrough
EOF
```

> Note: For passthrough to work, Application B would need to serve HTTPS itself on port 8443. This is shown for reference.

### Part D: Traffic Splitting Between Two Services (A/B Testing)

Routes support splitting traffic between two services using weights:

```bash
cat <<EOF | oc apply -f -
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: route-ab-split
  namespace: workshop
spec:
  host: split.apps-crc.testing
  to:
    kind: Service
    name: app-a-svc
    weight: 80
  alternateBackends:
  - kind: Service
    name: app-b-svc
    weight: 20
  port:
    targetPort: http
EOF
```

This sends 80% of traffic to App A and 20% to App B. Useful for canary deployments and A/B testing.

---

## 11. Lab 7 -- Testing & Verifying the Setup

### Step 7.1 -- Verify DNS Works

CRC automatically configures DNS for `*.apps-crc.testing`:

```bash
# Should resolve to CRC router IP
ping app-a.apps-crc.testing
nslookup app-a.apps-crc.testing
```

Expected: resolves to `192.168.130.11` (the CRC node/router IP).

### Step 7.2 -- Test Application A

```bash
# Plain HTTP via hostname-based Route
curl http://app-a.apps-crc.testing

# Verbose to see headers
curl -v http://app-a.apps-crc.testing
```

Expected: HTML page with "Application A"

### Step 7.3 -- Test Application B

```bash
# Should return JSON
curl http://app-b.apps-crc.testing | python3 -m json.tool
```

Expected output:

```json
{
  "application": "Application B",
  "type": "REST API",
  "message": "Hello from the API service!",
  "path": "/",
  "hostname": "app-b-xxxxxxxxx-xxxx",
  "timestamp": "2024-01-15T10:30:00Z",
  "routing": "OpenShift Route",
  "status": "healthy"
}
```

### Step 7.4 -- Test Path-Based Routes

```bash
# App A via path
curl http://workshop.apps-crc.testing/app-a

# App B via path
curl http://workshop.apps-crc.testing/app-b
```

### Step 7.5 -- Test TLS Route

```bash
# HTTPS with self-signed cert (use -k to skip cert verification in testing)
curl -k https://app-a-secure.apps-crc.testing

# Test HTTP to HTTPS redirect
curl -v http://app-a-secure.apps-crc.testing
# Should see: 302 redirect to https://
```

### Step 7.6 -- Observe Load Balancing Across Pods

```bash
# Hit App B multiple times and watch different pod hostnames
for i in $(seq 1 6); do
  curl -s http://app-b.apps-crc.testing | python3 -c \
    "import sys,json; print(json.load(sys.stdin)['hostname'])"
done
```

Expected: alternating pod names, showing load balancing is working.

### Step 7.7 -- List All Routes

```bash
oc get routes -n workshop -o wide
```

Expected:

```
NAME                  HOST/PORT                            PATH     SERVICES    PORT   TERMINATION     WILDCARD
route-app-a           app-a.apps-crc.testing                        app-a-svc   http                   None
route-app-b           app-b.apps-crc.testing                        app-b-svc   http                   None
route-app-a-path      workshop.apps-crc.testing            /app-a   app-a-svc   http                   None
route-app-b-path      workshop.apps-crc.testing            /app-b   app-b-svc   http                   None
route-app-a-tls       app-a-secure.apps-crc.testing                 app-a-svc   http   edge/Redirect   None
route-ab-split        split.apps-crc.testing                        app-a-svc   http                   None
```

### Step 7.8 -- View in OpenShift Web Console

```bash
crc console --credentials
```

Open `https://console-openshift-console.apps-crc.testing` and navigate to:
- **Networking → Routes** — see all your routes, their hostnames, and TLS status
- **Networking → Services** — see your ClusterIP services
- **Workloads → Pods** — see running pods

From the Routes page you can also click directly on the hostname to open the app in your browser.

---

## 12. Architecture Deep Dive

### Full Traffic Flow with Routes

```
Step 1: Browser requests http://app-a.apps-crc.testing
        |
        v
Step 2: DNS resolves app-a.apps-crc.testing --> 192.168.130.11
        (CRC pre-configures *.apps-crc.testing wildcard DNS)
        |
        v
Step 3: TCP connection reaches port 80 on 192.168.130.11
        This is the CRC node IP where the HAProxy Router pod runs
        |
        v
Step 4: HAProxy Router pod receives the HTTP request
        Reads Host header: "app-a.apps-crc.testing"
        Looks up its internal route table
        Finds: route-app-a --> app-a-svc
        |
        v
Step 5: Router forwards to Service app-a-svc (ClusterIP)
        iptables/IPVS picks one of the ready app-a pods
        |
        v
Step 6: app-a pod (Nginx) serves the HTML page
        Response travels back the same chain
        |
        v
Step 7: Browser renders the page
```

### How the Router Knows About Your Routes

The HAProxy Router pod **watches** the OpenShift API server for Route resources. Every time you create, update, or delete a Route, the Router automatically updates its HAProxy configuration without restarting.

```
You run: oc apply -f route.yaml
          |
          v
OpenShift API stores the Route resource
          |
          v
Router pod detects the change (watch event)
          |
          v
Router regenerates HAProxy config
          |
          v
New route is live (usually within 1-2 seconds)
```

### Route vs Service vs Pod

```
Route       = External-facing rule (hostname + path --> service)
Service     = Internal load balancer (stable IP/DNS --> pods)
Pod         = Where your actual app code runs
```

```
External traffic --> Route --> Service --> Pod
Internal traffic             --> Service --> Pod
```

Internal services (pod-to-pod) never need a Route. Only traffic coming from outside the cluster needs one.

### Comparing the Two Workshops

```
MetalLB + Ingress workshop:

Browser --> MetalLB IP --> Ingress Controller --> Service --> Pod
            (you set up    (reads Ingress YAML,
             IPAddressPool) managed separately)

OpenShift Routes workshop:

Browser --> Router IP --> Route --> Service --> Pod
            (CRC built-in, (reads Route YAML,
             auto-configured) native OpenShift)
```

The end result for the user is the same — they type a URL and get a response. The difference is how much you have to set up and what features are available.

---

## 13. Troubleshooting

### Route shows no ADDRESS

```bash
oc get route route-app-a -n workshop
# ADDRESS column is empty
```

Fix:

```bash
# Check the router is running
oc get pods -n openshift-ingress

# Check router logs
oc logs deployment/router-default -n openshift-ingress | tail -30

# Check the route has no errors
oc describe route route-app-a -n workshop
```

### 503 Service Unavailable

```bash
# Check pods are running and ready
oc get pods -n workshop

# Check endpoints -- if empty, service selector does not match pod labels
oc get endpoints app-a-svc -n workshop

# Verify labels
oc get pods --show-labels -n workshop
oc describe svc app-a-svc -n workshop | grep Selector
```

### DNS does not resolve

```bash
# CRC DNS should handle *.apps-crc.testing automatically
# Check CRC network DNS
crc status

# If DNS fails, add to /etc/hosts manually
echo "192.168.130.11 app-a.apps-crc.testing" | sudo tee -a /etc/hosts
echo "192.168.130.11 app-b.apps-crc.testing" | sudo tee -a /etc/hosts
echo "192.168.130.11 workshop.apps-crc.testing" | sudo tee -a /etc/hosts

# Or test bypassing DNS entirely
curl -v --resolve app-a.apps-crc.testing:80:192.168.130.11 \
     http://app-a.apps-crc.testing
```

### TLS certificate errors

```bash
# For testing, always use -k with self-signed certs
curl -k https://app-a-secure.apps-crc.testing

# Check the route has cert and key
oc get route route-app-a-tls -n workshop -o yaml | grep -A5 tls

# Re-create the route with correct cert paths
oc delete route route-app-a-tls -n workshop
oc create route edge route-app-a-tls \
  --service=app-a-svc \
  --hostname=app-a-secure.apps-crc.testing \
  --key=tls.key \
  --cert=tls.crt \
  -n workshop
```

### Route exists but app returns wrong content

```bash
# Confirm the route points to the right service
oc describe route route-app-a -n workshop | grep "Service:"

# Test the service directly from inside the cluster
oc run test --image=curlimages/curl --rm -it --restart=Never \
  -- curl http://app-a-svc.workshop.svc.cluster.local
```

### Redirect loop on TLS route

This happens when `insecureEdgeTerminationPolicy` is set to `Redirect` but the app also redirects. Fix:

```bash
oc patch route route-app-a-tls -n workshop \
  -p '{"spec":{"tls":{"insecureEdgeTerminationPolicy":"Allow"}}}'
```

---

## 14. Key Concepts Glossary

| Term | Definition |
|------|------------|
| **Route** | OpenShift resource that exposes a Service externally via a hostname and optional path |
| **HAProxy Router** | The OpenShift router pod that reads Route resources and forwards traffic |
| **Edge TLS** | TLS terminated at the Router; pod receives plain HTTP |
| **Passthrough TLS** | Router forwards encrypted traffic as-is; pod handles TLS itself |
| **Re-encryption** | Router decrypts then re-encrypts; both Router and pod use TLS |
| **Wildcard DNS** | `*.apps-crc.testing` resolves to the router IP; every Route gets a hostname automatically |
| **Host header** | HTTP header containing the domain name; used by the Router to match Routes |
| **Weight** | Percentage of traffic sent to a backend when using traffic splitting |
| **insecureEdgeTerminationPolicy** | What to do with HTTP when Route is HTTPS: Allow, Redirect, or None |
| **ClusterIP** | Service type for internal-only access; used by app-a-svc and app-b-svc |
| **oc expose** | CLI shortcut to create a Route from an existing Service |
| **alternateBackends** | Secondary services in a Route used for traffic splitting / A/B testing |
| **wildcardPolicy** | Whether the Route matches a single host or a wildcard (*.domain) |
| **CRC** | CodeReady Containers — runs a single-node OpenShift cluster on your laptop |
| **Namespace / Project** | Logical grouping of resources; OpenShift calls these Projects |

---

## Workshop Complete

Congratulations. You have successfully:

- Set up a local OpenShift cluster with CRC
- Deployed two different types of applications (web app + REST API)
- Created Routes to expose both applications externally using the OpenShift-native approach
- Configured path-based routing under a single hostname
- Secured a Route with Edge TLS termination
- Implemented traffic splitting between two applications
- Tested end-to-end traffic flow through the HAProxy Router

### Comparison: When to Use What

| Scenario | Use |
|----------|-----|
| Running on OpenShift | Routes (simpler, more features) |
| Running on plain Kubernetes | Ingress |
| Need TLS easily | Routes (built-in) |
| Need to run on bare metal Kubernetes | Ingress + MetalLB |
| Need A/B traffic splitting | Routes (weight-based) |
| Need to work across multiple clouds | Ingress (portable) |

### Next Steps

- **Explore Route annotations**: HAProxy annotations let you tune timeouts, rate limiting, and more
- **Try Re-encryption TLS**: Set up mutual TLS between Router and pod
- **Use cert-manager**: Automate TLS certificate issuance for Routes
- **Explore OpenShift Service Mesh**: Istio-based advanced traffic management
- **Try `oc expose` shortcuts**: Explore `oc expose deployment` directly

### Resources

- [OpenShift Routes Documentation](https://docs.openshift.com/container-platform/latest/networking/routes/route-configuration.html)
- [HAProxy Router Documentation](https://docs.openshift.com/container-platform/latest/networking/ingress-operator.html)
- [OpenShift CRC Documentation](https://crc.dev/crc/)
- [Route TLS Configuration](https://docs.openshift.com/container-platform/latest/networking/routes/secured-routes.html)

---

*Workshop created for OpenShift CRC beginners. All YAML manifests are tested on CRC v2.x with OpenShift 4.14+.*
