# Prometheus & Grafana Monitoring Lab — CRC (Single-Node OpenShift)

---

## Overview

This lab covers:
1. Installing Helm on Rocky Linux from tar.gz
2. Configuring OpenShift SCC permissions (anyuid)
3. Installing Prometheus using Helm on a CRC cluster
4. Installing Grafana using Helm
5. Configuring Grafana to use Prometheus as a data source
6. Writing a sample PromQL query on Grafana (dashboard)
7. Creating a Grafana alert rule

> **CRC Note:** This lab uses the default CRC StorageClass `crc-csi-hostpath-provisioner`. No NFS or external provisioner is required.

---

## Prerequisites

- CRC running and `oc` / `kubectl` connected
- Helm 3 installed (see Part 0 below)
- `oc` CLI with cluster-admin rights to assign SCCs (see Part 1)
- Default StorageClass `crc-csi-hostpath-provisioner` available (verify below)

```bash
oc get nodes && kubectl get storageclass && helm version
```

Expected StorageClass output:

```
NAME                                     PROVISIONER                        RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
crc-csi-hostpath-provisioner (default)   kubevirt.io.hostpath-provisioner   Retain          WaitForFirstConsumer   false                  42d
```

---

## Part 0 — Install Helm on Rocky Linux (from tar.gz)

### Step 1 — Download the Helm tar.gz

```bash
curl -LO https://get.helm.sh/helm-v3.17.3-linux-amd64.tar.gz
```

> Check the latest release at https://github.com/helm/helm/releases

### Step 2 — Extract the archive

```bash
tar -zxvf helm-v3.17.3-linux-amd64.tar.gz
```

### Step 3 — Move the binary to PATH

```bash
mv linux-amd64/helm /usr/local/bin/helm && chmod +x /usr/local/bin/helm
```

### Step 4 — Verify installation

```bash
helm version
```

### All-in-one command

```bash
curl -LO https://get.helm.sh/helm-v3.17.3-linux-amd64.tar.gz && tar -zxvf helm-v3.17.3-linux-amd64.tar.gz && mv linux-amd64/helm /usr/local/bin/helm && chmod +x /usr/local/bin/helm && helm version
```

### Cleanup

```bash
rm -rf helm-v3.17.3-linux-amd64.tar.gz linux-amd64
```

---

## Part 1 — OpenShift SCC Setup (Required before Helm installs)

OpenShift's Security Context Constraints (SCC) will block Prometheus and Grafana pods from starting because they run as UID `65534` (nobody), which falls outside the default namespace UID range `[1000690000, 1000699999]`. Grant the `anyuid` SCC to the relevant service accounts **before** running any Helm install.

### Step 1 — Grant anyuid SCC to Prometheus service accounts

```bash
oc adm policy add-scc-to-user anyuid -z prometheus-server -n default
```

```bash
oc adm policy add-scc-to-user anyuid -z prometheus-alertmanager -n default
```

```bash
oc adm policy add-scc-to-user anyuid -z prometheus-kube-state-metrics -n default
```

```bash
oc adm policy add-scc-to-user anyuid -z prometheus-prometheus-node-exporter -n default
```

> **Lab shortcut:** Grant to the `default` service account to cover all pods in one command:
> ```bash
> oc adm policy add-scc-to-user anyuid -z default -n default
> ```

### Step 2 — Grant anyuid SCC to the Grafana service account

```bash
oc adm policy add-scc-to-user anyuid -z grafana -n default
```

### Step 3 — Verify SCC assignments

```bash
oc get rolebindings -n default | grep anyuid
```

> **Note:** If using a custom namespace (e.g. `monitoring`), replace `-n default` with `-n monitoring` in all commands above.

---

## Part 2 — Prometheus Installation

### Step 1 — Add the Prometheus Community Helm Repo

### Step 1 — Add the Prometheus Community Helm Repo

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm repo list
```

---

### Step 2 — Install Prometheus

```bash
helm install prometheus prometheus-community/prometheus
```

> Installs Prometheus in the **default** namespace. Add `--namespace monitoring --create-namespace` for a dedicated namespace.

---

### Step 3 — Verify Pods

```bash
kubectl get po
```

Expected pods (may take 1–2 minutes):

```
NAME                                             READY   STATUS    RESTARTS
prometheus-server-xxxxxxxxxx-xxxxx               2/2     Running   0
prometheus-alertmanager-0                        1/1     Running   0
prometheus-kube-state-metrics-xxxxxxxxxx-xxxxx   1/1     Running   0
prometheus-prometheus-node-exporter-xxxxx        1/1     Running   0
```


---

### Step 4 — Access Prometheus UI

**Option A — Port Forward (quick test):**

```bash
kubectl port-forward --address 0.0.0.0 svc/prometheus-server 9090:80
```

Open browser: `http://<System01-IP>:9090`

**Option B — NodePort (persistent access):**

```bash
kubectl edit svc prometheus-server
```

Change `type: ClusterIP` to `type: NodePort`:

```yaml
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 9090
      nodePort: 30090
```

Access at: `http://$(crc ip):30090`

---

## Part 3 — Grafana Installation

### Step 1 — Add the Grafana Helm Repo

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

---

### Step 2 — Install Grafana

```bash
helm install grafana grafana/grafana
```

---

### Step 3 — Get the Admin Password

```bash
kubectl get secret --namespace default grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

Save this password for login.

---

### Step 4 — Access Grafana UI

**Option A — Port Forward (quick test):**

```bash
kubectl port-forward --address 0.0.0.0 svc/grafana 3000:80
```

Open browser: `http://<System01-IP>:3000`

**Option B — NodePort (persistent access):**

```bash
kubectl edit svc grafana
```

Change `type: ClusterIP` to `type: NodePort`:

```yaml
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 3000
      nodePort: 30030
```

Access Grafana at: `http://$(crc ip):30030`

Login:
- **Username:** `admin`
- **Password:** (from Step 3)

---

### Step 5 — Add Prometheus as a Data Source

1. In Grafana, go to **Connections → Data Sources → Add data source**
2. Select **Prometheus**
3. Get the Prometheus ClusterIP:

```bash
kubectl get svc prometheus-server -o jsonpath='{.spec.clusterIP}'
```

4. Set URL to:

```
http://<prometheus-server-ClusterIP>:80
```

5. Click **Save & Test** — confirm `Data source is working`

---

## Part 4 — Sample PromQL Query (Dashboard)

Go to **Dashboards → New → Add visualization** in Grafana, select the Prometheus data source, and run the following query.

---

### Node Memory Usage (%)

This single query gives a clear view of overall cluster memory health — ideal as the primary panel on a CRC monitoring dashboard.

```promql
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100
```

**Suggested panel settings:**

| Setting | Value |
|---|---|
| Visualization | Gauge or Time series |
| Unit | Percent (0–100) |
| Thresholds | Green < 70%, Yellow < 85%, Red ≥ 85% |
| Panel title | Node Memory Usage (%) |

Click **Apply** to save the panel to your dashboard.

---

## Part 5 — Grafana Alert Rule

Go to **Alerting → Alert Rules → New Alert Rule** and create the following:

---

### Alert — High Memory Usage

**Name:** `HighMemoryUsage`

**Query (A):**

```promql
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100
```

**Condition:** `IS ABOVE 90`  
**For:** `5m`  
**Severity:** `critical`

**Annotations:**

```
Summary: High memory usage detected
Description: Node memory usage is above 90%. Current value: {{ $value }}%
```

> This alert fires when memory stays above 90% for 5 continuous minutes, giving enough signal to distinguish a real pressure event from a transient spike. Pair it with a notification channel (email, Slack, etc.) under **Alerting → Contact Points**.

---

## Part 6 — Uninstall

```bash
helm uninstall prometheus
helm uninstall grafana
```

---

## Key Commands Reference

| Task | Command |
|---|---|
| Install Helm (all-in-one) | `curl -LO https://get.helm.sh/helm-v3.17.3-linux-amd64.tar.gz && tar -zxvf helm-v3.17.3-linux-amd64.tar.gz && mv linux-amd64/helm /usr/local/bin/helm && chmod +x /usr/local/bin/helm && helm version` |
| Add Prometheus repo | `helm repo add prometheus-community https://prometheus-community.github.io/helm-charts` |
| Install Prometheus | `helm install prometheus prometheus-community/prometheus` |
| Add Grafana repo | `helm repo add grafana https://grafana.github.io/helm-charts` |
| Install Grafana | `helm install grafana grafana/grafana` |
| Get Grafana password | `kubectl get secret grafana -o jsonpath="{.data.admin-password}" \| base64 --decode` |
| Port-forward Prometheus | `kubectl port-forward --address 0.0.0.0 svc/prometheus-server 9090:80` |
| Port-forward Grafana | `kubectl port-forward --address 0.0.0.0 svc/grafana 3000:80` |
| Check StorageClass | `kubectl get storageclass` |
| Check pods | `kubectl get po` |
| Check services | `kubectl get svc` |
| Uninstall Prometheus | `helm uninstall prometheus` |
| Uninstall Grafana | `helm uninstall grafana` |
| Grant anyuid SCC (Prometheus server) | `oc adm policy add-scc-to-user anyuid -z prometheus-server -n default` |
| Grant anyuid SCC (Grafana) | `oc adm policy add-scc-to-user anyuid -z grafana -n default` |
| Grant anyuid SCC (all, lab shortcut) | `oc adm policy add-scc-to-user anyuid -z default -n default` |
| Restart Prometheus after SCC fix | `kubectl rollout restart deployment/prometheus-server && kubectl rollout restart statefulset/prometheus-alertmanager` |
| Restart Grafana after SCC fix | `kubectl rollout restart deployment/grafana` |

---

## Troubleshooting

### Pods forbidden — SCC (Security Context Constraint) error

**Symptom:** Pods fail with errors like:

```
unable to validate against any security context constraint:
  provider "anyuid": Forbidden: not usable by user or serviceaccount
  provider restricted-v2: .containers[0].runAsUser: Invalid value: 65534: must be in the ranges: [1000690000, 1000699999]
  provider restricted-v2: .spec.securityContext.fsGroup: Invalid value: [65534]: 65534 is not an allowed group
```

**Cause:** OpenShift's SCC system blocks Prometheus pods because they run as UID `65534` (nobody), which falls outside the namespace's allowed UID range. The service accounts need the `anyuid` SCC granted explicitly.

**Fix — Grant `anyuid` SCC to all Prometheus service accounts:**

```bash
oc adm policy add-scc-to-user anyuid -z prometheus-server -n default
```

```bash
oc adm policy add-scc-to-user anyuid -z prometheus-alertmanager -n default
```

```bash
oc adm policy add-scc-to-user anyuid -z prometheus-kube-state-metrics -n default
```

```bash
oc adm policy add-scc-to-user anyuid -z prometheus-prometheus-node-exporter -n default
```

Or grant to all in one shot using the `default` service account (simpler for lab use):

```bash
oc adm policy add-scc-to-user anyuid -z default -n default
```

**Restart the pods after granting SCC:**

```bash
kubectl rollout restart deployment/prometheus-server && kubectl rollout restart statefulset/prometheus-alertmanager
```

**Verify pods are now running:**

```bash
kubectl get po
```

> **Note:** If using a custom namespace (e.g. `monitoring`), replace `-n default` with `-n monitoring` in all commands above.

---

### Grafana pod in Pending or SCC-forbidden state

```bash
kubectl describe pod -l app.kubernetes.io/name=grafana
```

If you see the same SCC UID range error, grant `anyuid` to the Grafana service account:

```bash
oc adm policy add-scc-to-user anyuid -z grafana -n default
```

Then restart:

```bash
kubectl rollout restart deployment/grafana
```


### Data source connection failed in Grafana

Use the ClusterIP of Prometheus, not `localhost`:

```bash
kubectl get svc prometheus-server -o jsonpath='{.spec.clusterIP}'
```

URL format in Grafana: `http://<ClusterIP>:80`

---

## Reference Links

- Prometheus Helm Charts: https://github.com/prometheus-community/helm-charts
- Grafana Helm Charts: https://github.com/grafana/helm-charts
- ArtifactHub: https://artifacthub.io/
- PromQL Basics: https://prometheus.io/docs/prometheus/latest/querying/basics/
