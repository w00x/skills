# Multimodal & Generation

## Table of Contents
- [Text Generation](#text-generation)
- [Generation Config](#generation-config)
- [Streaming](#streaming)
- [Image Input](#image-input)
- [Video & Audio Input](#video--audio-input)
- [System Instructions](#system-instructions)
- [Safety Settings](#safety-settings)
- [Token Counting](#token-counting)

---

## Text Generation

```go
func generateText(ctx context.Context, client *genai.Client, prompt string) (string, error) {
    result, err := client.Models.GenerateContent(ctx, "gemini-2.5-flash", genai.Text(prompt), nil)
    if err != nil {
        return "", fmt.Errorf("generate content: %w", err)
    }
    return result.Text(), nil
}
```

### Multi-Turn Conversation
```go
func chat(ctx context.Context, client *genai.Client) error {
    // Build conversation history
    history := []*genai.Content{
        genai.NewContentFromText("user", "Hello, I'm building a Go app"),
        genai.NewContentFromText("model", "Great! How can I help with your Go application?"),
    }

    // Send new message with history
    result, err := client.Models.GenerateContent(ctx, "gemini-2.5-flash",
        // Pass all history + new message as contents
        genai.Text("I need help with error handling patterns"),
        &genai.GenerateContentConfig{
            // Config options here
        },
    )
    if err != nil {
        return err
    }
    fmt.Println(result.Text())
    return nil
}
```

---

## Generation Config

```go
config := &genai.GenerateContentConfig{
    Temperature:     genai.Ptr(float32(0.7)),  // 0.0 = deterministic, 1.0 = creative
    TopP:            genai.Ptr(float32(0.95)),  // nucleus sampling
    TopK:            genai.Ptr(int32(40)),      // top-k sampling
    MaxOutputTokens: genai.Ptr(int32(8192)),    // limit response length
    StopSequences:   []string{"---", "END"},    // stop generation at these
}

result, err := client.Models.GenerateContent(ctx, "gemini-2.5-flash",
    genai.Text(prompt), config)
```

### Temperature Guide
| Value | Use Case |
|---|---|
| 0.0 | Factual extraction, classification, code generation |
| 0.3-0.5 | Summarization, structured output |
| 0.7-0.9 | Creative writing, brainstorming |
| 1.0+ | Maximum creativity (use with caution) |

---

## Streaming

```go
func generateStream(ctx context.Context, client *genai.Client, prompt string) error {
    // Use streaming to reduce time-to-first-byte
    for result, err := range client.Models.GenerateContentStream(ctx,
        "gemini-2.5-flash", genai.Text(prompt), nil) {
        if err != nil {
            return fmt.Errorf("stream chunk: %w", err)
        }
        // Process each chunk as it arrives
        for _, candidate := range result.Candidates {
            for _, part := range candidate.Content.Parts {
                if text, ok := part.(genai.Text); ok {
                    fmt.Print(string(text))
                }
            }
        }
    }
    return nil
}
```

### Rules
- Use streaming for responses >500 tokens to improve perceived latency
- Always handle errors on each chunk — partial failures are possible
- Check `candidate.FinishReason` on the final chunk
- For SSE endpoints, forward chunks directly to HTTP response

---

## Image Input

```go
func analyzeImage(ctx context.Context, client *genai.Client, imagePath string) (string, error) {
    // Read image file
    imageData, err := os.ReadFile(imagePath)
    if err != nil {
        return "", fmt.Errorf("read image: %w", err)
    }

    // Detect MIME type
    mimeType := http.DetectContentType(imageData)

    result, err := client.Models.GenerateContent(ctx, "gemini-2.5-flash",
        // Text prompt + inline image
        genai.Text("Describe this image in detail"),
        &genai.Blob{Data: imageData, MIMEType: mimeType},
        nil,
    )
    if err != nil {
        return "", fmt.Errorf("analyze image: %w", err)
    }
    return result.Text(), nil
}
```

### Image from URL
```go
imagePart := &genai.FileData{
    FileURI:  "gs://bucket/image.jpg",  // GCS URI
    MIMEType: "image/jpeg",
}
```

### Supported Formats
- JPEG, PNG, GIF, WebP, HEIC, HEIF
- Max 20 images per request (Flash), up to 3600 images (Pro)

---

## Video & Audio Input

### Video Analysis
```go
func analyzeVideo(ctx context.Context, client *genai.Client, videoURI string) (string, error) {
    // Upload file first for large videos
    // Or use GCS URI directly with Vertex AI
    result, err := client.Models.GenerateContent(ctx, "gemini-2.5-pro",
        genai.Text("Summarize the key events in this video"),
        &genai.FileData{
            FileURI:  videoURI,  // gs://bucket/video.mp4
            MIMEType: "video/mp4",
        },
        nil,
    )
    if err != nil {
        return "", fmt.Errorf("analyze video: %w", err)
    }
    return result.Text(), nil
}
```

### File Upload (for large files)
```go
func uploadFile(ctx context.Context, client *genai.Client, filePath, mimeType string) (*genai.File, error) {
    f, err := os.Open(filePath)
    if err != nil {
        return nil, err
    }
    defer f.Close()

    file, err := client.Files.Upload(ctx, f, &genai.UploadFileConfig{
        MIMEType:    mimeType,
        DisplayName: filepath.Base(filePath),
    })
    if err != nil {
        return nil, fmt.Errorf("upload file: %w", err)
    }
    // Wait for processing
    for file.State == genai.FileStateProcessing {
        time.Sleep(2 * time.Second)
        file, err = client.Files.Get(ctx, file.Name)
        if err != nil {
            return nil, err
        }
    }
    if file.State != genai.FileStateActive {
        return nil, fmt.Errorf("file processing failed: %s", file.State)
    }
    return file, nil
}
```

---

## System Instructions

```go
config := &genai.GenerateContentConfig{
    SystemInstruction: genai.NewContentFromText("model",
        `You are a senior Go developer. Follow these rules:
        1. Always handle errors explicitly
        2. Use context.Context for cancellation
        3. Write idiomatic Go code
        4. Include table-driven tests`),
}

result, err := client.Models.GenerateContent(ctx, "gemini-2.5-flash",
    genai.Text("Write a function to parse CSV files"),
    config,
)
```

### Rules
- System instructions persist across the conversation — set once
- Keep system instructions concise (<500 tokens) for cost efficiency
- Use them for: persona, output format, constraints, tone
- They count toward the context window but are cached efficiently

---

## Safety Settings

```go
config := &genai.GenerateContentConfig{
    SafetySettings: []*genai.SafetySetting{
        {
            Category:  genai.HarmCategoryHarassment,
            Threshold: genai.HarmBlockThresholdBlockMediumAndAbove,
        },
        {
            Category:  genai.HarmCategoryHateSpeech,
            Threshold: genai.HarmBlockThresholdBlockMediumAndAbove,
        },
        {
            Category:  genai.HarmCategoryDangerousContent,
            Threshold: genai.HarmBlockThresholdBlockOnlyHigh,
        },
        {
            Category:  genai.HarmCategorySexuallyExplicit,
            Threshold: genai.HarmBlockThresholdBlockMediumAndAbove,
        },
    },
}
```

### Handling Blocked Responses
```go
result, err := client.Models.GenerateContent(ctx, model, prompt, config)
if err != nil {
    return err
}
for _, candidate := range result.Candidates {
    if candidate.FinishReason == genai.FinishReasonSafety {
        log.Warn("response blocked by safety filters",
            "ratings", candidate.SafetyRatings)
        return ErrContentBlocked
    }
}
```

### Rules
- Always configure safety settings for production
- Handle `FinishReasonSafety` gracefully — don't expose raw errors to users
- Log blocked responses for monitoring (without logging the triggering content)
- Adjust thresholds per use case — stricter for user-facing, relaxed for internal analysis

---

## Token Counting

```go
// Count tokens before sending (cost estimation)
countResp, err := client.Models.CountTokens(ctx, "gemini-2.5-flash",
    genai.Text(prompt), nil)
if err != nil {
    return err
}
log.Info("token count",
    "total", countResp.TotalTokens,
    "billable_chars", countResp.TotalBillableCharacters,
)

// After generation — check usage metadata
result, _ := client.Models.GenerateContent(ctx, model, prompt, config)
if result.UsageMetadata != nil {
    log.Info("usage",
        "input_tokens", result.UsageMetadata.PromptTokenCount,
        "output_tokens", result.UsageMetadata.CandidatesTokenCount,
        "total", result.UsageMetadata.TotalTokenCount,
    )
}
```
