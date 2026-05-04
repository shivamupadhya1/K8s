# TUESDAY: Backup & Restore — Velero + VolumeSnapshot
### InfraThrone Study Notes — Complete with Hands-On Labs

---

## SECTION 1: WHY BACKUP IN KUBERNETES IS DIFFERENT

### The Problem with "Just Use etcd Backup"

Most newcomers think "etcd backup = cluster backup." That is dangerously wrong.

```
┌─────────────────────────────────────────────────────────────────────────┐
│              WHAT etcd HOLDS vs WHAT IT DOESN'T                        │
│                                                                         │
│  etcd HOLDS:                          etcd DOES NOT HOLD:              │
│  ──────────────────────────────        ──────────────────────────────   │
│  • Deployment manifests               • Actual container images        │
│  • ConfigMaps / Secrets               • Data inside PersistentVolumes  │
│  • Service definitions                • External DB data               │
│  • RBAC policies                      • Application state in memory    │
│  • Node registrations                 • Logs (unless in a PV)          │
│  • PVC *metadata* (not content)       • The PV content itself          │
│                                                                         │
│  CONCLUSION: etcd restore gives you the skeleton.                      │
│              Velero + Snapshots restore the flesh.                     │
└─────────────────────────────────────────────────────────────────────────┘
```

### The Three Layers of Kubernetes Data

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     THREE LAYERS TO BACK UP                            │
│                                                                         │
│  Layer 1 — Cluster State (etcd)                                        │
│  ─────────────────────────────                                         │
│  All Kubernetes API objects: Deployments, Services, Ingress, RBAC,     │
│  ConfigMaps, Secrets, etc.                                             │
│  Tool: Velero (resource backup) or etcdctl snapshot                    │
│                                                                         │
│  Layer 2 — Persistent Volume Data                                      │
│  ──────────────────────────────────                                    │
│  Actual bytes stored on disks attached to your pods.                   │
│  Tool: Velero with Restic/Kopia (file-level) OR CSI VolumeSnapshots   │
│                                                                         │
│  Layer 3 — External Dependencies                                       │
│  ─────────────────────────────────                                     │
│  Managed databases (RDS, CloudSQL), object storage, message queues.    │
│  Tool: Provider-native backup (RDS snapshots, etc.)                    │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## SECTION 2: VELERO ARCHITECTURE — DEEP DIVE

### What Velero Actually Is

Velero is a Kubernetes-native backup tool that:
1. Captures Kubernetes API objects (cluster state) as JSON/YAML
2. Optionally captures PVC data via file-level backup (Restic/Kopia) or snapshots
3. Stores everything in an object storage backend (S3, GCS, Azure Blob, MinIO)
4. Restores from that backend on demand

### Velero Component Architecture

```
┌───────────────────────────────────────────────────────────────────────────┐
│                        VELERO ARCHITECTURE                               │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │                     Kubernetes Cluster                              │ │
│  │                                                                     │ │
│  │  ┌──────────────────────────────────────────────────────────────┐  │ │
│  │  │  velero namespace                                            │  │ │
│  │  │                                                              │  │ │
│  │  │  ┌─────────────────┐    ┌──────────────────────────────┐    │  │ │
│  │  │  │  Velero Server  │    │  Node-Agent DaemonSet        │    │  │ │
│  │  │  │  (Deployment)   │    │  (Restic / Kopia per node)   │    │  │ │
│  │  │  │                 │    │                              │    │  │ │
│  │  │  │ • Backup logic  │    │ • Mounts host paths          │    │  │ │
│  │  │  │ • Restore logic │    │ • File-level PV backup       │    │  │ │
│  │  │  │ • Schedule mgmt │    │ • Streams to object storage  │    │  │ │
│  │  │  │ • Plugin loader │    │                              │    │  │ │
│  │  │  └────────┬────────┘    └──────────────┬───────────────┘    │  │ │
│  │  │           │                            │                     │  │ │
│  │  └───────────┼────────────────────────────┼─────────────────────┘  │ │
│  │              │                            │                         │ │
│  │              ▼                            ▼                         │ │
│  │  ┌─────────────────────────────────────────────────────────────┐   │ │
│  │  │  BackupStorageLocation CRD  │  VolumeSnapshotLocation CRD  │   │ │
│  │  └─────────────────────────────────────────────────────────────┘   │ │
│  │                                                                     │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                        │                           │                     │
│                        ▼                           ▼                     │
│              ┌──────────────────┐      ┌────────────────────────┐       │
│              │  Object Storage  │      │  Cloud Volume Snapshot │       │
│              │  (MinIO / S3 /   │      │  (EBS / GCE PD /       │       │
│              │   GCS / Blob)    │      │   Azure Disk / CSI)    │       │
│              └──────────────────┘      └────────────────────────┘       │
└───────────────────────────────────────────────────────────────────────────┘
```

### Velero Custom Resource Definitions (CRDs)

| CRD | Purpose |
|-----|---------|
| `Backup` | Defines a single backup operation |
| `Restore` | Defines a single restore operation |
| `Schedule` | Cron-based automated backup |
| `BackupStorageLocation` (BSL) | Where to store backup files |
| `VolumeSnapshotLocation` (VSL) | Where to store volume snapshots |
| `BackupRepository` | Tracks Restic/Kopia repositories |
| `DownloadRequest` | Request to download a backup file |
| `DeleteBackupRequest` | Request to delete a backup |

### Backup Flow — Step by Step

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     VELERO BACKUP FLOW                                 │
│                                                                         │
│  User: velero backup create my-backup --include-namespaces production  │
│                          │                                             │
│                          ▼                                             │
│  1. velero CLI creates a Backup CRD in the cluster                     │
│                          │                                             │
│                          ▼                                             │
│  2. Velero server watches for Backup CRDs → picks it up                │
│                          │                                             │
│                          ▼                                             │
│  3. Velero calls Kubernetes API:                                        │
│     GET all resources in selected namespaces/labels                    │
│     → Deployments, Services, PVCs, ConfigMaps, Secrets, etc.          │
│                          │                                             │
│                          ▼                                             │
│  4. Pre-backup hooks run (if annotated on pods)                        │
│     e.g., flush app caches, quiesce DB writes                         │
│                          │                                             │
│                          ▼                                             │
│  5a. PVC backup via Restic/Kopia:                                      │
│      Node-Agent pod on same node mounts the PVC volume                 │
│      → streams files to object storage                                 │
│                                                                         │
│  5b. OR: CSI VolumeSnapshot taken via snapshot plugin                  │
│      → Cloud provider creates disk snapshot                            │
│                          │                                             │
│                          ▼                                             │
│  6. Post-backup hooks run (e.g., resume DB writes)                     │
│                          │                                             │
│                          ▼                                             │
│  7. All resources + PV data uploaded to BackupStorageLocation          │
│     (MinIO bucket / S3 / GCS / Azure Blob)                            │
│                          │                                             │
│                          ▼                                             │
│  8. Backup CRD status → Completed / PartiallyFailed / Failed          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Restore Flow — Step by Step

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     VELERO RESTORE FLOW                                │
│                                                                         │
│  User: velero restore create --from-backup my-backup                   │
│                          │                                             │
│                          ▼                                             │
│  1. Restore CRD created in cluster                                     │
│                          │                                             │
│                          ▼                                             │
│  2. Velero downloads backup tarball from BSL                           │
│     (JSON files for each resource)                                     │
│                          │                                             │
│                          ▼                                             │
│  3. Velero applies resources to cluster via kubectl apply              │
│     • Respects existing resources (skip by default)                    │
│     • Can use --existing-resource-policy=update to overwrite          │
│                          │                                             │
│                          ▼                                             │
│  4a. PVC restore via Restic/Kopia:                                     │
│      New PVC created → Node-Agent streams files back from BSL          │
│                                                                         │
│  4b. OR: CSI VolumeSnapshot restore:                                   │
│      PVC created from VolumeSnapshot → cloud restores disk             │
│                          │                                             │
│                          ▼                                             │
│  5. Post-restore hooks run                                             │
│                          │                                             │
│                          ▼                                             │
│  6. Restore status → Completed / PartiallyFailed / Failed             │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## SECTION 3: PVC BACKUPS vs VOLUME SNAPSHOTS

### The Core Difference

```
┌──────────────────────────────────────────────────────────────────────────┐
│          VELERO RESTIC/KOPIA (File Backup) vs CSI SNAPSHOTS             │
│                                                                          │
│  ┌───────────────────────────────┐  ┌──────────────────────────────┐   │
│  │    Velero + Restic/Kopia      │  │    CSI VolumeSnapshot        │   │
│  │    (File-level backup)        │  │    (Block-level snapshot)    │   │
│  ├───────────────────────────────┤  ├──────────────────────────────┤   │
│  │ HOW:                          │  │ HOW:                         │   │
│  │ Node-Agent mounts PVC, reads  │  │ CSI driver calls cloud API   │   │
│  │ all files, compresses, sends  │  │ to snapshot the disk         │   │
│  │ to object storage             │  │ at block level               │   │
│  ├───────────────────────────────┤  ├──────────────────────────────┤   │
│  │ SPEED: Slow (file traversal)  │  │ SPEED: Fast (copy-on-write)  │   │
│  ├───────────────────────────────┤  ├──────────────────────────────┤   │
│  │ STORAGE: Object storage       │  │ STORAGE: Cloud snapshot      │   │
│  │ (cheap, portable)             │  │ service (provider-tied)      │   │
│  ├───────────────────────────────┤  ├──────────────────────────────┤   │
│  │ CROSS-CLUSTER: YES            │  │ CROSS-CLUSTER: HARD          │   │
│  │ (files in S3/MinIO)           │  │ (snapshot stays in region)   │   │
│  ├───────────────────────────────┤  ├──────────────────────────────┤   │
│  │ APP-CONSISTENT: With hooks    │  │ APP-CONSISTENT: With hooks   │   │
│  │ (pre/post backup hooks)       │  │ (pre/post backup hooks)      │   │
│  ├───────────────────────────────┤  ├──────────────────────────────┤   │
│  │ USE WHEN:                     │  │ USE WHEN:                    │   │
│  │ • Cross-cluster migration     │  │ • Large volumes (TBs)        │   │
│  │ • Cloud-agnostic backup       │  │ • Low RPO on same cloud      │   │
│  │ • Small to medium volumes     │  │ • Need fastest restore       │   │
│  └───────────────────────────────┘  └──────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────┘
```

### CSI VolumeSnapshot API Objects

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   CSI VOLUMESNAPSHOT API CHAIN                         │
│                                                                         │
│  VolumeSnapshotClass                                                    │
│  ─────────────────────                                                 │
│  • Defines the CSI driver to use                                        │
│  • Sets deletion policy (Delete vs Retain)                             │
│  • Like StorageClass but for snapshots                                 │
│                                                                         │
│         │  referenced by                                               │
│         ▼                                                               │
│  VolumeSnapshot  (namespaced)                                           │
│  ─────────────────────────────                                         │
│  • User-facing object: "take a snapshot of this PVC"                   │
│  • References: VolumeSnapshotClass + source PVC                        │
│  • Like PVC but for snapshots                                           │
│                                                                         │
│         │  creates                                                      │
│         ▼                                                               │
│  VolumeSnapshotContent  (cluster-scoped)                                │
│  ──────────────────────────────────────                                │
│  • The actual snapshot resource on the backend                         │
│  • Stores the snapshotHandle (cloud snapshot ID)                       │
│  • Like PersistentVolume but for snapshots                             │
│                                                                         │
│  RESTORE: Create new PVC with dataSource pointing to VolumeSnapshot    │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## SECTION 4: CROSS-REGION AND CROSS-CLUSTER RECOVERY

### Velero Cross-Cluster Recovery Model

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    CROSS-CLUSTER RECOVERY FLOW                            │
│                                                                            │
│  CLUSTER A (prod, us-east-1)              CLUSTER B (dr, us-west-2)       │
│  ───────────────────────────              ────────────────────────────     │
│                                                                            │
│  velero backup create prod-backup         velero restore create            │
│      --include-namespaces app             --from-backup prod-backup        │
│                │                                      ▲                   │
│                │                                      │                   │
│                ▼                                      │                   │
│         ┌─────────────────────┐                       │                   │
│         │   Shared S3 Bucket  │ ──────────────────────┘                   │
│         │   (or cross-region  │                                            │
│         │    replication)     │                                            │
│         └─────────────────────┘                                            │
│                                                                            │
│  REQUIREMENTS:                                                             │
│  • Both clusters have same BSL (or replicated bucket)                     │
│  • Cluster B has Velero installed with same plugins                       │
│  • StorageClasses with same name must exist in Cluster B                  │
│  • Secrets must be available (or recreated)                               │
│                                                                            │
│  GOTCHAS:                                                                  │
│  • Node Selectors / Taints may differ → pods may not schedule             │
│  • LoadBalancer IPs will be different                                      │
│  • PVC StorageClass must exist in target cluster                          │
│  • NamespaceMapping lets you rename: --namespace-mappings old:new         │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## SECTION 5: BACKUP STRATEGY — RTO, RPO, AND SCHEDULE

### Key Metrics

| Metric | Definition | Velero Lever |
|--------|-----------|-------------|
| **RPO** (Recovery Point Objective) | Max data loss tolerated | Backup schedule frequency |
| **RTO** (Recovery Time Objective) | Max downtime tolerated | Restore speed, runbooks |
| **Retention** | How long to keep backups | `--ttl` flag on backup |

### Backup Schedule Patterns

```yaml
# Schedule: every 6 hours, keep for 72 hours
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: production-6h
  namespace: velero
spec:
  schedule: "0 */6 * * *"         # every 6 hours
  template:
    ttl: 72h0m0s                  # keep for 3 days
    includedNamespaces:
      - production
      - staging
    storageLocation: default
    volumeSnapshotLocations:
      - default
```

---

## SECTION 6: HANDS-ON LABS

---

### LAB 1: Install Velero with MinIO Backend

**Goal:** Install MinIO as a local S3-compatible backend and install Velero in your cluster.

#### Step 1 — Deploy MinIO in the cluster

```yaml
# minio-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio
  namespace: velero
  labels:
    app: minio
spec:
  replicas: 1
  selector:
    matchLabels:
      app: minio
  template:
    metadata:
      labels:
        app: minio
    spec:
      containers:
        - name: minio
          image: minio/minio:latest
          args:
            - server
            - /data
            - --console-address
            - ":9001"
          env:
            - name: MINIO_ROOT_USER
              value: "minio"
            - name: MINIO_ROOT_PASSWORD
              value: "minio123"
          ports:
            - containerPort: 9000   # API
            - containerPort: 9001   # Console
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          emptyDir: {}    # use PVC in production
---
apiVersion: v1
kind: Service
metadata:
  name: minio
  namespace: velero
spec:
  selector:
    app: minio
  ports:
    - name: api
      port: 9000
      targetPort: 9000
    - name: console
      port: 9001
      targetPort: 9001
```

```bash
# Create velero namespace and deploy MinIO
kubectl create namespace velero
kubectl apply -f minio-deployment.yaml

# Wait for MinIO to be ready
kubectl rollout status deployment/minio -n velero

# Port-forward to create bucket via console
kubectl port-forward svc/minio 9001:9001 -n velero &
# Open http://localhost:9001 → login minio/minio123 → create bucket "velero"
```

#### Step 2 — Create MinIO credentials file

```bash
cat > /tmp/minio-credentials <<EOF
[default]
aws_access_key_id=minio
aws_secret_access_key=minio123
EOF
```

#### Step 3 — Install Velero CLI

```bash
# Linux / macOS
VELERO_VERSION=v1.13.2
curl -L https://github.com/vmware-tanzu/velero/releases/download/${VELERO_VERSION}/velero-${VELERO_VERSION}-linux-amd64.tar.gz \
  | tar -xz
sudo mv velero-${VELERO_VERSION}-linux-amd64/velero /usr/local/bin/

# Verify
velero version --client-only
```

#### Step 4 — Install Velero in cluster (with MinIO + Restic)

```bash
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.9.0 \
  --bucket velero \
  --secret-file /tmp/minio-credentials \
  --use-volume-snapshots=false \
  --backup-location-config \
      region=minio,s3ForcePathStyle=true,s3Url=http://minio.velero.svc:9000 \
  --use-node-agent \
  --default-volumes-to-fs-backup
```

**Flag breakdown:**

| Flag | Meaning |
|------|---------|
| `--provider aws` | Use AWS-compatible S3 plugin |
| `--use-volume-snapshots=false` | No cloud snapshots (using Restic) |
| `--use-node-agent` | Install Restic/Kopia DaemonSet |
| `--default-volumes-to-fs-backup` | All PVCs backed up via file system |

#### Step 5 — Verify installation

```bash
# All components should be Running
kubectl get all -n velero

# BSL should be Available
velero backup-location get

# Expected output:
# NAME      PROVIDER   BUCKET/PREFIX   PHASE       LAST VALIDATED
# default   aws        velero          Available   ...
```

---

### LAB 2: Take a Namespace Backup

**Goal:** Back up a complete namespace including Deployments, Services, PVCs, and ConfigMaps.

#### Step 1 — Create a sample application to back up

```yaml
# demo-app.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo-app
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: demo-app
data:
  DATABASE_HOST: "postgres.demo-app.svc"
  APP_ENV: "production"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
  namespace: demo-app
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: standard
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: demo-app
  labels:
    app: webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
      annotations:
        # Velero backup hooks — flush data before backup
        pre.hook.backup.velero.io/command: '["/bin/sh", "-c", "sync"]'
        pre.hook.backup.velero.io/timeout: "30s"
    spec:
      containers:
        - name: webapp
          image: nginx:1.25
          ports:
            - containerPort: 80
          volumeMounts:
            - name: data
              mountPath: /usr/share/nginx/html/data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: app-data
---
apiVersion: v1
kind: Service
metadata:
  name: webapp-svc
  namespace: demo-app
spec:
  selector:
    app: webapp
  ports:
    - port: 80
      targetPort: 80
```

```bash
kubectl apply -f demo-app.yaml

# Write some data to the PVC to verify restore later
POD=$(kubectl get pod -n demo-app -l app=webapp -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n demo-app $POD -- \
  sh -c 'echo "BACKUP TEST DATA - $(date)" > /usr/share/nginx/html/data/test.txt'

# Verify the data
kubectl exec -n demo-app $POD -- cat /usr/share/nginx/html/data/test.txt
```

#### Step 2 — Take the backup

```bash
velero backup create demo-backup \
  --include-namespaces demo-app \
  --default-volumes-to-fs-backup \
  --wait

# Check backup status
velero backup describe demo-backup --details

# Expected: Phase: Completed
# You should see PVC backup progress under "Restic Backups"
```

#### Step 3 — Verify backup in MinIO

```bash
# List objects in MinIO bucket
kubectl exec -n velero deploy/minio -- \
  sh -c 'ls /data/velero/backups/demo-backup/'

# You should see:
# demo-backup-csi-volumesnapshots.json.gz
# demo-backup-logs.gz
# demo-backup-podvolumebackups.json.gz
# demo-backup-resource-list.json.gz
# demo-backup-volumesnapshots.json.gz
# demo-backup.tar.gz    ← all Kubernetes manifests
# velero-backup.json    ← Backup metadata
```

---

### LAB 3: Delete the Namespace → Restore It

**Goal:** Simulate accidental namespace deletion and restore it completely, including PVC data.

#### Step 1 — Simulate the disaster

```bash
# CONFIRM current state before deletion
kubectl get all,pvc,configmap -n demo-app

# SIMULATE DISASTER — delete the entire namespace
kubectl delete namespace demo-app

# Verify it is gone
kubectl get namespace demo-app
# Error from server (NotFound): namespaces "demo-app" not found
```

#### Step 2 — Restore from backup

```bash
velero restore create demo-restore \
  --from-backup demo-backup \
  --wait

# Monitor restore progress
velero restore describe demo-restore --details
```

#### Step 3 — Verify the restore

```bash
# Check namespace is back
kubectl get namespace demo-app

# Check all resources restored
kubectl get all,pvc,configmap -n demo-app

# Wait for pods to be running
kubectl rollout status deployment/webapp -n demo-app

# CRITICAL: Verify PVC data returned
POD=$(kubectl get pod -n demo-app -l app=webapp -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n demo-app $POD -- cat /usr/share/nginx/html/data/test.txt

# You should see the SAME line you wrote before deletion:
# BACKUP TEST DATA - <original timestamp>
```

#### Step 4 — Partial restore (single resource)

```bash
# Restore only the ConfigMap (not everything)
velero restore create restore-configmap-only \
  --from-backup demo-backup \
  --include-resources configmaps \
  --wait

# Restore a specific resource by name
velero restore create restore-pvc-only \
  --from-backup demo-backup \
  --include-resources persistentvolumeclaims \
  --selector "app=webapp" \
  --wait
```

#### Restore Flags Cheat Sheet

```bash
# Restore to a different namespace
velero restore create \
  --from-backup demo-backup \
  --namespace-mappings demo-app:demo-app-restored

# Restore including cluster-scoped resources (RBAC)
velero restore create \
  --from-backup demo-backup \
  --include-cluster-resources=true

# Overwrite existing resources (instead of skipping)
velero restore create \
  --from-backup demo-backup \
  --existing-resource-policy=update

# Exclude secrets from restore
velero restore create \
  --from-backup demo-backup \
  --exclude-resources secrets
```

---

### LAB 4: Create a VolumeSnapshot and Restore the PVC Individually

**Goal:** Use the CSI VolumeSnapshot API to snapshot a PVC, delete it, and restore it without Velero.

#### Step 1 — Ensure CSI snapshot controller is installed

```bash
# Check if snapshot CRDs exist
kubectl get crd | grep snapshot

# Expected:
# volumesnapshotclasses.snapshot.storage.k8s.io
# volumesnapshotcontents.snapshot.storage.k8s.io
# volumesnapshots.snapshot.storage.k8s.io

# If NOT present (local cluster like kind/minikube), install:
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshotclasses.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshotcontents.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshots.yaml

# Install snapshot controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/deploy/kubernetes/snapshot-controller/rbac-snapshot-controller.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/deploy/kubernetes/snapshot-controller/setup-snapshot-controller.yaml
```

#### Step 2 — Create a VolumeSnapshotClass

```yaml
# snapshot-class.yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-snapclass
  annotations:
    # Mark as default snapshot class
    snapshot.storage.kubernetes.io/is-default-class: "true"
driver: hostpath.csi.k8s.io    # Change to your CSI driver:
                                 # ebs.csi.aws.com  (EKS)
                                 # pd.csi.storage.gke.io  (GKE)
                                 # disk.csi.azure.com  (AKS)
deletionPolicy: Retain          # Retain or Delete
```

```bash
kubectl apply -f snapshot-class.yaml
kubectl get volumesnapshotclass
```

#### Step 3 — Write data to a PVC, then snapshot it

```bash
# Reuse the demo-app namespace from Lab 2 (or recreate it)
# Write important data
POD=$(kubectl get pod -n demo-app -l app=webapp -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n demo-app $POD -- \
  sh -c 'echo "SNAPSHOT DATA V1 - $(date)" > /usr/share/nginx/html/data/snap-test.txt'
```

```yaml
# volume-snapshot.yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: app-data-snap-v1
  namespace: demo-app
spec:
  volumeSnapshotClassName: csi-snapclass
  source:
    persistentVolumeClaimName: app-data   # the PVC to snapshot
```

```bash
kubectl apply -f volume-snapshot.yaml

# Monitor snapshot readiness
kubectl get volumesnapshot -n demo-app -w

# When READYTOUSE = true, it is done:
# NAME               READYTOUSE   SOURCEPVC   RESTORESIZE   SNAPSHOTCONTENT
# app-data-snap-v1   true         app-data    1Gi           snapcontent-xxx
```

#### Step 4 — Check the VolumeSnapshotContent

```bash
# VolumeSnapshotContent is cluster-scoped
kubectl get volumesnapshotcontent

# Describe to see the actual snapshot handle (cloud snapshot ID)
kubectl describe volumesnapshotcontent snapcontent-<uid>

# Key fields:
# Status.SnapshotHandle: snap-xxxxxxxx   (actual cloud resource ID)
# Status.ReadyToUse: true
# Status.RestoreSize: 1073741824
```

#### Step 5 — Simulate PVC corruption / deletion

```bash
# Write "corrupt" data to simulate corruption
kubectl exec -n demo-app $POD -- \
  sh -c 'echo "CORRUPTED DATA" > /usr/share/nginx/html/data/snap-test.txt'
kubectl exec -n demo-app $POD -- cat /usr/share/nginx/html/data/snap-test.txt

# Or delete the PVC entirely (must delete pod first)
kubectl scale deployment webapp -n demo-app --replicas=0
kubectl delete pvc app-data -n demo-app
```

#### Step 6 — Restore PVC from VolumeSnapshot

```yaml
# pvc-from-snapshot.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data                # same name (or new name)
  namespace: demo-app
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: standard
  dataSource:                   # ← this is the restore mechanism
    name: app-data-snap-v1
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```

```bash
kubectl apply -f pvc-from-snapshot.yaml

# Restore the deployment
kubectl scale deployment webapp -n demo-app --replicas=2
kubectl rollout status deployment/webapp -n demo-app

# Verify the pre-corruption data is back
POD=$(kubectl get pod -n demo-app -l app=webapp -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n demo-app $POD -- cat /usr/share/nginx/html/data/snap-test.txt

# Expected: SNAPSHOT DATA V1 - <original timestamp>
# NOT "CORRUPTED DATA"
```

---

## SECTION 7: SCHEDULED BACKUPS AND BACKUP MANAGEMENT

### Create a Backup Schedule

```bash
# Backup every day at 2 AM, retain for 7 days
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --ttl 168h0m0s \
  --include-namespaces production,staging \
  --default-volumes-to-fs-backup

# List schedules
velero schedule get

# Manually trigger a scheduled backup now
velero backup create --from-schedule daily-backup
```

### Managing Backups

```bash
# List all backups
velero backup get

# Describe a backup with full details
velero backup describe my-backup --details

# Download backup logs
velero backup logs my-backup

# Delete a backup (removes from storage too)
velero backup delete my-backup

# Get restore history
velero restore get

# Delete a failed restore
velero restore delete failed-restore-1
```

---

## SECTION 8: BACKUP HOOKS — APPLICATION-CONSISTENT BACKUPS

### Why Hooks Matter

Without hooks, Velero snapshots the PVC while the application may still be writing data. This can result in a torn backup (data halfway through a write). Hooks let you quiesce the application before backup.

### Hook Annotations on Pod Templates

```yaml
# In your Deployment spec.template.metadata.annotations:
annotations:
  # Pre-backup: runs BEFORE the backup starts
  pre.hook.backup.velero.io/command: '["/bin/sh", "-c", "redis-cli BGSAVE && sleep 5"]'
  pre.hook.backup.velero.io/container: redis
  pre.hook.backup.velero.io/on-error: Fail     # Fail or Continue
  pre.hook.backup.velero.io/timeout: 60s

  # Post-backup: runs AFTER the backup completes
  post.hook.backup.velero.io/command: '["/bin/sh", "-c", "echo Backup done"]'
  post.hook.backup.velero.io/container: redis
  post.hook.backup.velero.io/timeout: 30s
```

### Real-world Hook Examples

```yaml
# PostgreSQL: freeze writes, backup, unfreeze
pre.hook.backup.velero.io/command: >
  ["/bin/sh", "-c",
   "psql -U postgres -c 'CHECKPOINT; SELECT pg_start_backup(''velero'');'"]

post.hook.backup.velero.io/command: >
  ["/bin/sh", "-c",
   "psql -U postgres -c 'SELECT pg_stop_backup();'"]

# MySQL: flush and lock tables
pre.hook.backup.velero.io/command: >
  ["/bin/sh", "-c",
   "mysql -u root -p${MYSQL_ROOT_PASSWORD} -e 'FLUSH TABLES WITH READ LOCK;'"]

post.hook.backup.velero.io/command: >
  ["/bin/sh", "-c",
   "mysql -u root -p${MYSQL_ROOT_PASSWORD} -e 'UNLOCK TABLES;'"]
```

---

## SECTION 9: TROUBLESHOOTING BACKUPS

### Backup Stuck in "InProgress"

```bash
# Check Velero server logs
kubectl logs -n velero deploy/velero -f

# Check node-agent (Restic) logs
kubectl logs -n velero daemonset/node-agent -f

# Check specific PodVolumeBackup objects
kubectl get podvolumebackup -n velero

# Force-delete a stuck backup (last resort)
kubectl patch backup my-backup -n velero \
  --type=merge -p '{"status":{"phase":"Failed"}}'
```

### Restore Fails with "already exists"

```bash
# By default Velero skips existing resources
# To overwrite:
velero restore create --from-backup my-backup \
  --existing-resource-policy=update

# To restore to a fresh namespace:
kubectl delete namespace demo-app   # if safe
velero restore create --from-backup my-backup \
  --include-namespaces demo-app
```

### PVC Data Not Restored

```bash
# Check PodVolumeRestore objects
kubectl get podvolumerestore -n velero

# Check if annotation is present on the backup
velero backup describe my-backup --details | grep -A5 "PV Backups"

# Common mistake: forgot --default-volumes-to-fs-backup on backup
# The backup will have no PV data at all — you must re-backup
```

### BackupStorageLocation Not Available

```bash
velero backup-location get

# If PHASE is Unavailable:
# 1. Check BSL credentials secret
kubectl get secret -n velero cloud-credentials -o yaml

# 2. Check connectivity to MinIO/S3
kubectl exec -n velero deploy/velero -- \
  curl -v http://minio.velero.svc:9000

# 3. Check BSL config
kubectl describe backupstoragelocation default -n velero
```

---

## SECTION 10: QUICK REFERENCE CHEAT SHEET

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    VELERO COMMAND CHEAT SHEET                          │
├─────────────────────────────────────────────────────────────────────────┤
│  BACKUPS                                                                │
│  velero backup create NAME --include-namespaces NS                     │
│  velero backup create NAME --selector app=myapp                        │
│  velero backup create NAME --default-volumes-to-fs-backup              │
│  velero backup create NAME --ttl 72h --wait                            │
│  velero backup get                                                      │
│  velero backup describe NAME --details                                  │
│  velero backup logs NAME                                                │
│  velero backup delete NAME                                              │
│                                                                         │
│  RESTORES                                                               │
│  velero restore create --from-backup NAME                              │
│  velero restore create --from-backup NAME --namespace-mappings old:new │
│  velero restore create --from-backup NAME --include-resources deploy   │
│  velero restore get                                                     │
│  velero restore describe NAME --details                                 │
│  velero restore logs NAME                                               │
│                                                                         │
│  SCHEDULES                                                              │
│  velero schedule create NAME --schedule="0 2 * * *" --ttl 168h        │
│  velero schedule get                                                    │
│  velero backup create --from-schedule SCHEDULE_NAME                    │
│                                                                         │
│  STATUS                                                                 │
│  velero backup-location get                                             │
│  velero version                                                         │
│  kubectl get all -n velero                                              │
│  kubectl get backup,restore,schedule -n velero                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## SECTION 11: INFRA MINDSET — BACKUP vs DR vs HA

```
┌─────────────────────────────────────────────────────────────────────────┐
│              BACKUP vs HA vs DR — WHAT THEY SOLVE                      │
│                                                                         │
│  HIGH AVAILABILITY (HA)                                                 │
│  ──────────────────────                                                │
│  • Multiple replicas, anti-affinity, PDB                               │
│  • Survives: pod crash, node failure, rolling updates                  │
│  • Does NOT protect: logical corruption, accidental deletion,          │
│    ransomware, config mistake applied to all replicas                  │
│  • RTO: seconds  |  RPO: zero (no data loss)                           │
│                                                                         │
│  BACKUP (Velero / Snapshots)                                            │
│  ────────────────────────────                                          │
│  • Point-in-time copy of state + data                                  │
│  • Survives: accidental deletion, config corruption, data corruption   │
│  • Does NOT protect: if backup itself is corrupted/deleted             │
│  • RTO: minutes to hours  |  RPO: time since last backup               │
│                                                                         │
│  DISASTER RECOVERY (DR)                                                 │
│  ────────────────────────                                              │
│  • Full runbook: backup + restore procedure + dependency recovery       │
│  • Includes: DNS failover, database promotion, external services       │
│  • Survives: complete region outage, full cluster loss                 │
│  • RTO: hours  |  RPO: depends on backup frequency + replication      │
│                                                                         │
│  KEY INSIGHT:                                                           │
│  Backup ≠ DR. Having Velero backups in the SAME region that failed    │
│  is NOT DR. True DR = backups in different region/account + tested    │
│  runbook + rehearsed restore + dependency recovery plan.               │
└─────────────────────────────────────────────────────────────────────────┘
```

### Snapshot Consistency Warning

```
┌─────────────────────────────────────────────────────────────────────────┐
│           SNAPSHOT CONSISTENCY: THE HIDDEN PROBLEM                     │
│                                                                         │
│  A snapshot taken while the app is running may capture:                │
│  • Write halfway through a transaction                                  │
│  • Index structures in an inconsistent state                           │
│  • Buffer pool contents not yet flushed to disk                        │
│                                                                         │
│  For file systems (ext4, xfs):  use pre-backup hook to run sync/fsfreeze
│  For databases (Postgres, MySQL): use pre-backup hook to checkpoint    │
│  For Redis: use pre-backup hook to trigger BGSAVE                      │
│                                                                         │
│  RULE: Any stateful workload needs backup hooks to be                  │
│        application-consistent. Raw snapshots are crash-consistent      │
│        (like pulling the power cord) — the DB will recover,           │
│        but you may lose the last N transactions.                       │
└─────────────────────────────────────────────────────────────────────────┘
```

### Always Test Recovery

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   THE GOLDEN RULE OF BACKUPS                           │
│                                                                         │
│  "AN UNTESTED BACKUP IS NOT A BACKUP. IT IS HOPE."                     │
│                                                                         │
│  Backup Testing Runbook:                                               │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ 1. Schedule: restore drill every 30 days minimum                 │  │
│  │ 2. Target: restore to isolated test namespace / cluster          │  │
│  │ 3. Verify: application starts AND data is correct               │  │
│  │ 4. Measure: record actual RTO (how long did restore take?)      │  │
│  │ 5. Document: update runbook with actual steps + gotchas         │  │
│  │ 6. Alert: alarm if scheduled backup fails or BSL is unavailable │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  Common failure modes discovered only during restore drills:           │
│  • Backup completed but PVC data was silently skipped                  │
│  • StorageClass name differs in DR cluster                             │
│  • Restore succeeds but app can't connect to external DB              │
│  • Secrets restored but encryption keys are wrong environment         │
│  • RTO was 4 hours, not the assumed 30 minutes                        │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## SECTION 12: EXAM-STYLE Q&A

**Q: What does Velero NOT back up by default?**
A: PVC data. Kubernetes API objects (manifests) are backed up. PVC data requires `--default-volumes-to-fs-backup` (Restic/Kopia) or a VolumeSnapshotLocation with CSI snapshots.

**Q: What is the difference between VolumeSnapshot and VolumeSnapshotContent?**
A: `VolumeSnapshot` is a namespaced, user-facing request (like PVC). `VolumeSnapshotContent` is the cluster-scoped actual snapshot resource (like PV). The snapshot controller binds them.

**Q: How do you restore a backup to a different namespace?**
A: `velero restore create --from-backup NAME --namespace-mappings source-ns:target-ns`

**Q: Backup is Completed but restore has no PVC data. Why?**
A: The backup was taken without `--default-volumes-to-fs-backup` and no VolumeSnapshotLocation was configured. The PVC metadata was backed up but not the data.

**Q: What is `deletionPolicy: Retain` on VolumeSnapshotClass?**
A: When the `VolumeSnapshot` object is deleted, the underlying `VolumeSnapshotContent` (and the actual cloud snapshot) is NOT deleted. Use `Delete` to auto-cleanup. `Retain` is safer for production.

**Q: When should you use CSI snapshots over Restic?**
A: Large volumes (TBs), need fastest restore, staying within same cloud provider. Restic is better for cross-cluster, cloud-agnostic, or small volumes.

**Q: How do you make a backup application-consistent for PostgreSQL?**
A: Add pre-backup hook annotation to the pod: run `CHECKPOINT; SELECT pg_start_backup(...)` before backup and `SELECT pg_stop_backup()` post-backup.

---

*Study Notes — TUESDAY: Backup & Restore with Velero + VolumeSnapshot*
*InfraThrone Mindset: Backup is not DR. Test your restore. Snapshots are crash-consistent by default.*
