# OpenShift CRC — Source-to-Image (S2I) Lab Workshop

> **Audience:** Developers & Platform Engineers new to OpenShift S2I  
> **Duration:** ~2 hours  
> **Environment:** OpenShift Local (CRC) 2.x | OpenShift 4.x  
> **Prerequisites:** CRC installed, `crc start` completed, `oc` CLI on PATH

---

## Workshop Overview

| Module | Topic | Time |
|--------|-------|------|
| 1 | Environment Setup & Login | 10 min |
| 2 | Understanding S2I | 15 min |
| 3 | Lab 1 — S2I from Git (Node.js) | 25 min |
| 4 | Lab 2 — S2I from Git (Python) | 20 min |
| 5 | Lab 3 — S2I with Environment Variables & Secrets | 20 min |
| 6 | Lab 4 — Custom S2I Builder Image | 20 min |
| 7 | Lab 5 — Webhooks & Automated Builds | 10 min |
| 8 | Cleanup | 5 min |

---

## Module 1 — Environment Setup

### 1.1 Start CRC & Verify

```bash
# Start CRC (if not already running)
crc start

# Check status
crc status
```

Expected output:
```
CRC VM:          Running
OpenShift:       Running (v4.x.x)
RAM Usage:       X.XGB of XGB
Disk Usage:      XXGiB of XXGiB
```

### 1.2 Configure Shell & Login

```bash
# Set up the oc CLI environment
eval $(crc oc-env)

# Login as developer
oc login -u developer -p developer https://api.crc.testing:6443

# Verify login
oc whoami          # → developer
oc version
```

### 1.3 Create a Workshop Project

```bash
oc new-project s2i-workshop --display-name="S2I Workshop Lab"

# Confirm active project
oc project
```

---

## Module 2 — Understanding S2I

### How S2I Works

Source-to-Image (**S2I**) is an OpenShift build strategy that takes your **application source code** and a **builder image**, and produces a ready-to-run container — without you writing a Dockerfile.

```
Your Source Code (Git / Binary)
          +
  Builder Image (Node.js, Python, Ruby…)
          │
          ▼
   ┌─────────────────────┐
   │   S2I Build Process  │
   │  1. Clone source     │
   │  2. Run assemble     │   ← script inside builder image
   │  3. Install deps     │
   │  4. Compile/build    │
   │  5. Commit layer     │
   └─────────────────────┘
          │
          ▼
   Output Image (app + runtime)
          │
          ▼
   Deployed as Pod → Service → Route
```

### S2I Scripts

| Script | When It Runs | Purpose |
|--------|-------------|---------|
| `assemble` | During build | Install dependencies, compile code |
| `run` | Container start | Launch the application |
| `save-artifacts` | Incremental build | Cache dependencies between builds |
| `usage` | `s2i usage` call | Print usage documentation |

### Available Builder Images in CRC

```bash
# List all S2I builder images available in the openshift namespace
oc get imagestream -n openshift | grep -v NAME

# Common builders
oc get is -n openshift | grep -E "nodejs|python|php|ruby|java|dotnet"
```

---

## Module 3 — Lab 1: S2I from Git (Node.js App)

### Objective
Deploy a Node.js application directly from a public Git repository using S2I.

---

### Step 1 — Explore Available Node.js Builder Tags

```bash
oc describe is nodejs -n openshift | grep -A5 "Tags:"
```

### Step 2 — Create the Application Using `oc new-app`

```bash
oc new-app \
  nodejs:18-ubi8~https://github.com/sclorg/nodejs-ex.git \
  --name=nodejs-app \
  --labels app=nodejs-app

# nodejs:18-ubi8       → builder image (ImageStreamTag)
# ~                    → separator between builder and source
# https://github.com/… → Git source URL
```

### Step 3 — Follow the Build Logs

```bash
# Watch the build in real time
oc logs -f bc/nodejs-app
```

You will see S2I phases:
```
Cloning source…
---> Installing application source…
---> Building your Node application from source…
npm install…
---> Build completed successfully
```

### Step 4 — Monitor Rollout

```bash
# Watch pods come up
oc get pods -w

# Check deployment status
oc rollout status deployment/nodejs-app
```

### Step 5 — Expose the Service

```bash
oc expose svc/nodejs-app

# Get the route URL
oc get route nodejs-app -o jsonpath='{.spec.host}'
```

Open the URL in your browser — you should see the Node.js welcome page.

### Step 6 — Inspect Build Objects

```bash
# BuildConfig created by new-app
oc describe bc/nodejs-app

# ImageStream created for output
oc describe is/nodejs-app

# See the generated build
oc get builds
oc describe build/nodejs-app-1
```

### ✅ Lab 1 Checkpoint

- [ ] Build completed successfully
- [ ] Pod is in `Running` state
- [ ] Route is accessible in browser
- [ ] Can describe the BuildConfig

---

## Module 4 — Lab 2: S2I from Git (Python App)

### Objective
Deploy a Python Flask application and observe S2I assembling a different runtime.

---

### Step 1 — Check Python Builder Tags

```bash
oc get imagestreamtag -n openshift | grep python
```

### Step 2 — Deploy Python App

```bash
oc new-app \
  python:3.9-ubi8~https://github.com/sclorg/django-ex.git \
  --name=python-app \
  --env DJANGO_SECRET_KEY=mysecretkey123 \
  --labels app=python-app
```

### Step 3 — Follow Build

```bash
oc logs -f bc/python-app
```

Expected S2I steps for Python:
```
---> Installing dependencies with pip…
Collecting Django…
---> Collecting application…
---> Build completed.
```

### Step 4 — Expose & Verify

```bash
oc expose svc/python-app
oc get route python-app -o jsonpath='{.spec.host}'
```

### Step 5 — Compare BuildConfigs

```bash
# Compare the two BuildConfigs side by side
oc get bc nodejs-app python-app -o yaml
```

Notice:
- Different `sourceStrategy.from` (nodejs vs python builder)
- Same `output.to` pattern (ImageStreamTag)
- Both created `ConfigChange` and `ImageChange` triggers automatically

### ✅ Lab 2 Checkpoint

- [ ] Python app builds successfully
- [ ] Django welcome page visible in browser
- [ ] Can identify the builder image used in the BC

---

## Module 5 — Lab 3: Environment Variables & Secrets in S2I Builds

### Objective
Inject configuration and sensitive data into S2I builds and running pods.

---

### 5.1 Build-time Environment Variables

Some S2I builders read env vars **during the build** (e.g., `NPM_MIRROR`, `PIP_INDEX_URL`).

```bash
# Patch the BuildConfig to add a build-time env var
oc set env bc/nodejs-app \
  NPM_CONFIG_LOGLEVEL=info \
  NODE_ENV=production

# Trigger a new build to apply
oc start-build nodejs-app --follow
```

### 5.2 Runtime Environment Variables

Runtime vars are set on the **Deployment**, not the BuildConfig.

```bash
# Set env vars on the deployment
oc set env deployment/nodejs-app \
  APP_PORT=8080 \
  LOG_LEVEL=debug

# Verify
oc set env deployment/nodejs-app --list
```

### 5.3 Using Secrets for Sensitive Values

```bash
# Create a Secret
oc create secret generic app-secrets \
  --from-literal=DB_PASSWORD=supersecretpass \
  --from-literal=API_KEY=abc123xyz

# Inject the Secret into the Deployment as env vars
oc set env deployment/nodejs-app \
  --from=secret/app-secrets

# Verify the vars are injected (values hidden)
oc set env deployment/nodejs-app --list
```

### 5.4 Using ConfigMaps

```bash
# Create a ConfigMap
oc create configmap app-config \
  --from-literal=APP_ENV=staging \
  --from-literal=APP_REGION=us-east

# Inject ConfigMap into Deployment
oc set env deployment/nodejs-app \
  --from=configmap/app-config

# Verify
oc describe pod -l app=nodejs-app | grep -A20 "Environment:"
```

### ✅ Lab 3 Checkpoint

- [ ] Build-time env var set and rebuild triggered
- [ ] Runtime env var visible in pod description
- [ ] Secret values injected without being exposed in plain text
- [ ] ConfigMap values available in running pod

---

## Module 6 — Lab 4: Custom S2I Builder Image

### Objective
Create a minimal custom S2I builder image and use it in a BuildConfig.

---

### 6.1 Project Structure

```bash
mkdir ~/custom-s2i-builder && cd ~/custom-s2i-builder
mkdir -p .s2i/bin
```

### 6.2 Create the S2I Scripts

**`.s2i/bin/assemble`**
```bash
cat > .s2i/bin/assemble << 'EOF'
#!/bin/bash
set -e
echo "---> Installing application from source..."
cp -Rf /tmp/src/. /opt/app-root/src/
echo "---> Installing dependencies..."
cd /opt/app-root/src && pip install -r requirements.txt --quiet
echo "---> Assemble complete."
EOF
chmod +x .s2i/bin/assemble
```

**`.s2i/bin/run`**
```bash
cat > .s2i/bin/run << 'EOF'
#!/bin/bash
set -e
echo "---> Starting the application..."
exec python /opt/app-root/src/app.py
EOF
chmod +x .s2i/bin/run
```

### 6.3 Create the Dockerfile for the Builder Image

```bash
cat > Dockerfile << 'EOF'
FROM python:3.11-slim

LABEL io.openshift.s2i.scripts-url="image:///usr/libexec/s2i"
LABEL io.k8s.description="Custom Python S2I Builder"
LABEL io.k8s.display-name="Custom Python S2I"
LABEL io.openshift.tags="builder,python,custom"

ENV APP_ROOT=/opt/app-root \
    PATH=/opt/app-root/bin:$PATH

RUN mkdir -p /opt/app-root/src /usr/libexec/s2i && \
    useradd -u 1001 -r -g 0 -d ${APP_ROOT} -s /sbin/nologin appuser && \
    chown -R 1001:0 /opt/app-root && \
    chmod -R g+rw /opt/app-root

COPY .s2i/bin/ /usr/libexec/s2i/

USER 1001
WORKDIR /opt/app-root/src

CMD ["/usr/libexec/s2i/usage"]
EOF
```

### 6.4 Build the Custom Builder on CRC

```bash
# Create a BuildConfig for the custom builder image itself
oc new-build \
  --name=custom-python-builder \
  --strategy=docker \
  --binary=true

# Upload and build
oc start-build custom-python-builder \
  --from-dir=. \
  --follow
```

### 6.5 Use the Custom Builder for an App

```bash
# Create a sample app
mkdir ~/my-python-app && cd ~/my-python-app

cat > app.py << 'EOF'
from http.server import HTTPServer, BaseHTTPRequestHandler

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.end_headers()
        self.wfile.write(b"Hello from Custom S2I Builder!")

HTTPServer(('0.0.0.0', 8080), Handler).serve_forever()
EOF

echo "flask" > requirements.txt

# Build app using our custom builder
oc new-build \
  --name=my-custom-app \
  --image-stream=s2i-workshop/custom-python-builder:latest \
  --binary=true

oc start-build my-custom-app --from-dir=. --follow
```

### ✅ Lab 4 Checkpoint

- [ ] Dockerfile for custom builder created
- [ ] Custom builder image built on CRC
- [ ] App built using the custom S2I builder
- [ ] `assemble` and `run` scripts execute correctly

---

## Module 7 — Lab 5: Webhooks & Automated Builds

### Objective
Configure GitHub webhooks to automatically trigger S2I builds on code push.

---

### Step 1 — Get Webhook URLs

```bash
# Get the GitHub webhook secret and URL for nodejs-app
oc describe bc/nodejs-app | grep -A3 "Webhook GitHub"

# Get just the secret
oc get bc nodejs-app \
  -o jsonpath='{.spec.triggers[?(@.type=="GitHub")].github.secret}'
echo ""
```

The webhook URL pattern is:
```
https://api.crc.testing:6443/apis/build.openshift.io/v1/namespaces/s2i-workshop/buildconfigs/nodejs-app/webhooks/<SECRET>/github
```

### Step 2 — Add a Generic Webhook Trigger (for testing without GitHub)

```bash
# Patch the BuildConfig to add a Generic webhook
oc patch bc/nodejs-app -p '{
  "spec": {
    "triggers": [
      {"type": "Generic", "generic": {"secret": "my-generic-secret-123"}}
    ]
  }
}'

# Trigger via curl (simulates a webhook call)
curl -k -X POST \
  https://api.crc.testing:6443/apis/build.openshift.io/v1/namespaces/s2i-workshop/buildconfigs/nodejs-app/webhooks/my-generic-secret-123/generic

# Watch the new build start
oc get builds -w
```

### Step 3 — Image Change Trigger

Every time the base `nodejs:18-ubi8` builder image is updated in the `openshift` namespace, a new build is automatically triggered.

```bash
# Confirm ImageChange trigger exists
oc describe bc/nodejs-app | grep -A5 "Image Change"

# Manually simulate by re-importing the builder image
oc import-image nodejs:18-ubi8 -n openshift --confirm
```

### Step 4 — ConfigChange Trigger

```bash
# Any change to the BuildConfig triggers a rebuild
# Let's change the Git reference to prove it
oc patch bc/nodejs-app \
  -p '{"spec":{"source":{"git":{"ref":"master"}}}}'

# This triggers a new build automatically
oc get builds -w
```

### ✅ Lab 5 Checkpoint

- [ ] GitHub webhook URL retrieved
- [ ] Generic webhook triggered a build via curl
- [ ] ImageChange trigger confirmed in BuildConfig
- [ ] ConfigChange trigger observed

---

## Quick Command Reference

### BuildConfig Commands

```bash
oc get bc                          # List all BuildConfigs
oc describe bc/<name>              # Full BC details
oc start-build <bc-name>           # Trigger a build
oc start-build <bc-name> --follow  # Trigger + stream logs
oc cancel-build <build-name>       # Cancel running build
oc delete bc/<name>                # Delete a BuildConfig
```

### Build Commands

```bash
oc get builds                      # List all builds
oc logs build/<name>               # Get logs for a build
oc logs -f bc/<bc-name>            # Follow latest build logs
oc describe build/<name>           # Detailed build info
```

### ImageStream Commands

```bash
oc get is                          # List ImageStreams
oc describe is/<name>              # IS details & tags
oc get istag                       # List ImageStreamTags
oc tag <src>:<tag> <dest>:<tag>    # Create/copy a tag
oc import-image <is>:<tag> --from=<registry/image> --confirm
```

### S2I Diagnostic Commands

```bash
# Check build pod logs directly
oc logs $(oc get pods | grep -i build | awk '{print $1}')

# Inspect the output image in the IS
oc get istag/<app>:latest -o yaml | grep -i image

# Get events for troubleshooting
oc get events --sort-by='.lastTimestamp'
```

---

## Troubleshooting Guide

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Build stuck at `Pending` | No available build node | `oc get nodes` — check CRC is running |
| `ImagePullBackOff` | ImageStream not found | `oc get is` — verify IS exists and has tags |
| `assemble` script fails | Missing deps or wrong builder | Check `oc logs -f bc/<name>` for pip/npm errors |
| Route not accessible | Service not exposed | `oc expose svc/<name>` |
| Webhook not triggering | Wrong secret or URL | Re-check with `oc describe bc/<name>` |
| Build pod `OOMKilled` | CRC resource limit | Increase CRC memory: `crc config set memory 12288` |

---

## Module 8 — Cleanup

```bash
# Remove all workshop resources
oc delete all --all -n s2i-workshop
oc delete secret app-secrets -n s2i-workshop
oc delete configmap app-config -n s2i-workshop

# Delete the project entirely
oc delete project s2i-workshop

# Stop CRC (optional)
crc stop
```

---

## Workshop Summary

In this workshop you:

1. **Set up** OpenShift Local (CRC) and logged in as a developer
2. **Learned** how S2I assembles source code + builder image into a runnable container
3. **Deployed** a Node.js app from Git using `oc new-app` with S2I
4. **Deployed** a Python/Django app and compared BuildConfig structures
5. **Injected** build-time vars, runtime env vars, Secrets, and ConfigMaps
6. **Built** a custom S2I builder image with `assemble` and `run` scripts
7. **Configured** Generic webhooks and observed ImageChange/ConfigChange triggers

---

*Lab tested on OpenShift Local (CRC) v2.x — OpenShift 4.13+*  
*For issues, run `crc status` and `oc get events` first.*
