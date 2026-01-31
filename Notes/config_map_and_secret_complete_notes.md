# 📘 Kubernetes ConfigMap & Secret – Complete Detailed Notes

> **These notes cover everything you need to know about ConfigMap and Secret for interviews and real production usage.**

---

# 📑 Table of Contents

1. Why ConfigMap & Secret are needed
2. What is ConfigMap
3. ConfigMap use cases
4. Creating ConfigMaps
5. Using ConfigMap in Pods
6. ConfigMap update behavior
7. Common ConfigMap issues
8. What is Secret
9. Types of Secrets
10. Creating Secrets
11. Using Secrets in Pods
12. Docker registry secret
13. ConfigMap vs Secret comparison
14. Production best practices
15. Common interview questions

---

# 1️⃣ Why ConfigMap & Secret are Needed

Before Kubernetes configuration management:

```dockerfile
ENV DB_HOST=10.0.0.10
ENV DB_USER=root
ENV DB_PASS=admin123
```

### ❌ Problems

- Image rebuild required for config change
- Same image cannot be reused in dev/test/prod
- Credentials exposed in image
- Not cloud native

---

### ✅ Kubernetes Solution

| Object | Purpose |
|------|------|
| ConfigMap | Store non-sensitive configuration |
| Secret | Store sensitive configuration |

---

# 2️⃣ What is ConfigMap?

> A **ConfigMap** stores non-confidential configuration data as **key–value pairs**.

---

### Examples of data stored

- Database host
- Application port
- Environment name
- Logging level
- URLs

---

### ❌ Should NOT store

- Passwords
- API tokens
- Certificates

---

# 3️⃣ ConfigMap Use Cases

- Application configuration
- Feature flags
- Environment-specific values
- Externalized config files

---

# 4️⃣ Creating ConfigMaps

## Method 1: From literals

```bash
kubectl create configmap app-config \
  --from-literal=DB_HOST=mysql \
  --from-literal=DB_PORT=3306
```

---

## Method 2: From file

```bash
kubectl create configmap app-config \
  --from-file=application.properties
```

---

## Method 3: YAML (Recommended)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DB_HOST: mysql-service
  DB_PORT: "3306"
  LOG_LEVEL: INFO
```

---

# 5️⃣ Using ConfigMap in Pods

## Method 1: As environment variables

```yaml
envFrom:
- configMapRef:
    name: app-config
```

---

## Method 2: Individual key

```yaml
env:
- name: DB_HOST
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: DB_HOST
```

---

## Method 3: As volume (config file)

```yaml
volumes:
- name: config-volume
  configMap:
    name: app-config

volumeMounts:
- name: config-volume
  mountPath: /app/config
```

Creates files like:

```
/app/config/DB_HOST
/app/config/DB_PORT
```

---

# 6️⃣ ConfigMap Update Behavior

| Usage | Auto Update |
|------|------|
| Environment variable | ❌ No |
| Volume mount | ✅ Yes (30–60 sec) |

Pods must restart to reload env variables.

---

# 7️⃣ Common ConfigMap Issues

- ConfigMap not created
- Typo in key name
- Wrong configMapRef
- Expecting env auto reload

---

# 8️⃣ What is a Secret?

> A **Secret** stores sensitive information in **base64-encoded format**.

⚠ Base64 is **encoding, not encryption**.

---

### Used for

- Passwords
- Tokens
- API keys
- Certificates
- Docker credentials

---

# 9️⃣ Types of Secrets

| Type | Purpose |
|------|------|
| Opaque | Generic secrets |
| kubernetes.io/dockerconfigjson | Registry auth |
| kubernetes.io/tls | TLS certificates |
| service-account-token | Auto created |

---

# 🔟 Creating Secrets

## From literals

```bash
kubectl create secret generic db-secret \
  --from-literal=DB_USER=root \
  --from-literal=DB_PASS=admin123
```

---

## Using YAML

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  DB_USER: cm9vdA==
  DB_PASS: YWRtaW4xMjM=
```

Encode value:

```bash
echo -n admin123 | base64
```

---

# 1️⃣1️⃣ Using Secrets in Pods

## As environment variables

```yaml
envFrom:
- secretRef:
    name: db-secret
```

---

## Individual key

```yaml
env:
- name: DB_PASS
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: DB_PASS
```

---

## As volume

```yaml
volumes:
- name: secret-vol
  secret:
    secretName: db-secret

volumeMounts:
- name: secret-vol
  mountPath: /secrets
```

---

# 1️⃣2️⃣ Docker Registry Secret

```bash
kubectl create secret docker-registry regcred \
  --docker-server=nexus.company.com \
  --docker-username=admin \
  --docker-password=pass123 \
  --docker-email=dev@company.com
```

Use in deployment:

```yaml
imagePullSecrets:
- name: regcred
```

---

# 1️⃣3️⃣ ConfigMap vs Secret

| Feature | ConfigMap | Secret |
|------|------|------|
| Data type | Plain text | Base64 |
| Used for | Configuration | Credentials |
| Encrypted | ❌ | ⚠ Optional |
| Git safe | ✅ | ❌ |
| Access restricted | ❌ | ✅ |

---

# 1️⃣4️⃣ Production Best Practices

✅ Never store secrets in ConfigMap  
✅ Never commit secrets to Git  
✅ Separate config per environment  
✅ Use RBAC to restrict secret access  
✅ Restart pods after env change  
✅ Prefer external secret managers

---

# 1️⃣5️⃣ Common Interview Questions

### Q1. ConfigMap vs Secret?
> ConfigMap stores non-sensitive data; Secret stores sensitive data.

---

### Q2. Are secrets encrypted?
> By default they are base64 encoded. Encryption at rest must be enabled in etcd.

---

### Q3. Do ConfigMap changes apply automatically?
> Only when mounted as volumes, not as environment variables.

---

### Q4. Can multiple pods use same ConfigMap?
> Yes, ConfigMaps and Secrets can be shared across pods.

---

# 🎯 Interview Golden Line

> “ConfigMaps externalize application configuration, while Secrets secure sensitive data, allowing the same container image to run across all environments.”

---

# ✅ Final Summary

- ConfigMap → configuration
- Secret → credentials
- Both decouple config from image
- Mandatory for production Kubernetes

---

🚀 **End of ConfigMap & Secret Complete Notes**

