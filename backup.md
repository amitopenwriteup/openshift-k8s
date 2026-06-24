Yes. In **OpenShift Local / CRC (CodeReady Containers)**, etcd is running as part of the single-node control plane, so you can demonstrate and inspect **etcd backups (snapshots)** much like on a regular OpenShift cluster.

### Check etcd pods

```bash
oc get pods -n openshift-etcd
```

You should see an etcd pod similar to:

```text
etcd-crc-master-0
```

### Access the node

```bash
oc debug node/$(oc get nodes -o name | head -1)
```

Then enter the host:

```bash
chroot /host
```

### Create an etcd snapshot

OpenShift provides a helper script:

```bash
/usr/local/bin/cluster-backup.sh /home/core/assets/backup
```

Example output:

```text
Snapshot saved to /home/core/assets/backup/snapshot.db
Static pod resources saved to /home/core/assets/backup/static_kuberesources.tar.gz
```

### Verify the backup files

```bash
ls -lh /home/core/assets/backup
```

Expected:

```text
snapshot.db
static_kuberesources.tar.gz
```

### Inspect the snapshot

You can use `etcdctl`:

```bash
ETCDCTL_API=3 etcdctl snapshot status \
/home/core/assets/backup/snapshot.db \
--write-out=table
```

Example output:

```text
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| 12345678 |   105432 |      12000 |      45 MB |
+----------+----------+------------+------------+
```

### Copy the backup to your workstation

From another terminal:

```bash
oc debug node/<node-name>
```

or use:

```bash
oc rsync
```

to move the snapshot outside CRC for demonstration purposes.

### Things to note about CRC

* CRC is a **single-node OpenShift cluster**, so etcd has only one member.
* The backup process is the same as production OpenShift, but recovery testing is usually less meaningful because CRC is intended for development.
* The standard OpenShift backup script creates:

  * `snapshot.db` (etcd data)
  * `static_kuberesources.tar.gz` (control-plane static pod resources)

If you're preparing a demo or interview, a common flow is:

1. Create a test project.
2. Create a ConfigMap or Deployment.
3. Run `cluster-backup.sh`.
4. Show `snapshot.db`.
5. Use `etcdctl snapshot status` to prove the backup contains data.
6. Explain how restoration would be performed on a production cluster.
