# Docker CI/CD Pipeline and Security Scanning

## A. Building a CI Pipeline for Docker Images

> A CI pipeline for Docker typically includes: building images, running tests, scanning for vulnerabilities, and pushing to a registry.

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

> Trivy is a comprehensive security scanner for containers. It detects vulnerabilities in OS packages and application dependencies.

1. Install Trivy (if not using GitHub Actions).
```
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
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

## C. Docker Scout for Vulnerability Detection

> Docker Scout is Docker's native security scanning tool integrated into Docker Desktop and CLI.

1. Enable Docker Scout (Docker Desktop users).
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

> Prevent sensitive files from being included in the image.

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

> Never store secrets in Docker images. Use runtime secrets instead.

### Using Docker Secrets (Swarm Mode)
```
# Create a secret
echo "mysecretpassword" | docker secret create db_password -

# Use in a service
docker service create --secret db_password myapp
```

### Using Environment Variables at Runtime
```
# Pass secrets at runtime, not build time
docker run -e DB_PASSWORD="$DB_PASSWORD" myapp
```

### Using Docker BuildKit Secrets (Build Time)
```
cat << EOF > Dockerfile.buildsecret
FROM python:3.9-slim
RUN --mount=type=secret,id=pip_token \
    pip install --index-url https://\$(cat /run/secrets/pip_token)@pypi.example.com/simple/ mypackage
EOF
```
```
# Build with secret
docker build --secret id=pip_token,src=./pip_token.txt -t myapp .
```

## G. Image Signing and Verification

> Docker Content Trust enables image signing and verification.

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
