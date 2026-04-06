# Presentation Script — How to Walk Through Your Solution

> This is your step-by-step speaking guide. Read it before the interview. It tells you **what to say, when to say it, and why**. It's written as if you're talking to the interviewer.

---

## Opening (2 minutes)

### What to Say:
> "Thank you for the opportunity. I'm going to walk you through how I approached this challenge — not just the final result, but the decisions I made along the way and the reasoning behind them. I structured my solution around three themes: **progressive hardening**, **supply chain transparency**, and **production readiness**."

### Why This Opening Works:
Interviewers at Chainguard care more about **how you think** than whether you completed every task. This opening signals that you'll explain your reasoning, not just show code.

---

## Part 1: Environment Setup (3 minutes)

### What to Present:
> "Before writing any code, I had to make infrastructure decisions. Let me walk you through the choices I made and what I considered."

### Decision 1: Windows + WSL2 + Docker Desktop

**What I considered:**
- Option A: Spin up a full Linux VM (VirtualBox/VMware)
- Option B: Use a cloud VM (EC2/GCE)
- Option C: Use Windows with WSL2 + Docker Desktop

**What I chose and why:**
> "I went with **Option C — WSL2**. The challenge says 'spin up a Linux VM,' but WSL2 **is** a real Linux kernel running inside a lightweight VM managed by Windows. I chose this because:
>
> 1. It provides a **fully compliant Linux execution environment** — Docker runs natively in the WSL2 Linux kernel, not in Windows emulation.
> 2. It avoids the overhead of managing a separate VM with snapshots, SSH, etc.
> 3. Docker Desktop seamlessly bridges Windows and WSL2, so I can use PowerShell or bash interchangeably.
>
> If this were a production environment, I'd use a proper Linux server, but for a take-home challenge, WSL2 gives identical results with less friction."

**Potential interview follow-up:** "Is WSL2 really Linux?"
> "Yes. WSL2 runs a real Linux kernel (Microsoft-maintained) in a lightweight Hyper-V VM. Docker recognizes it as native Linux. Container builds, networking, and file operations are identical to bare-metal Linux."

### Decision 2: kind vs Minikube vs k3s

**What I considered:**
| Option | Pros | Cons |
|---|---|---|
| **Minikube** | Full-featured, well-documented | Heavy, spins up a full VM, slower startup |
| **k3s** | Lightweight, production-like | Requires systemd, more complex setup on WSL2 |
| **kind** | Ultra-lightweight, runs inside Docker itself | Not for production |

**What I chose and why:**
> "I chose **kind** (Kubernetes IN Docker). For this challenge, I don't need a production-grade cluster — I need a fast, disposable environment to verify my deployment manifests. Kind spins up in under 10 seconds, runs entirely inside Docker (no extra VM), and behaves identically to real Kubernetes for deployment testing."

**What to say if asked "Would you use kind in production?":**
> "Absolutely not. Kind is for testing and CI/CD. In production, I'd use EKS, GKE, or a managed Kubernetes service with proper node pools, network policies, and cluster autoscaling."

### Decision 3: Local Registry (registry:2)

**What I chose and why:**
> "I used the standard Docker `registry:2` image on port 5000. I chose this because:
>
> 1. It's the **official, stateless OCI distribution** — it mirrors how real private registries work.
> 2. It let me simulate the complete supply chain locally: build → push → sign → attest → pull → verify.
> 3. No cloud credentials or internet dependency — the entire pipeline runs offline.
>
> I also pushed to Docker Hub as a secondary registry for easy evaluation."

---

## Part 2: Application Choice (2 minutes)

### The Most Important Decision to Explain

> "The challenge references the Go app from `chainguard-dev/hello-melange-apko`. I looked at it and made a deliberate decision to **not use it as-is**. Let me explain why."

**What the original app looks like:**
```go
package main
import "net/http"
func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintln(w, "Hello World!")
    })
    http.ListenAndServe(":8080", nil)
}
```

**The problem with it:**
> "This app uses only the Go standard library. It has **zero external dependencies**. That means:
> - Its SBOM would list only the Go stdlib — essentially empty
> - There's nothing interesting for Grype to analyze at the application level
> - It doesn't represent a real-world scenario
>
> For a Solutions Engineer role at Chainguard, I wanted to demonstrate that I understand what **real teams face**: applications with dozens of dependencies, complex SBOMs, and meaningful supply chain analysis."

**What I did instead:**
> "I built a web server using the **Gin framework** — the most popular Go web framework with 80k+ GitHub stars. This gave me:
> - **17 transitive dependencies** (validator, protobuf, crypto, etc.)
> - A **meaningful SBOM** showing real Go modules alongside OS packages
> - A more **realistic attack surface** to analyze
>
> I also added a `/healthz` endpoint for Kubernetes health probes — something production apps always need."

**Why this matters to Chainguard:**
> "Chainguard's customers aren't running Hello World binaries. They're running Gin, Echo, gRPC, and microservice apps with real dependency chains. By choosing a framework with actual dependencies, I'm showing that I understand the customer's reality."

---

## Part 3: The Three Dockerfiles — The Core Story (5 minutes)

### How to Frame This:

> "The heart of my solution is a **three-tier progression** that tells a story. I built three Dockerfiles, each one representing a different security maturity level. Think of it as: **The Problem → The Workaround → The Solution.**"

### Tier 1: Dockerfile.single — "The Problem" (The Baseline)

> "I started with the worst-case scenario on purpose. This is what happens when a developer doesn't think about security."

**What I did:**
```dockerfile
FROM golang:1.25
WORKDIR /app
COPY . .
RUN go mod download && go build -o hello-app .
CMD ["./hello-app"]
```

**The approach I considered:**
> "I could have skipped this and gone straight to the good solution. But I needed a **baseline for comparison**. Without showing how bad the default is, the hardened version has no context."

**The consequences:**
> "When I scanned this image with Grype, it found **756 known CVEs**. The image is **1.79 GB**. Here's why:
>
> The `golang:1.25` base image is built on Debian Bookworm. It includes the entire Go compiler toolchain, Python, C compilers, binutils, coreutils — hundreds of packages my app never uses. Each one is a potential attack vector.
>
> For example, if an attacker breaks into this container, they find:
> - `/bin/bash` — a full shell to run commands
> - `apt-get` — they can install any tool they want
> - `gcc` — they can compile exploits on the spot
> - `python3` — they can run scripts
>
> This is called a **Living Off The Land (LOTL) attack** — the attacker doesn't need to bring their own tools because the container already provides them."

**Key talking point:**
> "756 CVEs doesn't mean my app is broken. It means the OS packages surrounding my app have known vulnerabilities. Someone could exploit any one of them if they get into the container."

### Tier 2: Dockerfile.alpine — "The Workaround"

> "Now I applied the most common industry fix: multi-stage builds with an Alpine base."

**The approach I took:**
```dockerfile
# Stage 1 — Build (use the big image for compiling)
FROM golang:1.25 AS builder
WORKDIR /app
COPY . .
RUN go mod download && go build -o hello-app .

# Stage 2 — Run (use a tiny image for runtime)
FROM alpine:3.21
RUN adduser -D -u 65532 nonroot
WORKDIR /app
COPY --from=builder /app/hello-app .
USER nonroot
ENTRYPOINT ["./hello-app"]
```

**Why I chose this approach:**
> "Multi-stage builds are the **industry standard best practice**. The key insight is: you don't need the compiler at runtime. Phase 1 compiles the binary; Phase 2 starts fresh with just Alpine and copies only the binary.
>
> Result: **32 MB** image (98.2% reduction), only **2 unique CVEs** (99.7% reduction)."

**What I considered but rejected:**
> "I could have used Debian Slim (`debian:bookworm-slim`) instead of Alpine. It would be larger (~80MB) with more CVEs (~20-30). Alpine wins because it uses `musl` instead of `glibc` and has a much smaller package set."

**What's still wrong:**
> "Alpine still has:
> - `/bin/sh` — a shell an attacker can use
> - `apk` — a package manager to install tools
> - `busybox` — which has CVE-2025-60876
> - `zlib` — which has CVE-2026-27171
>
> It's dramatically better, but not zero. For this challenge — and for Chainguard — zero is the goal."

**Why I added HEALTHCHECK here:**
> "Alpine has `wget` built in, so I added a Docker HEALTHCHECK:
> ```dockerfile
> HEALTHCHECK --interval=30s CMD wget -qO- http://localhost:8080/healthz || exit 1
> ```
> For the single-stage, I used `curl` instead (Debian has curl but not wget by default). These are small details but they show awareness of what tools each base image provides."

### Tier 3: Dockerfile.chainguard — "The Solution"

> "Finally, the destination: a zero-CVE distroless image using Chainguard's hardened base."

**The approach I took:**
```dockerfile
FROM golang:1.25 AS builder
WORKDIR /app
COPY . .
RUN go mod download && CGO_ENABLED=0 GOOS=linux go build -o hello-app .

FROM cgr.dev/chainguard/static:latest
WORKDIR /app
COPY --from=builder /app/hello-app .
ENTRYPOINT ["./hello-app"]
```

**Why `CGO_ENABLED=0` — the critical decision:**
> "This is the most important build flag. Without it, Go may link to C libraries (like libc for DNS resolution). The Chainguard static image has **no C libraries** — it's truly empty. So I must produce a **fully static binary** that carries everything it needs.
>
> `CGO_ENABLED=0` tells Go: 'Use pure Go implementations for everything — DNS, TLS, compression. Don't depend on any C library.'
>
> If I forgot this flag, the binary would crash with 'not found' errors because it would try to load `libc.so` which doesn't exist."

**Why I chose `chainguard/static` over `chainguard/go`:**
> "Chainguard offers several base images:
> - `chainguard/go` — includes the Go runtime (for apps that need CGO)
> - `chainguard/glibc-dynamic` — includes glibc (for dynamically linked binaries)
> - `chainguard/static` — **absolutely nothing** except ca-certificates and tzdata
>
> Since my binary is fully static, I chose the smallest possible base. It's like choosing the smallest box that fits your product."

**Why no HEALTHCHECK:**
> "Distroless images have no shell, no `curl`, no `wget`. Docker's HEALTHCHECK directive requires executing a command inside the container. There's nothing to execute with. So I delegate health checking to **Kubernetes probes** instead, which make HTTP requests from outside the container."

**The consequences (The Twist!):**
> "When I scanned this distroless image, the OS vulnerabilities dropped to zero. But Grype still found **24 CVEs**! Why? Because those belonged to the original app's outdated Go dependencies (like `golang.org/x/net`).
>
> Distroless fixed the infrastructure, but it exposed the application's hygiene issues.
>
> So, I created a **Phase 2**: I updated all the Go dependencies to modern, secure versions. When I rebuilt the distroless image, it finally reached true **0 known CVEs**. This proves that zero-CVE status requires a defense-in-depth approach covering both the base image AND the application code."

---

## Part 4: Supply Chain Security (4 minutes)

### How to Present This:

> "Building a secure image isn't enough. I need to **prove** it's secure and that it hasn't been tampered with. That's supply chain security."

### Step 1: SBOM Generation (Transparency)

> "I generated SBOMs for all three images using Syft in SPDX 2.3 JSON format."

**Why SPDX and not CycloneDX:**
> "Both are valid. I chose SPDX because:
> 1. It's an **ISO standard** (ISO/IEC 5962:2021)
> 2. It's required by **Executive Order 14028** for US government software
> 3. It's maintained by the **Linux Foundation**, giving it broad industry trust
>
> CycloneDX (from OWASP) is also excellent and more developer-friendly. In a real project, I'd choose based on the customer's compliance requirements."

**What the SBOM reveals:**
> "The Chainguard SBOM is 930KB and lists:
> - **OS packages**: `ca-certificates-bundle`, `tzdata`, `wolfi-baselayout` — just 3 packages
> - **Go modules**: `gin-gonic/gin`, `go-playground/validator`, `protobuf`, `crypto`, etc. — 17 modules
> - **File checksums**: SHA256 hashes of every file in the image
>
> Compare this to the single-stage SBOM at 10.7MB — it must catalog hundreds of unnecessary Debian packages."

### Step 2: Image Signing (Authenticity)

> "After pushing images to the registry, I signed each one with cosign."

**My approach:**
```bash
cosign generate-key-pair              # Create private/public keys
cosign sign --key cosign.key <image>  # Sign the image
cosign verify --key cosign.pub <image> # Anyone can verify
```

**Why this matters:**
> "Signing creates a cryptographic bond between the image's SHA256 digest and my private key. If someone pushes a malicious image to the registry with the same tag, verification would fail because the digest wouldn't match my signature.
>
> In production, I wouldn't use local key files. I'd use **keyless signing** with Sigstore's Fulcio CA and Rekor transparency log. This eliminates key management entirely — you authenticate via OIDC (like Google/GitHub login), get a short-lived certificate, and the signature is recorded in a public, immutable log."

### Step 3: SBOM Attestation (Binding Content to Identity)

> "The final step ties everything together. I attached each SBOM as a cosign attestation."

**What this does:**
> "An attestation answers: 'What's inside this exact image?' It cryptographically binds the SBOM to the image digest. So if you pull `pankajsayal81/go-app-chainguard:v1` from Docker Hub, you can verify:
> 1. WHO signed it (my key)
> 2. WHAT's inside it (the SBOM)
> 3. THAT it hasn't changed (digest verification)
>
> This is the complete trust chain that enterprises need for compliance."

---

## Part 5: Kubernetes Hardening (3 minutes)

### How to Present This:

> "I didn't just deploy the container — I hardened the deployment manifest with defense-in-depth principles."

### Walk Through Each Config:

**1. Non-root execution:**
> "`runAsNonRoot: true` and `runAsUser: 65532` ensure the process **never runs as root**. Even if an attacker finds a privilege escalation bug, `allowPrivilegeEscalation: false` blocks it."

**2. Read-only filesystem:**
> "`readOnlyRootFilesystem: true` means the attacker **can't write files** to the container. No dropping malware, no modifying configs, no creating cron jobs."

**3. Dropped capabilities:**
> "`drop: [ALL]` removes every Linux capability. The process can't send raw network packets, mount filesystems, or perform any privileged operation."

**4. Health probes:**
> "Liveness and readiness probes hit `/healthz` every 10 seconds. This is **especially important for distroless** because we can't use Docker HEALTHCHECK."

**5. Resource limits:**
> "CPU and memory limits prevent a compromised container from consuming all node resources — a simple but effective denial-of-service defense."

**Why this matters for Chainguard customers:**
> "Chainguard provides the secure base image, but **customers are responsible for the deployment config**. An insecure deployment can undermine even a zero-CVE image. This shows I understand security is a spectrum, not a checkbox."

---

## Part 6: Wrapping Up (2 minutes)

### Summary Statement:

> "To summarize, I took a deliberate progression approach:
>
> 1. **Started with the problem** — 756 CVEs in a bloated single-stage build
> 2. **Applied industry best practices** — multi-stage Alpine brought it to 2 OS CVEs (but 24 Go CVEs remained)
> 3. **Adopted the Chainguard approach** — distroless eliminated all OS CVEs
> 4. **Addressed application hygiene** — updating Go dependencies brought the total to 0 CVEs
> 5. **Proved the security claims** — SBOMs, signatures, and attestations provide cryptographic evidence
> 6. **Hardened the deployment** — Kubernetes security context, probes, and resource limits
>
> The key insight is that **security requires both infrastructure reduction AND code hygiene**. Remove the OS attack surface with Chainguard, then fix your own code. What's left is your application and nothing else."

### Close With:

> "I'm happy to deep-dive into any area — the Dockerfile patterns, the supply chain tooling, the Kubernetes configuration, or how this would integrate into a CI/CD pipeline. What would you like to explore?"

---

## Bonus: Handling Tough Questions

### "What didn't go well?"
> "The main challenge was running cosign and syft from Windows. These tools are Linux-native, so I used Docker-based versions (`anchore/grype`, `anchore/syft`, `gcr.io/projectsigstore/cosign`). I also ran into Docker credential store issues when signing Docker Hub images — the cosign container couldn't access the Windows credential manager. I solved it by mounting a separate Docker config directory."

### "What would you add with more time?"
> "Three things:
> 1. **GitHub Actions CI/CD pipeline** — automate the entire build → scan → sign → deploy workflow
> 2. **Kubernetes admission controller** (Sigstore Policy Controller or Kyverno) — reject any unsigned images from being deployed
> 3. **Keyless signing with Fulcio/Rekor** — eliminate key management and add transparency log support"

### "How would you sell Chainguard images to a skeptical customer?"
> "I'd show them exactly what I showed here. 'You have 756 CVEs today. With Chainguard, you have zero. Your security team is spending weeks triaging vulnerabilities that don't even have fixes. With distroless images, there's nothing to triage. You can redeploy your engineering time from patching to building features. And the SBOMs + signatures give you compliance documentation out of the box.'"

### "What's the ROI argument?"
> "Three dimensions:
> 1. **Engineering time**: Stop patching 756 CVEs manually every release cycle
> 2. **Compliance**: Automated SBOMs satisfy Executive Order 14028, SOC2, and FedRAMP requirements
> 3. **Incident risk**: Zero attack surface = fewer breaches = lower insurance and remediation costs
>
> One data point: the average cost of a container breach is $4.24M (IBM Cost of a Data Breach Report). A Chainguard subscription costs a fraction of that."

---

*Remember: They're evaluating how you THINK, not just what you built. Explain the WHY behind every decision.*
