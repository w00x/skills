# Cloud Security Reference (AWS/GCP/Azure)

## Table of Contents
- [IAM & Least Privilege](#iam--least-privilege)
- [Encryption at Rest](#encryption-at-rest)
- [Encryption in Transit](#encryption-in-transit)
- [Secrets Management](#secrets-management)
- [Network Security](#network-security)
- [Logging & Monitoring](#logging--monitoring)

---

## IAM & Least Privilege

### Principles
- Grant minimum required permissions per service account
- Use short-lived credentials (STS tokens, workload identity) over static keys
- Separate service accounts per microservice — never share
- Audit IAM policies monthly; remove unused permissions

### Go Pattern — AWS STS Assume Role
```go
cfg, err := config.LoadDefaultConfig(ctx,
    config.WithRegion("us-east-1"),
)
stsClient := sts.NewFromConfig(cfg)
creds := stscreds.NewAssumeRoleProvider(stsClient, "arn:aws:iam::123456:role/limited-role",
    func(o *stscreds.AssumeRoleOptions) {
        o.Duration = 15 * time.Minute
    },
)
cfg.Credentials = aws.NewCredentialsCache(creds)
```

### Go Pattern — GCP Workload Identity
```go
// In GKE with Workload Identity, the SDK auto-detects
client, err := storage.NewClient(ctx) // uses pod's K8s service account
```

### Anti-Patterns
- ❌ `"Action": "*"` or `"Resource": "*"` in IAM policies
- ❌ Static access keys in environment variables
- ❌ Sharing service accounts across services
- ❌ Long-lived credentials without rotation

---

## Encryption at Rest

### Cloud-Provider Managed Keys
- **AWS**: Use KMS with CMK, enable automatic rotation
- **GCP**: Use Cloud KMS or CMEK for GCS/BigQuery
- **Azure**: Use Key Vault with HSM-backed keys

### Go Pattern — AWS KMS Envelope Encryption
```go
kmsClient := kms.NewFromConfig(cfg)

// Generate data key
result, err := kmsClient.GenerateDataKey(ctx, &kms.GenerateDataKeyInput{
    KeyId:   aws.String("alias/my-key"),
    KeySpec: types.DataKeySpecAes256,
})
// result.Plaintext = DEK (use to encrypt, then discard)
// result.CiphertextBlob = encrypted DEK (store alongside ciphertext)
```

### Go Pattern — AES-GCM Encryption
```go
func encrypt(key, plaintext []byte) ([]byte, error) {
    block, err := aes.NewCipher(key)
    if err != nil {
        return nil, err
    }
    aesGCM, err := cipher.NewGCM(block)
    if err != nil {
        return nil, err
    }
    nonce := make([]byte, aesGCM.NonceSize())
    if _, err := io.ReadFull(crypto_rand.Reader, nonce); err != nil {
        return nil, err
    }
    return aesGCM.Seal(nonce, nonce, plaintext, nil), nil
}
```

### Rules
- Always use AES-256-GCM (authenticated encryption)
- Never reuse nonces — generate with `crypto/rand`
- Use envelope encryption: KMS encrypts DEK, DEK encrypts data
- Enable server-side encryption for all object storage buckets

---

## Encryption in Transit

### TLS Configuration
```go
tlsCfg := &tls.Config{
    MinVersion:               tls.VersionTLS12,
    PreferServerCipherSuites: true,
    CipherSuites: []uint16{
        tls.TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,
        tls.TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,
        tls.TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,
    },
}
server := &http.Server{
    Addr:      ":443",
    TLSConfig: tlsCfg,
    ReadTimeout:  10 * time.Second,
    WriteTimeout: 10 * time.Second,
    IdleTimeout:  120 * time.Second,
}
```

### mTLS for Service-to-Service
```go
func loadMTLSConfig(certFile, keyFile, caFile string) (*tls.Config, error) {
    cert, err := tls.LoadX509KeyPair(certFile, keyFile)
    if err != nil {
        return nil, err
    }
    caCert, err := os.ReadFile(caFile)
    if err != nil {
        return nil, err
    }
    caPool := x509.NewCertPool()
    caPool.AppendCertsFromPEM(caCert)
    return &tls.Config{
        Certificates: []tls.Certificate{cert},
        ClientCAs:    caPool,
        ClientAuth:   tls.RequireAndVerifyClientCert,
        MinVersion:   tls.VersionTLS12,
    }, nil
}
```

### Rules
- Enforce TLS 1.2+ everywhere — no exceptions
- Use mTLS for service-to-service in zero-trust architectures
- Never set `InsecureSkipVerify: true` in production
- Terminate TLS at the ingress/load balancer or at the app — not both without reason
- Set timeouts: `ReadTimeout`, `WriteTimeout`, `IdleTimeout`

---

## Secrets Management

### Cloud-Native Secret Stores
- **AWS**: Secrets Manager or SSM Parameter Store (SecureString)
- **GCP**: Secret Manager
- **Azure**: Key Vault
- **K8s**: External Secrets Operator → syncs cloud secrets to K8s Secrets

### Go Pattern — GCP Secret Manager
```go
func getSecret(ctx context.Context, name string) (string, error) {
    client, err := secretmanager.NewClient(ctx)
    if err != nil {
        return "", err
    }
    defer client.Close()
    result, err := client.AccessSecretVersion(ctx, &secretmanagerpb.AccessSecretVersionRequest{
        Name: name, // "projects/proj/secrets/my-secret/versions/latest"
    })
    if err != nil {
        return "", fmt.Errorf("access secret: %w", err)
    }
    return string(result.Payload.Data), nil
}
```

### Anti-Patterns
- ❌ Secrets in environment variables visible via `/proc`
- ❌ Secrets in Docker image layers (even if deleted in later layers)
- ❌ Secrets in git history
- ❌ Logging secret values (even partially)

### Rules
- Load secrets at startup, cache in memory, rotate periodically
- Use mounted volumes (tmpfs) in K8s, not env vars
- Scan repos with `trufflehog` or `gitleaks` pre-commit
- Zero secrets in source code — ever

---

## Network Security

### Principles
- Default-deny ingress/egress
- Use private subnets for all backend services
- Expose only the ingress controller/load balancer publicly
- Use VPC Service Controls (GCP) or VPC Endpoints (AWS) for cloud API access

### Go — HTTP Client Hardening
```go
transport := &http.Transport{
    TLSClientConfig:     &tls.Config{MinVersion: tls.VersionTLS12},
    MaxIdleConns:         100,
    MaxIdleConnsPerHost:  10,
    IdleConnTimeout:      90 * time.Second,
    DisableCompression:   false,
    ForceAttemptHTTP2:    true,
}
client := &http.Client{
    Transport: transport,
    Timeout:   30 * time.Second,
}
```

### Network Policies (K8s)
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
spec:
  podSelector: {}
  policyTypes: [Ingress]
  ingress: [] # deny all
```

Then explicitly allow only needed traffic per service.

---

## Logging & Monitoring

### Secure Logging Pattern
```go
import "log/slog"

logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelInfo,
}))

// DO: structured, no PII
logger.Info("request processed",
    "request_id", reqID,
    "user_id", userID,
    "status", statusCode,
    "duration_ms", elapsed.Milliseconds(),
)

// DON'T: logging sensitive data
// logger.Info("login", "password", password, "token", token)
```

### Rules
- Use structured logging (`log/slog` in Go 1.21+)
- Never log passwords, tokens, PII, or credit card numbers
- Include request/correlation IDs for tracing
- Ship logs to a centralized system (CloudWatch, Stackdriver, ELK)
- Set up alerts for: auth failures, 5xx spikes, permission denials
