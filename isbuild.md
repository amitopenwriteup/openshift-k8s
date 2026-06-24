# OpenShift CRC Lab: ImageStreams & BuildConfigs

> **Prerequisites:** OpenShift Local (CRC) installed and running, `oc` CLI available, CRC started with `crc start`.

---

## 1. Login & Setup

```bash
# Login with developer credentials
oc login -u developer -p developer https://api.crc.testing:6443

# Create a dedicated namespace for this lab
oc new-project imagestream-lab
```

---

## 2. What Is an ImageStream?

An **ImageStream** is an OpenShift abstraction that tracks container images. It does not store image data itself — it holds *references* (tags) to images stored in a registry.

| Concept | Description |
|---|---|
| `ImageStream` | Tracks one or more image tags |
| `ImageStreamTag` | A named pointer to a specific image |
| `ImageStreamImage` | Reference to an image by digest |

---

## 3. Create an ImageStream

### 3.1 Manually Define an ImageStream

```yaml
# imagestream.yaml
apiVersion: image.openshift.io/v1
kind: ImageStream
metadata:
  name: my-app
  namespace: imagestream-lab
spec:
  lookupPolicy:
    local: false   # set to true to allow pods to resolve by IS name
```

```bash
oc apply -f imagestream.yaml

# Verify
oc get imagestream my-app
oc describe imagestream my-app
```

### 3.2 Import an External Image into the ImageStream

```bash
# Import an image tag from Docker Hub
oc import-image my-app:latest \
  --from=docker.io/library/nginx:latest \
  --confirm \
  --scheduled          # auto re-import periodically

# Check imported tags
oc get imagestreamtag my-app:latest
```

---

## 4. What Is a BuildConfig?

A **BuildConfig** defines how OpenShift builds a container image. It specifies:
- **Source** — Git repo, binary, or Dockerfile
- **Strategy** — Source-to-Image (S2I), Docker, or Custom
- **Output** — where to push the built image (usually an ImageStream)
- **Triggers** — what kicks off a new build

---

## 5. BuildConfig — Source-to-Image (S2I) Strategy

### 5.1 Create a BuildConfig (S2I)

```yaml
# buildconfig-s2i.yaml
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  name: my-app-s2i
  namespace: imagestream-lab
spec:
  source:
    type: Git
    git:
      uri: "https://github.com/sclorg/nodejs-ex.git"
      ref: "master"
  strategy:
    type: Source
    sourceStrategy:
      from:
        kind: ImageStreamTag
        namespace: openshift       # built-in OpenShift namespace
        name: "nodejs:18-ubi8"    # base builder image
  output:
    to:
      kind: ImageStreamTag
      name: "my-app:latest"       # push result to our ImageStream
  triggers:
    - type: ConfigChange           # rebuild on BuildConfig change
    - type: ImageChange            # rebuild when builder image updates
      imageChange: {}
```

```bash
oc apply -f buildconfig-s2i.yaml

# Verify
oc get buildconfig my-app-s2i
oc describe buildconfig my-app-s2i
```

### 5.2 Start a Build Manually

```bash
oc start-build my-app-s2i

# Stream build logs in real time
oc logs -f bc/my-app-s2i

# List all builds
oc get builds
```

---

## 6. BuildConfig — Docker Strategy

```yaml
# buildconfig-docker.yaml
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  name: my-app-docker
  namespace: imagestream-lab
spec:
  source:
    type: Git
    git:
      uri: "https://github.com/sclorg/httpd-ex.git"
      ref: "master"
    contextDir: "."
  strategy:
    type: Docker
    dockerStrategy:
      dockerfilePath: Dockerfile   # relative to contextDir
  output:
    to:
      kind: ImageStreamTag
      name: "my-app:docker-latest"
  triggers:
    - type: ConfigChange
```

```bash
oc apply -f buildconfig-docker.yaml
oc start-build my-app-docker --follow
```

---

## 7. BuildConfig — Binary Strategy (Upload Local Code)

```bash
# Create a simple app directory
mkdir myapp && echo '<h1>Hello CRC!</h1>' > myapp/index.html

# Create BuildConfig for binary input
oc new-build \
  --name=my-app-binary \
  --strategy=docker \
  --binary=true \
  -l app=my-app-binary

# Start build by uploading local directory
oc start-build my-app-binary \
  --from-dir=./myapp \
  --follow
```

---

## 8. Explore Build Objects

```bash
# Get all BuildConfigs in the project
oc get bc

# Get all Builds (instances of a BuildConfig run)
oc get builds

# Describe a specific build
oc describe build my-app-s2i-1

# Get build logs for a specific build number
oc logs build/my-app-s2i-1

# Cancel a running build
oc cancel-build my-app-s2i-1
```

---

## 9. ImageStream Triggers & Automatic Deployments

When an ImageStream tag is updated (after a build), any **Deployment** or **DeploymentConfig** watching that tag can automatically redeploy.

```bash
# Deploy the app from the ImageStream
oc new-app --image-stream=imagestream-lab/my-app:latest \
  --name=my-web-app

# Expose a route
oc expose svc/my-web-app

# Get the app URL
oc get route my-web-app
```

Now re-run a build — the deployment will automatically roll out:

```bash
oc start-build my-app-s2i --follow
# Watch the rollout
oc rollout status deployment/my-web-app
```

---

## 10. Tagging ImageStreams

```bash
# Tag an existing image as 'stable'
oc tag my-app:latest my-app:stable

# Tag an external image into your ImageStream
oc tag docker.io/library/httpd:2.4 my-app:httpd-2.4

# List all tags on an ImageStream
oc get imagestreamtag | grep my-app

# Delete a tag
oc tag my-app:stable -d
```

---

## 11. Webhook Triggers (CI/CD Integration)

```bash
# Get the GitHub webhook URL for your BuildConfig
oc describe bc/my-app-s2i | grep -A2 "Webhook"

# Or extract it with a template
oc get bc my-app-s2i \
  -o jsonpath='{.spec.triggers[?(@.type=="GitHub")].github.secret}'
```

Add this URL to your GitHub repo under **Settings → Webhooks** to trigger builds on every push.

---

## 12. Cleanup

```bash
# Delete specific resources
oc delete bc my-app-s2i my-app-docker my-app-binary
oc delete imagestream my-app
oc delete all -l app=my-web-app

# Or delete the entire project
oc delete project imagestream-lab
```

---

## 13. Quick Reference Cheat Sheet

| Task | Command |
|---|---|
| List ImageStreams | `oc get imagestream` |
| Describe ImageStream | `oc describe is <name>` |
| Import image | `oc import-image <is>:<tag> --from=<registry/image> --confirm` |
| List BuildConfigs | `oc get bc` |
| Start build | `oc start-build <bc-name>` |
| Follow build logs | `oc logs -f bc/<bc-name>` |
| List builds | `oc get builds` |
| Tag image | `oc tag <src-is>:<tag> <dest-is>:<tag>` |
| Get webhooks | `oc describe bc/<name>` |

---

## 14. Architecture Summary

```
Git Repo / Local Code
        │
        ▼
  [ BuildConfig ]
   Strategy: S2I / Docker / Binary
        │
        │  runs Build(s)
        ▼
  [ Build Pod ]  ← pulls builder image from ImageStream
        │
        │  pushes output image
        ▼
  [ ImageStream ]  ←── tracks tags (latest, stable, v1.0…)
        │
        │  triggers via ImageChange
        ▼
  [ Deployment / DeploymentConfig ]
        │
        ▼
  [ Running Pods ]  →  Route → Browser
```

---

*Lab tested on OpenShift Local (CRC) v2.x with OpenShift 4.x cluster.*
