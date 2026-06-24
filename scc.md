# Workshop: OpenShift CRC — Security Context Constraints (SCC) Basic Scenario

**Goal:** Understand why default SCCs block certain pods, create a custom SCC, bind it to a ServiceAccount, and prove the difference by deploying a workload before and after the SCC is applied.

**Audience:** Anyone with a working CRC cluster who needs pods to run with a specific UID, group, or relaxed security setting (e.g., legacy containers that expect to run as root or a fixed UID).

---

## 0. Background: What is an SCC?

A **Security Context Constraint (SCC)** is an OpenShift-specific resource (no direct Kubernetes equivalent) that controls what a pod is *allowed* to do at the security-context level — things like:

- Which UID/GID range a container can run as
- Whether containers can run as a privileged container
- Whether `hostNetwork`, `hostPID`, `hostIPC`, or `hostPath` volumes are allowed
- Which Linux capabilities can be added/dropped
- What SELinux context applies

By default, regular ServiceAccounts and users are bound to the **`restricted-v2`** SCC, which forces:

- A randomly assigned UID from the namespace's allocated range
- No privileged escalation
- No host namespace access

This is great for security but breaks images that hardcode a specific UID (a very common real-world scenario, e.g. UID `1001` baked into a Dockerfile).

This workshop walks through that exact failure and the fix.

---

## 1. Prerequisites

- CRC running on Rocky Linux (`crc start` completed)
- `oc` CLI in `$PATH`
- Cluster-admin access via `kubeadmin` (SCCs are cluster-scoped and require elevated privileges to create)

```bash
eval $(crc oc-env)
oc login -u kubeadmin -p <kubeadmin-password> https://api.crc.testing:6443
oc whoami
```

---

## 2. Create a Project and ServiceAccount

```bash
oc new-project scc-workshop
oc create serviceaccount fixed-uid-sa -n scc-workshop
```

Confirm:

```bash
oc get sa fixed-uid-sa -n scc-workshop
```

---

## 3. Reproduce the Failure (Before SCC)

Deploy a pod that explicitly requests UID `1001` using the default ServiceAccount restrictions.

```yaml
# pod-fixed-uid.yaml
apiVersion: v1
kind: Pod
metadata:
  name: fixed-uid-pod
  namespace: scc-workshop
spec:
  serviceAccountName: fixed-uid-sa
  containers:
    - name: app
      image: registry.access.redhat.com/ubi9/ubi-minimal
      command: ["sleep", "3600"]
      securityContext:
        runAsUser: 1001
```

Apply it:

```bash
oc apply -f pod-fixed-uid.yaml
oc get pod fixed-uid-pod -n scc-workshop
oc describe pod fixed-uid-pod -n scc-workshop
```

**Expected result:** the pod is rejected or fails to start. Look for an event like:

```
Error creating: pods "fixed-uid-pod" is forbidden: unable to validate against any security context constraint:
[provider "restricted-v2": .spec.containers[0].securityContext.runAsUser: Invalid value: 1001: must be in the ranges: [1000680000, 1000689999]]
```

This confirms the `restricted-v2` SCC is blocking the fixed UID. This is the problem the rest of the workshop fixes.

Clean up the failed pod before continuing:

```bash
oc delete pod fixed-uid-pod -n scc-workshop --ignore-not-found
```

---

## 4. Create a Custom SCC

Create an SCC that allows a specific fixed UID range while keeping everything else as locked-down as the default.

```yaml
# fixed-uid-scc.yaml
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: fixed-uid-scc
allowHostDirVolumePlugin: false
allowHostIPC: false
allowHostNetwork: false
allowHostPID: false
allowHostPorts: false
allowPrivilegeEscalation: false
allowPrivilegedContainer: false
allowedCapabilities: []
defaultAddCapabilities: []
requiredDropCapabilities:
  - ALL
readOnlyRootFilesystem: false
runAsUser:
  type: MustRunAsRange
  uidRangeMin: 1001
  uidRangeMax: 1001
seLinuxContext:
  type: MustRunAs
fsGroup:
  type: MustRunAs
supplementalGroups:
  type: RunAsAny
users: []
groups: []
volumes:
  - configMap
  - downwardAPI
  - emptyDir
  - persistentVolumeClaim
  - projected
  - secret
```

Apply it:

```bash
oc apply -f fixed-uid-scc.yaml
oc get scc fixed-uid-scc
```

> **Note:** `users` and `groups` are intentionally left empty here — SCC-to-identity binding in OpenShift is done via RBAC (`use` verb on the SCC resource), not by listing users directly inside the SCC manifest. That's the next step.

---

## 5. Grant the ServiceAccount Permission to Use the SCC

SCC access is itself governed by RBAC. The ServiceAccount needs the `use` verb on the new SCC.

**Option A — quick imperative command:**

```bash
oc adm policy add-scc-to-user fixed-uid-scc -z fixed-uid-sa -n scc-workshop
```

**Option B — explicit Role + RoleBinding (more transparent, GitOps-friendly):**

```yaml
# use-fixed-uid-scc-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: use-fixed-uid-scc
  namespace: scc-workshop
rules:
  - apiGroups: ["security.openshift.io"]
    resources: ["securitycontextconstraints"]
    resourceNames: ["fixed-uid-scc"]
    verbs: ["use"]
```

```yaml
# use-fixed-uid-scc-rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: fixed-uid-sa-use-scc
  namespace: scc-workshop
subjects:
  - kind: ServiceAccount
    name: fixed-uid-sa
    namespace: scc-workshop
roleRef:
  kind: Role
  apiGroup: rbac.authorization.k8s.io
  name: use-fixed-uid-scc
```

```bash
oc apply -f use-fixed-uid-scc-role.yaml
oc apply -f use-fixed-uid-scc-rolebinding.yaml
```

Verify the grant:

```bash
oc get rolebinding fixed-uid-sa-use-scc -n scc-workshop
oc describe scc fixed-uid-scc
```

---

## 6. Redeploy the Pod (After SCC)

Reapply the same pod manifest from Step 3 — nothing about the pod spec changes, only the permissions behind it have changed.

```bash
oc apply -f pod-fixed-uid.yaml
oc get pod fixed-uid-pod -n scc-workshop -w
```

Wait for `STATUS` to reach `Running`, then `Ctrl+C`.

---

## 7. Test and Verify

### 7.1 Confirm the pod is actually running as UID 1001

```bash
oc exec fixed-uid-pod -n scc-workshop -- id
```

Expected output:

```
uid=1001 gid=0(root) groups=0(root)
```

### 7.2 Confirm which SCC was applied

```bash
oc get pod fixed-uid-pod -n scc-workshop -o jsonpath='{.metadata.annotations.openshift\.io/scc}{"\n"}'
```

Expected output:

```
fixed-uid-scc
```

### 7.3 Confirm the boundary still holds (negative test)

Try requesting a UID outside the allowed range to prove the SCC isn't wide open:

```bash
oc run bad-uid-pod -n scc-workshop \
  --image=registry.access.redhat.com/ubi9/ubi-minimal \
  --overrides='{"spec":{"serviceAccountName":"fixed-uid-sa","containers":[{"name":"app","image":"registry.access.redhat.com/ubi9/ubi-minimal","command":["sleep","3600"],"securityContext":{"runAsUser":0}}]}}'

oc get pod bad-uid-pod -n scc-workshop
oc describe pod bad-uid-pod -n scc-workshop
```

**Expected result:** rejected, because `fixed-uid-scc` only permits UID `1001` (`MustRunAsRange` with min=max=1001), and `allowPrivilegedContainer`/root are still disallowed.

Clean up the negative test:

```bash
oc delete pod bad-uid-pod -n scc-workshop --ignore-not-found
```

### 7.4 Confirm a different ServiceAccount is still blocked

```bash
oc create serviceaccount no-access-sa -n scc-workshop
oc run no-access-pod -n scc-workshop \
  --image=registry.access.redhat.com/ubi9/ubi-minimal \
  --overrides='{"spec":{"serviceAccountName":"no-access-sa","containers":[{"name":"app","image":"registry.access.redhat.com/ubi9/ubi-minimal","command":["sleep","3600"],"securityContext":{"runAsUser":1001}}]}}'

oc describe pod no-access-pod -n scc-workshop
```

**Expected result:** still rejected — `no-access-sa` was never granted the `use` verb on `fixed-uid-scc`, so it falls back to `restricted-v2`, which forbids UID `1001`.

Clean up:

```bash
oc delete pod no-access-pod -n scc-workshop --ignore-not-found
oc delete sa no-access-sa -n scc-workshop
```

---

## 8. Cleanup (optional)

```bash
oc delete project scc-workshop
oc delete scc fixed-uid-scc
```

> Deleting the project removes the namespaced Role, RoleBinding, ServiceAccount, and Pod automatically. The SCC itself is cluster-scoped and must be deleted separately.

---

## Summary Table

| Step | Resource | Command |
|---|---|---|
| 2 | Project + SA | `oc new-project scc-workshop` / `oc create sa fixed-uid-sa` |
| 3 | Reproduce failure | `oc apply -f pod-fixed-uid.yaml` (fails under `restricted-v2`) |
| 4 | Custom SCC | `oc apply -f fixed-uid-scc.yaml` |
| 5 | Grant `use` access | `oc adm policy add-scc-to-user fixed-uid-scc -z fixed-uid-sa -n scc-workshop` |
| 6 | Redeploy pod | `oc apply -f pod-fixed-uid.yaml` (now succeeds) |
| 7 | Verify | `oc exec ... -- id`, `oc get pod -o jsonpath='{...annotations.openshift.io/scc}'` |

---

## Troubleshooting Notes

- **"unable to validate against any security context constraint":** the ServiceAccount running the pod has no SCC granting the requested security context. Check `oc describe scc <name>` and confirm the RoleBinding/`add-scc-to-user` grant targets the *exact* ServiceAccount (`-z <name>`) and namespace.
- **Pod runs but under the wrong SCC:** the `openshift.io/scc` annotation on the pod shows which SCC actually won admission. If multiple SCCs are available to the same identity, OpenShift picks the most restrictive one that satisfies the pod spec — so an overly permissive SCC granted elsewhere can silently take priority over a narrowly-scoped one if it's also more aligned with the pod's defaults. Use `oc describe scc` on each candidate to compare priority and constraints.
- **`oc adm policy add-scc-to-user` vs explicit RBAC:** the imperative command is faster for workshops/demos; explicit Role/RoleBinding YAML is preferred for GitOps repos since it's reviewable and versioned like any other manifest.
- **Don't bind SCCs to `system:authenticated` or broad groups** in real environments — that defeats the purpose of having a restricted default and effectively grants the relaxed SCC cluster-wide.
