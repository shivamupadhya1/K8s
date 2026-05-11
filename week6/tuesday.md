# Tuesday: SBOM Generation + Vulnerability Scanning + Image Signing

> **Goal:** Master the three pillars of container security — know what's inside your image (SBOM), find what's broken (scanning), and prove it hasn't been tampered with (signing). This is shift-left security in practice.

---

## Table of Contents

1. [Core Concepts](#1-core-concepts)
2. [The Three Pillars of Container Security](#2-the-three-pillars-of-container-security)
3. [Key Terminology](#3-key-terminology)
4. [Environment Setup](#4-environment-setup)
5. [Lab 1 – SBOM Generation with Syft](#5-lab-1--sbom-generation-with-syft)
6. [Lab 2 – Vulnerability Scanning with Trivy](#6-lab-2--vulnerability-scanning-with-trivy)
7. [Lab 3 – Image Signing with Cosign](#7-lab-3--image-signing-with-cosign)
8. [Lab 4 – Full Pipeline Integration](#8-lab-4--full-pipeline-integration)
9. [Observation Checklist](#9-observation-checklist)
10. [Common Errors & Fixes](#10-common-errors--fixes)
11. [Tools Comparison](#11-tools-comparison)
12. [Revision Questions](#12-revision-questions)
13. [Key Takeaways](#13-key-takeaways)

---

## 1. Core Concepts

### What is Shift-Left Security?
Traditional security checked code **after** deployment. Shift-left moves security checks **earlier** in the pipeline — at build time, not after release.

```
Traditional (Shift-Right):
  Code → Build → Deploy → [Security Check] → Production
                                               ↑
                                          Too late!

Shift-Left (Modern DevSecOps):
  Code → [SBOM] → [Scan] → [Sign] → Build → Deploy → Production
          ↑          ↑        ↑
     What's in    CVEs    Trusted?
      my image?   found!
```

### Why These Three Tools Together?
| Question | Tool | Answer |
|----------|------|--------|
| "What is inside my image?" | **Syft** (SBOM) | Full inventory of packages, libraries, OS components |
| "What vulnerabilities exist?" | **Trivy** (Scanner) | CVE list with severity ratings |
| "Can I trust this image?" | **Cosign** (Signing) | Cryptographic proof of authenticity |

### The Supply Chain Attack Problem
Without these tools:
1. You build an image — but what's actually in it?
2. You pull an image from a registry — was it tampered with?
3. A dependency gets compromised — would you know?

These three tools together close those gaps.

---

## 2. The Three Pillars of Container Security

### Pillar 1: SBOM (Software Bill of Materials)
An SBOM is a **complete inventory** of every component in your software — like a nutrition label for your container image.

```
coreops-app:1.0
├── python 3.9.18
├── pip 23.0.1
├── libc6 2.31-13
├── libssl1.1 1.1.1w-0
├── ca-certificates 20210119
└── ... (50+ more components)
```

**Why it matters:**
- Regulators (US Executive Order 14028) now require SBOMs for software sold to the US government
- When a new CVE drops, you can instantly check if you're affected
- Auditors can verify your software supply chain

### Pillar 2: Vulnerability Scanning
Scanning compares your SBOM against **CVE databases** (NVD, OSV, GitHub Advisory) to find known vulnerabilities.

```
CVE-2023-1234  python:3.9.18  HIGH    Remote Code Execution
CVE-2023-5678  libssl:1.1.1w  CRITICAL  Buffer Overflow
```

**Severity Levels (CVSS Score):**
| Severity | Score Range | Action |
|----------|-------------|--------|
| CRITICAL | 9.0 – 10.0 | Block pipeline immediately |
| HIGH | 7.0 – 8.9 | Fix before merging |
| MEDIUM | 4.0 – 6.9 | Fix within sprint |
| LOW | 0.1 – 3.9 | Track and monitor |
| NONE | 0.0 | Informational only |

### Pillar 3: Image Signing
Signing uses **asymmetric cryptography** to attach a digital signature to your image. Anyone can verify the image came from you and was not tampered with.

```
Your CI pipeline:
  Build image → Sign with private key → Push to registry

Consumer/Deployment:
  Pull image → Verify with public key → Deploy (or reject)
```

---

## 3. Key Terminology

| Term | Definition |
|------|-----------|
| **SBOM** | Software Bill of Materials — inventory of all components in a software artifact |
| **CVE** | Common Vulnerabilities and Exposures — standardized ID for known vulnerabilities |
| **CVSS** | Common Vulnerability Scoring System — 0–10 severity score |
| **NVD** | National Vulnerability Database — US NIST database of CVEs |
| **Syft** | Open-source SBOM generator by Anchore |
| **Grype** | Anchore's vulnerability scanner (pairs with Syft) |
| **Trivy** | All-in-one security scanner by Aqua Security |
| **Cosign** | Container signing tool by the Sigstore project |
| **Sigstore** | Open-source project for signing software artifacts (Cosign, Fulcio, Rekor) |
| **OCI** | Open Container Initiative — standards for container images and registries |
| **Attestation** | Cryptographically signed statement about an artifact (e.g., "this SBOM belongs to this image") |
| **Keyless signing** | Signing using short-lived certificates from Fulcio (OIDC-based, no key management) |
| **cosign.key** | Private key — keep secret, used to sign |
| **cosign.pub** | Public key — share freely, used to verify |

---

## 4. Environment Setup

### Install Syft
```bash
# Linux/macOS (recommended)
curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b /usr/local/bin

# Verify installation
syft version

# Windows (via Chocolatey)
choco install syft

# Windows (via Scoop)
scoop install syft
```

### Install Trivy
```bash
# Linux (Debian/Ubuntu)
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt-get update && sudo apt-get install trivy

# macOS
brew install trivy

# Windows (via Chocolatey)
choco install trivy

# Verify installation
trivy --version
```

### Install Cosign
```bash
# Linux (download binary)
curl -O -L "https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64"
sudo mv cosign-linux-amd64 /usr/local/bin/cosign
sudo chmod +x /usr/local/bin/cosign

# macOS
brew install cosign

# Windows (via Chocolatey)
choco install cosign

# Verify installation
cosign version
```

### Build the Image (Starting Point)
```bash
# Navigate to your project
cd week6-coreops

# Build with a versioned tag (important — scanners need a tag)
docker build -t coreops-app:1.0 .

# Verify the image exists
docker images coreops-app
```

> **Why tag with `1.0` not `latest`?** Signing and attestations are attached per tag. Using a specific version makes verification deterministic.

---

## 5. Lab 1 – SBOM Generation with Syft

### What Syft Does
Syft inspects a container image layer by layer, extracts all installed packages (OS packages, language packages, binaries), and outputs them in a structured format.

### Lab 1a: Basic SBOM Generation
```bash
# Generate SBOM in JSON format
syft coreops-app:1.0 -o json > sbom.json

# Quick peek at the output
cat sbom.json | head -50

# Count total packages found
cat sbom.json | python3 -c "import json,sys; data=json.load(sys.stdin); print(f'Total packages: {len(data[\"artifacts\"])}')"
```

**Example output snippet:**
```json
{
  "schema": { "version": "16.0.8" },
  "source": {
    "type": "image",
    "target": { "tags": ["coreops-app:1.0"] }
  },
  "artifacts": [
    {
      "name": "python",
      "version": "3.9.18",
      "type": "binary",
      "cpes": ["cpe:2.3:a:python:python:3.9.18:*:*:*:*:*:*:*"]
    },
    {
      "name": "pip",
      "version": "23.0.1",
      "type": "python"
    }
    // ... many more
  ]
}
```

### Lab 1b: SBOM in Multiple Formats
```bash
# SPDX format (industry standard, required by some regulations)
syft coreops-app:1.0 -o spdx-json > sbom.spdx.json

# CycloneDX format (OWASP standard, used by many scanners)
syft coreops-app:1.0 -o cyclonedx-json > sbom.cyclonedx.json

# Human-readable table format
syft coreops-app:1.0 -o table

# List just package names and versions
syft coreops-app:1.0 -o table | grep -v "^NAME" | awk '{print $1, $2}'
```

### SBOM Format Comparison
| Format | Standard | Use Case |
|--------|----------|----------|
| `json` (Syft native) | Syft | Detailed, human-readable |
| `spdx-json` | Linux Foundation | Regulatory compliance, sharing |
| `cyclonedx-json` | OWASP | Integration with security tools |
| `table` | None | Quick terminal review |

### Lab 1c: SBOM for a Filesystem (not just images)
```bash
# Scan your local project directory
syft dir:. -o table

# Scan a specific file
syft file:./requirements.txt -o table
```

### Lab 1d: Understanding SBOM Fields
```bash
# Pretty-print and explore
cat sbom.json | python3 -m json.tool | less

# Extract just OS packages
cat sbom.json | python3 -c "
import json, sys
data = json.load(sys.stdin)
os_pkgs = [a for a in data['artifacts'] if a['type'] == 'deb']
for p in os_pkgs:
    print(f\"{p['name']}:{p['version']}\")
"
```

**Key SBOM fields:**
| Field | Meaning |
|-------|---------|
| `name` | Package name |
| `version` | Installed version |
| `type` | Package manager type (`deb`, `python`, `binary`, etc.) |
| `cpes` | CPE identifiers — used to match against CVE databases |
| `purl` | Package URL — standardized identifier (`pkg:pypi/requests@2.28.0`) |
| `licenses` | License information |

---

## 6. Lab 2 – Vulnerability Scanning with Trivy

### What Trivy Does
Trivy compares every package in your image against multiple CVE databases simultaneously, reporting which packages have known vulnerabilities and how severe they are.

### Lab 2a: Basic Image Scan
```bash
# Scan the image (outputs to terminal)
trivy image coreops-app:1.0
```

**Expected output structure:**
```
coreops-app:1.0 (debian 11.8)
================================
Total: 47 (UNKNOWN: 0, LOW: 30, MEDIUM: 10, HIGH: 5, CRITICAL: 2)

┌──────────────────────┬────────────────┬──────────┬───────────────────┬───────────────┬──────────────────────────────────────────────────┐
│       Library        │ Vulnerability  │ Severity │ Installed Version │ Fixed Version │                      Title                       │
├──────────────────────┼────────────────┼──────────┼───────────────────┼───────────────┼──────────────────────────────────────────────────┤
│ libssl1.1            │ CVE-2023-5678  │ CRITICAL │ 1.1.1n-0+deb11u4  │ 1.1.1w-0+... │ OpenSSL: Buffer overflow in X.509 parsing        │
│ python3.9            │ CVE-2023-1234  │ HIGH     │ 3.9.2-1           │ 3.9.18-1      │ Python: Remote code execution via pickle         │
└──────────────────────┴────────────────┴──────────┴───────────────────┴───────────────┴──────────────────────────────────────────────────┘
```

### Lab 2b: Filter by Severity
```bash
# Only show CRITICAL and HIGH
trivy image --severity HIGH,CRITICAL coreops-app:1.0

# Only show CRITICAL
trivy image --severity CRITICAL coreops-app:1.0

# Exit with error code if any CRITICAL found (useful in CI)
trivy image --severity CRITICAL --exit-code 1 coreops-app:1.0
echo "Exit code: $?"
```

### Lab 2c: Output to JSON (for CI artifact storage)
```bash
# JSON output
trivy image --format json -o trivy-report.json coreops-app:1.0

# Inspect the report
cat trivy-report.json | python3 -c "
import json, sys
data = json.load(sys.stdin)
for result in data.get('Results', []):
    vulns = result.get('Vulnerabilities', [])
    critical = [v for v in vulns if v['Severity'] == 'CRITICAL']
    high = [v for v in vulns if v['Severity'] == 'HIGH']
    print(f\"Target: {result['Target']}\")
    print(f\"  CRITICAL: {len(critical)}, HIGH: {len(high)}\")
"
```

### Lab 2d: Scan an SBOM file (instead of image)
```bash
# Scan the SBOM we generated in Lab 1
trivy sbom sbom.cyclonedx.json

# This is faster (no need to re-inspect the image)
# Useful when the image isn't available locally
```

### Lab 2e: Ignore Known Accepted Vulnerabilities
Create a `.trivyignore` file:
```bash
cat > .trivyignore << 'EOF'
# CVEs we have accepted risk for (with reason)
CVE-2021-12345  # False positive - not exploitable in our config
CVE-2022-67890  # No fix available, mitigated by network policy
EOF
```

Then:
```bash
trivy image --ignorefile .trivyignore coreops-app:1.0
```

### Lab 2f: Fix Vulnerabilities — Update Base Image
The most common fix is updating the base image:

```dockerfile
# Before (potentially vulnerable)
FROM python:3.9-slim

# After (latest patch releases)
FROM python:3.9-slim-bookworm

# Even better — specify exact digest for reproducibility
FROM python:3.9-slim-bookworm@sha256:abc123...
```

```bash
# Rebuild with updated base
docker build -t coreops-app:1.1 .

# Scan again to see improvement
trivy image --severity HIGH,CRITICAL coreops-app:1.1
```

### Trivy Scan Targets Reference
| Target | Command | What it Scans |
|--------|---------|---------------|
| Container image | `trivy image NAME` | OS packages, language libs |
| Filesystem | `trivy fs .` | Source code dependencies |
| Git repo | `trivy repo https://github.com/...` | Remote repository |
| Terraform | `trivy config infra/` | IaC misconfigurations |
| SBOM file | `trivy sbom sbom.json` | Pre-generated SBOM |
| Kubernetes | `trivy k8s cluster` | Live cluster resources |

---

## 7. Lab 3 – Image Signing with Cosign

### How Cosign Works
```
                    SIGNING
You (CI Pipeline)
       │
       ├─ cosign generate-key-pair → cosign.key (private) + cosign.pub (public)
       │
       ├─ docker build → coreops-app:1.0
       │
       └─ cosign sign --key cosign.key coreops-app:1.0
                              │
                              ▼
                    Registry stores signature
                    alongside image as OCI artifact


                   VERIFICATION
Consumer/Deployment
       │
       └─ cosign verify --key cosign.pub coreops-app:1.0
                              │
                    ✓ Signature valid → deploy
                    ✗ Signature invalid → reject
```

### Lab 3a: Generate a Key Pair
```bash
# Generate keys (will prompt for password)
cosign generate-key-pair

# This creates:
# cosign.key  ← PRIVATE — never commit this to Git
# cosign.pub  ← PUBLIC  — safe to share/commit
```

> **Security warning:** Add `cosign.key` to `.gitignore` immediately.

```bash
echo "cosign.key" >> .gitignore
git add .gitignore
git commit -m "security: ignore cosign private key"
```

### Lab 3b: Sign a Local Image
```bash
# Sign the image
cosign sign --key cosign.key coreops-app:1.0

# NOTE: For local images not pushed to a registry, use --no-tuf flag
# For OCI registries, the signature is stored alongside the image
```

### Lab 3c: Sign an Image in a Registry
```bash
# First push the image to a registry (Docker Hub example)
docker tag coreops-app:1.0 YOUR_DOCKERHUB_USER/coreops-app:1.0
docker push YOUR_DOCKERHUB_USER/coreops-app:1.0

# Sign (signs the image in the registry)
cosign sign --key cosign.key YOUR_DOCKERHUB_USER/coreops-app:1.0

# The signature is pushed as a separate OCI artifact:
# YOUR_DOCKERHUB_USER/coreops-app:sha256-<digest>.sig
```

### Lab 3d: Verify the Signature
```bash
# Verify using public key
cosign verify --key cosign.pub YOUR_DOCKERHUB_USER/coreops-app:1.0

# Expected output:
# Verification for index.docker.io/YOUR_USER/coreops-app:1.0 --
# The following checks were performed on each of these signatures:
#   - The cosign claims were validated
#   - The signatures were verified against the specified public key
#
# [{"critical":{"identity":{"docker-reference":"..."},...}}]
```

```bash
# What happens with a tampered image (try modifying a layer)
cosign verify --key cosign.pub tampered-image:1.0
# Error: no matching signatures
```

### Lab 3e: Attach SBOM as Attestation
This links your SBOM to the image cryptographically — proving the SBOM belongs to that exact image:

```bash
# Convert SBOM to attestation format and attach
cosign attest --key cosign.key \
  --predicate sbom.cyclonedx.json \
  --type cyclonedx \
  YOUR_DOCKERHUB_USER/coreops-app:1.0

# Verify the attestation
cosign verify-attestation --key cosign.pub \
  --type cyclonedx \
  YOUR_DOCKERHUB_USER/coreops-app:1.0
```

### Lab 3f: Keyless Signing (Advanced — CI Best Practice)
In GitHub Actions, you don't need key files. Cosign uses GitHub's OIDC token for keyless signing via Sigstore's Fulcio CA:

```yaml
- name: Sign image (keyless)
  run: |
    cosign sign \
      --yes \
      YOUR_DOCKERHUB_USER/coreops-app:${{ github.sha }}
  env:
    COSIGN_EXPERIMENTAL: "true"
```

```bash
# Verify keyless signature
cosign verify \
  --certificate-identity "https://github.com/YOUR_ORG/YOUR_REPO/.github/workflows/ci.yml@refs/heads/main" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  YOUR_DOCKERHUB_USER/coreops-app:SHA
```

### Key-based vs Keyless Signing
| Aspect | Key-based | Keyless (Sigstore) |
|--------|-----------|-------------------|
| Key storage | You manage `cosign.key` in a secret | No key to manage |
| Identity | Key ownership | OIDC identity (GitHub, Google, etc.) |
| Auditability | Manual | Transparent log (Rekor) |
| Best for | Air-gapped, self-hosted | Cloud CI (GitHub Actions, etc.) |
| Setup complexity | Low | Low (built into GitHub Actions) |

---

## 8. Lab 4 – Full Pipeline Integration

### Complete CI Pipeline with All Three Tools

Update `.github/workflows/ci.yml`:

```yaml
name: CI Pipeline - DevSecOps

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  docker-build:
    runs-on: ubuntu-latest

    permissions:
      contents: read
      packages: write
      id-token: write        # Required for keyless cosign signing

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      # ─── BUILD ───────────────────────────────────────────────
      - name: Build Docker image
        run: docker build -t coreops-app:${{ github.sha }} .

      - name: Run container test
        run: |
          OUTPUT=$(docker run coreops-app:${{ github.sha }})
          echo "Output: $OUTPUT"
          echo "$OUTPUT" | grep -q "Hello CoreOps" || (echo "Test failed" && exit 1)

      # ─── SBOM GENERATION ─────────────────────────────────────
      - name: Generate SBOM
        uses: anchore/sbom-action/download-syft@v0

      - name: Create SBOM
        run: syft coreops-app:${{ github.sha }} -o cyclonedx-json > sbom.json

      - name: Upload SBOM artifact
        uses: actions/upload-artifact@v4
        with:
          name: sbom
          path: sbom.json

      # ─── VULNERABILITY SCAN ──────────────────────────────────
      - name: Vulnerability Scan
        uses: aquasecurity/trivy-action@v0.36.0
        with:
          image-ref: coreops-app:${{ github.sha }}
          format: json
          output: trivy-report.json
          severity: HIGH,CRITICAL
          exit-code: 1            # Fail pipeline on HIGH/CRITICAL

      - name: Upload Trivy report
        uses: actions/upload-artifact@v4
        if: always()              # Upload even if scan failed
        with:
          name: trivy-report
          path: trivy-report.json

      # ─── IMAGE SIGNING ───────────────────────────────────────
      - name: Install cosign
        uses: sigstore/cosign-installer@v3

      - name: Verify cosign installed
        run: cosign version

      # ─── TERRAFORM VALIDATION ────────────────────────────────
  terraform-validate:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: infra
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install Terraform
        uses: hashicorp/setup-terraform@v3

      - name: Terraform Init
        run: terraform init

      - name: Terraform Validate
        run: terraform validate

      - name: Terraform Format
        run: terraform fmt -check
```

### Pipeline Flow Diagram
```
Push to main
     │
     ▼
┌─────────────────────────────────────────────┐
│              docker-build job               │
│                                             │
│  Checkout → Build → Test                   │
│                │                            │
│                ▼                            │
│         Generate SBOM (Syft)               │
│                │                            │
│                ▼                            │
│      Vulnerability Scan (Trivy)            │
│         CRITICAL found?                    │
│         YES → ✗ FAIL                       │
│         NO  → ↓                            │
│                │                            │
│                ▼                            │
│       Install & Verify Cosign              │
│                                             │
│  Artifacts: sbom.json, trivy-report.json   │
└─────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────┐
│           terraform-validate job            │
│  Init → Validate → Format Check            │
└─────────────────────────────────────────────┘
```

### Download Artifacts After Pipeline
1. Go to your GitHub repo → **Actions** tab
2. Click the completed workflow run
3. Scroll to **Artifacts** section at the bottom
4. Download `sbom` and `trivy-report`

---

## 9. Observation Checklist

After running the full pipeline, verify each item:

### SBOM (Syft)
- [ ] `sbom.json` artifact is created and downloadable
- [ ] SBOM lists Python and OS packages
- [ ] SBOM includes `purl` and `cpe` fields for each package
- [ ] SBOM format is valid CycloneDX JSON

```bash
# Validate CycloneDX format locally
cat sbom.json | python3 -c "
import json, sys
data = json.load(sys.stdin)
print('Format:', data.get('bomFormat', 'Unknown'))
print('Spec version:', data.get('specVersion', 'Unknown'))
print('Components:', len(data.get('components', [])))
"
```

### Vulnerability Scan (Trivy)
- [ ] `trivy-report.json` artifact is created
- [ ] Trivy reports CVEs with severity levels
- [ ] HIGH/CRITICAL findings are listed (or absent if image is clean)
- [ ] Pipeline fails if CRITICAL vulnerabilities are found (when `exit-code: 1` is set)

```bash
# Count vulnerabilities by severity locally
cat trivy-report.json | python3 -c "
import json, sys
data = json.load(sys.stdin)
counts = {}
for result in data.get('Results', []):
    for v in result.get('Vulnerabilities') or []:
        sev = v['Severity']
        counts[sev] = counts.get(sev, 0) + 1
for sev, count in sorted(counts.items()):
    print(f'{sev}: {count}')
"
```

### Image Signing (Cosign)
- [ ] `cosign version` prints version without error
- [ ] Key pair generated (`cosign.key` + `cosign.pub`)
- [ ] `cosign.key` is in `.gitignore` and NOT committed
- [ ] Signing command completes successfully
- [ ] `cosign verify` confirms signature validity
- [ ] Tampered image fails verification

### Security
- [ ] No private keys in workflow files or committed to repo
- [ ] Trivy report uploaded even on scan failure (`if: always()`)
- [ ] `id-token: write` permission only added when using keyless signing
- [ ] Scan results reviewed — vulnerabilities tracked

---

## 10. Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `syft: command not found` | Syft not installed / action not run first | Use `anchore/sbom-action/download-syft@v0` step before running `syft` |
| `Error: unknown command "" for "trivy"` | Using `trivy-action` as installer, then running trivy manually | Use `trivy-action` with `image-ref` input — it handles scanning internally |
| `Error: no such image: coreops-app:1.0` | Image not built yet | Ensure `docker build` step runs before Syft/Trivy steps |
| `cosign: error generating key pair: key already exists` | `cosign.key` already in directory | Delete existing keys or use `--output-key-prefix` flag |
| `Error: signing coreops-app:1.0: image is not in a registry` | Trying to sign a local-only image | Push to registry first, then sign the registry reference |
| `WARN: 1 fixable vulnerabilities found` with exit-code 0 | `exit-code: 1` not set in trivy-action | Add `exit-code: '1'` to trivy-action inputs |
| `Error: failed to verify signature: no signatures found` | Image was not signed | Run `cosign sign` before `cosign verify` |
| `permission denied: id-token` | Missing `id-token: write` permission | Add to job `permissions:` block when using keyless signing |
| `sbom.json: No such file or directory` | Syft output path issue | Ensure `syft` command runs in same directory as next steps |
| `cosign.key: permission denied` | Key file not readable | Run `chmod 600 cosign.key` |

---

## 11. Tools Comparison

### SBOM Tools
| Tool | Vendor | Formats | Best For |
|------|--------|---------|----------|
| **Syft** | Anchore | SPDX, CycloneDX, JSON | Container images, filesystems |
| **cdxgen** | OWASP | CycloneDX | Multi-language projects |
| **Tern** | VMware | SPDX | Docker layer analysis |
| **docker sbom** | Docker | SPDX | Quick Docker CLI integration |

### Vulnerability Scanners
| Tool | Vendor | Databases | Best For |
|------|--------|-----------|----------|
| **Trivy** | Aqua Security | NVD, OSV, GitHub Advisory, many more | All-in-one: images, IaC, code |
| **Grype** | Anchore | NVD, GitHub Advisory | Pairs naturally with Syft |
| **Snyk** | Snyk | Snyk proprietary DB | Developer-focused, IDE integration |
| **Clair** | Quay/RedHat | NVD | Registry-side scanning |
| **Anchore Enterprise** | Anchore | Multiple | Enterprise policy enforcement |

### Signing Tools
| Tool | Standard | Key Management | Best For |
|------|----------|----------------|----------|
| **Cosign** | Sigstore/OCI | Keyless or key-based | Container images (modern standard) |
| **Notary v2** | CNCF/OCI | TUF-based | Enterprise registries |
| **GPG** | OpenPGP | Manual key management | Legacy systems |
| **Docker Content Trust** | Notary v1 | Notary server | Basic Docker Hub signing |

---

## 12. Revision Questions

Test your understanding — answer without looking:

1. What is an SBOM and why is it compared to a "nutrition label"?
2. What is the difference between `syft` and `trivy` — can one replace the other?
3. What does a CVE number like `CVE-2023-1234` represent?
4. If Trivy finds a CRITICAL vulnerability, what are your options?
5. What is the difference between `cosign.key` and `cosign.pub` — which one is safe to share?
6. Why should you never commit `cosign.key` to a Git repository?
7. What does `cosign attest` do differently than `cosign sign`?
8. What is keyless signing and what makes it more secure than key-based signing?
9. Why is `if: always()` important on the Trivy report upload step?
10. What is the difference between `trivy image` and `trivy sbom`?
11. What CVSS score range is considered CRITICAL?
12. What happens to a supply chain if an attacker pushes a malicious image to your registry without signing? How does Cosign prevent this?

### Answers
<details>
<summary>Click to reveal answers</summary>

1. An SBOM is a complete inventory of all components (packages, libraries, OS components) in your software. Like a nutrition label, it tells you exactly what's "inside" — so you can check for harmful ingredients (vulnerabilities).

2. Syft **generates** SBOMs (inventories). Trivy **scans for vulnerabilities**. Trivy can generate SBOMs too, but Syft is more detailed. They complement each other — Syft for thorough SBOM, Trivy for scanning. You can scan a Syft-generated SBOM with Trivy: `trivy sbom sbom.json`.

3. CVE-2023-1234 is a standardized identifier: CVE = Common Vulnerabilities and Exposures, 2023 = year disclosed, 1234 = sequential number. It's a globally unique ID for a known security flaw.

4. Options: (a) Update the package/base image to a fixed version, (b) Accept the risk with a `.trivyignore` entry if not exploitable in your context, (c) Use `--ignore-unfixed` to suppress unfixed CVEs, (d) Change the base image entirely.

5. `cosign.key` is the **private key** — used to sign, must be kept secret. `cosign.pub` is the **public key** — used to verify, safe to share, commit to repo, and distribute widely.

6. If `cosign.key` is in a public Git repo, anyone can sign images pretending to be you. Your entire trust model collapses — attackers could sign malicious images that pass `cosign verify`.

7. `cosign sign` signs the image digest. `cosign attest` attaches a signed statement (predicate) — like "this SBOM belongs to this image" or "this image passed security scan." Attestations carry richer metadata than a plain signature.

8. Keyless signing uses short-lived certificates from Sigstore's Fulcio CA, tied to an OIDC identity (like your GitHub Actions workflow). No private key to store or rotate. Every signing event is logged in Rekor (transparent log), making it auditable. Compromise of a runner doesn't compromise a long-lived key.

9. Without `if: always()`, if Trivy exits with code 1 (CRITICAL found), GitHub Actions skips all subsequent steps — including the report upload. `if: always()` ensures the report is uploaded regardless, so you can review what was found even on a failed run.

10. `trivy image` pulls and inspects a container image directly, analyzing all layers. `trivy sbom` scans a pre-generated SBOM file — faster (no image needed locally), useful in pipelines where the image has already been analyzed.

11. CRITICAL = CVSS score 9.0–10.0.

12. Without signing, an attacker who compromises your registry can push a malicious image with the same tag — your deployment pipeline would pull and run it unknowingly. With Cosign, your deployment policy only allows images with a valid signature from your CI pipeline's key/identity. The unsigned malicious image fails verification and is rejected before deployment.

</details>

---

## 13. Key Takeaways

```
┌─────────────────────────────────────────────────────────────┐
│                    TUESDAY SUMMARY                          │
├─────────────────────────────────────────────────────────────┤
│  ✓ SBOM = complete inventory of image components (Syft)     │
│  ✓ Vulnerability scanning = match SBOM against CVE DBs      │
│  ✓ Trivy scans images, filesystems, IaC, and SBOMs          │
│  ✓ CRITICAL CVEs should block the pipeline (exit-code: 1)   │
│  ✓ Cosign signs images with private key, verifies with pub  │
│  ✓ NEVER commit cosign.key — add to .gitignore              │
│  ✓ Keyless signing is preferred in GitHub Actions (OIDC)    │
│  ✓ Attestations link SBOMs to images cryptographically      │
│  ✓ Upload scan reports as artifacts even on failure         │
│  ✓ Shift-left = validate security before shipping           │
└─────────────────────────────────────────────────────────────┘
```

### The Full DevSecOps Pipeline Picture
```
Code Push
    │
    ▼
Checkout → Build Image → Run Tests
                │
                ▼
          Generate SBOM ──────────────────→ sbom.json (artifact)
          (Syft)
                │
                ▼
          Scan for CVEs ──────────────────→ trivy-report.json (artifact)
          (Trivy)          CRITICAL found?
                           YES → ✗ FAIL PIPELINE
                           NO  → ↓
                │
                ▼
          Sign Image ─────────────────────→ Signature in registry
          (Cosign)
                │
                ▼
          Push to Registry
                │
                ▼
          Deploy (only if signed & clean)
```

### What Each Tool Prevents
| Tool | Attack it Prevents |
|------|--------------------|
| Syft (SBOM) | Unknown dependency sprawl; untracked components |
| Trivy | Shipping known vulnerable packages to production |
| Cosign | Supply chain tampering; unsigned/malicious image execution |
| All three together | Full software supply chain compromise |

---

### Connecting Monday and Tuesday

| Monday | Tuesday |
|--------|---------|
| Build the image | Inventory the image (SBOM) |
| Run container test | Scan for vulnerabilities |
| Validate Terraform | Sign the image |
| Secrets management | Key management (cosign.key) |
| Pipeline as automation | Pipeline as security gate |

> The CI pipeline built on Monday becomes the **security enforcement point** on Tuesday. Every image that passes Tuesday's pipeline has a known inventory, no critical CVEs, and a cryptographic proof of origin.

---

*Study both days together — Monday's pipeline is the vehicle; Tuesday's tools are the security layer on top.*
