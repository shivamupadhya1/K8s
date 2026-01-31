
# 📦 Volumes, PersistentVolume (PV) & PersistentVolumeClaim (PVC)

> **One of the most important Kubernetes topics for interviews and real production.**

---

# ❓ Why Storage is Needed in Kubernetes

By default, containers are **ephemeral**.

If a pod restarts:

❌ Logs lost\
❌ Uploaded files lost\
❌ Database data lost

---

### Example

```text
Pod deleted → data gone
```

Kubernetes solves this using:

- Volumes
- PersistentVolume (PV)
- PersistentVolumeClaim (PVC)

---

# 🧩 Kubernetes Storage Architecture

```
Application
   ↓
Pod
   ↓
Volume
   ↓
PVC (request)
   ↓
PV (actual storage)
   ↓
Disk (EBS / NFS / Azure Disk)
```

---

# 📦 VOLUMES

## What is a Volume?

> A volume provides **storage inside a pod** that can be shared between containers.

---

## 🔹 emptyDir Volume

- Created when pod starts
- Deleted when pod dies

```yaml
volumes:
- name: cache
  emptyDir: {}
```

Used for:

- temporary files
- cache

---

## 🔹 hostPath Volume

Mounts node filesystem into pod.

```yaml
volumes:
- name: host-data
  hostPath:
    path: /data
```

⚠ Not recommended in production.

---

# 🧠 Problem with Normal Volumes

| Issue          | Reason             |
| -------------- | ------------------ |
| Pod restart    | Data lost          |
| Pod reschedule | New node = no data |
| No portability | Node dependent     |

👉 This is why **persistent storage** is needed.

---

# 💾 PersistentVolume (PV)

## What is PV?

> A PersistentVolume is a **cluster-level storage resource** created by administrators.

---

## Example PV

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-demo
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/data
```

---

## PV Access Modes

| Mode          | Meaning          |
| ------------- | ---------------- |
| ReadWriteOnce | One node         |
| ReadOnlyMany  | Many nodes read  |
| ReadWriteMany | Many nodes write |

---

# 🧾 PersistentVolumeClaim (PVC)

## What is PVC?

> PVC is a **request for storage by a pod**.

Pod never talks directly to PV.

---

## PVC Example

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-demo
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

---

# 🔗 Binding Process

```
PVC created
   ↓
Kubernetes searches PV
   ↓
Matching size + accessMode
   ↓
PVC bound to PV
```

---

# 🔗 Using PVC in Pod

```yaml
volumes:
- name: app-storage
  persistentVolumeClaim:
    claimName: pvc-demo

volumeMounts:
- name: app-storage
  mountPath: /data
```

---

# 🔄 Dynamic Provisioning (Production)

No manual PV creation required.

---

## StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
```

PVC automatically creates PV.

---

# 🔥 REAL PRODUCTION FLOW

```
Deployment
  ↓
PVC
  ↓
StorageClass
  ↓
Cloud Disk Created
  ↓
PV Bound
```

---

# 🧠 StatefulSet Always Uses PVC

Each pod gets its own volume:

```
mysql-0 → pvc-mysql-0
mysql-1 → pvc-mysql-1
```

---

# ⚠ Common Production Issues

| Issue             | Cause                |
| ----------------- | -------------------- |
| PVC Pending       | No matching PV       |
| Pod Pending       | PVC not bound        |
| Data lost         | reclaimPolicy=Delete |
| Permission denied | fsGroup missing      |

---

# ✅ Best Practices

✔ Always use StorageClass ✔ Avoid hostPath ✔ Use Retain for critical data ✔ Use PVC — never PV directly ✔ Backup volumes

---

# 🎯 Interview Golden Lines

> "Pods are ephemeral; volumes make data persistent. PV is the storage, PVC is the request, and StorageClass automates provisioning."

---

# ✅ FINAL SUMMARY

| Component    | Role              |
| ------------ | ----------------- |
| Volume       | Pod-level storage |
| PV           | Actual disk       |
| PVC          | Storage request   |
| StorageClass | Auto-provisioner  |

---

🚀 **End of Volumes, PV & PVC Notes**

