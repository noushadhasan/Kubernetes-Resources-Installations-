# Runbook: Persistent Storage (PV/PVC with NFS) for API Pods in Kubernetes

**Purpose:** Persist application data (e.g., uploaded files in `/app/App_Data`) across pod restarts, redeployments, and rescheduling, so data is not lost when a new pod replaces an old one.

**Example used throughout:** `test-api` in namespace `staging`, NFS server `10.200.90.211`.

---

## 1. Architecture Overview

```
Azure Pipeline → Deployment (pod) → PVC → PV → NFS Server (10.200.90.211)
                                                  └── /data/<app-name>/app_data
```

- **NFS server** holds the actual data on disk.
- **PV (PersistentVolume)** is the cluster-level object pointing to the NFS export.
- **PVC (PersistentVolumeClaim)** is the namespace-level claim the pod mounts.
- PV + PVC are created **once, manually** — they are NOT part of the CI/CD release file.
- The Deployment (from the pipeline) only references the PVC by name.

Naming convention: `<app-name>-appdata-pv` / `<app-name>-appdata-pvc`.

---

## 2. Prerequisites

### 2.1 NFS Server side (10.200.90.211)

1. Install NFS server (Ubuntu):
   ```bash
   sudo apt-get install -y nfs-kernel-server
   ```
2. Create the export directory:
   ```bash
   sudo mkdir -p /data/test-api/app_data
   ```
  Set Permissions:
  ```bash
  sudo chown nobody:nogroup /data/test-api/app_data
  sudo chmod 777 /data/test-api/app_data
  ```
3. Add to `/etc/exports` (adjust CIDR to the cluster node network):
   ```
   /data/test-api/app_data  *(rw,sync,no_subtree_check,no_root_squash)
   ```
   `no_root_squash` is required because the container runs as root.
4. Apply exports and verify:
   ```bash
   sudo exportfs -ra        #apply
   ```
   ```bash
   sudo exportfs -v         # verify
   ```
   sudo systemctl status nfs-server
   showmount -e localhost
   ```

### 2.2 Every Kubernetes worker node

NFS **client** utilities must be installed on ALL worker nodes, otherwise the pod fails with:

> `mount: ... bad option; for several filesystems (e.g. nfs, cifs) you might need a /sbin/mount.<type> helper program`

Install on each node:

```bash
# Ubuntu/Debian
sudo apt-get update && sudo apt-get install -y nfs-common
```

Verify from a node before deploying:

```bash
showmount -e 10.200.90.211
sudo mkdir -p /mnt/nfstest
sudo mount -t nfs 10.200.90.211:/data/test-api/app_data /mnt/nfstest
ls /mnt/nfstest && sudo umount /mnt/nfstest
```

> **Note for new nodes:** when a new worker node is added to the cluster, install `nfs-common` on it as part of node provisioning.

---

## 3. Create PV and PVC (one-time, manual)

Save as `<app-name>-pv-pvc.yaml` and apply with `kubectl apply -f <file>`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: <app-name>-appdata-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: 10.200.90.211
    path: /data/test-api/app_data
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-api-appdata-pvc
  namespace: staging
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: ""          # must be empty string to bind static PV
  resources:
    requests:
      storage: 10Gi
  volumeName: test-api-appdata-pv
```

Key points:
- `persistentVolumeReclaimPolicy: Retain` — data on NFS is kept even if PVC is deleted.
- `storageClassName: ""` — prevents the dynamic provisioner from intercepting the claim; forces binding to our static PV.
- `ReadWriteMany` — NFS supports multi-node access, so pods can move between nodes freely.

Verify binding:

```bash
kubectl get pv test-api-appdata-pv
kubectl get pvc test-api-appdata-pvc -n staging
# STATUS should be "Bound" for both
```

---

## 4. Update the Azure Pipeline Release File (Deployment)

Changes vs. the old template:
1. Added `strategy: Recreate`
2. Added `volumeMounts` in the container
3. Added `volumes` in the pod spec referencing the PVC

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: #{Deployment}#
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app:   #{Deployment}#
  template:
    metadata:
      labels:
        app:  #{Deployment}#
        type:  #{Deployment}#
    spec:
      containers:
      - name:  #{Deployment}#
        image: iboslimitedbd/#{Deployment}#:#{Build.BuildId}#
        resources:
          requests:
            cpu: "250m"
            memory: "64Mi"
          limits:
            cpu: "600m"
            memory: "1Gi"
        env:
        - name: #{AppEnvName}#
          value: #{AppEnvValue}#
        - name:  "ConnectionString"
          value:  #{ConnectionString}#
        - name: "REACT_APP_KEY_NAME"
          value: #{REACT_APP_KEY_NAME}#
        - name:  "REACT_APP_IV_NAME"
          value:  #{REACT_APP_IV_NAME}#
        - name:  "REACT_APP_SECRET_NAME"
          value:  #{REACT_APP_SECRET_NAME}#
        volumeMounts:
        - name: appdata
          mountPath: /app/App_Data
      volumes:
      - name: appdata
        persistentVolumeClaim:
          claimName: #{Deployment}#-appdata-pvc
      imagePullSecrets:
      - name: dockercred
---
apiVersion: v1
kind: Service
metadata:
  name: #{Deployment}#
spec:
  selector:
    app: #{Deployment}#
  ports:
  - port: 80
```

Why `Recreate` strategy: the old pod releases the volume before the new pod starts. With RollingUpdate, the new pod can get stuck waiting for the volume. Trade-off: a few seconds of downtime per deploy.

> Because the claim name is templated as `#{Deployment}#-appdata-pvc`, the same release file works for any app — just create a matching PV/PVC first (Section 3).

---

## 5. Migrating Existing Data (first deployment only)

**Important:** when the volume mounts over `/app/App_Data`, files baked into the container image at that path become hidden. Back up before redeploying.

1. Backup from the OLD running pod:
   ```bash
   kubectl cp staging/<old-pod-name>:/app/App_Data ./App_Data_backup
   ```
2. Deploy via the pipeline (new pod comes up with empty NFS volume).
3. Restore into the new pod (or copy directly into the NFS export directory on the server):
   ```bash
   kubectl cp ./App_Data_backup staging/<new-pod-name>:/app/App_Data
   ```

---

## 6. Verification

```bash
# 1. Confirm NFS is mounted inside the pod
kubectl exec -it <pod> -n staging -- df -h /app/App_Data
# Expected filesystem: 10.209.99.21:/data/<app-name>/app_data

# 2. Persistence test
kubectl exec -it <pod> -n staging -- touch /app/App_Data/persist-test.txt
kubectl rollout restart deploy <app-name> -n staging
kubectl exec -it <new-pod> -n staging -- ls /app/App_Data/persist-test.txt
# File should still exist → persistence confirmed
```

---

## 7. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `bad option; ... need a /sbin/mount.<type> helper program` | `nfs-common` not installed on the worker node | Install `nfs-common` (apt) / `nfs-utils` (yum) on ALL worker nodes |
| `access denied by server` | Export missing/wrong CIDR in `/etc/exports` | Fix `/etc/exports`, run `exportfs -ra` |
| `Permission denied` writing files | `root_squash` active on export | Add `no_root_squash` to the export options |
| PVC stuck `Pending` | `storageClassName` not set to `""`, or PV/PVC size/accessMode mismatch | Ensure `storageClassName: ""`, matching size and access modes |
| New pod stuck `ContainerCreating` after deploy | Volume held by old pod (RollingUpdate) | Use `strategy: Recreate`; delete the stuck pod |
| Connection timed out on mount | Firewall blocking NFS (port 2049) between nodes and NFS server | Open TCP/UDP 2049 (and 111 for rpcbind) |
| Old files missing after first deploy | Volume mount hid the image's App_Data content | Restore from backup (Section 5) |

---

## 8. Checklist for Onboarding a New App

- [ ] Create directory on NFS server: `/data/<app-name>/app_data`
- [ ] Add export to `/etc/exports` + `exportfs -ra`
- [ ] Confirm `nfs-common` installed on all worker nodes
- [ ] Apply PV + PVC manifest (`<app-name>-appdata-pv` / `<app-name>-appdata-pvc` in correct namespace)
- [ ] Verify PV/PVC `Bound`
- [ ] Back up existing pod data (if app already running)
- [ ] Release file already templated — just run the pipeline
- [ ] Restore backed-up data
- [ ] Run persistence test (Section 6)

---

## 9. Security Recommendation

Database credentials are currently passed as plain env vars in the release file (`ConnectionString`). Recommended improvement: store them in a Kubernetes Secret and reference via `secretKeyRef`, and rotate any password that has been exposed in pipeline logs or pod descriptions.

---

*Last updated: June 2026 — based on rdms-api (staging) implementation.*
