# Container Security Reference (Docker / Kubernetes)

## Table of Contents
- [Dockerfile Hardening](#dockerfile-hardening)
- [Image Security](#image-security)
- [Kubernetes RBAC](#kubernetes-rbac)
- [Pod Security](#pod-security)
- [Runtime Security](#runtime-security)
- [Supply Chain Security](#supply-chain-security)

---

## Dockerfile Hardening

### Secure Multi-Stage Build for Go
```dockerfile
# Build stage
FROM golang:1.22-alpine AS builder
RUN apk add --no-cache ca-certificates git
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -ldflags="-s -w" -trimpath -o /app/server ./cmd/server

# Runtime stage
FROM scratch
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /app/server /server
USER 65534:65534
EXPOSE 8080
ENTRYPOINT ["/server"]
```

### Rules
- Use multi-stage builds — final image from `scratch` or `distroless`
- `CGO_ENABLED=0` for static binaries (eliminates C dependency attack surface)
- `-ldflags="-s -w"` strips debug info, `-trimpath` removes build paths
- Never run as root — use `USER 65534` (nobody) or a dedicated UID
- Copy only `ca-certificates.crt` and the binary into final stage
- Pin base image versions (e.g., `golang:1.22.1-alpine`, not `golang:latest`)
- No secrets in build args or layers — use build-time secret mounts

### Anti-Patterns
- ❌ `FROM ubuntu:latest` as runtime image
- ❌ `RUN apt-get install -y curl wget` in production image
- ❌ `COPY . .` in single-stage build (includes .git, tests, docs)
- ❌ Credentials in `ENV` or `ARG` instructions

---

## Image Security

### Scanning
- Scan images in CI: `trivy image myapp:latest`
- Block images with critical/high CVEs from deploying
- Use signed images with cosign/Notary

### Minimal Base Images (preference order)
1. `scratch` — Go static binaries (ideal)
2. `gcr.io/distroless/static-debian12` — when you need CA certs/timezone data
3. `alpine` — when you need a shell for debugging

### Registry Security
- Use private registries (ECR, GCR, ACR)
- Enable vulnerability scanning on push
- Set image pull policies to `Always` in prod
- Implement image admission control (OPA Gatekeeper, Kyverno)

---

## Kubernetes RBAC

### Least-Privilege ServiceAccount
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp
  annotations:
    # GCP Workload Identity
    iam.gke.io/gcp-service-account: myapp@project.iam.gserviceaccount.com
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: myapp-role
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list"]
    resourceNames: ["myapp-config"]  # scope to specific resources
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: myapp-binding
subjects:
  - kind: ServiceAccount
    name: myapp
roleRef:
  kind: Role
  name: myapp-role
  apiGroup: rbac.authorization.k8s.io
```

### Rules
- One ServiceAccount per workload — never use `default`
- Use `Role`/`RoleBinding` (namespace-scoped), not `ClusterRole` unless required
- Scope `resourceNames` when accessing specific resources
- Never grant `*` verbs or resources
- Audit with `kubectl auth can-i --list --as=system:serviceaccount:ns:name`

---

## Pod Security

### Hardened Pod Spec
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  serviceAccountName: myapp
  automountServiceAccountToken: false  # disable unless needed
  securityContext:
    runAsNonRoot: true
    runAsUser: 65534
    runAsGroup: 65534
    fsGroup: 65534
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: myapp
      image: myapp:v1.2.3@sha256:abc123...  # pin by digest
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
      resources:
        limits:
          cpu: "500m"
          memory: "256Mi"
        requests:
          cpu: "100m"
          memory: "128Mi"
      livenessProbe:
        httpGet:
          path: /healthz
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 10
      readinessProbe:
        httpGet:
          path: /readyz
          port: 8080
        initialDelaySeconds: 3
        periodSeconds: 5
      volumeMounts:
        - name: tmp
          mountPath: /tmp
  volumes:
    - name: tmp
      emptyDir:
        medium: Memory
        sizeLimit: "64Mi"
```

### Rules
- `readOnlyRootFilesystem: true` — mount writable tmpfs only where needed
- Drop ALL capabilities, add back only what's required
- `allowPrivilegeEscalation: false` always
- Pin images by digest (`@sha256:...`), not just tag
- Set resource limits to prevent noisy-neighbor / DoS
- `automountServiceAccountToken: false` unless K8s API access is needed
- Use `seccompProfile: RuntimeDefault` at minimum

---

## Runtime Security

### Health Check Endpoints in Go
```go
func healthzHandler(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
    w.Write([]byte("ok"))
}

func readyzHandler(db *sql.DB) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
        defer cancel()
        if err := db.PingContext(ctx); err != nil {
            w.WriteHeader(http.StatusServiceUnavailable)
            return
        }
        w.WriteHeader(http.StatusOK)
    }
}
```

### Graceful Shutdown
```go
func main() {
    srv := &http.Server{Addr: ":8080", Handler: mux}
    go func() { srv.ListenAndServe() }()

    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)
    <-quit

    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    srv.Shutdown(ctx)
}
```

### Rules
- Implement `/healthz` (liveness) and `/readyz` (readiness) endpoints
- Handle `SIGTERM` gracefully — drain connections before exit
- Set `terminationGracePeriodSeconds` in pod spec to match shutdown timeout
- Use PodDisruptionBudgets for high-availability services

---

## Supply Chain Security

### Go Module Security
```bash
# Verify checksums
GONOSUMCHECK= go mod tidy
GONOSUMCHECK= go mod verify

# Scan for vulnerabilities
govulncheck ./...

# Use Go's checksum database
GONOSUMDB= GOFLAGS=-mod=readonly go build ./...
```

### Rules
- Run `govulncheck` in CI pipelines
- Use `go.sum` for reproducible builds — commit it always
- Vendor dependencies for air-gapped builds: `go mod vendor`
- Pin dependency versions in `go.mod` — avoid `latest`
- Sign container images with `cosign` in CI
- Use SBOM generation (Syft, GoReleaser) for compliance
