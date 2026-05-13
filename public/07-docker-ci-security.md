# Docker CI/CD Pipeline and Security Scanning

## A. Building a CI Pipeline for Docker Images

> A CI pipeline for Docker typically chains four stages: **test** (unit and integration tests against the source), **build** (assemble the image), **scan** (check for known vulnerabilities), and **push** (publish to a registry). The whole point of running them in this order — gated by `needs:` in GitHub Actions, `requires:` in CircleCI, or `dependencies:` in GitLab — is to prevent broken or vulnerable images from ever reaching the registry. If `scan` finds a critical CVE, `push` never runs.

### GitHub Actions CI Pipeline Example

1. Create a sample project structure.
```
mkdir ~/myciproject
cd ~/myciproject
git init
```

2. Create a simple Python application.
```
cat << 'EOF' > app.py
def greet(name):
    return f"Hello, {name}!"

def add(a, b):
    return a + b

if __name__ == "__main__":
    print(greet("World"))
EOF
```

3. Create a test file.
```
cat << 'EOF' > test_app.py
import unittest
from app import greet, add

class TestApp(unittest.TestCase):
    def test_greet(self):
        self.assertEqual(greet("Docker"), "Hello, Docker!")

    def test_add(self):
        self.assertEqual(add(2, 3), 5)

if __name__ == "__main__":
    unittest.main()
EOF
```

4. Create a Dockerfile.
```
cat << EOF > Dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY . .

RUN pip install --no-cache-dir pytest

CMD ["python", "app.py"]
EOF
```

5. Create the GitHub Actions workflow directory.
```
mkdir -p .github/workflows
```

6. Create the CI workflow file.
```
cat << 'EOF' > .github/workflows/ci.yml
name: Docker CI Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.9'

      - name: Run tests
        run: python -m pytest test_app.py -v

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build Docker image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Run container test
        run: |
          docker run --rm myapp:${{ github.sha }} python -c "from app import greet; print(greet('CI'))"

  scan:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build image for scanning
        run: docker build -t myapp:scan .

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'myapp:scan'
          format: 'table'
          exit-code: '1'
          severity: 'CRITICAL,HIGH'

  push:
    needs: [test, build, scan]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_TOKEN }}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: ${{ secrets.DOCKER_USERNAME }}/myapp:latest
EOF
```

> This workflow demonstrates the four key stages: test, build, scan, and push.

## B. Security Scanning with Trivy

> **Trivy** is an open-source vulnerability scanner that compares the packages installed in your image against public **CVE databases** (NVD, vendor advisories, GitHub's database, and others) — those databases list every publicly-disclosed vulnerability with a severity rating. Running Trivy in CI lets you catch a vulnerable dependency *before* it reaches production, rather than discovering it from a Slack alert at 3 AM. The `--exit-code 1` flag is what turns Trivy into a **CI gate**: a non-zero exit fails the job, which (via `needs:`) blocks the push step.

1. Install Trivy (if not using GitHub Actions).
```
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sudo sh -s -- -b /usr/local/bin
```

2. Scan a local Docker image.
```
docker pull nginx:latest
trivy image nginx:latest
```

3. Scan with specific severity levels.
```
trivy image --severity HIGH,CRITICAL nginx:latest
```

4. Generate a JSON report.
```
trivy image --format json --output report.json nginx:latest
```

5. Scan an image and fail on vulnerabilities.
```
trivy image --exit-code 1 --severity CRITICAL nginx:latest
```

> A non-zero exit code indicates vulnerabilities were found, useful for CI/CD pipeline gates.

## C. (OPTIONAL: REQUIRES DOCKER CLOUD LOGIN) - Docker Scout for Vulnerability Detection

> **Docker Scout** is Docker's first-party scanner, integrated directly into Docker Desktop and the CLI. Functionally it overlaps with Trivy: both query CVE databases against your image's package list. The practical difference is ecosystem — Scout is convenient if you're already on Docker Desktop, while Trivy is the de-facto open-source standard and works equally well outside it (in CI, against non-Docker registries, against filesystems and Kubernetes manifests). Most teams pick one; some run both for defense in depth.

1. Install and enable Docker Scout (Docker Desktop users).
```
curl -fsSL https://raw.githubusercontent.com/docker/scout-cli/main/install.sh -o install-scout.sh
sh install-scout.sh
```
```
docker scout quickview nginx:latest
```

2. Get detailed CVE information.
```
docker scout cves nginx:latest
```

3. Compare images for security improvements.
```
docker scout compare nginx:latest nginx:1.24-alpine
```

## D. Dockerfile Security Best Practices

> Three of the highest-leverage hardening practices: **pin specific base image versions** (so a `latest` tag floating to a compromised build can't poison your image overnight), **run as a non-root user** (so a process escape doesn't immediately get container-root capabilities), and **use multi-stage builds** (so the build toolchain and source code don't ship to production along with the binary). The examples below show each.

### Use Specific Base Image Versions
```
# Bad - unpredictable
FROM python:latest

# Good - specific and reproducible
FROM python:3.9.18-slim-bookworm
```

### Run as Non-Root User
```
cat << EOF > Dockerfile.secure
FROM python:3.9-slim

# Create non-root user
RUN groupadd -r appgroup && useradd -r -g appgroup appuser

WORKDIR /app

COPY --chown=appuser:appgroup . .

# Switch to non-root user
USER appuser

CMD ["python", "app.py"]
EOF
```

### Use Multi-Stage Builds
```
cat << EOF > Dockerfile.multistage
# Build stage
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o myapp

# Production stage - minimal image
FROM gcr.io/distroless/static-debian12
COPY --from=builder /app/myapp /
USER nonroot:nonroot
CMD ["/myapp"]
EOF
```

### Scan During Build
```
# Example: Hadolint for Dockerfile linting
docker run --rm -i hadolint/hadolint < Dockerfile.secure #or Dockerfile.multistage
```

## E. Using .dockerignore

> The `.dockerignore` file does for `docker build` what `.gitignore` does for `git`: it excludes files from the **build context** sent to the Docker daemon. There are two reasons to care. First, **secrets hygiene** — keep `.env`, credentials, and key files out of the image entirely, since image layers are recoverable forever. Second, **build speed** — smaller contexts upload faster, and excluding `node_modules` or `__pycache__` can cut a multi-gigabyte transfer to a few megabytes.

1. Create a .dockerignore file.
```
cat << EOF > .dockerignore
# Git
.git
.gitignore

# Environment files with secrets
.env
.env.*
*.env

# IDE files
.vscode
.idea

# Test files (for production images)
*_test.go
test_*.py
__pycache__

# Documentation
*.md
docs/

# CI/CD configs
.github
.gitlab-ci.yml
Jenkinsfile
EOF
```

## F. Secrets Management in Docker

> The cardinal rule: **never bake secrets into image layers**. Layers are content-addressed and cached — once a secret is in a layer, it's recoverable from the image even if a later `RUN rm` "deletes" it, and anyone who pulls the image gets a copy. The patterns below show three safer alternatives: Docker Swarm secrets (mounted as files under `/run/secrets/...` at runtime), runtime environment variables, and **BuildKit build secrets** (which mount a secret into a single `RUN` step without persisting it to any layer).

### Using Docker Secrets (Swarm Mode)
1. Initialize docker swarm
```
docker swarm init
```

3. Create a secret
```
# Create a secret
echo "mysecretpassword" | docker secret create db_password -
```

## G. (REFERENCE-ONLY): Image Signing and Verification

> Image signing answers the question, "is this image actually the one the maintainer published, or was it tampered with along the way?" **Docker Content Trust** (DCT) is Docker's original implementation, based on Notary v1. Most modern signing workflows have moved to **Sigstore / cosign**, which integrates with OIDC identities and a public transparency log (Rekor) — but DCT is still supported and is what's wired into the `docker push` command directly, so it remains a useful reference point.

1. Enable Docker Content Trust.
```
export DOCKER_CONTENT_TRUST=1
```

2. Push a signed image.
```
docker push myregistry/myimage:v1
```

> When DCT is enabled, Docker will prompt for signing keys when pushing.

## H. CI Pipeline Security Checklist

| **Check** | **Tool/Method** | **Stage** |
|-----------|-----------------|-----------|
| Lint Dockerfile | Hadolint | Build |
| Scan for vulnerabilities | Trivy, Docker Scout | Build |
| Check for secrets in code | git-secrets, truffleHog | Pre-commit |
| Static code analysis | SonarQube, Snyk | Test |
| Base image updates | Dependabot, Renovate | Continuous |
| Image signing | Docker Content Trust | Push |
| Runtime security | Falco, Sysdig | Deploy |

## I. Clean Up

```
cd ~
rm -rf myciproject
docker system prune -f
```
