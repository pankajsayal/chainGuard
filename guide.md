# Chainguard Challenge — Complete Beginner's Guide & Demo Playbook

> This guide explains **every concept, every file, and every decision** in this project. After reading it, you should be able to demonstrate the project live and confidently answer any question.

---

## Table of Contents
1. [Core Concepts (What You Need to Know)](#1-core-concepts)
2. [Project File-by-File Breakdown](#2-project-files)
3. [The Three Dockerfiles — Why Three?](#3-the-three-dockerfiles)
4. [Security Tools Explained](#4-security-tools)
5. [Kubernetes Security](#5-kubernetes-security)
6. [Supply Chain Security (The Big Picture)](#6-supply-chain-security)
7. [Live Demo Script](#7-live-demo-script)
8. [Interview Questions & Answers](#8-interview-qa)
9. [Key Numbers to Remember](#9-key-numbers)
10. [Glossary](#10-glossary)

---

## 1. Core Concepts

### What is a Container?
A container is a lightweight, isolated package that includes your application and everything it needs to run (libraries, dependencies, config). Think of it as a "shipping container" for software — it runs the same everywhere.

### What is Docker?
Docker is the tool that **builds and runs** containers. You write a `Dockerfile` (a recipe), and Docker builds an **image** (the blueprint). When you run an image, it becomes a **container** (the live process).

```
Dockerfile (recipe) → Image (blueprint) → Container (running app)
```

### What is a Base Image?
Every Dockerfile starts with `FROM some-image`. That's the **base image** — the foundation your app is built on. Examples:
- `golang:1.25` — Full Debian Linux + Go compiler (~1.8GB). Has 756 known vulnerabilities.
- `alpine:3.21` — Tiny Linux (~7MB). Has 2-4 vulnerabilities.
- `cgr.dev/chainguard/static:latest` — **Distroless** (~2MB). Has **0 vulnerabilities**.

### What is "Distroless"?
A distroless image contains **only your application binary**. No shell, no package manager, no extra tools. If an attacker breaks into your app, there's nothing else in the container for them to use.

### What is Chainguard?
Chainguard is a company that builds **hardened, distroless container base images** with zero known CVEs. They rebuild images daily, patching vulnerabilities before they can be exploited. Their business model: companies pay for guaranteed zero-CVE base images.

### What is a CVE?
**Common Vulnerabilities and Exposures** — a standardized ID for a known security bug. Example: `CVE-2025-60876` is a specific bug in `busybox`. The more CVEs in your image, the more ways an attacker can exploit it.

### What is an SBOM?
**Software Bill of Materials** — a complete list of every component inside your container (OS packages, libraries, Go modules). Think of it as an "ingredient list" for software. It's required for compliance and helps you know exactly what you're shipping.

### What is Cosign?
Cosign is a tool for **cryptographically signing** container images. It proves:
1. **Who built the image** (authenticity)
2. **The image hasn't been tampered with** (integrity)

You sign with a **private key** (kept secret) and anyone can verify with the **public key** (shared).

### What is a Multi-Stage Build?
A Dockerfile pattern where you use one stage to **compile** your code, then copy only the resulting binary into a **smaller, cleaner** final image. The compiler and build tools are thrown away.

```dockerfile
# Stage 1: Build (big image with compiler)
FROM golang:1.25 AS builder
RUN go build -o myapp

# Stage 2: Run (tiny image, binary only)
FROM alpine:3.21
COPY --from=builder /app/myapp .
```

---

## 2. Project Files

### Source Code
| File | What It Does |
|---|---|
| `main.go` | The Go web server. Uses the Gin framework. Has two endpoints: `/` returns "Hello World!", `/healthz` returns `{"status":"healthy"}` |
| `go.mod` | Lists Go dependencies (like `package.json` for Node.js) |
| `go.sum` | Checksums of dependencies (ensures nobody tampered with them) |

### Dockerfiles
| File | Strategy | Image Size | CVEs |
|---|---|---|---|
| `Dockerfile.single` | Single-stage, full Go SDK | ~1.79 GB | 756 OS + 24 Go |
| `Dockerfile.alpine` | Multi-stage, Alpine runtime | ~32 MB | 4 OS + 24 Go |
| `Dockerfile.chainguard` | Multi-stage, distroless (Phase 1 / v1) | ~26.3 MB | 0 OS + 24 Go |
| `Dockerfile.chainguard` | Multi-stage, distroless (Phase 2 / v2) | ~26.3 MB | **0 total** |

### Security Artifacts
| File | What It Is |
|---|---|
| `scan-single-report.txt` | Grype vulnerability scan of the single-stage image |
| `scan-alpine-report.txt` | Grype vulnerability scan of the Alpine image |
| `scan-chainguard-report.txt` | Grype vulnerability scan — says "No vulnerabilities found" |
| `sbom-single.spdx.json` | Full ingredient list of the single-stage image (10.7MB — huge because it includes the entire Go SDK) |
| `sbom-alpine.spdx.json` | Ingredient list of the Alpine image (123KB) |
| `sbom-chainguard.spdx.json` | Ingredient list of the Chainguard image (930KB) |
| `cosign.pub` | Public key for verifying image signatures (safe to share) |
| `cosign.key` | **Private key** for signing — NEVER committed (gitignored) |

### Kubernetes
| File | What It Does |
|---|---|
| `k8s-deployment.yaml` | Deploys the Chainguard image to Kubernetes with all security hardening |

### Config & Docs
| File | What It Does |
|---|---|
| `.gitignore` | Prevents committing secrets (`cosign.key`) |
| `.dockerignore` | Prevents secrets from leaking into Docker image layers |
| `README.md` | Technical documentation of the project |
| `presentation-deck.html` | 22-slide Reveal.js presentation for the interview |

---

## 3. The Three Dockerfiles — Why Three?

This is the **core story** of the project. You're showing a progression: bad → better → best.

### Dockerfile.single (The Problem)
```dockerfile
FROM golang:1.25        # Full Debian OS + Go compiler
COPY . .                # Copy source code
RUN go build            # Compile the app
CMD ["./hello-app"]     # Run it
```
**What's wrong:** The final image contains the entire Go compiler, Debian packages, shells, package managers — everything an attacker needs. 756 CVEs. 1.79 GB.

**Why it exists:** This is what most beginners do. It works, but it's wildly insecure.

### Dockerfile.alpine (The Workaround)
```dockerfile
# Stage 1: Build
FROM golang:1.25 AS builder
RUN go build -o hello-app

# Stage 2: Run
FROM alpine:3.21       # Tiny Linux (7MB)
COPY --from=builder /app/hello-app .
CMD ["./hello-app"]
```
**What's better:** The compiler is thrown away. The final image is just Alpine Linux + your binary. 32MB. Only 2 unique CVEs.

**What's still wrong:** Alpine still has a shell (`/bin/sh`), a package manager (`apk`), and basic utilities like `busybox`. An attacker could still use these.

### Dockerfile.chainguard (The Solution)
```dockerfile
# Stage 1: Build
FROM golang:1.25 AS builder
RUN CGO_ENABLED=0 go build -o hello-app  # Static binary, no C dependencies

# Stage 2: Run
FROM cgr.dev/chainguard/static:latest    # Nothing but the binary
COPY --from=builder /app/hello-app .
ENTRYPOINT ["./hello-app"]
```
**Why it's best:**
- **No shell** — attacker can't run commands
- **No package manager** — attacker can't install tools
- **No utilities** — nothing to exploit
- **0 CVEs** — no known vulnerabilities
- **26.3 MB** — pulls in seconds during scaling

**Why `CGO_ENABLED=0`?** Go can link to C libraries. Setting this to 0 creates a **fully static binary** that doesn't need any external libraries. This is required for distroless images that have no C libraries.

**Why `ENTRYPOINT` instead of `CMD`?** `ENTRYPOINT` makes the container act as a fixed executable. `CMD` can be overridden. For distroless, there's no shell to override with anyway, but `ENTRYPOINT` is the best practice.

---

## 4. Security Tools

### Grype (Vulnerability Scanner)
```bash
grype go-app-single:v1 -o table
```
**What it does:** Scans every package inside a container image and checks it against known CVE databases.

**What to say:** "I used Grype because it's developed by Anchore and is an industry standard. It detects both OS-level vulnerabilities (like in Debian packages) and application-level vulnerabilities (like in Go modules)."

### Syft (SBOM Generator)
```bash
syft go-app-chainguard:v1 -o spdx-json --file sbom-chainguard.spdx.json
```
**What it does:** Creates a complete inventory of every component in the image — OS packages, Go modules, file checksums.

**Output format:** SPDX 2.3 JSON — an industry standard format recognized by NIST, the US government, and enterprise compliance frameworks.

**What to say:** "I chose SPDX-2.3 because it satisfies Executive Order 14028 requirements for software supply chain transparency. CycloneDX would also work, but SPDX has broader government adoption."

### Cosign (Image Signing)
```bash
# Generate keys
cosign generate-key-pair

# Sign an image
cosign sign --key cosign.key localhost:5000/go-app-chainguard:v1

# Verify a signature
cosign verify --key cosign.pub localhost:5000/go-app-chainguard:v1
```
**What it does:** Creates a cryptographic signature tied to the image's SHA256 digest. If anyone changes even one byte of the image, verification fails.

**What to say:** "Cosign is part of the Sigstore project, which Chainguard co-founded. In production, I'd use keyless signing with Fulcio and Rekor for full transparency log integration, but local key files work for this demo."

### Cosign Attestation (SBOM attached to image)
```bash
# Attach SBOM as attestation
cosign attest --key cosign.key --type spdxjson --predicate sbom-chainguard.spdx.json localhost:5000/go-app-chainguard:v1

# Verify attestation
cosign verify-attestation --key cosign.pub --type spdxjson localhost:5000/go-app-chainguard:v1
```
**What it does:** Binds the SBOM to the image cryptographically. Anyone pulling the image can also retrieve and verify its exact bill of materials.

**Why it matters:** It proves "this exact image contains exactly these components" — not just "we scanned it once." It's cryptographically verifiable at any time in the future.

---

## 5. Kubernetes Security

### The Deployment Manifest (`k8s-deployment.yaml`)

Every line in the security configuration has a reason:

```yaml
securityContext:
  runAsNonRoot: true              # Prevents running as root
  runAsUser: 65532                # Uses the built-in nonroot user
  readOnlyRootFilesystem: true    # Attacker can't write files
  allowPrivilegeEscalation: false # Attacker can't become root
  capabilities:
    drop:
      - ALL                       # Removes all Linux capabilities
```

**Why `runAsUser: 65532`?** This is the UID of the `nonroot` user built into Chainguard images. It has minimal permissions.

**Why `readOnlyRootFilesystem`?** If an attacker exploits the app, they can't write malware, cron jobs, or SSH keys to the filesystem.

**Why `drop: ALL` capabilities?** Linux capabilities are fine-grained root powers (like `NET_RAW`, `SYS_ADMIN`). Dropping all of them means the process can only do basic operations — no raw network packets, no mounting filesystems.

### Health Probes
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
readinessProbe:
  httpGet:
    path: /healthz
    port: 8080
```
**Liveness probe:** "Is the app still alive?" If it fails 3 times, Kubernetes **restarts** the pod.

**Readiness probe:** "Is the app ready to take traffic?" If it fails, Kubernetes **stops sending traffic** to that pod.

**Why `/healthz`?** We added this endpoint to `main.go`. It returns `{"status":"healthy"}` with HTTP 200. Kubernetes calls it every 10 seconds.

**Why not use Docker HEALTHCHECK for Chainguard?** Docker HEALTHCHECK requires a shell or tool (like `curl` or `wget`) to execute the check. Distroless images have neither. So we delegate health checking to Kubernetes probes instead.

### Resource Limits
```yaml
resources:
  requests:
    memory: "32Mi"
    cpu: "50m"
  limits:
    memory: "128Mi"
    cpu: "200m"
```
**Requests:** "I need at least this much." The scheduler uses this to place the pod on a node.

**Limits:** "Never exceed this." Prevents a single pod from consuming all cluster resources (denial of service).

### automountServiceAccountToken: false
By default, every pod gets a Kubernetes API token. This token can be used to query the Kubernetes API. If your app doesn't need it (most don't), disable it to prevent attackers from using it.

---

## 6. Supply Chain Security (The Big Picture)

### What is a Software Supply Chain?
The entire pipeline from source code to running container:
```
Source Code → Build → Image → Registry → Deployment
```

Each step is a potential attack point:
- **Source code:** Malicious dependency (typosquatting)
- **Build:** Compromised CI/CD pipeline
- **Image:** Tampered image in transit
- **Registry:** Malicious image pushed
- **Deployment:** Wrong image pulled

### How We Secured Each Step

| Step | Attack | Our Defense |
|---|---|---|
| Source Code | Malicious Go module | `go.sum` checksums verify integrity |
| Build | Bloated image with tools | Multi-stage build, distroless base |
| Image | Unknown components | SBOM (syft) documents everything |
| Image | Unpatched vulnerabilities | Grype scan shows 0 CVEs |
| Registry | Tampered image | Cosign signature proves authenticity |
| Registry | Unknown contents | SBOM attestation proves bill of materials |
| Deployment | Privilege escalation | K8s securityContext locks it down |

### The Trust Chain
```
cosign.key (private) ──signs──→ Image Digest (sha256:d85a...)
cosign.pub (public)  ──verifies──→ "Yes, this image is authentic"

cosign.key ──attests──→ SBOM + Image Digest
cosign.pub ──verifies──→ "Yes, this SBOM belongs to this image"
```

This is called **provenance**: you can prove who built what, when, and with which components.

---

## 7. Live Demo Script

### Quick Demo (5 minutes)

1. **Show the scan results side by side:**
```powershell
Get-Content scan-single-report.txt | Select-Object -First 10      # 756 OS CVEs
Get-Content scan-chainguard-report.txt                            # Phase 1: 0 OS CVEs, but 24 Go CVEs!
Get-Content scan-chainguard-v2-report.txt                         # Phase 2: "No vulnerabilities found" (0 total!)
```

2. **Show image sizes:**
```powershell
docker images | Select-String "go-app"
```
Say: "We went from 1.79GB to 26MB — a 98.5% reduction."

3. **Run the app:**
```powershell
docker run -d -p 8080:8080 --name demo go-app-chainguard:v1
curl http://localhost:8080         # Hello World!
curl http://localhost:8080/healthz # {"status":"healthy"}
docker stop demo; docker rm demo
```

4. **Verify the signature on Docker Hub:**
```powershell
docker run --rm gcr.io/projectsigstore/cosign:latest verify --key /dev/stdin docker.io/pankajsayal81/go-app-chainguard:v1
```
(Paste cosign.pub content when prompted, or mount the file)

5. **Show the SBOM:**
```powershell
Get-Content sbom-chainguard.spdx.json | Select-Object -First 20
```
Say: "This SBOM lists every component — Go modules like Gin, OS packages like tzdata and ca-certificates."

### If Asked to Show Kubernetes:
```powershell
# Make sure kind cluster is running
kind create cluster --name demo
kind load docker-image go-app-chainguard:v1 --name demo
kubectl apply -f k8s-deployment.yaml
kubectl get pods
kubectl describe pod -l app=secure-go-app  # Show securityContext
```

---

## 8. Interview Questions & Answers

### Q: Why didn't you use the original hello-melange-apko app?
**A:** "The original app is a pure stdlib Go binary with zero external dependencies. Its SBOM would be almost empty, which doesn't demonstrate real supply chain analysis. I deliberately chose a Gin-based server because it has real dependencies (validator, protobuf, crypto, etc.), producing a meaningful SBOM that reflects what teams actually deal with in production."

### Q: Why Grype and not Trivy?
**A:** "Both are excellent. I chose Grype because it's developed by Anchore and has native integration with Syft for SBOM-based scanning. Trivy is Aqua Security's tool and also works well, but the Grype+Syft pairing gave me a cleaner workflow with consistent data formats."

### Q: What's the difference between signing and attestation?
**A:** "Signing proves **who built the image** and that it **hasn't been tampered with**. Attestation goes further — it attaches structured metadata (like an SBOM) to the image and signs that too. So attestation answers: 'What's inside this image?' while signing answers: 'Is this image genuine?'"

### Q: Why cosign and not Notation or GPG?
**A:** "Cosign is part of Sigstore, which Chainguard co-founded. It's purpose-built for container signing with OCI registry support. GPG was designed for emails and files, not container workflows. Notation (from Microsoft/Notary v2) is fine but has less ecosystem adoption than Sigstore."

### Q: What would you do differently in production?
**A:** "Three things: (1) Use **keyless signing** with Fulcio/Rekor instead of local key files — this provides an immutable transparency log. (2) Add a **Kubernetes admission controller** (like Kyverno or Sigstore Policy Controller) that rejects unsigned images. (3) Integrate scanning into **CI/CD** so vulnerabilities are caught before images are published."

### Q: Why 0 CVEs with Chainguard but 756 with golang?
**A:** "The 756 CVEs are in **OS packages** bundled into the golang image — things like libpython, binutils, coreutils. Your app doesn't use any of them, but they're there because the base image is a full Debian system. Chainguard images contain **only what's needed** — no extra packages means no extra vulnerabilities."

### Q: Can you explain the SBOM format you used?
**A:** "SPDX 2.3 JSON. SPDX is maintained by the Linux Foundation and is an ISO standard (ISO/IEC 5962:2021). It lists every package, its version, checksums, and license. I chose it over CycloneDX because SPDX has broader adoption in government compliance (Executive Order 14028)."

### Q: What is a distroless image exactly?
**A:** "A distroless image contains no Linux distribution — no apt, no apk, no bash, no coreutils. It's just your application binary plus minimal runtime files (like TLS certificates and timezone data). The term was coined by Google, and Chainguard builds on this concept with their hardened images."

### Q: Why `CGO_ENABLED=0`?
**A:** "Go can optionally link to C libraries for things like DNS resolution or TLS. Setting CGO_ENABLED=0 forces Go to use its pure-Go implementations instead, producing a fully **static binary** that has zero external dependencies. This is required for distroless images because they don't have the C standard library (glibc/musl)."

### Q: What does `readOnlyRootFilesystem` actually prevent?
**A:** "If an attacker exploits a vulnerability in the app, they typically want to: write a reverse shell script, drop SSH keys, install crypto miners, or modify app configs. A read-only filesystem prevents all of these. The app can still write to mounted volumes if needed (like for logs), but the root filesystem is immutable."

### Q: What are Linux capabilities and why drop them all?
**A:** "Linux capabilities are fine-grained permissions that break up root's power into 40+ individual privileges. Examples: CAP_NET_RAW (send raw packets), CAP_SYS_ADMIN (mount filesystems), CAP_DAC_OVERRIDE (bypass file permissions). By dropping ALL, we ensure the container process has **minimum privileges**. If the app needed network binding on port 80, we'd add back only CAP_NET_BIND_SERVICE."

### Q: What's the difference between liveness and readiness probes?
**A:** "Liveness: 'Is the process still working?' If it fails, Kubernetes **kills and restarts** the pod. Example: the app entered a deadlock.

Readiness: 'Is the pod ready to serve requests?' If it fails, Kubernetes **removes the pod from the service's load balancer** but doesn't restart it. Example: the app is still initializing or waiting for a database connection."

### Q: Why did you use a local registry instead of Docker Hub?
**A:** "The challenge asked to deploy a local registry. Using registry:2 simulates a private registry that enterprises use internally. It also let me demonstrate the full workflow (push → sign → attest → verify) without needing internet access or cloud credentials. I also pushed to Docker Hub to make the images publicly accessible."

### Q: What is the `--allow-insecure-registry` flag?
**A:** "Our local registry runs on HTTP (not HTTPS). Cosign by default requires HTTPS for security. The `--allow-insecure-registry` flag tells cosign to accept HTTP connections. In production, you'd always use HTTPS with proper TLS certificates."

### Q: How would you integrate this into CI/CD?
**A:** "In a GitHub Actions pipeline:
1. `go build` → compile the binary
2. `docker build` → create the image
3. `grype` → scan for vulnerabilities (fail the pipeline if Critical/High found)
4. `syft` → generate SBOM
5. `docker push` → push to registry
6. `cosign sign` → sign with keyless (Fulcio + OIDC identity)
7. `cosign attest` → attach SBOM attestation
8. Deploy to K8s with admission controller that verifies signatures"

### Q: What is Sigstore?
**A:** "Sigstore is an open-source project (co-founded by Chainguard's CEO, Dan Lorenc) that provides free tools for signing, verifying, and protecting software. Its components:
- **Cosign**: Signs container images
- **Fulcio**: Issues short-lived signing certificates (keyless)
- **Rekor**: Public transparency log (proves a signature existed at a specific time)
This eliminates the need to manage long-lived signing keys."

---

## 9. Key Numbers to Remember

| Metric | Single-Stage | Alpine | Chainguard (v1) | Chainguard (v2) |
|---|---|---|---|---|
| **Image Size** | 1.79 GB | 32 MB | 26.3 MB | 26.3 MB |
| **CVEs (OS-Level)** | 756 | 4 | **0** | **0** |
| **CVEs (App-Level)** | 24 | 24 | 24 | **0** |
| **Total CVEs** | 780 | 28 | 24 | **0** |
| **Has Shell?** | Yes (bash) | Yes (sh) | **No** | **No** |
| **Has Pkg Manager?** | Yes (apt) | Yes (apk) | **No** | **No** |
| **SBOM Size** | 10.7 MB | 123 KB | 930 KB | 930 KB |

---

## 10. Glossary

| Term | Definition |
|---|---|
| **CVE** | Common Vulnerabilities and Exposures — a cataloged security bug |
| **SBOM** | Software Bill of Materials — ingredient list for software |
| **SPDX** | Software Package Data Exchange — an SBOM format (ISO standard) |
| **OCI** | Open Container Initiative — the standard for container images |
| **Distroless** | Container image with no OS distribution, just the app binary |
| **Multi-stage build** | Dockerfile pattern: compile in one stage, run in a minimal stage |
| **Cosign** | Tool for signing and verifying container images |
| **Sigstore** | Open-source project for software supply chain security |
| **Fulcio** | Certificate authority for keyless signing |
| **Rekor** | Transparency log for recording signatures |
| **Attestation** | Signed statement about an artifact (e.g., "this image has this SBOM") |
| **Provenance** | Proof of where software came from and how it was built |
| **CGO** | Go's interface to C code; disabling it makes pure Go binaries |
| **Static binary** | Executable with no external library dependencies |
| **Kind** | Kubernetes IN Docker — runs a K8s cluster inside Docker containers |
| **SecurityContext** | K8s config that restricts what a container process can do |
| **Capabilities** | Fine-grained Linux permissions (subset of root powers) |
| **Liveness probe** | K8s check: "Is the app alive?" Restart if not. |
| **Readiness probe** | K8s check: "Is the app ready?" Remove from load balancer if not. |
| **Registry** | Storage server for container images (Docker Hub, local registry:2) |
| **Digest** | SHA256 hash uniquely identifying a container image |
| **LOTL** | Living Off The Land — attacker uses tools already in the system |

---

*Good luck, Pankaj! 🚀*
