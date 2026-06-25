# Workshop: MariaDB StatefulSet on OpenShift Developer Sandbox

> **Environment:** OpenShift Developer Sandbox  
> **Image:** `mariadb:13.0.1-resolute-rc`  
> **Storage Options:** AWS EBS (block) and AWS EFS (shared filesystem)

---

## Prerequisites

- Access to [OpenShift Developer Sandbox](https://developers.redhat.com/developer-sandbox)
- `oc` CLI installed and logged in
- Basic understanding of Kubernetes/OpenShift concepts

Verify your login:

```bash
oc whoami
oc project   # Note your assigned namespace
```

---

## Concepts Review

### Why StatefulSet for Databases?

A StatefulSet differs from a Deployment in three key ways:

| Feature | Deployment | StatefulSet |
|---|---|---|
| Pod identity | Random names | Stable, ordered names (`db-0`, `db-1`) |
| Storage | Shared or ephemeral | Each pod gets its own PVC |
| Startup/shutdown | Parallel | Ordered (0 → N) |
| DNS | Single service | Stable per-pod DNS entries |

MariaDB requires stable storage and identity, making StatefulSet the correct workload type.

### EBS vs EFS on OpenShift Sandbox

| | EBS (Elastic Block Store) | EFS (Elastic File System) |
|---|---|---|
| Access Mode | `ReadWriteOnce` (one node) | `ReadWriteMany` (multi-node) |
| Performance | High IOPS, low latency | Moderate, shared |
| Use case | **Primary DB data directory** | Shared config, backups, logs |
| StorageClass | `gp2` / `gp3` | `efs-sc` |
| Reclaim | Delete or Retain | Delete or Retain |

---

## Step 1 — Explore Available StorageClasses

```bash
oc get storageclass
```

Expected output (sandbox may vary):

```
NAME            PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE
gp2 (default)   kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer
gp3             ebs.csi.aws.com         Delete          WaitForFirstConsumer
efs-sc          efs.csi.aws.com         Delete          Immediate
```

Note which StorageClass names are available — you'll reference them in the PVC templates below.

---

## Step 2 — Create the Namespace Resources

### Secret for MariaDB Credentials

```yaml
# mariadb-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mariadb-secret
type: Opaque
stringData:
  MARIADB_ROOT_PASSWORD: "r00tP@ssw0rd"
  MARIADB_DATABASE: "workshopdb"
  MARIADB_USER: "appuser"
  MARIADB_PASSWORD: "appP@ssw0rd"
```

```bash
oc apply -f mariadb-secret.yaml
```

### ConfigMap for Custom MariaDB Config

```yaml
# mariadb-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mariadb-config
data:
  my.cnf: |
    [mysqld]
    innodb_buffer_pool_size = 256M
    innodb_log_file_size    = 64M
    max_connections         = 100
    character-set-server    = utf8mb4
    collation-server        = utf8mb4_unicode_ci
    bind-address            = 0.0.0.0
```

```bash
oc apply -f mariadb-config.yaml
```

---

## Step 3 — Headless Service

A headless Service (`clusterIP: None`) enables stable DNS per pod:  
`mariadb-0.mariadb-headless.<namespace>.svc.cluster.local`

```yaml
# mariadb-headless-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: mariadb-headless
  labels:
    app: mariadb
spec:
  clusterIP: None
  selector:
    app: mariadb
  ports:
    - name: mysql
      port: 3306
      targetPort: 3306
```

Also create a regular Service for application access:

```yaml
# mariadb-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: mariadb
  labels:
    app: mariadb
spec:
  selector:
    app: mariadb
  ports:
    - name: mysql
      port: 3306
      targetPort: 3306
```

```bash
oc apply -f mariadb-headless-svc.yaml
oc apply -f mariadb-svc.yaml
```

---

## Step 4 — StatefulSet with EBS (Primary Data)

This is the core of the workshop. The `volumeClaimTemplates` section automatically provisions an EBS PVC per pod.

```yaml
# mariadb-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mariadb
  labels:
    app: mariadb
spec:
  serviceName: mariadb-headless        # Must match headless Service name
  replicas: 1                          # Start with 1; increase for replication setups
  selector:
    matchLabels:
      app: mariadb
  template:
    metadata:
      labels:
        app: mariadb
    spec:
      securityContext:
        fsGroup: 999                   # mariadb group in the container
      containers:
        - name: mariadb
          image: mariadb:13.0.1-resolute-rc
          ports:
            - containerPort: 3306
              name: mysql
          envFrom:
            - secretRef:
                name: mariadb-secret
          volumeMounts:
            - name: data              # EBS — MariaDB data directory
              mountPath: /var/lib/mysql
            - name: config            # ConfigMap volume
              mountPath: /etc/mysql/conf.d/my.cnf
              subPath: my.cnf
            - name: backups           # EFS — shared backup directory
              mountPath: /var/backups/mysql
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "500m"
          livenessProbe:
            exec:
              command:
                - sh
                - -c
                - "mysqladmin ping -u root -p${MARIADB_ROOT_PASSWORD}"
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
          readinessProbe:
            exec:
              command:
                - sh
                - -c
                - "mysql -u root -p${MARIADB_ROOT_PASSWORD} -e 'SELECT 1'"
            initialDelaySeconds: 20
            periodSeconds: 10
            timeoutSeconds: 5
      volumes:
        - name: config
          configMap:
            name: mariadb-config
        - name: backups               # EFS PVC (pre-created, shared across pods)
          persistentVolumeClaim:
            claimName: mariadb-efs-backups

  # EBS PVC — one per pod, automatically named: data-mariadb-0, data-mariadb-1 ...
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: gp2          # Change to gp3 if available
        resources:
          requests:
            storage: 5Gi
```

---

## Step 5 — EFS PVC for Shared Backups

Unlike EBS (provisioned per pod automatically), the EFS PVC is created manually and shared across all pods.

```yaml
# mariadb-efs-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mariadb-efs-backups
spec:
  accessModes:
    - ReadWriteMany                   # EFS supports multi-pod access
  storageClassName: efs-sc
  resources:
    requests:
      storage: 10Gi
```

```bash
oc apply -f mariadb-efs-pvc.yaml
```

Verify the PVC binds before applying the StatefulSet:

```bash
oc get pvc mariadb-efs-backups
# STATUS should be Bound
```

---

## Step 6 — Deploy the StatefulSet

```bash
oc apply -f mariadb-statefulset.yaml
```

Watch pod come up in order:

```bash
oc get pods -l app=mariadb -w
```

Expected progression:

```
NAME        READY   STATUS    RESTARTS   AGE
mariadb-0   0/1     Pending   0          2s
mariadb-0   0/1     Init:0/1  0          5s
mariadb-0   1/1     Running   0          30s
```

Check auto-provisioned EBS PVC:

```bash
oc get pvc
```

You should see both:

```
NAME                STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS
data-mariadb-0      Bound    pvc-...  5Gi        RWO            gp2
mariadb-efs-backups Bound    pvc-...  10Gi       RWX            efs-sc
```

---

## Step 7 — Verify MariaDB is Running

```bash
# Exec into the pod
oc exec -it mariadb-0 -- bash

# Inside the pod — connect to MariaDB
mysql -u root -p${MARIADB_ROOT_PASSWORD}
```

Run a quick check inside the MariaDB shell:

```sql
SHOW DATABASES;
SELECT @@version;
CREATE TABLE workshopdb.test_table (id INT AUTO_INCREMENT PRIMARY KEY, name VARCHAR(50));
INSERT INTO workshopdb.test_table (name) VALUES ('OpenShift Workshop');
SELECT * FROM workshopdb.test_table;
EXIT;
```

---

## Step 8 — Test Storage Persistence

Verify data survives a pod restart (the critical stateful guarantee):

```bash
# Delete the pod (StatefulSet will recreate it)
oc delete pod mariadb-0

# Wait for it to come back
oc get pods -l app=mariadb -w

# Reconnect and verify data is still there
oc exec -it mariadb-0 -- mysql -u root -p${MARIADB_ROOT_PASSWORD} \
  -e "SELECT * FROM workshopdb.test_table;"
```

---

## Step 9 — Test EFS Shared Backup Volume

Write a file to the EFS volume and confirm it is accessible from the pod:

```bash
oc exec -it mariadb-0 -- bash -c \
  "mysqldump -u root -p\${MARIADB_ROOT_PASSWORD} workshopdb > /var/backups/mysql/workshopdb-backup.sql"

# Verify the file exists
oc exec -it mariadb-0 -- ls -lh /var/backups/mysql/
```

If you scale to multiple replicas, all pods can read the same backup directory because EFS uses `ReadWriteMany`.

---

## Step 10 — Scale the StatefulSet

> **Note:** Scaling a standalone MariaDB StatefulSet does NOT set up automatic replication. Additional replicas will boot with their own empty EBS volumes unless you configure MariaDB replication or use a Galera cluster setup. This step demonstrates the StatefulSet scaling mechanism.

```bash
oc scale statefulset mariadb --replicas=2

# Watch pods start in order (mariadb-1 only starts after mariadb-0 is Ready)
oc get pods -l app=mariadb -w

# New EBS PVC is automatically created for mariadb-1
oc get pvc
```

Scale back down:

```bash
oc scale statefulset mariadb --replicas=1
# mariadb-1 pod is deleted; its PVC (data-mariadb-1) is RETAINED for safety
oc get pvc   # data-mariadb-1 still exists
```

---

## Cleanup

Remove all resources when done:

```bash
oc delete statefulset mariadb
oc delete service mariadb mariadb-headless
oc delete secret mariadb-secret
oc delete configmap mariadb-config
oc delete pvc mariadb-efs-backups

# StatefulSet PVCs must be deleted manually (by design — prevents accidental data loss)
oc delete pvc data-mariadb-0
oc delete pvc data-mariadb-1   # if scaled up earlier
```

---

## Troubleshooting

### Pod stuck in Pending

```bash
oc describe pod mariadb-0
# Look for: "0/1 nodes are available" or PVC binding issues
oc describe pvc data-mariadb-0
```

Common causes: StorageClass name mismatch, sandbox quota exceeded, or `WaitForFirstConsumer` binding mode requiring the pod to be scheduled first.

### EFS PVC not binding

```bash
oc describe pvc mariadb-efs-backups
# Look for: "no persistent volumes available" or provisioner errors
oc get events --sort-by='.lastTimestamp'
```

Verify `efs-sc` StorageClass exists: `oc get sc`. If not available in your sandbox, replace the EFS volume with an `emptyDir` for the workshop exercises.

### Permission denied on /var/lib/mysql

The `fsGroup: 999` in `securityContext` sets ownership of mounted volumes to the `mysql` group. If you see permission errors:

```bash
oc exec -it mariadb-0 -- ls -la /var/lib/mysql
# uid/gid should include 999
```

On OpenShift, if a custom SCC blocks fsGroup, contact your sandbox administrator or use a SCC that permits fsGroup ranges.

### MariaDB init errors

```bash
oc logs mariadb-0 --previous   # logs from last failed attempt
oc logs mariadb-0              # current logs
```

---

## Summary

| Component | Purpose |
|---|---|
| `StatefulSet` | Ordered, stable identity pods for MariaDB |
| `Headless Service` | Stable per-pod DNS (`mariadb-0.mariadb-headless...`) |
| `ClusterIP Service` | Single endpoint for application connections |
| `Secret` | Credentials injected via `envFrom` |
| `ConfigMap` | Custom `my.cnf` mounted into container |
| `volumeClaimTemplates` (EBS `gp2`) | Auto-provisioned per-pod data volume |
| `PVC` (EFS `efs-sc`) | Shared backup volume across all replicas |

---

*Workshop created for OpenShift Developer Sandbox — MariaDB `13.0.1-resolute-rc`*
