# Workshop: OpenShift CRC (Rocky Linux) — htpasswd Authentication, Roles & RoleBindings

**Goal:** Add htpasswd-based user authentication to an OpenShift CRC cluster running on Rocky Linux, create a custom Role and RoleBinding, and verify access with a test user.

**Audience:** Anyone with a working CRC cluster who wants local users with scoped permissions instead of relying solely on `kubeadmin`.

---

## 0. Prerequisites

- CRC already installed and running on Rocky Linux (`crc start` completed successfully)
- `oc` CLI available in `$PATH`
- `htpasswd` utility installed (see Step 1 below)
- Cluster-admin access via `kubeadmin`

Confirm CRC is up and log in as `kubeadmin`:

```bash
crc status
eval $(crc oc-env)
oc login -u kubeadmin -p $(crc console --credentials | grep kubeadmin | awk -F"-p " '{print $2}') https://api.crc.testing:6443
```

> If the one-liner above doesn't parse cleanly, just run `crc console --credentials` and copy the `kubeadmin` password manually.

Verify:

```bash
oc whoami
oc get nodes
```

---

## 1. Install the `htpasswd` Utility (Rocky Linux)

`htpasswd` ships in the `httpd-tools` package, which is separate from the full Apache web server — installing it does **not** start or enable httpd.

```bash
sudo dnf install -y httpd-tools
```

Verify the install and check the version:

```bash
which htpasswd
htpasswd
```

You should see usage output (`Usage: htpasswd [-cimBdpsDv] ...`) rather than a "command not found" error.

> **Note:** `httpd-tools` is included in Rocky Linux's default AppStream repo, so no extra repo configuration is needed. If `dnf` reports the package isn't found, refresh the metadata first: `sudo dnf makecache`.

---

## 2. Create the htpasswd File

Create a working directory and generate the htpasswd file with your first user.

```bash
mkdir -p ~/ocp-htpasswd && cd ~/ocp-htpasswd

# -c creates a new file, -B uses bcrypt, -b takes password on command line
htpasswd -c -B -b users.htpasswd devuser DevUser@123
```

Add additional users (omit `-c` so you don't overwrite the file):

```bash
htpasswd -B -b users.htpasswd opsuser OpsUser@123
htpasswd -B -b users.htpasswd vieweruser ViewUser@123
```

Check the file contents:

```bash
cat users.htpasswd
```

You should see one bcrypt-hashed line per user.

---

## 3. Create the htpasswd Secret in OpenShift

The secret must live in the `openshift-config` namespace and use the key name `htpasswd`.

```bash
oc create secret generic htpass-secret  --from-file=htpasswd=users.htpasswd  -n openshift-config
```

If the secret already exists and you're adding users later, update it instead:

```bash
oc create secret generic htpass-secret --from-file=htpasswd=users.htpasswd -n openshift-config --dry-run=client -o yaml | oc replace -f -
```

---

## 4. Configure the OAuth Identity Provider

Check the current OAuth configuration:

```bash
oc get oauth cluster -o yaml
```

Edit it to add the htpasswd identity provider:

```bash
oc edit oauth cluster
```

Add/merge this under `spec.identityProviders`:

```yaml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
    - name: htpasswd_provider
      mappingMethod: claim
      type: HTPasswd
      htpasswd:
        fileData:
          name: htpass-secret
```



## 5. Verify Login as the New Users

```bash
oc login -u devuser -p 'DevUser@123' https://api.crc.testing:6443
oc whoami
```

At this point `devuser` exists as a cluster user but has **no permissions** beyond the default authenticated-user role (e.g., can't list projects' resources, can't create anything meaningful).

Log back in as `kubeadmin` for the next steps:

```bash
oc login -u kubeadmin https://api.crc.testing:6443
```

---

## 6. Create a Project to Scope Permissions

```bash
oc new-project dev-workshop
```

---

## 7. Create a Role

This example Role grants typical "developer" permissions on Pods, Deployments, Services, and Routes within the `dev-workshop` namespace.

**Option A — imperative command:**

```bash
oc create role pod-deployment-manager   --verb=get,list,watch,create,update,patch,delete  --resource=pods,deployments,services,routes,configmaps  -n dev-workshop
```

**Option B — YAML manifest (recommended for repeatability):**

```yaml
# pod-deployment-manager-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-deployment-manager
  namespace: dev-workshop
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["route.openshift.io"]
    resources: ["routes"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

Apply it:

```bash
oc apply -f pod-deployment-manager-role.yaml
```

Also create a read-only Role for `vieweruser`:

```bash
oc create role project-viewer \
  --verb=get,list,watch \
  --resource=pods,deployments,services,routes,configmaps \
  -n dev-workshop
```

---

## 8. Create the RoleBinding

Bind `devuser` to the `pod-deployment-manager` Role in `dev-workshop`:

```bash
oc create rolebinding devuser-pod-deployment-manager \
  --role=pod-deployment-manager \
  --user=devuser \
  -n dev-workshop
```

Bind `vieweruser` to the read-only Role:

```bash
oc create rolebinding vieweruser-project-viewer \
  --role=project-viewer \
  --user=vieweruser \
  -n dev-workshop
```

Equivalent YAML version (for `devuser`):

```yaml
# devuser-rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: devuser-pod-deployment-manager
  namespace: dev-workshop
subjects:
  - kind: User
    apiGroup: rbac.authorization.k8s.io
    name: devuser
roleRef:
  kind: Role
  apiGroup: rbac.authorization.k8s.io
  name: pod-deployment-manager
```

```bash
oc apply -f devuser-rolebinding.yaml
```

Confirm both bindings:

```bash
oc get rolebinding -n dev-workshop
oc describe rolebinding devuser-pod-deployment-manager -n dev-workshop
```

---

## 9. Test the Setup

### 8.1 Test as `devuser` (should succeed)

```bash
oc login -u devuser -p 'DevUser@123' https://api.crc.testing:6443
oc project dev-workshop

# Should succeed
oc auth can-i create deployments -n dev-workshop
oc auth can-i delete pods -n dev-workshop

# Try deploying something real
oc new-app --image=registry.access.redhat.com/ubi9/httpd-24 --name=test-app -n dev-workshop
oc get pods -n dev-workshop
oc expose svc/test-app -n dev-workshop
oc get route test-app -n dev-workshop
```

### 8.2 Test permission boundary (should fail)

`devuser` was not granted cluster-wide or cross-namespace rights:

```bash
oc auth can-i create projects
# Expected: no

oc get pods -n openshift-config
# Expected: Error from server (Forbidden)
```

### 8.3 Test as `vieweruser` (read-only)

```bash
oc login -u vieweruser -p 'ViewUser@123' https://api.crc.testing:6443
oc project dev-workshop

oc auth can-i list pods -n dev-workshop        # Expected: yes
oc auth can-i create deployments -n dev-workshop  # Expected: no
oc delete pod <any-pod> -n dev-workshop        # Expected: Forbidden
```

### 8.4 Test as `opsuser` (no bindings yet)

```bash
oc login -u opsuser -p 'OpsUser@123' https://api.crc.testing:6443
oc auth can-i get pods -n dev-workshop
# Expected: no — opsuser has no RoleBinding in this namespace
```

This confirms RBAC is scoped per-user and per-namespace exactly as configured.

---

## 10. Cleanup (optional)

```bash
oc login -u kubeadmin -p <kubeadmin-password> https://api.crc.testing:6443
oc delete project dev-workshop
oc delete secret htpass-secret -n openshift-config
```

To fully remove the identity provider, edit the OAuth `cluster` resource again and remove the `htpasswd_provider` entry from `identityProviders`.

---

## Summary Table

| Step | Resource | Command |
|---|---|---|
| 1 | Install htpasswd | `sudo dnf install -y httpd-tools` |
| 2 | htpasswd file | `htpasswd -c -B -b users.htpasswd <user> <pass>` |
| 3 | Secret | `oc create secret generic htpass-secret --from-file=htpasswd=users.htpasswd -n openshift-config` |
| 4 | OAuth IdP | `oc edit oauth cluster` |
| 5 | Login test | `oc login -u <user> -p <pass>` |
| 6 | Project | `oc new-project dev-workshop` |
| 7 | Role | `oc create role <name> --verb=... --resource=... -n <ns>` |
| 8 | RoleBinding | `oc create rolebinding <name> --role=<role> --user=<user> -n <ns>` |
| 9 | Verify | `oc auth can-i <verb> <resource> -n <ns>` |

---

## Troubleshooting Notes

- **Login hangs / "x509 certificate" errors:** add `--insecure-skip-tls-verify=true` to `oc login` for CRC's self-signed cert, or trust CRC's CA via `crc setup`.
- **OAuth pods not restarting:** check `oc get pods -n openshift-authentication` and `oc logs` on the pod; also confirm the secret key is literally named `htpasswd`.
- **User shows "no projects":** that's expected immediately after first login before any RoleBinding exists — it's not an error.
- **Bcrypt vs other htpasswd hash types:** OpenShift's HTPasswd identity provider supports bcrypt (`-B`), MD5, and SHA1 hashes, but bcrypt (`-B`) is the recommended default.
