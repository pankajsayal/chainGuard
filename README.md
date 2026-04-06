# Strategic Security Assessment: Go Application Modernization

**Objective:** Remediate systemic vulnerabilities and secure the software supply chain using Chainguard distroless images, demonstrating an advanced progression from bloated single-stage containers to zero-CVE deployments.

> **Note:** This implementation is based on the Go application from the [chainguard-dev/hello-melange-apko](https://github.com/chainguard-dev/hello-melange-apko/tree/main/go) repository. The original Gin-based web server was updated with current dependency versions (Go 1.25, Gin v1.12.0) and enhanced with a `/healthz` health check endpoint to support Kubernetes liveness and readiness probes — a production best practice not present in the original demo.

---

## 1. Environment & Architecture
### Setup Decisions & Rationale
- **Operating System:** Windows with WSL2 / Docker Desktop. Rationale: Provides a fully compliant Linux-compatible execution environment without the overhead of maintaining a heavy standalone Linux VM, allowing fluid execution of the required native toolchains.
- **Local Kubernetes:** `kind` (Kubernetes IN Docker). Rationale: Extremely lightweight compared to Minikube or k3s for temporary verifications; it spins up in seconds and behaves identically to production for our testing schemas.
- **Local Registry:** `registry:2` (standard Docker Hub image). Rationale: Official stateless distribution. Allowed simulation of the complete supply chain (pushing, signing, and consuming images locally inside Kind) without managing cloud credentials.

### Tool Selection Justification
- **Vulnerability Scanner:** `grype`. Rationale: Developed by Anchore, it is heavily optimized to detect OS vulnerabilities and application-level dependencies rapidly, making it an industry standard.
- **SBOM Generator:** `syft`. Rationale: Possesses native synergy with Grype, is widely accepted, and effortlessly generates standard compliance formats (SPDX-2.3 JSON).
- **Signing Tool:** `cosign` (Sigstore). Rationale: The leading standard for container signing. Seamlessly integrates with OCI-compatible registries.

---

## 2. Deep Dive: Dockerfile Design Choices
We used three distinct containerization strategies to highlight the progression from worst to best practices:

### Security Findings and Two-Phase Remediation

Security is a multi-layered challenge. We demonstrated this by scanning the application in two distinct phases:

#### Phase 1: Fixing the OS Base (Original Dependencies)
Initially, we used the exact, outdated dependency versions from the reference repository (Go 1.18, Gin v1.8.1).

| Image Variant | OS CVEs | Go App CVEs | Total CVEs | Attack Surface |
| :--- | :--- | :--- | :--- | :--- |
| **Baseline (`v1`)** | 756 | 24 | **780** | Full Debian OS |
| **Alpine (`v1`)** | 4 | 24 | **28** | Minimal OS + shell |
| **Chainguard (`v1`)** | **0** | **24** | **24** | **Binary Only (Distroless)** |

> **Finding:** Migrating to Chainguard completely eliminated the 756 OS-level vulnerabilities. However, the original application's outdated dependencies inherently possessed 24 known Go-level vulnerabilities (e.g., `golang.org/x/net`, `golang.org/x/crypto`). Distroless fixes the infrastructure, but *application hygiene is equally critical*.

#### Phase 2: Zero-CVE Target (Updated Dependencies)
We updated the Go module dependencies to current secure versions (Go 1.25, Gin v1.12.0) and rebuilt the images.

| Image Variant | OS CVEs | Go App CVEs | Total CVEs |
| :--- | :--- | :--- | :--- |
| **Baseline (`v2`)** | 756 | 0 | **756** |
| **Alpine (`v2`)** | 4 | 0 | **4** |
| **Chainguard (`v2`)** | **0** | **0** | **0 (Zero CVEs)** |

> **Conclusion:** True "Zero CVE" status requires a defense-in-depth approach. By combining a hardened Distroless container image (Chainguard) with strict application dependency hygiene, we successfully eliminated the entire known attack surface.

---

## 3. Software Supply Chain Integrity & SBOM Insights
We moved beyond "security by obscurity" to **Security by Transparency**:

- **SBOM (Software Bill of Materials):** Generated via `syft` for all three container variants in SPDX-2.3 JSON format. The Chainguard SBOM successfully traces our components, explicitly proving that standard OS utility packages (`tzdata`, `ca-certificates-bundle`) coexist cleanly alongside our isolated `github.com/gin-gonic/gin` binary bindings. 
- **Cryptographic Provenance:** Used `cosign` to map cryptographic signatures against our built artifacts. Implementing signatures physically protects against malicious image injection between the build server and deployment cluster.
- **SBOM Attestations:** SBOM attestations are attached to container images via `cosign attest`, creating a verifiable link between the container digest and its bill of materials. Verification is performed with `cosign verify-attestation`.

> **Production note:** In a production environment, cosign keys would be managed via a KMS (e.g., AWS KMS, GCP KMS, or HashiCorp Vault) rather than local key files. The local file-based signing here is strictly for demonstration purposes.

---

## 4. Q&A and Production Challenges

### Production Challenges for Traditional Container Builds
- **Bloat & Attack Vectors:** Storing compilers and core Linux utilities in production images gives attackers immediate tools to escalate privileges or exfiltrate data (Living off the Land attacks) if they bypass application logic.
- **Slow Scalability:** Pulling 1.8GB images repeatedly severely slows down Kubernetes node scaling during traffic spikes compared to rapid 26MB distroless pulls.
- **Maintenance Nightmare:** Remediating 750+ vulnerabilities manually requires unfeasible operational overhead; distroless models shift these responsibilities left directly to the base maintainer (Chainguard).

### Kubernetes Security & Best Practices Implemented
- **Least Privilege Execution:** The application executes under a predefined restricted user (UID 65532).
- **Defense-in-Depth Manifest:** Pod configurations enforce `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem: true`, and explicitly `drop: [ALL]` capabilities.
- **Service Account Token Restriction:** `automountServiceAccountToken: false` prevents unnecessary API server access.
- **Health Probes:** Liveness and readiness probes target the `/healthz` endpoint, enabling Kubernetes to detect and recover from application failures automatically.
- **Resource Boundaries:** Explicit CPU and memory requests/limits prevent unbounded resource consumption.

**Kubernetes securityContext Example:**

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 65532
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
```

---

## 5. Pre-Built Images (Docker Hub)

All three images are published on Docker Hub for direct evaluation:

```bash
docker pull pankajsayal81/go-app-single:v2      # Baseline (756 CVEs)
docker pull pankajsayal81/go-app-alpine:v2      # Alpine (4 CVEs)
docker pull pankajsayal81/go-app-chainguard:v2  # Chainguard (0 CVEs) — Signed & Attested
```

The Chainguard image is signed and carries an SBOM attestation on Docker Hub:

```bash
cosign verify --key cosign.pub docker.io/pankajsayal81/go-app-chainguard:v2
cosign verify-attestation --key cosign.pub --type spdxjson docker.io/pankajsayal81/go-app-chainguard:v2
```

---

## 6. Quickstart: Build, Scan & Deploy

```bash
# ---------- 1. Build The Images ----------
docker build -f Dockerfile.single -t go-app-single:v2 .
docker build -f Dockerfile.alpine -t go-app-alpine:v2 .
docker build -f Dockerfile.chainguard -t go-app-chainguard:v2 .

# ---------- 2. Scanning ----------
grype go-app-single:v2 -o table > scan-single-report.txt
grype go-app-alpine:v2 -o table > scan-alpine-report.txt
grype go-app-chainguard:v2 -o table > scan-chainguard-report.txt

# ---------- 3. Generate SBOMs ----------
syft go-app-single:v2 -o spdx-json --file sbom-single.spdx.json
syft go-app-alpine:v2 -o spdx-json --file sbom-alpine.spdx.json
syft go-app-chainguard:v2 -o spdx-json --file sbom-chainguard.spdx.json

# ---------- 4. Sign & Push to Local Registry ----------
# Start a local registry (if not already running):
# docker run -d -p 5000:5000 --name registry registry:2

docker tag go-app-single:v2 localhost:5000/go-app-single:v2
docker tag go-app-alpine:v2 localhost:5000/go-app-alpine:v2
docker tag go-app-chainguard:v2 localhost:5000/go-app-chainguard:v2

docker push localhost:5000/go-app-single:v2
docker push localhost:5000/go-app-alpine:v2
docker push localhost:5000/go-app-chainguard:v2

# Setup Signing Key (production: use KMS instead of local keys)
export COSIGN_PASSWORD=""  # for local demo; use KMS in production
cosign generate-key-pair

# Sign all three images (--allow-insecure-registry for local HTTP registry)
cosign sign --yes --key cosign.key --allow-insecure-registry localhost:5000/go-app-single:v2
cosign sign --yes --key cosign.key --allow-insecure-registry localhost:5000/go-app-alpine:v2
cosign sign --yes --key cosign.key --allow-insecure-registry localhost:5000/go-app-chainguard:v2

# ---------- 4b. SBOM Attestation & Verification ----------
# Attach SBOM as attestation to each image
cosign attest --key cosign.key --allow-insecure-registry \
  --type spdxjson \
  --predicate sbom-single.spdx.json \
  localhost:5000/go-app-single:v2

cosign attest --key cosign.key --allow-insecure-registry \
  --type spdxjson \
  --predicate sbom-alpine.spdx.json \
  localhost:5000/go-app-alpine:v2

cosign attest --key cosign.key --allow-insecure-registry \
  --type spdxjson \
  --predicate sbom-chainguard.spdx.json \
  localhost:5000/go-app-chainguard:v2

# Verify signatures and attestations
cosign verify --key cosign.pub --allow-insecure-registry localhost:5000/go-app-chainguard:v2
cosign verify-attestation --key cosign.pub --allow-insecure-registry \
  --type spdxjson \
  localhost:5000/go-app-chainguard:v2

# ---------- 5. Standalone Validation ----------
docker run -d -p 8080:8080 --name test-app go-app-chainguard:v2
curl http://localhost:8080          # → "Hello World!"
curl http://localhost:8080/healthz  # → {"status":"healthy"}
docker stop test-app && docker rm test-app

# ---------- 6. Kubernetes Deployment ----------
# Ensure your kind cluster 'chainguard-challenge' is active
kind load docker-image go-app-chainguard:v2 --name chainguard-challenge
kubectl apply -f k8s-deployment.yaml
kubectl wait --for=condition=ready pod -l app=secure-go-app --timeout=60s

# Forward the port and test
kubectl port-forward svc/secure-go-app-service 9090:8080 &
curl http://localhost:9090          # → "Hello World!"
curl http://localhost:9090/healthz  # → {"status":"healthy"}
```

---

## 7. Conclusion
This exercise demonstrates that modern container security is not about incremental patching routines, but strict architectural decisions. By adopting minimal, distroless images and enforcing supply chain cryptographic integrity, we can build systems that are aggressively secure by design. Ultimately, security is not a feature we continuously add—it is an immutable property we trace and enforce throughout the entire DevOps lifecycle.

---

*Author: Pankaj Sayal*
