## Trivy — Very Basic Practice Lab

[Trivy](https://trivy.dev/?utm_source=chatgpt.com) is an open-source security scanner from Aqua Security. For a beginner lab, focus on **container image scanning** and **filesystem/project scanning**. Trivy can scan for vulnerabilities, misconfigurations, secrets, and other security issues. ([Trivy][1])

### 1. Architecture

```text
Developer
   |
   v
Application Code
   |
   v
Docker Build
   |
   v
Docker Image
   |
   v
+-------------+
|    Trivy    |
| Security    |
|   Scan      |
+-------------+
   |
   v
Vulnerabilities
CRITICAL / HIGH / MEDIUM / LOW
```

### 2. Install Trivy

For **macOS**:

```bash
brew install trivy
```

Verify:

```bash
trivy --version
```

Homebrew is an officially documented installation method for macOS/Linux. ([Trivy][2])

### 3. First Practice — Scan an Existing Docker Image

Make sure Docker Desktop is running.

```bash
docker pull nginx
```

Check:

```bash
docker images
```

Now scan:

```bash
trivy image nginx
```

Basic flow:

```text
Docker Hub
    |
    v
nginx image
    |
    v
  Trivy
    |
    v
Vulnerability Report
```

Look at these fields in the output:

```text
Library
Vulnerability
Severity
Installed Version
Fixed Version
```

### 4. Show Only HIGH and CRITICAL

Instead of looking through every vulnerability:

```bash
trivy image --severity HIGH,CRITICAL nginx
```

This is much easier to demonstrate to students because you can focus on the important findings.

### 5. Scan Your Own Application

Example:

```bash
git clone https://github.com/atulkamble/FlaskApp-ACR-ACI.git
cd FlaskApp-ACR-ACI
```

Build the image:

```bash
docker build -t flaskapp:v1 .
```

Check:

```bash
docker images
```

Scan:

```bash
trivy image flaskapp:v1
```

Only important vulnerabilities:

```bash
trivy image --severity HIGH,CRITICAL flaskapp:v1
```

### 6. Scan Source Code / Project Directory

Trivy also supports scanning a local filesystem/project. The basic syntax is `trivy fs PATH`. ([Trivy][3])

Inside the project:

```bash
trivy fs .
```

Or:

```bash
trivy filesystem .
```

This is useful **before building the Docker image**.

```text
Source Code
    |
    +---- trivy fs .
    |
    v
Docker Build
    |
    v
Docker Image
    |
    +---- trivy image flaskapp:v1
```

### 7. Simple DevSecOps Workflow

The key idea for students is:

```text
Developer
   |
   v
GitHub
   |
   v
Source Code
   |
   +----> Trivy FS Scan
   |
   v
Docker Build
   |
   v
Docker Image
   |
   +----> Trivy Image Scan
   |
   v
Container Registry
   |
   v
Kubernetes / EKS
```

**Trivy introduces security scanning before deployment.**

### 8. Commands to Remember

```bash
# Check installation
trivy --version

# Scan public/local container image
trivy image nginx

# HIGH and CRITICAL only
trivy image --severity HIGH,CRITICAL nginx

# Scan project
trivy fs .

# Build your application
docker build -t flaskapp:v1 .

# Scan your application image
trivy image flaskapp:v1

# Important findings only
trivy image --severity HIGH,CRITICAL flaskapp:v1
```

### Basic 10–15 Minute Student Demo

Use this exact sequence:

```bash
brew install trivy

trivy --version

docker pull nginx

docker images

trivy image nginx

trivy image --severity HIGH,CRITICAL nginx

git clone https://github.com/atulkamble/FlaskApp-ACR-ACI.git

cd FlaskApp-ACR-ACI

trivy fs .

docker build -t flaskapp:v1 .

trivy image flaskapp:v1

trivy image --severity HIGH,CRITICAL flaskapp:v1
```

**Points to remember:** `trivy fs .` scans the project/filesystem, while `trivy image <image>` scans a container image. In a CI/CD pipeline, you can use the scan as a security gate before pushing or deploying an image. ([Trivy][3])

[Official Trivy documentation](https://trivy.dev/docs/?utm_source=chatgpt.com)

[1]: https://trivy.dev/docs/dev/getting-started/?utm_source=chatgpt.com "First steps - Trivy"
[2]: https://www.trivy.dev/docs/latest/getting-started/installation/?utm_source=chatgpt.com "Installation - Trivy"
[3]: https://www.trivy.dev/docs/latest/guide/references/configuration/cli/trivy_filesystem/?utm_source=chatgpt.com "Filesystem - Trivy"
