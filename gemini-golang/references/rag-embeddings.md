# RAG & Embeddings

## Table of Contents
- [Embedding Models](#embedding-models)
- [Generating Embeddings](#generating-embeddings)
- [RAG Pipeline](#rag-pipeline)
- [Context Caching](#context-caching)
- [Grounding with Google Search](#grounding-with-google-search)

---

## Embedding Models

| Model | Dimensions | Max Tokens | Use Case |
|---|---|---|---|
| `text-embedding-004` | 768 | 2048 | General-purpose embeddings |
| `text-embedding-005` | 768 | 2048 | Improved quality (latest) |

### Task Types
| Task Type | Description |
|---|---|
| `RETRIEVAL_DOCUMENT` | Embed documents for search index |
| `RETRIEVAL_QUERY` | Embed user queries for search |
| `SEMANTIC_SIMILARITY` | Compare text similarity |
| `CLASSIFICATION` | Text classification |
| `CLUSTERING` | Group similar texts |

---

## Generating Embeddings

### Single Text
```go
func embedText(ctx context.Context, client *genai.Client, text, taskType string) ([]float32, error) {
    result, err := client.Models.EmbedContent(ctx, "text-embedding-005",
        genai.Text(text),
        &genai.EmbedContentConfig{
            TaskType: taskType,
        },
    )
    if err != nil {
        return nil, fmt.Errorf("embed content: %w", err)
    }
    return result.Embedding.Values, nil
}
```

### Batch Embeddings
```go
func embedBatch(ctx context.Context, client *genai.Client, texts []string) ([][]float32, error) {
    var contents []*genai.Content
    for _, t := range texts {
        contents = append(contents, genai.NewContentFromText("user", t))
    }

    result, err := client.Models.BatchEmbedContents(ctx, "text-embedding-005",
        contents,
        &genai.EmbedContentConfig{
            TaskType: "RETRIEVAL_DOCUMENT",
        },
    )
    if err != nil {
        return nil, fmt.Errorf("batch embed: %w", err)
    }

    embeddings := make([][]float32, len(result.Embeddings))
    for i, emb := range result.Embeddings {
        embeddings[i] = emb.Values
    }
    return embeddings, nil
}
```

### Rules
- Use `RETRIEVAL_DOCUMENT` for indexing, `RETRIEVAL_QUERY` for searching
- Batch requests (up to 100 texts) for throughput
- Embeddings are deterministic — cache results for repeated texts
- Chunk documents to ~500-1000 tokens for optimal retrieval

---

## RAG Pipeline

### Architecture
```
1. Ingest: Document → Chunk → Embed → Store in Vector DB
2. Query:  User query → Embed → Vector search → Top-K chunks
3. Generate: System instruction + retrieved chunks + user query → Gemini → Answer
```

### Document Chunking
```go
func chunkText(text string, chunkSize, overlap int) []string {
    words := strings.Fields(text)
    var chunks []string
    for i := 0; i < len(words); i += chunkSize - overlap {
        end := i + chunkSize
        if end > len(words) {
            end = len(words)
        }
        chunks = append(chunks, strings.Join(words[i:end], " "))
        if end == len(words) {
            break
        }
    }
    return chunks
}
```

### RAG Generation
```go
func ragQuery(ctx context.Context, client *genai.Client, query string, relevantChunks []string) (string, error) {
    // Build context from retrieved chunks
    var contextBuilder strings.Builder
    for i, chunk := range relevantChunks {
        fmt.Fprintf(&contextBuilder, "[Source %d]: %s\n\n", i+1, chunk)
    }

    config := &genai.GenerateContentConfig{
        SystemInstruction: genai.NewContentFromText("model",
            `You are a helpful assistant. Answer questions based ONLY on the provided context.
            If the context doesn't contain the answer, say "I don't have enough information."
            Cite your sources using [Source N] notation.`),
        Temperature: genai.Ptr(float32(0.2)), // Low for factual accuracy
    }

    prompt := fmt.Sprintf("Context:\n%s\nQuestion: %s", contextBuilder.String(), query)

    result, err := client.Models.GenerateContent(ctx, "gemini-2.5-flash",
        genai.Text(prompt), config)
    if err != nil {
        return "", fmt.Errorf("rag query: %w", err)
    }
    return result.Text(), nil
}
```

### Vector Store Options
| Store | Type | Strengths |
|---|---|---|
| PostgreSQL + pgvector | Self-hosted | ACID, familiar SQL, hybrid search |
| Vertex AI Vector Search | Managed (GCP) | Auto-scaling, low-latency, integrated |
| Pinecone | Managed | Simple API, serverless |
| Weaviate | Self-hosted/Cloud | Hybrid search, GraphQL |
| ChromaDB | Embedded | Local dev, lightweight |

---

## Context Caching

Cache large, reusable prompt content to reduce token costs and latency.

```go
func createCache(ctx context.Context, client *genai.Client, systemPrompt, largeDocument string) (*genai.CachedContent, error) {
    cache, err := client.CachedContents.Create(ctx, &genai.CachedContent{
        Model: "gemini-2.5-flash",
        Contents: []*genai.Content{
            genai.NewContentFromText("user", largeDocument),
        },
        SystemInstruction: genai.NewContentFromText("model", systemPrompt),
        ExpireTime:        time.Now().Add(1 * time.Hour),
    })
    if err != nil {
        return nil, fmt.Errorf("create cache: %w", err)
    }
    return cache, nil
}

func queryWithCache(ctx context.Context, client *genai.Client, cacheName, query string) (string, error) {
    result, err := client.Models.GenerateContent(ctx, "gemini-2.5-flash",
        genai.Text(query),
        &genai.GenerateContentConfig{
            CachedContent: cacheName,
        },
    )
    if err != nil {
        return "", fmt.Errorf("query with cache: %w", err)
    }
    return result.Text(), nil
}
```

### When to Use Context Caching
| Scenario | Benefit |
|---|---|
| Large document Q&A (>10K tokens) | Pay for document tokens once, query many times |
| System instructions + examples | Cache few-shot examples |
| Code repository analysis | Cache repo contents, query repeatedly |
| Legal/compliance document review | Cache contract, ask multiple questions |

### Rules
- Minimum cache size: 32K tokens (below this, caching isn't cost-effective)
- Set appropriate TTL — default 1 hour, max 24 hours
- Cached content is billed at a reduced rate per hour of storage
- Cache is model-specific — can't share between Flash and Pro

---

## Grounding with Google Search

```go
config := &genai.GenerateContentConfig{
    Tools: []*genai.Tool{
        {GoogleSearch: &genai.GoogleSearch{}},
    },
}

result, err := client.Models.GenerateContent(ctx, "gemini-2.5-flash",
    genai.Text("What were the latest Go releases in 2025?"),
    config,
)

// Access grounding metadata
if result.Candidates[0].GroundingMetadata != nil {
    for _, chunk := range result.Candidates[0].GroundingMetadata.GroundingChunks {
        fmt.Printf("Source: %s (%s)\n", chunk.Web.Title, chunk.Web.URI)
    }
}
```

### Rules
- Use Grounding for real-time or recent information queries
- Always display source attributions to the user
- Grounding adds latency (~1-3s) — don't use for cached/static knowledge
- Available on both AI Studio and Vertex AI
