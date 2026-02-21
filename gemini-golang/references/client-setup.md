# Client Setup & Authentication

## Table of Contents
- [SDK Installation](#sdk-installation)
- [API Key Authentication (AI Studio)](#api-key-authentication-ai-studio)
- [Vertex AI Authentication (GCP)](#vertex-ai-authentication-gcp)
- [Retry Logic](#retry-logic)
- [Model Selection](#model-selection)

---

## SDK Installation

```bash
go get google.golang.org/genai
```

The unified `genai` package supports both Google AI Studio (API Key) and Vertex AI (GCP IAM) backends.

---

## API Key Authentication (AI Studio)

```go
package main

import (
    "context"
    "fmt"
    "log"
    "os"
    "time"

    "google.golang.org/genai"
)

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    // Load API key from environment — NEVER hardcode
    apiKey := os.Getenv("GEMINI_API_KEY")
    if apiKey == "" {
        log.Fatal("GEMINI_API_KEY environment variable not set")
    }

    client, err := genai.NewClient(ctx, &genai.ClientConfig{
        APIKey:  apiKey,
        Backend: genai.BackendGoogleAI,
    })
    if err != nil {
        log.Fatalf("create client: %v", err)
    }
    defer client.Close()

    // Use client...
}
```

### Rules
- Store API key in env var `GEMINI_API_KEY` or GCP Secret Manager
- In containers, mount as K8s Secret → env var or volume
- Rotate keys periodically via Google AI Studio console

---

## Vertex AI Authentication (GCP)

```go
func newVertexClient(ctx context.Context) (*genai.Client, error) {
    // Uses Application Default Credentials (ADC)
    // Locally: gcloud auth application-default login
    // In GKE: Workload Identity auto-provides credentials
    client, err := genai.NewClient(ctx, &genai.ClientConfig{
        Project:  os.Getenv("GCP_PROJECT_ID"),
        Location: os.Getenv("GCP_LOCATION"), // e.g., "us-central1"
        Backend:  genai.BackendVertexAI,
    })
    if err != nil {
        return nil, fmt.Errorf("create vertex client: %w", err)
    }
    return client, nil
}
```

### When to Use Vertex AI vs AI Studio
| Feature | AI Studio (API Key) | Vertex AI (GCP) |
|---|---|---|
| Auth | API key | IAM / service account |
| Data residency | No control | Region-specific |
| VPC-SC | No | Yes |
| Enterprise SLA | No | Yes |
| Grounding with Google Search | Limited | Full |
| Context Caching | Yes | Yes |
| Pricing | Pay-per-use | Pay-per-use + GCP billing |

**Rule:** Use Vertex AI for production workloads. Use AI Studio for prototyping and development.

---

## Retry Logic

```go
import (
    "math"
    "math/rand"
    "time"
)

func withRetry[T any](ctx context.Context, maxRetries int, fn func(ctx context.Context) (T, error)) (T, error) {
    var zero T
    for attempt := 0; attempt <= maxRetries; attempt++ {
        result, err := fn(ctx)
        if err == nil {
            return result, nil
        }

        // Check if retryable (429 Too Many Requests, 500+ Server Error)
        if !isRetryable(err) || attempt == maxRetries {
            return zero, fmt.Errorf("after %d attempts: %w", attempt+1, err)
        }

        // Exponential backoff with jitter
        backoff := time.Duration(math.Pow(2, float64(attempt))) * time.Second
        jitter := time.Duration(rand.Int63n(int64(time.Second)))
        select {
        case <-ctx.Done():
            return zero, ctx.Err()
        case <-time.After(backoff + jitter):
        }
    }
    return zero, fmt.Errorf("max retries exceeded")
}

func isRetryable(err error) bool {
    errStr := err.Error()
    return strings.Contains(errStr, "429") ||
           strings.Contains(errStr, "500") ||
           strings.Contains(errStr, "503") ||
           strings.Contains(errStr, "UNAVAILABLE") ||
           strings.Contains(errStr, "RESOURCE_EXHAUSTED")
}
```

### Rules
- Always wrap API calls with retry logic in production
- Use exponential backoff: 1s, 2s, 4s, 8s... with jitter
- Max 3-5 retries for interactive, more for batch
- Respect `context.Context` cancellation during backoff
- Log each retry attempt with the error for debugging

---

## Model Selection

| Model | Strengths | Use When | Cost |
|---|---|---|---|
| `gemini-2.5-pro` | Best quality, complex reasoning, large context | Complex analysis, code generation, long documents | $$$ |
| `gemini-2.5-flash` | Fast, good quality, cost-effective | Most production use cases, chat, summarization | $ |
| `gemini-2.0-flash-lite` | Fastest, cheapest | Simple tasks, classification, extraction | ¢ |
| `text-embedding-004` | Embeddings | RAG, semantic search, similarity | ¢ |

### Decision Framework
1. Start with **Flash** — upgrade to Pro only if quality is insufficient
2. Use **Flash Lite** for high-volume, low-complexity tasks (classification, extraction)
3. Use **Pro** for: multi-step reasoning, code generation, complex analysis, long-context (>100K tokens)
4. Always benchmark Flash vs Pro on your specific task before committing
