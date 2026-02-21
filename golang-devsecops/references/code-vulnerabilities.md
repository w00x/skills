# Go Code Vulnerabilities Reference

## Table of Contents
- [SQL Injection](#sql-injection)
- [Cross-Site Scripting (XSS)](#cross-site-scripting-xss)
- [Server-Side Request Forgery (SSRF)](#server-side-request-forgery-ssrf)
- [Path Traversal](#path-traversal)
- [Race Conditions](#race-conditions)
- [Unsafe Deserialization](#unsafe-deserialization)
- [Improper Error Handling](#improper-error-handling)
- [Cryptographic Failures](#cryptographic-failures)
- [Authentication & Session Management](#authentication--session-management)

---

## SQL Injection
**CWE-89 | OWASP A03:2021 Injection**

### Insecure
```go
query := fmt.Sprintf("SELECT * FROM users WHERE id = '%s'", userInput)
rows, err := db.Query(query)
```

### Secure
```go
rows, err := db.QueryContext(ctx, "SELECT * FROM users WHERE id = $1", userInput)
if err != nil {
    return fmt.Errorf("query users: %w", err)
}
defer rows.Close()
```

**Rules:**
- Always use parameterized queries (`$1`, `?` placeholders)
- Use `QueryContext`/`ExecContext` with timeout context
- For dynamic column/table names, use an allowlist — never interpolate

---

## Cross-Site Scripting (XSS)
**CWE-79 | OWASP A03:2021 Injection**

### Insecure
```go
import "text/template"
tmpl := template.Must(template.New("page").Parse("<h1>{{.Title}}</h1>"))
```

### Secure
```go
import "html/template"
tmpl := template.Must(template.New("page").Parse("<h1>{{.Title}}</h1>"))
```

**Rules:**
- Always use `html/template` for HTML output — it auto-escapes by default
- Never use `template.HTML()` to cast untrusted input
- Set `Content-Type: text/html; charset=utf-8` header explicitly
- For JSON API responses, set `Content-Type: application/json` and use `json.Encoder`

---

## Server-Side Request Forgery (SSRF)
**CWE-918 | OWASP A10:2021**

### Insecure
```go
resp, err := http.Get(userProvidedURL)
```

### Secure
```go
func safeFetch(ctx context.Context, rawURL string) (*http.Response, error) {
    parsed, err := url.Parse(rawURL)
    if err != nil {
        return nil, fmt.Errorf("invalid URL: %w", err)
    }
    // Block internal/private networks
    if parsed.Hostname() == "localhost" || 
       strings.HasPrefix(parsed.Hostname(), "127.") ||
       strings.HasPrefix(parsed.Hostname(), "10.") ||
       strings.HasPrefix(parsed.Hostname(), "169.254.") ||
       parsed.Hostname() == "metadata.google.internal" {
        return nil, errors.New("blocked: internal address")
    }
    if parsed.Scheme != "https" {
        return nil, errors.New("blocked: only HTTPS allowed")
    }
    client := &http.Client{
        Timeout: 10 * time.Second,
        CheckRedirect: func(req *http.Request, via []*http.Request) error {
            if len(via) >= 3 {
                return errors.New("too many redirects")
            }
            return nil
        },
    }
    req, _ := http.NewRequestWithContext(ctx, http.MethodGet, parsed.String(), nil)
    return client.Do(req)
}
```

**Rules:**
- Validate and allowlist URLs/domains before fetching
- Block `169.254.169.254` (cloud metadata endpoint), `localhost`, RFC 1918 ranges
- Enforce HTTPS only, limit redirects
- Use DNS resolution validation to prevent DNS rebinding

---

## Path Traversal
**CWE-22 | OWASP A01:2021**

### Insecure
```go
filePath := filepath.Join("/uploads", userInput)
data, err := os.ReadFile(filePath)
```

### Secure
```go
func safeReadFile(basedir, userInput string) ([]byte, error) {
    cleaned := filepath.Clean(userInput)
    absPath := filepath.Join(basedir, cleaned)
    // Verify the resolved path is within basedir
    if !strings.HasPrefix(absPath, filepath.Clean(basedir)+string(os.PathSeparator)) {
        return nil, errors.New("path traversal blocked")
    }
    return os.ReadFile(absPath)
}
```

**Rules:**
- Always `filepath.Clean` user input
- Verify resolved path stays within allowed base directory
- Never use `os.Symlink` targets from user input without validation
- In containers, use read-only filesystem mounts where possible

---

## Race Conditions
**CWE-362 | OWASP A04:2021**

### Insecure — Data Race
```go
var counter int
func handler(w http.ResponseWriter, r *http.Request) {
    counter++ // unsynchronized access
    fmt.Fprintf(w, "Count: %d", counter)
}
```

### Secure — Atomic Operations
```go
var counter atomic.Int64
func handler(w http.ResponseWriter, r *http.Request) {
    val := counter.Add(1)
    fmt.Fprintf(w, "Count: %d", val)
}
```

### Secure — Mutex (complex state)
```go
type SafeState struct {
    mu    sync.RWMutex
    data  map[string]int
}

func (s *SafeState) Get(key string) (int, bool) {
    s.mu.RLock()
    defer s.mu.RUnlock()
    v, ok := s.data[key]
    return v, ok
}

func (s *SafeState) Set(key string, val int) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.data[key] = val
}
```

**Rules:**
- Always run tests with `-race` flag
- Prefer `sync/atomic` for simple counters
- Use `sync.RWMutex` for read-heavy shared state
- Prefer channels for goroutine communication patterns
- Use `sync.Once` for one-time initialization

---

## Unsafe Deserialization
**CWE-502 | OWASP A08:2021**

### Insecure
```go
var config map[string]interface{}
json.NewDecoder(r.Body).Decode(&config) // unbounded, untyped
```

### Secure
```go
type Config struct {
    Name    string `json:"name" validate:"required,max=100"`
    Timeout int    `json:"timeout" validate:"min=1,max=300"`
}

func decodeConfig(r *http.Request) (Config, error) {
    r.Body = http.MaxBytesReader(nil, r.Body, 1<<20) // 1MB limit
    var cfg Config
    if err := json.NewDecoder(r.Body).Decode(&cfg); err != nil {
        return Config{}, fmt.Errorf("decode config: %w", err)
    }
    // validate struct fields
    return cfg, nil
}
```

**Rules:**
- Always decode into typed structs, never `interface{}`
- Use `http.MaxBytesReader` to limit request body size
- Validate decoded values (range checks, allowlists)
- Never use `encoding/gob` or `encoding/xml` with untrusted input without size limits

---

## Improper Error Handling
**CWE-209 | OWASP A09:2021**

### Insecure — Leaking internals
```go
func handler(w http.ResponseWriter, r *http.Request) {
    _, err := db.QueryContext(ctx, query, args...)
    if err != nil {
        http.Error(w, err.Error(), 500) // exposes DB details
    }
}
```

### Secure
```go
func handler(w http.ResponseWriter, r *http.Request) {
    _, err := db.QueryContext(ctx, query, args...)
    if err != nil {
        log.Error("db query failed",
            "error", err,
            "request_id", requestID(r),
        )
        http.Error(w, "internal error", http.StatusInternalServerError)
    }
}
```

**Rules:**
- Log full error details server-side with structured logging
- Return generic error messages to clients
- Include correlation/request IDs for debugging
- Never expose stack traces, SQL errors, or file paths to consumers

---

## Cryptographic Failures
**CWE-327 | OWASP A02:2021**

### Password Hashing
```go
import "golang.org/x/crypto/bcrypt"

func hashPassword(password string) (string, error) {
    hash, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    return string(hash), err
}

func checkPassword(hash, password string) bool {
    return bcrypt.CompareHashAndPassword([]byte(hash), []byte(password)) == nil
}
```

### Token Generation
```go
func generateToken(n int) (string, error) {
    b := make([]byte, n)
    if _, err := crypto_rand.Read(b); err != nil {
        return "", fmt.Errorf("generate token: %w", err)
    }
    return base64.URLEncoding.EncodeToString(b), nil
}
```

### Constant-Time Comparison
```go
import "crypto/subtle"

func verifyAPIKey(provided, expected string) bool {
    return subtle.ConstantTimeCompare([]byte(provided), []byte(expected)) == 1
}
```

**Rules:**
- Use `crypto/rand` — never `math/rand` for security-sensitive values
- Use bcrypt/scrypt/argon2 for passwords (never SHA/MD5)
- Use `crypto/subtle.ConstantTimeCompare` for secret comparison
- Enforce TLS 1.2+ with `tls.Config{MinVersion: tls.VersionTLS12}`

---

## Authentication & Session Management
**OWASP A07:2021**

### JWT Validation Pattern
```go
func validateJWT(tokenString, secret string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenString, &Claims{}, func(t *jwt.Token) (interface{}, error) {
        // Verify signing algorithm
        if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
        }
        return []byte(secret), nil
    })
    if err != nil {
        return nil, fmt.Errorf("parse token: %w", err)
    }
    claims, ok := token.Claims.(*Claims)
    if !ok || !token.Valid {
        return nil, errors.New("invalid token")
    }
    return claims, nil
}
```

**Rules:**
- Always verify the signing algorithm in JWT validation (`alg` header)
- Set short expiration times, use refresh tokens for long sessions
- Store session tokens server-side when possible (Redis/memcached)
- Invalidate tokens on logout and password change
