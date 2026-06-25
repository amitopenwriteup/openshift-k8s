

### 1. Multiple master nodes (3 etcd members)
CRC is single-node, so etcd has one member. UPI typically has **3 master nodes**, each running an etcd member.

- Run `cluster-backup.sh` on **one master node only** — the snapshot captures the full cluster state
- Run `cluster-restore.sh` on **all 3 masters**, starting with the one you backed up on

```bash
# On master-0 first
sudo /usr/local/bin/cluster-restore.sh /home/core/etcd-backup

# Then master-1 and master-2
sudo /usr/local/bin/cluster-restore.sh /home/core/etcd-backup
```

### 2. Node access — SSH not oc debug
In UPI you typically SSH directly to master nodes using their IPs or hostnames:

```bash
ssh core@master-0.example.com
ssh core@master-1.example.com
ssh core@master-2.example.com
```

`oc debug node` may or may not work depending on cluster state during a restore scenario — SSH is more reliable when the API is down.

### 3. Copy backup to all masters before restore
Since the backup file sits on one master, copy it to the other two before running restore:

```bash
scp -r core@master-0.example.com:/home/core/etcd-backup /tmp/etcd-backup
scp -r /tmp/etcd-backup core@master-1.example.com:/home/core/
scp -r /tmp/etcd-backup core@master-2.example.com:/home/core/
```

### 4. Restore order matters
Always restore in this order:
1. **master-0** (or whichever node you took the backup on) — run restore first and wait for etcd to come up
2. **master-1** — run restore
3. **master-2** — run restore

### 5. Force etcd quorum restart after restore
In UPI multi-member etcd, after restore you may need to restart the etcd operator to force peer discovery:

```bash
oc patch etcd cluster -p='{"spec": {"forceRedeploymentReason": "recovery-'"$( date --rfc-3339=ns )"'"}}' --type=merge
```

---

## Summary Table

| Aspect | CRC (Single Node) | UPI (3 Masters) |
|---|---|---|
| etcd members | 1 | 3 |
| Backup on | 1 node | 1 node only |
| Restore on | 1 node | All 3 masters |
| Node access | `oc debug` or SSH | SSH (preferred) |
| Copy backup | Not needed | Copy to all masters |
| Restore order | Any | master-0 first |
| Force redeploy | Usually not needed | Recommended |

---

The official Red Hat doc for UPI etcd restore is at **docs.openshift.com → Backup and restore → Restoring to a previous cluster state** — worth bookmarking as it has version-specific nuances for OCP 4.x.
