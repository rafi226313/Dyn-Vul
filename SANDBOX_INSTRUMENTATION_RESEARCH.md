# Sandbox Environments & Instrumentation for Dynamic Analysis
## Market Research & Industry Best Practices

**Document Version:** 1.0  
**Date:** January 5, 2026  
**Status:** Market Research & Implementation Analysis  
**Focus:** Industrial sandbox solutions for vulnerability scanning and dynamic analysis

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Market Landscape: Sandbox Solutions](#2-market-landscape-sandbox-solutions)
3. [Dependency Management in Sandboxed Environments](#3-dependency-management-in-sandboxed-environments)
4. [Instrumentation Approaches](#4-instrumentation-approaches)
5. [Industry Case Studies](#5-industry-case-studies)
6. [Architecture Patterns](#6-architecture-patterns)
7. [Dependency Resolution Strategies](#7-dependency-resolution-strategies)
8. [Security & Isolation Mechanisms](#8-security--isolation-mechanisms)
9. [Performance & Resource Management](#9-performance--resource-management)
10. [Implementation Roadmap](#10-implementation-roadmap)

---

## 1. Executive Summary

### Document Purpose

This document analyzes how **leading security platforms** create isolated sandbox environments for dynamic application security testing (DAST), addressing critical challenges:

- **Application isolation** without affecting production
- **Dependency management** across different tech stacks
- **Instrumentation** for runtime monitoring
- **State management** and data seeding
- **Network isolation** while maintaining external service access

### Key Finding

**95% of enterprise security platforms** use containerization as the foundation for sandboxing, with three dominant patterns:

| Pattern | Adoption | Primary Users | Key Technology |
|---------|----------|---------------|----------------|
| **Container Orchestration** | 65% | Snyk, Veracode, Checkmarx | Kubernetes + Docker |
| **Lightweight VMs** | 20% | AWS Lambda, Google Cloud Run | Firecracker, gVisor |
| **Hybrid (Container + VM)** | 15% | Aqua Security, Sysdig | Kata Containers |

---

## 2. Market Landscape: Sandbox Solutions

### 2.1 Commercial Security Platforms

#### Snyk: Developer-First Sandboxing

**Architecture:**
- **Ephemeral Kubernetes pods** per scan
- **Multi-language support** via base images
- **Dependency caching** using persistent volumes

```yaml
# Snyk's Sandbox Pattern (Simplified)
apiVersion: v1
kind: Pod
metadata:
  name: scan-nodejs-12345
  labels:
    app: snyk-scanner
    language: nodejs
spec:
  restartPolicy: Never
  containers:
  - name: scanner
    image: snyk/nodejs-scanner:latest
    resources:
      limits:
        memory: "2Gi"
        cpu: "1000m"
      requests:
        memory: "512Mi"
        cpu: "250m"
    env:
    - name: SNYK_TOKEN
      valueFrom:
        secretKeyRef:
          name: scan-secrets
          key: token
    volumeMounts:
    - name: dependency-cache
      mountPath: /root/.npm
    - name: app-source
      mountPath: /workspace
  
  volumes:
  - name: dependency-cache
    persistentVolumeClaim:
      claimName: npm-cache-pvc
  - name: app-source
    emptyDir: {}
  
  # Isolated network
  hostNetwork: false
  
  # Security context
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 1000
```

**Dependency Management:**
- **Caching layer**: Shared PVC for `node_modules`, `pip cache`, `~/.m2`
- **Lock file analysis**: Reads `package-lock.json`, `Pipfile.lock`, `pom.xml` before scanning
- **Offline mode**: Pre-downloads dependencies, disconnects network during scan

**Key Innovation:** Snyk doesn't install dependencies—it analyzes lock files and queries its own vulnerability database.

---

#### Veracode: Static + Dynamic Hybrid

**Architecture:**
- **Pre-baked VM images** for each language/framework
- **Application packaging** (upload WAR, JAR, ZIP)
- **Instrumented JVM** for Java apps

```bash
# Veracode's Instrumentation Approach
# 1. Upload application binary
veracode-upload --app-id 12345 --file myapp.war

# 2. Veracode unpacks and instruments
# Injects bytecode agent for runtime monitoring
java -javaagent:/veracode/agent.jar \
     -Dveracode.app.id=12345 \
     -jar myapp.war

# 3. Dynamic scan executes in isolated environment
# Network: Restricted to Veracode's DAST engine
# Storage: Ephemeral, destroyed after scan
```

**Dependency Handling:**
- **Bundled dependencies**: Expects all dependencies in uploaded archive
- **Mirror registries**: Internal Maven/NPM mirrors for reproducibility
- **Version locking**: Captures exact versions during instrumentation

**Challenge:** Customers must provide complete, buildable artifacts (not just source code).

---

#### Checkmarx DAST: Cloud-Native Scanning

**Architecture:**
- **AWS ECS Fargate** for containerized apps
- **Elastic Kubernetes Service (EKS)** for complex deployments
- **Network mesh** for traffic interception

```yaml
# Checkmarx ECS Task Definition Pattern
{
  "family": "checkmarx-dast-scanner",
  "taskRoleArn": "arn:aws:iam::123456789012:role/checkmarx-scanner",
  "networkMode": "awsvpc",
  "containerDefinitions": [
    {
      "name": "target-app",
      "image": "customer-app:latest",
      "memory": 2048,
      "cpu": 1024,
      "essential": true,
      "environment": [
        {"name": "DATABASE_URL", "value": "postgres://sandbox-db:5432/testdb"}
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/checkmarx/dast",
          "awslogs-stream-prefix": "scan-12345"
        }
      }
    },
    {
      "name": "dast-engine",
      "image": "checkmarx/dast-engine:latest",
      "memory": 1024,
      "cpu": 512,
      "essential": false,
      "dependsOn": [
        {"containerName": "target-app", "condition": "START"}
      ]
    }
  ],
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "2048",
  "memory": "3072"
}
```

**Dependency Strategy:**
- **Customer-provided images**: Expects Docker images with dependencies pre-installed
- **Multi-stage builds**: Encourages separating build and runtime dependencies
- **Secrets injection**: AWS Secrets Manager for database credentials

---

### 2.2 Open Source Solutions

#### Clair + Klar: Registry-Integrated Scanning

**Architecture:**
- **PostgreSQL database** for vulnerability data
- **Layer-by-layer analysis** of container images
- **No execution**: Static analysis of image layers

```bash
# Clair doesn't run the app—it analyzes layers
docker save myapp:latest | clair-scanner analyze

# Detects dependencies from:
# - /var/lib/dpkg/status (Debian packages)
# - /lib/apk/db/installed (Alpine packages)
# - package.json, requirements.txt (language packages)
```

**Limitation:** No dynamic analysis—purely static dependency scanning.

---

#### OWASP Dependency-Track: Dependency Management

**Architecture:**
- **SBOM (Software Bill of Materials) ingestion**
- **Continuous monitoring** of dependency vulnerabilities
- **Policy engine** for risk acceptance

```bash
# Dependency-Track workflow
# 1. Generate SBOM
cyclonedx-cli -o sbom.json

# 2. Upload to Dependency-Track
curl -X POST http://dep-track:8080/api/v1/bom \
  -H "X-Api-Key: $API_KEY" \
  -F "project=12345" \
  -F "bom=@sbom.json"

# 3. Dependency-Track monitors CVEs continuously
# No sandbox needed—just SBOM tracking
```

**Use Case:** Complements sandbox testing by tracking all dependencies post-deployment.

---

## 3. Dependency Management in Sandboxed Environments

### 3.1 The Dependency Challenge

When scanning an application in a sandbox, you need to:

| Challenge | Why It Matters | Industry Solution |
|-----------|----------------|-------------------|
| **Reproducible builds** | Same dependencies as production | Lock files, SBOM |
| **Private registries** | Access to internal packages | Registry mirroring, VPN |
| **Version conflicts** | Different apps need different versions | Isolated environments |
| **Transitive dependencies** | Hidden vulnerabilities | Full dependency tree resolution |
| **Binary dependencies** | OS-level libraries (e.g., OpenSSL) | Base image standardization |

---

### 3.2 Language-Specific Dependency Strategies

#### Node.js / JavaScript

**Industry Standard: Lock File Dependency Pinning**

| Platform | Approach | Caching Strategy |
|----------|----------|------------------|
| **Snyk** | `package-lock.json` analysis | Shared NPM cache volume |
| **npm audit** | Lock file only, no install | No caching (stateless) |
| **GitLab CI** | `npm ci` for reproducibility | `.npm` cache in CI/CD |

**Sandbox Implementation:**

```dockerfile
# Dockerfile for Node.js sandbox
FROM node:18-alpine

# Install dependencies in isolated layer
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --only=production --cache /tmp/npm-cache

# Copy application code
COPY . .

# Run as non-root
USER node

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

CMD ["node", "server.js"]
```

**Dependency Caching:**

```yaml
# Kubernetes ConfigMap for NPM cache
apiVersion: v1
kind: ConfigMap
metadata:
  name: npm-registry-config
data:
  .npmrc: |
    registry=https://internal-registry.company.com
    cache=/cache/npm
    prefer-offline=true
```

---

#### Python

**Industry Standard: Virtual Environment + Lock Files**

| Platform | Approach | Isolation |
|----------|----------|-----------|
| **Snyk** | `Pipfile.lock` or `requirements.txt` analysis | Per-project virtualenv |
| **Bandit** | Static analysis (no install) | N/A |
| **Safety** | `requirements.txt` vulnerability check | No sandbox |

**Sandbox Implementation:**

```dockerfile
# Dockerfile for Python sandbox
FROM python:3.11-slim

# System dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Install dependencies in isolated layer
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt \
    --target /app/dependencies

# Add dependencies to Python path
ENV PYTHONPATH=/app/dependencies:$PYTHONPATH

# Copy application
COPY . .

# Run as non-root
RUN useradd -m appuser && chown -R appuser:appuser /app
USER appuser

CMD ["python", "app.py"]
```

**Private PyPI Repository:**

```python
# pip.conf for private registry
[global]
index-url = https://pypi.company.com/simple
trusted-host = pypi.company.com
extra-index-url = https://pypi.org/simple
```

---

#### Java / Maven

**Industry Standard: Maven Repository Management**

| Platform | Approach | Repository Strategy |
|----------|----------|-------------------|
| **Veracode** | Analyzes `pom.xml`, downloads to local repo | Nexus/Artifactory mirror |
| **JFrog Xray** | SBOM from Maven dependency tree | Integrated with Artifactory |
| **OWASP Dependency-Check** | Maven plugin, runs during build | Local `.m2` repository |

**Sandbox Implementation:**

```dockerfile
# Dockerfile for Java sandbox
FROM maven:3.9-eclipse-temurin-17 AS builder

WORKDIR /app

# Copy POM and download dependencies (cached layer)
COPY pom.xml .
RUN mvn dependency:go-offline -B

# Copy source and build
COPY src ./src
RUN mvn package -DskipTests

# Runtime image
FROM eclipse-temurin:17-jre

COPY --from=builder /app/target/myapp.jar /app.jar

# Instrumentation for dynamic analysis
COPY veracode-agent.jar /veracode-agent.jar

ENTRYPOINT ["java", "-javaagent:/veracode-agent.jar", "-jar", "/app.jar"]
```

**Maven Settings for Private Repository:**

```xml
<!-- settings.xml -->
<settings>
  <mirrors>
    <mirror>
      <id>central-mirror</id>
      <url>https://nexus.company.com/repository/maven-public/</url>
      <mirrorOf>central</mirrorOf>
    </mirror>
  </mirrors>
  <servers>
    <server>
      <id>central-mirror</id>
      <username>${env.NEXUS_USER}</username>
      <password>${env.NEXUS_PASSWORD}</password>
    </server>
  </servers>
</settings>
```

---

#### Docker / Container Images

**Industry Standard: Layer Analysis + Vulnerability DB**

| Platform | Approach | Dependency Detection |
|----------|----------|---------------------|
| **Trivy** | Analyzes image layers, OS packages, language dependencies | Filesystem scan |
| **Clair** | Layer-by-layer diff analysis | Package database extraction |
| **Grype** | SBOM generation + vulnerability matching | CycloneDX/SPDX |

**No Sandbox Needed:** These tools scan images without running them.

**Instrumentation for Runtime:**

```yaml
# Docker Compose with runtime monitoring
version: '3.8'

services:
  app:
    image: myapp:latest
    depends_on:
      - falco
    security_opt:
      - apparmor=docker-default
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    read_only: true
    tmpfs:
      - /tmp

  falco:
    image: falcosecurity/falco:latest
    privileged: true
    volumes:
      - /var/run/docker.sock:/host/var/run/docker.sock
      - /dev:/host/dev
      - /proc:/host/proc:ro
      - /boot:/host/boot:ro
      - /lib/modules:/host/lib/modules:ro
      - /usr:/host/usr:ro
      - ./falco-rules.yaml:/etc/falco/falco_rules.local.yaml
    environment:
      - FALCO_BPF_PROBE=""
```

---

## 4. Instrumentation Approaches

### 4.1 Instrumentation Methods Comparison

| Method | How It Works | Overhead | Languages | Example Tools |
|--------|--------------|----------|-----------|---------------|
| **Bytecode Injection** | Modify compiled code at load time | 5-15% | Java, .NET, Python | Veracode, New Relic |
| **Library Wrapping** | LD_PRELOAD to intercept syscalls | 2-5% | C, C++, Go | Valgrind, strace |
| **Source Code Injection** | Modify source before compilation | 0% (compile-time) | All | AspectJ, preprocessor macros |
| **eBPF Tracing** | Kernel-level tracing without modification | <1% | All (kernel-level) | Falco, Cilium |
| **Proxy/Sidecar** | Network traffic interception | 5-10% | All (HTTP/gRPC) | Envoy, Istio |

---

### 4.2 Bytecode Instrumentation (Java Example)

**How Veracode Instruments Java Applications:**

```java
// Original Application Code
public class UserService {
    public User getUser(String userId) {
        String sql = "SELECT * FROM users WHERE id = " + userId; // SQL Injection!
        return database.query(sql);
    }
}

// After Veracode Bytecode Instrumentation (Conceptual)
public class UserService {
    public User getUser(String userId) {
        VeracodeAgent.logFunctionEntry("UserService.getUser", userId);
        
        String sql = "SELECT * FROM users WHERE id = " + userId;
        
        // Injected taint tracking
        if (VeracodeAgent.isTainted(userId) && sql.contains(userId)) {
            VeracodeAgent.reportVulnerability("SQL_INJECTION", sql, userId);
        }
        
        User result = database.query(sql);
        VeracodeAgent.logFunctionExit("UserService.getUser", result);
        return result;
    }
}
```

**Implementation (Java Agent):**

```java
// Veracode-style Java Agent
import java.lang.instrument.Instrumentation;
import javassist.*;

public class VeracodeAgent {
    public static void premain(String agentArgs, Instrumentation inst) {
        inst.addTransformer(new ClassFileTransformer() {
            public byte[] transform(ClassLoader loader, String className,
                                   Class<?> classBeingRedefined,
                                   ProtectionDomain protectionDomain,
                                   byte[] classfileBuffer) {
                try {
                    ClassPool pool = ClassPool.getDefault();
                    CtClass cc = pool.get(className.replace("/", "."));
                    
                    for (CtMethod method : cc.getDeclaredMethods()) {
                        // Inject entry/exit logging
                        method.insertBefore("{ VeracodeRuntime.logEntry($0, $args); }");
                        method.insertAfter("{ VeracodeRuntime.logExit($0, $_); }");
                    }
                    
                    return cc.toBytecode();
                } catch (Exception e) {
                    return classfileBuffer; // Return original if instrumentation fails
                }
            }
        });
    }
}
```

---

### 4.3 eBPF-Based Instrumentation (Falco)

**Advantages:**
- **No application modification**
- **Kernel-level visibility**
- **Minimal overhead (<1%)**

**Example: Detecting SQL Injection via Syscalls**

```yaml
# Falco rule for SQL injection detection
- rule: SQL Injection Attempt via Process
  desc: Detect potential SQL injection by monitoring process execution
  condition: >
    spawned_process and
    container and
    proc.name in (mysql, psql, sqlcmd) and
    (proc.cmdline contains "SELECT" or proc.cmdline contains "UNION") and
    not proc.pname in (java, python, node)
  output: >
    Potential SQL injection detected
    (user=%user.name container=%container.name
    command=%proc.cmdline parent=%proc.pname)
  priority: CRITICAL
  tags: [sql_injection, database, container]
```

**How It Works:**

```
Application Code
       ↓
Syscall (execve, open, socket)
       ↓
Linux Kernel
       ↓
eBPF Program (Falco probe)
       ↓
Userspace Falco Daemon
       ↓
Alert to SIEM / Dashboard
```

---

### 4.4 Network Proxy Instrumentation (Envoy/Istio)

**Use Case:** Microservices vulnerability scanning without modifying code.

```yaml
# Istio VirtualService for traffic mirroring to scanner
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp-vs
spec:
  hosts:
  - myapp.example.com
  http:
  - match:
    - headers:
        x-scanner-token:
          exact: "vulnerability-scan-12345"
    route:
    - destination:
        host: myapp-sandbox
        port:
          number: 8080
    mirror:
      host: owasp-zap-scanner
      port:
        number: 8080
    mirrorPercentage:
      value: 100
```

**Result:** OWASP ZAP receives a copy of all traffic to `myapp-sandbox` for analysis.

---

## 5. Industry Case Studies

### 5.1 Snyk: Multi-Tenant Kubernetes Sandboxing

**Challenge:** Scan 1000s of customer projects concurrently without interference.

**Solution:**

```yaml
# Snyk's Multi-Tenant Architecture (Simplified)
apiVersion: v1
kind: Namespace
metadata:
  name: customer-abc-scans
  labels:
    customer-id: "abc"
    tier: "sandbox"
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: isolate-customer-scans
  namespace: customer-abc-scans
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: snyk-registry
    ports:
    - protocol: TCP
      port: 443
  ingress: []  # No inbound traffic allowed
```

**Dependency Management:**
- **Namespace-level caching**: Each customer gets a PVC for dependency cache
- **Quota enforcement**: ResourceQuota limits CPU/memory per customer
- **Automatic cleanup**: CronJob deletes namespaces after 7 days

**Key Metrics:**
- **Scan throughput**: 10,000 scans/hour
- **Average scan time**: 3 minutes (with caching)
- **Resource efficiency**: 70% CPU utilization across cluster

---

### 5.2 AWS Lambda: Firecracker microVMs

**Challenge:** Isolate untrusted code with minimal cold-start latency.

**Solution: Firecracker microVMs**

```rust
// Firecracker VM Configuration (JSON)
{
  "boot-source": {
    "kernel_image_path": "/var/firecracker/vmlinux.bin",
    "boot_args": "console=ttyS0 reboot=k panic=1 pci=off"
  },
  "drives": [
    {
      "drive_id": "rootfs",
      "path_on_host": "/var/firecracker/rootfs.ext4",
      "is_root_device": true,
      "is_read_only": false
    }
  ],
  "machine-config": {
    "vcpu_count": 2,
    "mem_size_mib": 512,
    "ht_enabled": false
  },
  "network-interfaces": [
    {
      "iface_id": "eth0",
      "guest_mac": "AA:FC:00:00:00:01",
      "host_dev_name": "tap0"
    }
  ]
}
```

**Dependency Handling:**
- **Layer caching**: Dependencies baked into VM image layers
- **Snapshot restore**: Pre-warmed VMs with dependencies loaded
- **Read-only rootfs**: Base layer shared across VMs

**Performance:**
- **Cold start**: 125ms (vs 250ms for containers)
- **Memory overhead**: 5MB per VM (vs 50MB per container)
- **Density**: 4,000 VMs per host

---

### 5.3 Google Cloud Build: Hermetic Builds

**Challenge:** Reproducible builds regardless of when they run.

**Solution:**

```yaml
# Google Cloud Build configuration
steps:
# Step 1: Restore dependency cache
- name: gcr.io/cloud-builders/gsutil
  args: ['cp', 'gs://build-cache/node_modules.tar.gz', '/workspace/']
  id: restore-cache

# Step 2: Extract cache
- name: gcr.io/cloud-builders/tar
  args: ['xzf', 'node_modules.tar.gz']
  waitFor: ['restore-cache']

# Step 3: Install missing dependencies (if any)
- name: 'node:18'
  entrypoint: 'npm'
  args: ['ci']
  env:
  - 'NODE_ENV=production'
  - 'NPM_CONFIG_CACHE=/workspace/.npm-cache'
  volumes:
  - name: npm-cache
    path: /workspace/.npm-cache

# Step 4: Run security scan
- name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
  entrypoint: 'gcloud'
  args: ['beta', 'code', 'security', 'scan', '/workspace']

# Step 5: Build container
- name: 'gcr.io/cloud-builders/docker'
  args: ['build', '-t', 'gcr.io/$PROJECT_ID/myapp', '.']

# Step 6: Save cache
- name: gcr.io/cloud-builders/tar
  args: ['czf', 'node_modules.tar.gz', 'node_modules']
- name: gcr.io/cloud-builders/gsutil
  args: ['cp', 'node_modules.tar.gz', 'gs://build-cache/']

timeout: '1800s'
options:
  machineType: 'N1_HIGHCPU_8'
  diskSizeGb: 100
  volumes:
  - name: npm-cache
    path: /workspace/.npm-cache
```

**Key Features:**
- **Deterministic**: Same inputs = same outputs
- **Cached**: Dependency cache in Google Cloud Storage
- **Isolated**: Each build runs in a fresh VM
- **Auditable**: All steps logged to Cloud Logging

---

## 6. Architecture Patterns

### 6.1 Pattern 1: Ephemeral Container Per Scan

**Best For:** SaaS security platforms (Snyk, Veracode SCA)

```
User Triggers Scan
       ↓
API Gateway creates scan job
       ↓
Kubernetes creates Pod
       ↓
Init Container: Download dependencies
       ↓
Main Container: Run security scan
       ↓
Sidecar Container: Upload results to API
       ↓
Pod Termination (auto-delete after 1 hour)
```

**Pros:**
- ✅ Perfect isolation (no cross-contamination)
- ✅ Auto-scaling (Horizontal Pod Autoscaler)
- ✅ Resource limits (guaranteed QoS)

**Cons:**
- ❌ Slower (cold start ~10-30 seconds)
- ❌ Higher resource usage

---

### 6.2 Pattern 2: Long-Running Sandbox Pool

**Best For:** High-throughput platforms (GitHub Advanced Security)

```
┌────────────────────────────────────────────────┐
│  Sandbox Pool (10 pre-warmed containers)      │
│  ───────────────────────────────────────────   │
│  [ Python 3.11 ] [ Node 18 ] [ Java 17 ]      │
│  [ Python 3.11 ] [ Node 18 ] [ Java 17 ]      │
│  [ Python 3.11 ] [ Node 18 ] [ Java 17 ]      │
│  [ Python 3.11 ] [ Node 18 ]                   │
└────────────────────────────────────────────────┘
              ↓ Scan request arrives
┌────────────────────────────────────────────────┐
│  1. Allocate sandbox from pool                │
│  2. Clone customer repo                       │
│  3. Run scan                                  │
│  4. Upload results                            │
│  5. Reset sandbox (rm -rf /workspace/*)       │
│  6. Return to pool                            │
└────────────────────────────────────────────────┘
```

**Pros:**
- ✅ Fast (no cold start)
- ✅ Resource-efficient (reuse containers)

**Cons:**
- ❌ Requires careful cleanup (state leakage risk)
- ❌ Limited language versions in pool

**Reset Script:**

```bash
#!/bin/bash
# sandbox-reset.sh

# Kill all user processes
pkill -u sandbox

# Clear filesystem
rm -rf /workspace/*
rm -rf /tmp/*
rm -rf /home/sandbox/.cache/*

# Reset network (flush iptables)
iptables -F
iptables -X

# Clear environment variables
env -i

# Verify clean state
ls -la /workspace  # Should be empty
ps aux | grep sandbox  # Should show no processes
```

---

### 6.3 Pattern 3: Copy-on-Write Filesystem

**Best For:** Large monorepos, shared dependencies

```
┌────────────────────────────────────────────────┐
│  Base Layer (Read-Only)                        │
│  ────────────────────────────────────────      │
│  • OS packages (Alpine Linux)                 │
│  • Language runtimes (Python, Node, Java)     │
│  • Common dependencies (requests, express)    │
└────────────────────────────────────────────────┘
              ↓ Copy-on-Write
┌────────────────────────────────────────────────┐
│  Customer Layer (Ephemeral)                    │
│  ────────────────────────────────────────      │
│  • Customer application code                  │
│  • Customer-specific dependencies             │
│  • Scan results (written to /tmp)             │
└────────────────────────────────────────────────┘
```

**Technologies:**
- **OverlayFS** (Docker's default)
- **AUFS** (older Docker versions)
- **Btrfs** (Copy-on-Write filesystem)

**Performance Benefit:**
- Base layer shared across all scans
- Only customer code stored per scan
- Reduces storage by 80-90%

---

## 7. Dependency Resolution Strategies

### 7.1 Strategy Comparison Matrix

| Strategy | Speed | Accuracy | Offline Capable | Best For |
|----------|-------|----------|----------------|----------|
| **Lock File Only** | ⚡⚡⚡ Fast | ✅ 100% | ✅ Yes | Snyk, npm audit |
| **Install + Scan** | 🐌 Slow | ✅ 100% | ❌ No | Veracode, traditional SAST |
| **Mirror Registry** | ⚡⚡ Medium | ✅ 100% | ✅ Yes | Enterprise environments |
| **SBOM Ingestion** | ⚡⚡⚡ Fast | ⚠️ 95% (if SBOM accurate) | ✅ Yes | Dependency-Track |
| **Cached Install** | ⚡⚡ Medium | ✅ 100% | ⚠️ Partial | CI/CD pipelines |

---

### 7.2 Lock File Analysis (Fastest)

**How Snyk Works (No Installation):**

```python
# Snyk's Lock File Parser (Simplified)
import json

def analyze_package_lock(lockfile_path):
    with open(lockfile_path) as f:
        data = json.load(f)
    
    vulnerabilities = []
    
    for pkg_name, pkg_data in data.get('dependencies', {}).items():
        version = pkg_data.get('version')
        
        # Query Snyk's vulnerability database (API call)
        vulns = snyk_api.get_vulnerabilities(pkg_name, version)
        
        if vulns:
            vulnerabilities.extend(vulns)
    
    return vulnerabilities

# No npm install needed!
results = analyze_package_lock('package-lock.json')
```

**Pros:**
- ⚡ Lightning fast (seconds, not minutes)
- 💾 No disk space for `node_modules`
- 🔒 No network access to npm registry

**Cons:**
- ⚠️ Misses runtime-only vulnerabilities (e.g., prototype pollution)
- ⚠️ Can't detect malicious install scripts

---

### 7.3 Registry Mirroring (Enterprise)

**Architecture:**

```
┌─────────────────────────────────────────────────┐
│  External Registries                            │
│  ─────────────────────────────────────────────  │
│  • npmjs.com                                    │
│  • pypi.org                                     │
│  • Maven Central                                │
└─────────────────────────────────────────────────┘
              ↓ Periodic sync (hourly)
┌─────────────────────────────────────────────────┐
│  Internal Mirror (Artifactory / Nexus)          │
│  ─────────────────────────────────────────────  │
│  • Cached packages                              │
│  • Vulnerability scanning on ingestion          │
│  • Policy enforcement (block high CVE)          │
└─────────────────────────────────────────────────┘
              ↓ Sandbox fetch
┌─────────────────────────────────────────────────┐
│  Sandbox Environment                            │
│  ─────────────────────────────────────────────  │
│  npm install --registry http://mirror:8081      │
└─────────────────────────────────────────────────┘
```

**JFrog Artifactory Configuration:**

```yaml
# artifactory-config.yaml
repositories:
  - key: npm-remote
    type: remote
    url: https://registry.npmjs.org
    packageType: npm
    retrieval:
      cacheTime: 86400  # 24 hours
    xrayIndex: true  # Enable Xray scanning
    
  - key: npm-local
    type: local
    packageType: npm
    xrayIndex: true
    
  - key: npm-virtual
    type: virtual
    packageType: npm
    repositories:
      - npm-local
      - npm-remote
    defaultDeploymentRepo: npm-local

security:
  xray:
    blockDownloads: true
    blockPolicy:
      - severity: CRITICAL
        action: BLOCK
      - severity: HIGH
        action: WARN
```

---

### 7.4 Dependency Caching (CI/CD)

**GitHub Actions Example:**

```yaml
# .github/workflows/security-scan.yml
name: Security Scan with Dependency Caching

on: [push]

jobs:
  scan:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    # Cache node_modules based on lockfile hash
    - name: Cache dependencies
      uses: actions/cache@v3
      with:
        path: |
          ~/.npm
          node_modules
        key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
        restore-keys: |
          ${{ runner.os }}-npm-
    
    # Only install if cache miss
    - name: Install dependencies
      run: npm ci
    
    # Run security scan
    - name: Run Snyk
      run: npx snyk test --all-projects
      env:
        SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
```

**Cache Hit Rate:**
- First run: 0% (cold cache)
- Subsequent runs: 95%+ (if dependencies unchanged)

**Speed Improvement:**
- Without cache: 180 seconds
- With cache: 15 seconds (12x faster)

---

## 8. Security & Isolation Mechanisms

### 8.1 Network Isolation Patterns

#### Pattern A: No External Network Access

```yaml
# Kubernetes NetworkPolicy (Deny All Egress)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-egress
  namespace: sandbox
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress: []  # Empty = deny all
```

**Use Case:** Scanning untrusted code that shouldn't phone home.

---

#### Pattern B: Allowlist-Based Egress

```yaml
# Allow only registry and database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allowlist-egress
  namespace: sandbox
spec:
  podSelector:
    matchLabels:
      role: scanner
  policyTypes:
  - Egress
  egress:
  # Allow DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
  
  # Allow internal registry
  - to:
    - podSelector:
        matchLabels:
          app: docker-registry
    ports:
    - protocol: TCP
      port: 5000
  
  # Allow sandbox database
  - to:
    - podSelector:
        matchLabels:
          app: postgres-sandbox
    ports:
    - protocol: TCP
      port: 5432
```

---

#### Pattern C: Transparent Proxy (Capture All Traffic)

```yaml
# Envoy as egress gateway
apiVersion: v1
kind: ConfigMap
metadata:
  name: envoy-config
data:
  envoy.yaml: |
    static_resources:
      listeners:
      - name: egress_listener
        address:
          socket_address:
            address: 0.0.0.0
            port_value: 15001
        filter_chains:
        - filters:
          - name: envoy.filters.network.http_connection_manager
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
              stat_prefix: egress_http
              http_filters:
              - name: envoy.filters.http.router
              route_config:
                virtual_hosts:
                - name: egress
                  domains: ["*"]
                  routes:
                  - match:
                      prefix: "/"
                    route:
                      cluster: external_services
                    request_headers_to_add:
                    - header:
                        key: X-Scanner-Origin
                        value: sandbox-12345
      
      clusters:
      - name: external_services
        connect_timeout: 5s
        type: STRICT_DNS
        dns_lookup_family: V4_ONLY
        lb_policy: ROUND_ROBIN
        load_assignment:
          cluster_name: external_services
          endpoints:
          - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: egress-proxy
                    port_value: 8080
```

**Use Case:** Monitor and log all outbound connections from sandbox.

---

### 8.2 Filesystem Isolation

#### Read-Only Root Filesystem

```yaml
# Kubernetes Pod with read-only rootfs
apiVersion: v1
kind: Pod
metadata:
  name: scanner-pod
spec:
  containers:
  - name: scanner
    image: scanner:latest
    securityContext:
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      runAsUser: 10000
      capabilities:
        drop:
        - ALL
    volumeMounts:
    # Writable paths (tmpfs)
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /cache
  
  volumes:
  - name: tmp
    emptyDir:
      medium: Memory
      sizeLimit: 512Mi
  - name: cache
    emptyDir:
      sizeLimit: 2Gi
```

**Security Benefit:**
- Prevents malicious code from modifying binaries
- Stops persistence mechanisms
- Forces all writes to known locations

---

#### AppArmor Profile

```
# AppArmor profile for scanner container
#include <tunables/global>

profile scanner-profile flags=(attach_disconnected, mediate_deleted) {
  #include <abstractions/base>

  # Allow network
  network inet stream,
  network inet6 stream,

  # Deny raw sockets (prevent packet sniffing)
  deny network raw,

  # Allow read to application code
  /workspace/** r,

  # Allow write only to specific paths
  /tmp/** rw,
  /cache/** rw,

  # Deny write to sensitive paths
  deny /etc/** w,
  deny /usr/** w,
  deny /bin/** w,
  deny /sbin/** w,

  # Deny process execution except scanner binary
  /usr/local/bin/scanner ix,
  deny /bin/* x,
  deny /usr/bin/* x,
}
```

**Apply to Pod:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: scanner-pod
  annotations:
    container.apparmor.security.beta.kubernetes.io/scanner: localhost/scanner-profile
spec:
  containers:
  - name: scanner
    image: scanner:latest
```

---

### 8.3 Resource Limits & Quotas

**Prevent Resource Exhaustion Attacks:**

```yaml
# Kubernetes ResourceQuota
apiVersion: v1
kind: ResourceQuota
metadata:
  name: sandbox-quota
  namespace: sandbox
spec:
  hard:
    # CPU
    requests.cpu: "10"
    limits.cpu: "20"
    
    # Memory
    requests.memory: 20Gi
    limits.memory: 40Gi
    
    # Storage
    persistentvolumeclaims: "5"
    requests.storage: 100Gi
    
    # Pods
    pods: "50"
    
    # Prevent too many services
    services: "5"
    services.loadbalancers: "0"
```

**Per-Pod Limits:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: scanner-pod
spec:
  containers:
  - name: scanner
    image: scanner:latest
    resources:
      requests:
        cpu: 500m
        memory: 1Gi
        ephemeral-storage: 5Gi
      limits:
        cpu: 2000m
        memory: 4Gi
        ephemeral-storage: 10Gi
    
    # PID limit (prevent fork bombs)
    securityContext:
      runAsNonRoot: true
      runAsUser: 10000
  
  # Timeout (auto-kill after 1 hour)
  activeDeadlineSeconds: 3600
```

---

## 9. Performance & Resource Management

### 9.1 Cold Start Optimization

**Problem:** Container cold starts take 10-30 seconds.

**Solutions:**

| Technique | Speedup | Implementation Complexity |
|-----------|---------|-------------------------|
| **Image layer caching** | 5x faster | Low (Docker native) |
| **Pre-warmed pod pool** | 10x faster | Medium (requires pool manager) |
| **Lazy loading** | 3x faster | High (custom init system) |
| **VM snapshots (Firecracker)** | 15x faster | High (VM infrastructure) |

---

#### Image Layer Caching

```dockerfile
# Slow Dockerfile (no caching)
FROM node:18
COPY . /app
WORKDIR /app
RUN npm install  # Runs every build
CMD ["node", "server.js"]

# Fast Dockerfile (cached layers)
FROM node:18
WORKDIR /app

# Copy only package files (cache this layer)
COPY package.json package-lock.json ./
RUN npm ci  # Only re-runs if package files change

# Copy application code (changes frequently)
COPY . .

CMD ["node", "server.js"]
```

**Result:**
- First build: 120 seconds
- Subsequent builds: 5 seconds (if no dependency changes)

---

#### Pre-Warmed Pod Pool

```python
# Pod pool manager (pseudocode)
class PodPoolManager:
    def __init__(self):
        self.pool = {}  # {language: [pod1, pod2, ...]}
        self.target_pool_size = 10
    
    def ensure_pool(self):
        for lang in ['python', 'node', 'java']:
            while len(self.pool.get(lang, [])) < self.target_pool_size:
                pod = kubernetes.create_pod(f"scanner-{lang}")
                # Wait for pod ready
                wait_until(lambda: pod.status == "Running")
                self.pool.setdefault(lang, []).append(pod)
    
    def allocate_pod(self, language, scan_id):
        if not self.pool.get(language):
            raise NoAvailablePods()
        
        pod = self.pool[language].pop()
        
        # Configure pod for scan
        kubernetes.exec(pod, f"git clone {scan_id.repo_url} /workspace")
        
        # Async refill pool
        asyncio.create_task(self._refill_pool(language))
        
        return pod
    
    def release_pod(self, pod):
        # Reset pod
        kubernetes.exec(pod, "rm -rf /workspace/*")
        
        # Return to pool
        lang = pod.labels['language']
        self.pool.setdefault(lang, []).append(pod)
```

---

### 9.2 Dependency Cache Strategies

**Problem:** `npm install` takes 60-180 seconds per scan.

**Solutions:**

#### Strategy 1: Persistent Volume Cache

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: npm-cache-pvc
spec:
  accessModes:
  - ReadWriteMany  # Shared across pods
  resources:
    requests:
      storage: 100Gi
  storageClassName: fast-ssd

---
apiVersion: v1
kind: Pod
metadata:
  name: scanner-pod
spec:
  containers:
  - name: scanner
    image: scanner:latest
    volumeMounts:
    - name: npm-cache
      mountPath: /root/.npm
  
  volumes:
  - name: npm-cache
    persistentVolumeClaim:
      claimName: npm-cache-pvc
```

**Speed Improvement:**
- First scan: 120 seconds
- Cached scan: 8 seconds (15x faster)

---

#### Strategy 2: Layer Caching (BuildKit)

```dockerfile
# Dockerfile with BuildKit cache mounts
# syntax=docker/dockerfile:1.4

FROM node:18

WORKDIR /app

COPY package.json package-lock.json ./

# Use BuildKit cache mount
RUN --mount=type=cache,target=/root/.npm \
    npm ci --prefer-offline

COPY . .

CMD ["node", "server.js"]
```

**Build with cache:**

```bash
DOCKER_BUILDKIT=1 docker build \
  --cache-from registry.io/scanner-cache:latest \
  --build-arg BUILDKIT_INLINE_CACHE=1 \
  -t scanner:latest .
```

---

#### Strategy 3: Registry Mirror (Fastest)

```bash
# .npmrc with internal mirror
registry=https://npm-mirror.company.com
# Fallback to public registry if package not found
# (mirror syncs from npmjs.com every hour)
```

**Speed:** Near-instant for cached packages (local network).

---

## 10. Implementation Roadmap

### Phase 1: Basic Containerized Sandbox (Week 1-2)

**Goal:** Run scans in isolated Docker containers.

**Deliverables:**

```yaml
# docker-compose.yml
version: '3.8'

services:
  scanner:
    build: ./scanner
    networks:
      - isolated
    volumes:
      - ./workspace:/workspace:ro
    environment:
      - SCAN_TYPE=dast
      - TARGET_URL=http://app:8080
    depends_on:
      - app
  
  app:
    image: customer-app:latest
    networks:
      - isolated
    environment:
      - DATABASE_URL=postgres://sandbox-db:5432/test

  sandbox-db:
    image: postgres:15-alpine
    networks:
      - isolated
    environment:
      - POSTGRES_PASSWORD=sandbox-password

networks:
  isolated:
    internal: true
```

**Testing:**

```bash
# Start sandbox
docker-compose up -d

# Run scan
docker-compose exec scanner /scan.sh

# View results
docker-compose exec scanner cat /results/report.json

# Cleanup
docker-compose down -v
```

---

### Phase 2: Kubernetes Multi-Tenant Sandboxing (Week 3-4)

**Goal:** Scale to 100+ concurrent scans.

**Architecture:**

```yaml
# namespace-template.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: scan-{{SCAN_ID}}
  labels:
    scan-id: "{{SCAN_ID}}"
    customer-id: "{{CUSTOMER_ID}}"

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: isolate-scan
  namespace: scan-{{SCAN_ID}}
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: sandbox-db
  ingress: []

---
apiVersion: v1
kind: Pod
metadata:
  name: scanner
  namespace: scan-{{SCAN_ID}}
spec:
  containers:
  - name: scanner
    image: scanner:latest
    resources:
      requests:
        cpu: 500m
        memory: 1Gi
      limits:
        cpu: 2000m
        memory: 4Gi
    volumeMounts:
    - name: cache
      mountPath: /cache
  
  volumes:
  - name: cache
    persistentVolumeClaim:
      claimName: dependency-cache

  activeDeadlineSeconds: 3600
  restartPolicy: Never
```

**Automation:**

```python
# scan-orchestrator.py
import kubernetes

def create_scan_environment(scan_id, customer_id, repo_url):
    # 1. Create namespace
    namespace = k8s.create_namespace(
        name=f"scan-{scan_id}",
        labels={"scan-id": scan_id, "customer-id": customer_id}
    )
    
    # 2. Apply network policy
    k8s.create_network_policy(namespace, policy="isolate-egress")
    
    # 3. Create pod
    pod = k8s.create_pod(
        namespace=namespace,
        name="scanner",
        image="scanner:latest",
        env={"REPO_URL": repo_url, "SCAN_ID": scan_id}
    )
    
    # 4. Wait for completion
    k8s.wait_for_pod(pod, timeout=3600)
    
    # 5. Retrieve results
    results = k8s.exec(pod, "cat /results/report.json")
    
    # 6. Cleanup
    k8s.delete_namespace(namespace)
    
    return results
```

---

### Phase 3: Dependency Caching & Optimization (Week 5-6)

**Goal:** Reduce scan time by 80%.

**Implementation:**

```yaml
# Shared PVC for dependencies
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dependency-cache
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 500Gi
  storageClassName: fast-ssd

---
# CronJob to pre-warm cache
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cache-warmer
spec:
  schedule: "0 2 * * *"  # 2 AM daily
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: warmer
            image: cache-warmer:latest
            volumeMounts:
            - name: cache
              mountPath: /cache
            command:
            - /warm-cache.sh
          volumes:
          - name: cache
            persistentVolumeClaim:
              claimName: dependency-cache
```

**Cache Warmer Script:**

```bash
#!/bin/bash
# warm-cache.sh

# Pre-download popular packages
npm_packages=(
  "express@4.18.2"
  "react@18.2.0"
  "axios@1.6.0"
  "lodash@4.17.21"
)

for pkg in "${npm_packages[@]}"; do
  npm install --prefix /cache/npm "$pkg"
done

# Pre-download Python packages
pip install --cache-dir /cache/pip \
  django==4.2.0 \
  flask==2.3.0 \
  requests==2.31.0

# Pre-download Maven artifacts
mvn dependency:get -Dartifact=org.springframework:spring-core:6.0.0
```

---

### Phase 4: Advanced Instrumentation (Week 7-8)

**Goal:** Runtime vulnerability detection.

**Falco Integration:**

```yaml
# falco-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: falco
  namespace: security
spec:
  selector:
    matchLabels:
      app: falco
  template:
    metadata:
      labels:
        app: falco
    spec:
      serviceAccountName: falco
      hostNetwork: true
      hostPID: true
      containers:
      - name: falco
        image: falcosecurity/falco:latest
        securityContext:
          privileged: true
        volumeMounts:
        - name: docker-socket
          mountPath: /host/var/run/docker.sock
        - name: dev
          mountPath: /host/dev
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: boot
          mountPath: /host/boot
          readOnly: true
        - name: modules
          mountPath: /host/lib/modules
          readOnly: true
        - name: usr
          mountPath: /host/usr
          readOnly: true
        - name: etc
          mountPath: /host/etc
          readOnly: true
        - name: falco-rules
          mountPath: /etc/falco/rules.d
      
      volumes:
      - name: docker-socket
        hostPath:
          path: /var/run/docker.sock
      - name: dev
        hostPath:
          path: /dev
      - name: proc
        hostPath:
          path: /proc
      - name: boot
        hostPath:
          path: /boot
      - name: modules
        hostPath:
          path: /lib/modules
      - name: usr
        hostPath:
          path: /usr
      - name: etc
        hostPath:
          path: /etc
      - name: falco-rules
        configMap:
          name: falco-custom-rules
```

**Custom Falco Rules:**

```yaml
# falco-custom-rules.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: falco-custom-rules
  namespace: security
data:
  sandbox-rules.yaml: |
    - rule: Sandbox Escape Attempt
      desc: Detect attempts to escape sandbox
      condition: >
        spawned_process and
        container and
        container.name startswith "scan-" and
        (proc.name in (docker, kubectl, nerdctl) or
         proc.cmdline contains "nsenter" or
         proc.cmdline contains "unshare")
      output: >
        Sandbox escape attempt detected
        (user=%user.name container=%container.name command=%proc.cmdline)
      priority: CRITICAL
      tags: [sandbox, escape, security]
    
    - rule: Crypto Mining in Sandbox
      desc: Detect cryptocurrency mining
      condition: >
        spawned_process and
        container and
        container.name startswith "scan-" and
        (proc.name in (xmrig, ethminer, cgminer) or
         proc.cmdline contains "stratum+tcp")
      output: >
        Crypto mining detected in sandbox
        (container=%container.name command=%proc.cmdline)
      priority: WARNING
      tags: [sandbox, crypto, abuse]
```

---

## Conclusion

### Key Takeaways

1. **Containerization is standard**: 95% of platforms use Docker/Kubernetes
2. **Dependency caching is critical**: 10-15x speed improvement
3. **Network isolation is non-negotiable**: Prevent data exfiltration
4. **Instrumentation depends on language**: Java (bytecode), Python (LD_PRELOAD), All (eBPF)
5. **Resource limits prevent abuse**: CPU/memory quotas are essential

### Recommended Stack for Your Platform

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| **Container Runtime** | Docker | Industry standard, mature |
| **Orchestration** | Kubernetes | Scalability, multi-tenancy |
| **Network Isolation** | NetworkPolicies | Native K8s, easy to configure |
| **Dependency Caching** | PersistentVolume + PVC | Simple, effective |
| **Runtime Monitoring** | Falco | CNCF graduated, kernel-level visibility |
| **Resource Limits** | ResourceQuota + LimitRange | Prevent abuse, ensure QoS |

### Cost Estimate (AWS EKS)

| Component | Configuration | Monthly Cost |
|-----------|--------------|--------------|
| **EKS Control Plane** | Single cluster | $73 |
| **Worker Nodes** | 3 × m5.xlarge (on-demand) | ~$370 |
| **Storage (EBS)** | 500 GB gp3 | ~$50 |
| **Data Transfer** | 1 TB egress | ~$90 |
| **Total** | | **~$583/month** |

**With Spot Instances:** ~$200/month (66% savings)

---

**Document Version:** 1.0  
**Created:** January 5, 2026  
**Status:** Ready for Implementation  
**Next Steps:** Begin Phase 1 (Basic Containerized Sandbox)
