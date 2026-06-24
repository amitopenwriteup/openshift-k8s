# Workshop: Running WordPress on OpenShift CRC with a Custom SCC

**Goal:** Deploy a fully functional WordPress + MySQL stack on CRC by creating a custom Security Context Constraint that grants the exact permissions these workloads need — no more, no less.

**Audience:** Developers and platform engineers who need to run third-party or legacy container images on OpenShift that expect fixed UIDs or filesystem write access.

**Time:** ~45 minutes

---

## 0. Background: Why WordPress Fails Out of the Box

The official WordPress image runs as UID `33` (`www-data`) and MySQL/MariaDB as UID `999`. Both write to the filesystem at runtime. OpenShift's default **`restricted-v2`** SCC blocks both:

| What the image expects | What `restricted-v2` enforces | Result |
|---|---|---|
| Run as UID `33` or `999` | Random UID from namespace range (e.g. `1000680000+`) | Pod rejected |
| Write to `/var/www/html` | Read-only root or random-UID ownership | Permission denied |
| `allowPrivilegeEscalation` | Denied | Container fails to start |

A custom SCC lets these workloads run while keeping everything else locked down.

---

## 1. Prerequisites

- CRC running on your machine (`crc start` completed)
- `oc` CLI available in `$PATH`
- `kubeadmin` credentials (SCCs are cluster-scoped)

```bash
eval $(crc oc-env)
oc login -u kubeadmin -p <kubeadmin-password> https://api.crc.testing:6443
oc whoami   # should print: kubeadmin
```

---

## 2. Create the Project and ServiceAccount

```bash
oc new-project wordpress-workshop
oc create serviceaccount wordpress-sa -n wordpress-workshop
```

Verify:

```bash
oc get sa wordpress-sa -n wordpress-workshop
```

---

## 3. Reproduce the Failure (Before SCC)

Deploy a bare WordPress pod to confirm the default SCC blocks it.

```yaml
# wordpress-test-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: wordpress-test-pod
  namespace: wordpress-workshop
spec:
  serviceAccountName: wordpress-sa
  containers:
    - name: wordpress
      image: docker.io/library/wordpress:latest
      securityContext:
        runAsUser: 33
```

```bash
oc apply -f wordpress-test-pod.yaml
oc describe pod wordpress-test-pod -n wordpress-workshop
```

**Expected error:**

```
Error creating: pods "wordpress-test-pod" is forbidden: unable to validate against
any security context constraint: [provider "restricted-v2":
.spec.containers[0].securityContext.runAsUser: Invalid value: 33:
must be in the ranges: [1000680000, 1000689999]]
```

Clean up before continuing:

```bash
oc delete pod wordpress-test-pod -n wordpress-workshop --ignore-not-found
```

---

## 4. Create the Custom SCC

This SCC allows UIDs `33–999` (covering `www-data` and `mysql`), permits filesystem writes, and keeps everything else as locked down as possible.

```yaml
# wordpress-scc.yaml
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: wordpress-scc
  annotations:
    kubernetes.io/description: >
      Allows WordPress and MySQL workloads to run with fixed UIDs (33, 999)
      and write to container filesystems. Grants no host-level access.

# --- Privilege escalation ---
allowPrivilegedContainer: false
allowPrivilegeEscalation: true          # Required: WordPress/MySQL drop privs internally

# --- Host access: all denied ---
allowHostDirVolumePlugin: false
allowHostIPC: false
allowHostNetwork: false
allowHostPID: false
allowHostPorts: false

# --- Linux capabilities ---
allowedCapabilities: []
defaultAddCapabilities: []
requiredDropCapabilities:
  - KILL
  - MKNOD
  - SETUID
  - SETGID

# --- Filesystem ---
readOnlyRootFilesystem: false           # WordPress writes to /var/www/html at runtime

# --- UID/GID ---
runAsUser:
  type: MustRunAsRange
  uidRangeMin: 33                       # www-data (WordPress)
  uidRangeMax: 999                      # mysql (MySQL/MariaDB)

fsGroup:
  type: MustRunAs
  ranges:
    - min: 33
      max: 999

supplementalGroups:
  type: MustRunAs
  ranges:
    - min: 33
      max: 999

# --- SELinux ---
seLinuxContext:
  type: MustRunAs

# --- Volumes: only what WordPress actually needs ---
volumes:
  - configMap
  - downwardAPI
  - emptyDir
  - persistentVolumeClaim
  - projected
  - secret

users: []
groups: []
```

Apply and confirm:

```bash
oc apply -f wordpress-scc.yaml
oc get scc wordpress-scc
oc describe scc wordpress-scc
```

---

## 5. Grant the ServiceAccount Access to the SCC

SCC access is controlled by RBAC. The ServiceAccount needs the `use` verb on the SCC resource.

**Option A — quick imperative (great for workshops):**

```bash
oc adm policy add-scc-to-user wordpress-scc -z wordpress-sa -n wordpress-workshop
```

**Option B — explicit Role + RoleBinding (recommended for GitOps):**

```yaml
# wordpress-scc-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: use-wordpress-scc
  namespace: wordpress-workshop
rules:
  - apiGroups: ["security.openshift.io"]
    resources: ["securitycontextconstraints"]
    resourceNames: ["wordpress-scc"]
    verbs: ["use"]
```

```yaml
# wordpress-scc-rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: wordpress-sa-use-scc
  namespace: wordpress-workshop
subjects:
  - kind: ServiceAccount
    name: wordpress-sa
    namespace: wordpress-workshop
roleRef:
  kind: Role
  apiGroup: rbac.authorization.k8s.io
  name: use-wordpress-scc
```

```bash
oc apply -f wordpress-scc-role.yaml
oc apply -f wordpress-scc-rolebinding.yaml
```

Verify:

```bash
oc get rolebinding wordpress-sa-use-scc -n wordpress-workshop
```

---

## 6. Deploy MySQL

WordPress needs a database first. Deploy MySQL with a persistent volume and a Secret for credentials.

```yaml
# mysql-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: wordpress-workshop
type: Opaque
stringData:
  MYSQL_ROOT_PASSWORD: rootpassword123
  MYSQL_DATABASE: wordpress
  MYSQL_USER: wpuser
  MYSQL_PASSWORD: wppassword123
```

```yaml
# mysql-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  namespace: wordpress-workshop
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

```yaml
# mysql-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  namespace: wordpress-workshop
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      serviceAccountName: wordpress-sa
      containers:
        - name: mysql
          image: docker.io/library/mysql:8.0
          envFrom:
            - secretRef:
                name: mysql-secret
          ports:
            - containerPort: 3306
          securityContext:
            runAsUser: 999                # mysql user inside the official image
            allowPrivilegeEscalation: true
          volumeMounts:
            - name: mysql-storage
              mountPath: /var/lib/mysql
      volumes:
        - name: mysql-storage
          persistentVolumeClaim:
            claimName: mysql-pvc
```

```yaml
# mysql-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: wordpress-workshop
spec:
  selector:
    app: mysql
  ports:
    - port: 3306
      targetPort: 3306
```

Apply everything:

```bash
oc apply -f mysql-secret.yaml
oc apply -f mysql-pvc.yaml
oc apply -f mysql-deployment.yaml
oc apply -f mysql-service.yaml
```

Wait for MySQL to be ready:

```bash
oc rollout status deployment/mysql -n wordpress-workshop
```

---

## 7. Deploy WordPress

```yaml
# wordpress-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wordpress-pvc
  namespace: wordpress-workshop
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```

```yaml
# wordpress-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  namespace: wordpress-workshop
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      serviceAccountName: wordpress-sa
      containers:
        - name: wordpress
          image: docker.io/library/wordpress:latest
          env:
            - name: WORDPRESS_DB_HOST
              value: mysql:3306
            - name: WORDPRESS_DB_NAME
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_DATABASE
            - name: WORDPRESS_DB_USER
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_USER
            - name: WORDPRESS_DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_PASSWORD
          ports:
            - containerPort: 80
          securityContext:
            runAsUser: 33               # www-data
            allowPrivilegeEscalation: true
          volumeMounts:
            - name: wordpress-storage
              mountPath: /var/www/html
      volumes:
        - name: wordpress-storage
          persistentVolumeClaim:
            claimName: wordpress-pvc
```

```yaml
# wordpress-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  namespace: wordpress-workshop
spec:
  selector:
    app: wordpress
  ports:
    - port: 80
      targetPort: 80
```

```yaml
# wordpress-route.yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: wordpress
  namespace: wordpress-workshop
spec:
  to:
    kind: Service
    name: wordpress
  port:
    targetPort: 80
```

Apply:

```bash
oc apply -f wordpress-pvc.yaml
oc apply -f wordpress-deployment.yaml
oc apply -f wordpress-service.yaml
oc apply -f wordpress-route.yaml
```

Wait for WordPress to be ready:

```bash
oc rollout status deployment/wordpress -n wordpress-workshop
```

Get the URL:

```bash
oc get route wordpress -n wordpress-workshop
```

Open the URL in your browser — you should see the WordPress setup wizard.

---

## 8. Verify

### 8.1 Confirm pods are running under the correct UIDs

```bash
# WordPress should report uid=33
oc exec deployment/wordpress -n wordpress-workshop -- id

# MySQL should report uid=999
oc exec deployment/mysql -n wordpress-workshop -- id
```

Expected output for WordPress:
```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Expected output for MySQL:
```
uid=999(mysql) gid=999(mysql) groups=999(mysql)
```

### 8.2 Confirm which SCC was applied

```bash
oc get pod -n wordpress-workshop -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.annotations.openshift\.io/scc}{"\n"}{end}'
```

Both pods should show `wordpress-scc`.

### 8.3 Confirm filesystem writes work

```bash
oc exec deployment/wordpress -n wordpress-workshop -- ls -la /var/www/html/wp-content/
```

The directory should be owned by `www-data` (33) and contain writable plugin/theme/upload subdirectories.

### 8.4 Negative test — confirm the SCC boundary holds

Attempt to run a pod as root (`UID 0`) using the same ServiceAccount:

```bash
oc run root-test -n wordpress-workshop \
  --image=docker.io/library/wordpress:latest \
  --overrides='{"spec":{"serviceAccountName":"wordpress-sa","containers":[{"name":"app","image":"docker.io/library/wordpress:latest","command":["sleep","3600"],"securityContext":{"runAsUser":0}}]}}'

oc describe pod root-test -n wordpress-workshop
```

**Expected:** rejected. UID `0` falls outside the `33–999` range defined in `wordpress-scc`.

Clean up:

```bash
oc delete pod root-test -n wordpress-workshop --ignore-not-found
```

### 8.5 Negative test — confirm an unbound ServiceAccount is still blocked

```bash
oc create serviceaccount no-access-sa -n wordpress-workshop

oc run no-access-test -n wordpress-workshop \
  --image=docker.io/library/wordpress:latest \
  --overrides='{"spec":{"serviceAccountName":"no-access-sa","containers":[{"name":"app","image":"docker.io/library/wordpress:latest","command":["sleep","3600"],"securityContext":{"runAsUser":33}}]}}'

oc describe pod no-access-test -n wordpress-workshop
```

**Expected:** still rejected — `no-access-sa` was never granted the `use` verb on `wordpress-scc`, so it falls back to `restricted-v2`.

Clean up:

```bash
oc delete pod no-access-test -n wordpress-workshop --ignore-not-found
oc delete sa no-access-sa -n wordpress-workshop
```

---

## 9. Cleanup

```bash
# Delete the project (removes all namespaced resources)
oc delete project wordpress-workshop

# Delete the cluster-scoped SCC separately
oc delete scc wordpress-scc
```

---

## Summary

| Step | What you did | Key resource |
|---|---|---|
| 2 | Created project and ServiceAccount | `wordpress-sa` |
| 3 | Reproduced the `restricted-v2` rejection | Pod event log |
| 4 | Created custom SCC allowing UIDs 33–999 | `wordpress-scc` |
| 5 | Bound ServiceAccount to SCC via RBAC | Role + RoleBinding |
| 6 | Deployed MySQL as UID 999 | `mysql` Deployment |
| 7 | Deployed WordPress as UID 33 + Route | `wordpress` Deployment + Route |
| 8 | Verified UIDs, SCC annotation, boundary tests | `oc exec`, `oc get pod -o jsonpath` |

---

## Troubleshooting

**Pod stuck in `Pending` with SCC error:**
Check the exact error with `oc describe pod <name>`. Confirm the RoleBinding targets the exact ServiceAccount name and namespace. Run `oc adm policy who-can use scc wordpress-scc` to list all identities with access.

**Pod starts but gets `Permission denied` on filesystem:**
The SCC allowed the UID but the PVC was pre-provisioned with different ownership. Run `oc exec <pod> -- ls -lan /var/www/html` to check. Add an `initContainer` that runs `chown -R 33:33 /var/www/html` as a workaround.

**Wrong SCC applied (annotation shows something unexpected):**
OpenShift picks the most restrictive SCC that satisfies the pod spec from all SCCs available to the identity. If another SCC is also bound to `wordpress-sa` (e.g., via a group), it may win. Run `oc describe scc <name>` on each candidate and compare priorities.

**`oc adm policy add-scc-to-user` vs. explicit RBAC:**
The imperative command is fine for workshops and quick iteration. Use explicit Role/RoleBinding YAML in production so changes are reviewable, versioned, and auditable in Git.

**Never bind SCCs to `system:authenticated` or broad groups in real environments** — that silently grants relaxed permissions cluster-wide and defeats the purpose of having a restricted default.
