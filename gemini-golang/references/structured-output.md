# Structured Output

## Table of Contents
- [JSON Response Mode](#json-response-mode)
- [Schema Enforcement](#schema-enforcement)
- [Enum Constraints](#enum-constraints)
- [Complex Nested Schemas](#complex-nested-schemas)
- [Parsing Responses](#parsing-responses)

---

## JSON Response Mode

Force the model to return valid JSON:

```go
config := &genai.GenerateContentConfig{
    ResponseMIMEType: "application/json",
}

result, err := client.Models.GenerateContent(ctx, "gemini-2.5-flash",
    genai.Text("List 3 programming languages with their year of creation"),
    config,
)
// result.Text() will be valid JSON
```

### Supported MIME Types
| MIME Type | Output |
|---|---|
| `text/plain` | Free-form text (default) |
| `application/json` | JSON object/array |
| `text/x.enum` | Single enum value |

---

## Schema Enforcement

Define a strict schema that the model must follow:

```go
type ProductReview struct {
    ProductName string   `json:"product_name"`
    Rating      int      `json:"rating"`
    Sentiment   string   `json:"sentiment"`
    Pros        []string `json:"pros"`
    Cons        []string `json:"cons"`
    Summary     string   `json:"summary"`
}

config := &genai.GenerateContentConfig{
    ResponseMIMEType: "application/json",
    ResponseSchema: &genai.Schema{
        Type: genai.TypeObject,
        Properties: map[string]*genai.Schema{
            "product_name": {Type: genai.TypeString, Description: "Name of the product"},
            "rating":       {Type: genai.TypeInteger, Description: "Rating from 1 to 5"},
            "sentiment":    {Type: genai.TypeString, Enum: []string{"positive", "negative", "neutral"}},
            "pros":         {Type: genai.TypeArray, Items: &genai.Schema{Type: genai.TypeString}},
            "cons":         {Type: genai.TypeArray, Items: &genai.Schema{Type: genai.TypeString}},
            "summary":      {Type: genai.TypeString, Description: "One-sentence summary"},
        },
        Required: []string{"product_name", "rating", "sentiment", "summary"},
    },
}

result, err := client.Models.GenerateContent(ctx, "gemini-2.5-flash",
    genai.Text("Analyze this review: 'The new headphones are amazing! Great sound quality but the battery only lasts 4 hours.'"),
    config,
)
```

---

## Enum Constraints

### Single Enum Value
```go
config := &genai.GenerateContentConfig{
    ResponseMIMEType: "text/x.enum",
    ResponseSchema: &genai.Schema{
        Type: genai.TypeString,
        Enum: []string{"spam", "not_spam", "uncertain"},
    },
}

result, err := client.Models.GenerateContent(ctx, "gemini-2.5-flash",
    genai.Text("Classify this email: 'You won $1,000,000! Click here to claim!'"),
    config,
)
// result.Text() → "spam"
```

### Enum in Object
```go
"priority": &genai.Schema{
    Type:        genai.TypeString,
    Enum:        []string{"low", "medium", "high", "critical"},
    Description: "Priority level of the issue",
},
```

---

## Complex Nested Schemas

### Array of Objects
```go
config := &genai.GenerateContentConfig{
    ResponseMIMEType: "application/json",
    ResponseSchema: &genai.Schema{
        Type: genai.TypeArray,
        Items: &genai.Schema{
            Type: genai.TypeObject,
            Properties: map[string]*genai.Schema{
                "entity":   {Type: genai.TypeString, Description: "Named entity"},
                "type":     {Type: genai.TypeString, Enum: []string{"person", "organization", "location", "date"}},
                "context":  {Type: genai.TypeString, Description: "Surrounding context"},
            },
            Required: []string{"entity", "type"},
        },
    },
}
```

### Nested Objects
```go
"address": &genai.Schema{
    Type: genai.TypeObject,
    Properties: map[string]*genai.Schema{
        "street":  {Type: genai.TypeString},
        "city":    {Type: genai.TypeString},
        "country": {Type: genai.TypeString},
        "zip":     {Type: genai.TypeString},
    },
    Required: []string{"city", "country"},
},
```

---

## Parsing Responses

```go
func parseStructured[T any](result *genai.GenerateContentResponse) (T, error) {
    var parsed T
    text := result.Text()
    if err := json.Unmarshal([]byte(text), &parsed); err != nil {
        return parsed, fmt.Errorf("parse response: %w (raw: %s)", err, text)
    }
    return parsed, nil
}

// Usage
review, err := parseStructured[ProductReview](result)
if err != nil {
    return err
}
fmt.Printf("Product: %s, Rating: %d\n", review.ProductName, review.Rating)
```

### Rules
- Always define `ResponseSchema` alongside `ResponseMIMEType: "application/json"` for reliability
- Use `Required` fields to ensure critical data is always present
- Use `Enum` for classification/category fields — prevents hallucination
- Use `Description` on schema properties — helps the model understand intent
- Parse into typed Go structs, never `map[string]interface{}`
- Validate parsed values (range checks, format validation) before use
- Set `Temperature: 0.0` for deterministic extraction tasks
