# etcd Backup and Restore Lab — CRC (Single-Node OpenShift)

---

## Overview

This lab walks through taking a manual etcd snapshot on a CRC (single-node OpenShift) cluster using `oc debug node`, simulating data loss, and restoring from the snapshot.

**What you will do:**
1. Verify cluster health and fix disk pressure if needed
2. Create a test namespace and resource
3. Access the CRC node via `oc debug node/crc` and `chroot`
4. Take an etcd snapshot using the built-in backup script
5. Simulate data loss by deleting the test resource
6. Restore etcd from the snapshot
7. Verify the resource is recovered

---

## Prerequisites

- CRC running and `oc` connected (`oc whoami` returns a valid user)
- `oc debug` access to the CRC node (no SSH required)

---

## Step 1 — Verify Cluster Health

```bash
oc get nodes
oc get co | grep -v "True.*False.*False"
```

Expected: node `crc` in `Ready` state, all cluster operators showing `Available=True`.

### Check for Disk Pressure

```bash
oc describe node/crc | grep -A5 Conditions
```

> **Important:** If `DiskPressure` is `True`, the `oc debug` pod will fail to start.  
> Fix it before proceeding — see the **Troubleshooting** section at the end of this lab.

---

## Step 2 — Create a Test Resource

This gives you something to lose and recover after the restore.

```bash
oc new-project etcd-test
oc create configmap workshop-data \
  --from-literal=key1=hello \
  --from-literal=key2=world \
  -n etcd-test
oc get configmap workshop-data -n etcd-test -o yaml
```

Note the `resourceVersion` value in the output — it will change after a successful restore, confirming etcd was rolled back.

---

## Step 3 — Access the CRC Node via oc debug

Instead of SSH, use `oc debug node` to get a privileged shell on the node:

```bash
oc debug node/crc
```

Once the debug pod starts, you will see:

```
Starting pod/crc-debug-xxxxx ...
To use host binaries, run `chroot /host`.
```

Chroot into the host filesystem to access node binaries:

```bash
chroot /host
```

You are now operating as root inside the CRC node's filesystem.

> **Note:** All commands and output in this session are recorded in container logs.  
> The debug pod is temporary — do not exit until all node-side steps are complete.

---

## Step 4 — Take an etcd Snapshot

Still inside the `chroot /host` shell, run OpenShift's built-in etcd backup script:

```bash
/usr/local/bin/cluster-backup.sh /home/core/etcd-backup
```

The script will snapshot etcd and save the static pod resources. This takes about 30–60 seconds.

Verify the backup files were created:

```bash
ls -lh /home/core/etcd-backup/
```

Expected output (filenames include a timestamp):

```
-rw-------. 1 root root  95M Jun 25 12:00 snapshot_2026-06-25_120000.db
-rw-------. 1 root root 112K Jun 25 12:00 static_kuberesources_2026-06-25_120000.tar.gz
```

Two files must be present:
- `.db` — the etcd snapshot
- `.tar.gz` — the static pod resource backup

> **Keep the node shell open** — you will need it again for the restore step.

---

## Step 5 — Simulate Data Loss

Open a **second terminal** on System01 (leave the debug shell running in the first).

Delete the configmap to simulate data loss:

```bash
oc delete configmap workshop-data -n etcd-test
oc get configmap -n etcd-test
```

Expected:

```
No resources found in etcd-test namespace.
```

The data is gone. Now restore it from the snapshot.

---

## Step 6 — Restore etcd from Snapshot

Switch back to your **first terminal** (the `oc debug` / `chroot /host` shell).

### 6a — Confirm etcd is running before restore

```bash
crictl ps | grep etcd
```

You should see the etcd container in a `Running` state.

### 6b — Run the restore script

Pass the **directory path** (not the `.db` file directly):

```bash
/usr/local/bin/cluster-restore.sh /home/core/etcd-backup
```

The script will automatically:
- Stop the etcd, kube-apiserver, kube-controller-manager, and kube-scheduler static pods
- Restore the snapshot data into etcd
- Restart all static pods

You will see output similar to:

```
...stopping kube-apiserver-pod.yaml
...stopping kube-controller-manager-pod.yaml
...stopping kube-scheduler-pod.yaml
...stopping etcd-pod.yaml
Restoring etcd member from snapshot...
...starting etcd-pod.yaml
...starting kube-apiserver-pod.yaml
...starting kube-controller-manager-pod.yaml
...starting kube-scheduler-pod.yaml
```

This takes **2–5 minutes**. The API server will be unavailable during restart — this is expected.

### 6c — Exit the debug shell

```bash
exit   # exit chroot
exit   # exit debug pod
```

---

## Step 7 — Wait for API Server to Recover

On System01, watch the node status:

```bash
watch oc get nodes
```

The API will be unreachable for 1–3 minutes. Once `crc` shows `Ready`, proceed.

Then check cluster operators are recovering:

```bash
oc get co
```

Allow 5–10 minutes for all operators to return to `Available=True`.

---

## Step 8 — Verify the Restore

Confirm the deleted configmap is back:

```bash
oc get configmap workshop-data -n etcd-test -o yaml
```

Expected: configmap is present with `key1=hello` and `key2=world`.

Compare the `resourceVersion` to what you noted in Step 2 — it should be **different**, confirming etcd was rolled back to the pre-delete snapshot.

---

## Step 9 — Cleanup

Delete the test project:

```bash
oc delete project etcd-test
```

Remove the backup files from the node:

```bash
oc debug node/crc
chroot /host
rm -rf /home/core/etcd-backup
exit
exit
```

---

## Key Commands Reference

| Task | Command |
|---|---|
| Access node shell | `oc debug node/crc` then `chroot /host` |
| Take backup | `/usr/local/bin/cluster-backup.sh /home/core/etcd-backup` |
| Restore backup | `/usr/local/bin/cluster-restore.sh /home/core/etcd-backup` |
| Check etcd containers | `crictl ps \| grep etcd` |
| Check node conditions | `oc describe node/crc \| grep -A5 Conditions` |
| Check cluster operators | `oc get co` |

---

## Troubleshooting

### oc debug pod fails to start / "container not available"

This is almost always caused by **DiskPressure** on the node. Check:

```bash
oc describe node/crc | grep DiskPressure
```

If `DiskPressure: True`, free up disk space via SSH (SSH is only needed here as a fallback):

```bash
ssh -i ~/.crc/machines/crc/id_ed25519 core@$(crc ip)
```

Inside the node, clean up space:

```bash
# Check disk usage
df -h

# Prune unused container images
sudo crictl rmi --prune

# Vacuum journal logs
sudo journalctl --vacuum-size=200M

# Prune podman images
sudo podman image prune -a
```

Exit SSH and wait for DiskPressure to clear:

```bash
watch "oc describe node/crc | grep DiskPressure"
```

Once it shows `False`, retry `oc debug node/crc`.

---

### cluster-backup.sh not found

On some CRC versions the script may be at a different path:

```bash
find / -name cluster-backup.sh 2>/dev/null
```

---

### Restore script errors — snapshot file not found

Always pass the **directory**, not the `.db` file:

```bash
# Correct
/usr/local/bin/cluster-restore.sh /home/core/etcd-backup

# Wrong
/usr/local/bin/cluster-restore.sh /home/core/etcd-backup/snapshot_xxx.db
```

---

### API server does not come back after restore

Wait up to 10 minutes. If still unavailable, open a debug shell and check container status:

```bash
oc debug node/crc
chroot /host
crictl ps -a | grep -E "etcd|kube-apiserver"
crictl logs <container-id>
```

---

### Cluster operators degraded after restore

This is normal for up to 10 minutes post-restore as each operator reconciles. Monitor with:

```bash
watch "oc get co | grep -v 'True.*False.*False'"
```
