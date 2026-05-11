# Monday: GitHub Actions / GitLab CI Pipelines with Terraform & Docker

> **Goal:** Master CI pipeline fundamentals — Docker builds, container testing, Terraform validation, and safe secret handling — as the foundation of DevSecOps automation.

---

## Table of Contents

1. [Core Concepts](#1-core-concepts)
2. [How GitHub Actions Works](#2-how-github-actions-works)
3. [Key Terminology](#3-key-terminology)
4. [Environment Setup](#4-environment-setup)
5. [Lab 1 – Basic CI Pipeline](#5-lab-1--basic-ci-pipeline)
6. [Lab 2 – Docker Deep Dive](#6-lab-2--docker-deep-dive)
7. [Lab 3 – Terraform Validation in CI](#7-lab-3--terraform-validation-in-ci)
8. [Lab 4 – Secrets & Security Hardening](#8-lab-4--secrets--security-hardening)
9. [Observation Checklist](#9-observation-checklist)
10. [Common Errors & Fixes](#10-common-errors--fixes)
11. [GitLab CI Equivalent](#11-gitlab-ci-equivalent)
12. [Revision Questions](#12-revision-questions)
13. [Key Takeaways](#13-key-takeaways)

---

## 1. Core Concepts

### What is CI (Continuous Integration)?
CI is the practice of **automatically building, testing, and validating** code every time a change is pushed to a repository. It catches bugs early, enforces standards, and reduces "it works on my machine" problems.

### CI vs CD vs CD
| Term | Stands For | What it Does |
|------|-----------|--------------|
| CI | Continuous Integration | Build + Test on every push |
| CD | Continuous Delivery | Auto-deploy to staging, manual approval to prod |
| CD | Continuous Deployment | Fully automated deploy all the way to production |

### Why GitHub Actions?
- **Native to GitHub** — no external service needed
- **YAML-based** — easy to read and version-control
- **Free tier** — 2,000 minutes/month for public repos
- **Huge marketplace** — thousands of pre-built actions
- **Matrix builds** — test against multiple OS/runtime combos

### DevSecOps Triangle
```
        Security
           /\
          /  \
         /    \
        /  CI  \
       /Pipeline\
      /____________\
   Dev            Ops
```
The CI pipeline is where all three meet: code quality (Dev), deployment automation (Ops), and vulnerability scanning (Sec).

---

## 2. How GitHub Actions Works

### Execution Flow
```
Code Push / PR / Schedule
        │
        ▼
  Trigger Event (on:)
        │
        ▼
  Workflow File (.github/workflows/*.yml)
        │
        ▼
  Job 1 ──────── Job 2 ──────── Job 3
  (runner)       (runner)       (runner)
     │
     ▼
  Step 1 → Step 2 → Step 3 → Step 4
```

### Runner Types
| Type | Description | Use Case |
|------|-------------|----------|
| `ubuntu-latest` | GitHub-hosted Ubuntu VM | Most CI tasks |
| `windows-latest` | GitHub-hosted Windows VM | .NET / Windows builds |
| `macos-latest` | GitHub-hosted macOS VM | iOS/macOS builds |
| `self-hosted` | Your own machine/server | Private networks, custom hardware |

### Workflow File Location
```
your-repo/
└── .github/
    └── workflows/
        ├── ci.yml         ← main pipeline
        ├── release.yml    ← release automation
        └── security.yml   ← security scans
```

---

## 3. Key Terminology

| Term | Definition |
|------|-----------|
| **Workflow** | The entire `.yml` file — defines the automation |
| **Event** | What triggers the workflow (`push`, `pull_request`, `schedule`) |
| **Job** | A group of steps that run on the same runner |
| **Step** | A single task inside a job — either a `run` command or `uses` action |
| **Action** | A reusable unit of work from the marketplace (e.g., `actions/checkout@v3`) |
| **Runner** | The virtual machine that executes jobs |
| **Artifact** | Files saved from a job (test reports, binaries, SBOMs) |
| **Secret** | Encrypted environment variable stored in GitHub settings |
| **Context** | Objects like `github`, `env`, `secrets` available in expressions |
| **Expression** | `${{ }}` syntax for dynamic values |

---

## 4. Environment Setup

### Prerequisites
```bash
# Verify Git is installed
git --version

# Verify Docker is installed
docker --version

# Verify Terraform is installed
terraform --version
```

### Project Structure Setup
```bash
# Create project directory
mkdir week6-coreops && cd week6-coreops

# Initialize Git repo
git init

# Create the Python application
echo 'print("Hello CoreOps Week 6!")' > app.py

# Verify it runs locally first
python app.py
# Expected output: Hello CoreOps Week 6!
```

### Create the Dockerfile
```bash
cat > Dockerfile << 'EOF'
FROM python:3.9-slim
COPY app.py /app/app.py
CMD ["python", "/app/app.py"]
EOF
```

**Dockerfile breakdown:**
| Line | Meaning |
|------|---------|
| `FROM python:3.9-slim` | Base image — minimal Python 3.9 (slim = smaller size) |
| `COPY app.py /app/app.py` | Copy local file into the container at `/app/app.py` |
| `CMD [...]` | Default command to run when the container starts |

### Create Terraform Infrastructure
```bash
# Create infra directory
mkdir infra

# Create main.tf
cat > infra/main.tf << 'EOF'
terraform {
  required_version = ">= 1.0"
}

resource "null_resource" "example" {}
EOF
```

**Terraform file breakdown:**
| Block | Meaning |
|-------|---------|
| `terraform {}` | Meta-configuration for Terraform itself |
| `required_version = ">= 1.0"` | Enforces minimum Terraform version |
| `null_resource "example"` | A resource that does nothing — used for testing/demos |

### Create GitHub Actions Workflow
```bash
# Create directories
mkdir -p .github/workflows

# Create the CI file
touch .github/workflows/ci.yml
```

### Final Directory Structure
```
week6-coreops/
├── .github/
│   └── workflows/
│       └── ci.yml
├── infra/
│   └── main.tf
├── app.py
└── Dockerfile
```

---

## 5. Lab 1 – Basic CI Pipeline

### Step 1: Write the CI Workflow

Create `.github/workflows/ci.yml`:

```yaml
name: CI Pipeline

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Build Docker image
        run: docker build -t coreops-app .

      - name: Run container test
        run: docker run coreops-app

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3

      - name: Terraform Init
        run: terraform -chdir=infra init

      - name: Terraform Validate
        run: terraform -chdir=infra validate
```

### Step 2: Push to GitHub
```bash
# Stage all files
git add .

# Commit
git commit -m "feat: initial CI pipeline with Docker and Terraform"

# Add remote (replace with your repo URL)
git remote add origin https://github.com/YOUR_USERNAME/week6-coreops.git

# Push
git push -u origin main
```

### Step 3: Observe the Pipeline
1. Go to your GitHub repo → **Actions** tab
2. Click the running workflow
3. Expand each step to see logs

### Expected Output per Step

**Checkout:**
```
Cloning into '/home/runner/work/week6-coreops/week6-coreops'...
```

**Build Docker image:**
```
Step 1/3 : FROM python:3.9-slim
Step 2/3 : COPY app.py /app/app.py
Step 3/3 : CMD ["python", "/app/app.py"]
Successfully built a1b2c3d4e5f6
Successfully tagged coreops-app:latest
```

**Run container test:**
```
Hello CoreOps Week 6!
```

**Terraform Init:**
```
Initializing the backend...
Initializing provider plugins...
Terraform has been successfully initialized!
```

**Terraform Validate:**
```
Success! The configuration is valid.
```

---

## 6. Lab 2 – Docker Deep Dive

### Understanding Docker in CI

When GitHub Actions runs `docker build`, it:
1. Spins up a fresh Ubuntu VM
2. Docker daemon is pre-installed on `ubuntu-latest`
3. Builds the image from scratch (no cache by default)
4. The image exists only for the duration of that job

### Lab 2a: Add a `.dockerignore`
Prevent unnecessary files from being sent to Docker build context:

```bash
cat > .dockerignore << 'EOF'
.github/
infra/
*.md
.git/
EOF
```

### Lab 2b: Add Build Arguments
Modify your Dockerfile to accept a build argument:

```dockerfile
FROM python:3.9-slim

ARG APP_VERSION=1.0.0
ENV APP_VERSION=${APP_VERSION}

COPY app.py /app/app.py
CMD ["python", "/app/app.py"]
```

Update `ci.yml` to pass the argument:
```yaml
- name: Build Docker image
  run: docker build --build-arg APP_VERSION=1.0.${{ github.run_number }} -t coreops-app .
```

> `github.run_number` is a built-in GitHub Actions context — auto-incrementing integer for each run.

### Lab 2c: Tag Images Properly
```yaml
- name: Build Docker image
  run: |
    IMAGE_TAG=${{ github.sha }}
    docker build -t coreops-app:latest -t coreops-app:${IMAGE_TAG:0:7} .
    echo "Built image with tags: latest and ${IMAGE_TAG:0:7}"
```

### Docker Image Tagging Best Practices
| Tag | Example | When to Use |
|-----|---------|-------------|
| `latest` | `coreops-app:latest` | Latest main branch build |
| Git SHA | `coreops-app:a1b2c3d` | Immutable, traceable |
| Semver | `coreops-app:1.2.3` | Release versions |
| Branch | `coreops-app:feature-login` | Feature branch testing |

### Lab 2d: Run Container with Health Check
```yaml
- name: Run container test
  run: |
    OUTPUT=$(docker run coreops-app)
    echo "Container output: $OUTPUT"
    if [[ "$OUTPUT" == *"Hello CoreOps"* ]]; then
      echo "✓ Test passed"
    else
      echo "✗ Test failed"
      exit 1
    fi
```

---

## 7. Lab 3 – Terraform Validation in CI

### Why Validate Terraform in CI?
- Catch syntax errors before they reach production
- Enforce formatting standards across the team
- Prevent invalid provider configurations from being merged

### Lab 3a: Full Terraform Validation Pipeline
Update the Terraform section in `ci.yml`:

```yaml
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.9.0"

      - name: Terraform Format Check
        run: terraform -chdir=infra fmt -check
        # Fails if files are not properly formatted

      - name: Terraform Init
        run: terraform -chdir=infra init

      - name: Terraform Validate
        run: terraform -chdir=infra validate
```

### Lab 3b: Auto-format Terraform on Push
Add this to `infra/main.tf` with bad formatting to test:
```hcl
terraform{required_version=">= 1.0"}
resource "null_resource" "example"{}
```

Then run locally:
```bash
terraform fmt infra/
```
This auto-fixes formatting. In CI, `fmt -check` fails the pipeline if formatting is wrong — enforcing the team to format locally before pushing.

### Terraform Commands Reference
| Command | What it Does |
|---------|-------------|
| `terraform init` | Downloads providers, sets up backend |
| `terraform validate` | Checks syntax and internal consistency |
| `terraform fmt` | Formats `.tf` files to canonical style |
| `terraform fmt -check` | Returns non-zero exit if files need formatting |
| `terraform plan` | Shows what changes would be made |
| `terraform apply` | Applies the changes (never in CI without approval) |

### Lab 3c: Add a `required_providers` block
Update `infra/main.tf`:
```hcl
terraform {
  required_version = ">= 1.0"

  required_providers {
    null = {
      source  = "hashicorp/null"
      version = "~> 3.0"
    }
  }
}

resource "null_resource" "example" {
  triggers = {
    always_run = timestamp()
  }
}
```

This teaches CI to download the specific provider version on `terraform init`.

---

## 8. Lab 4 – Secrets & Security Hardening

### The Golden Rule
> **Never hardcode secrets in workflow files or application code.**

GitHub Actions provides encrypted secrets — they are masked in logs automatically.

### Lab 4a: Add a Secret in GitHub
1. Go to your repo → **Settings** → **Secrets and variables** → **Actions**
2. Click **New repository secret**
3. Name: `MY_API_KEY`, Value: `super-secret-value`

### Lab 4b: Use the Secret in CI
```yaml
      - name: Use API Key
        run: |
          echo "Key length: ${#MY_API_KEY}"
          # Never echo the actual value!
        env:
          MY_API_KEY: ${{ secrets.MY_API_KEY }}
```

> GitHub automatically replaces `super-secret-value` with `***` in logs.

### Lab 4c: Security Checklist for Workflows
```yaml
jobs:
  build:
    runs-on: ubuntu-latest

    # SECURITY: Restrict GITHUB_TOKEN permissions to minimum needed
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v3
        # SECURITY: Pin actions to full SHA for supply chain security
        # uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683

      - name: Build Docker image
        run: docker build -t coreops-app .
        # SECURITY: Never use --privileged flag
        # SECURITY: Never mount host paths like -v /:/host
```

### What NOT to Do
```yaml
# ❌ NEVER DO THIS
- name: Bad practice examples
  run: |
    echo "Password: mypassword123"           # Hardcoded secret
    docker run --privileged coreops-app       # Privileged container
    curl http://internal-service/secret       # Unencrypted HTTP
  env:
    DB_PASSWORD: "hardcoded-password"         # Secret in env without secrets context
```

---

## 9. Observation Checklist

After pushing your code, verify each item in the GitHub Actions logs:

### Docker
- [ ] `docker build` completes without errors
- [ ] Image is tagged correctly (`coreops-app:latest`)
- [ ] `docker run` prints `Hello CoreOps Week 6!`
- [ ] No secrets appear in build logs

### Terraform
- [ ] `terraform init` downloads providers successfully
- [ ] `terraform validate` returns `Success! The configuration is valid.`
- [ ] `terraform fmt -check` passes (no formatting issues)
- [ ] No Terraform state files committed to repo

### Pipeline Health
- [ ] All steps show green checkmarks
- [ ] Total pipeline runs in under 3 minutes
- [ ] No hardcoded credentials anywhere in `.yml` files
- [ ] Workflow is triggered on both `push` and `pull_request`

---

## 10. Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `COPY failed: file not found` | Dockerfile context issue | Ensure `app.py` is in the same directory as Dockerfile |
| `Error: Unable to resolve action` | Wrong action name or version | Check GitHub Marketplace for correct name |
| `terraform: command not found` | Missing setup step | Add `hashicorp/setup-terraform@v3` before terraform commands |
| `Error: No configuration files found` | Wrong working directory | Use `terraform -chdir=infra` or set `working-directory` |
| `exit code 1` on fmt -check | Unformatted `.tf` files | Run `terraform fmt infra/` locally and commit |
| `docker: Cannot connect to Docker daemon` | Using wrong runner | Ensure `runs-on: ubuntu-latest` (not alpine-based) |
| `permission denied` on `docker build` | Rootless Docker | Add `sudo` or use GitHub-hosted runner |

---

## 11. GitLab CI Equivalent

The same pipeline in **GitLab CI** (`.gitlab-ci.yml`):

```yaml
# .gitlab-ci.yml

stages:
  - build
  - test
  - validate

variables:
  IMAGE_NAME: coreops-app

build-docker:
  stage: build
  image: docker:24
  services:
    - docker:24-dind              # Docker-in-Docker
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  script:
    - docker build -t $IMAGE_NAME .
    - docker run $IMAGE_NAME

terraform-validate:
  stage: validate
  image:
    name: hashicorp/terraform:1.9
    entrypoint: [""]
  script:
    - cd infra
    - terraform init
    - terraform fmt -check
    - terraform validate
```

### GitHub Actions vs GitLab CI Comparison
| Feature | GitHub Actions | GitLab CI |
|---------|---------------|-----------|
| Config file | `.github/workflows/*.yml` | `.gitlab-ci.yml` |
| Trigger keyword | `on:` | `rules:` / `only:` |
| Job grouping | `jobs:` | `stages:` + job definitions |
| Reusable units | Actions (marketplace) | CI/CD Components / `include:` |
| Docker-in-Docker | Built-in on `ubuntu-latest` | Requires `docker:dind` service |
| Secrets | `${{ secrets.NAME }}` | `$NAME` (CI/CD Variables) |
| Self-hosted runners | `self-hosted` label | GitLab Runner |

---

## 12. Revision Questions

Test your understanding — answer without looking:

1. What is the difference between a **job** and a **step** in GitHub Actions?
2. Why do we use `actions/checkout@v3` as the first step?
3. What does `terraform -chdir=infra init` do differently than `cd infra && terraform init`?
4. Why is `runs-on: ubuntu-latest` preferred over `runs-on: alpine`?
5. What happens if you `echo ${{ secrets.MY_KEY }}` in a step? Is it safe?
6. What is the purpose of `terraform fmt -check` in CI (not `terraform fmt`)?
7. How does GitHub Actions know to run a workflow — what triggers it?
8. What is the difference between `uses:` and `run:` in a step?
9. Why should you never run `terraform apply` in a basic CI job?
10. What does pinning an action to a SHA (instead of a tag) protect against?

### Answers
<details>
<summary>Click to reveal answers</summary>

1. A **job** is a group of steps running on one runner VM. A **step** is a single command or action within a job. Jobs run in parallel by default; steps run sequentially.
2. `actions/checkout` clones your repository code onto the runner. Without it, no files exist in the working directory.
3. `-chdir=infra` changes Terraform's working directory without changing the shell's `$PWD`. It's cleaner and more explicit in CI.
4. `ubuntu-latest` has Docker, curl, git, and common tools pre-installed. Alpine is minimal and would require installing everything.
5. GitHub automatically masks the secret value as `***` in logs, so it won't be exposed. But it's still bad practice — avoid echoing secrets at all.
6. `fmt -check` exits with code 1 if files are not formatted, **failing the pipeline**. `fmt` silently fixes files, which would change files on the runner but not in your repo.
7. The `on:` block defines triggers: `push`, `pull_request`, `schedule`, `workflow_dispatch`, etc.
8. `uses:` runs a pre-built GitHub Action (JavaScript or Docker-based). `run:` executes shell commands directly on the runner.
9. `terraform apply` makes real infrastructure changes. In CI without proper approval gates, it could destroy or misconfigure production resources.
10. Tags can be moved to point to different commits (tag hijacking supply chain attack). A full SHA is immutable and always points to the exact same code.

</details>

---

## 13. Key Takeaways

```
┌─────────────────────────────────────────────────────────────┐
│                    MONDAY SUMMARY                           │
├─────────────────────────────────────────────────────────────┤
│  ✓ GitHub Actions workflows live in .github/workflows/      │
│  ✓ Jobs run in parallel; steps run sequentially             │
│  ✓ ubuntu-latest has Docker pre-installed                   │
│  ✓ docker build + docker run = build and test in CI         │
│  ✓ hashicorp/setup-terraform installs Terraform on runner   │
│  ✓ terraform validate catches config errors early           │
│  ✓ terraform fmt -check enforces team formatting standards  │
│  ✓ NEVER hardcode secrets — use ${{ secrets.NAME }}         │
│  ✓ Pin action versions to prevent supply chain attacks      │
│  ✓ This pattern is the foundation of DevSecOps automation   │
└─────────────────────────────────────────────────────────────┘
```

### What This Pipeline Protects Against
| Risk | Protection |
|------|-----------|
| Broken Docker builds | `docker build` fails the pipeline |
| App not starting | `docker run` output checked |
| Invalid Terraform | `validate` + `fmt -check` |
| Leaked credentials | GitHub Secrets + masked logs |
| Tampered actions | Pin to SHA (advanced) |
| Unreviewed code | Trigger on `pull_request` |

---

*Next: Tuesday — Advanced pipeline features: caching, matrix builds, artifacts, and multi-environment deployments.*
