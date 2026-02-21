# Function Calling (Tool Use)

## Table of Contents
- [Overview](#overview)
- [Defining Tools](#defining-tools)
- [Automatic Function Calling](#automatic-function-calling)
- [Manual Function Calling](#manual-function-calling)
- [Multiple Tools](#multiple-tools)
- [Best Practices](#best-practices)

---

## Overview

Function Calling allows Gemini to invoke external functions/APIs by generating structured arguments. Gemini never executes functions — it returns a `FunctionCall` with the name and arguments, and your code executes the function and returns the result.

**Flow:**
```
User prompt → Gemini → FunctionCall{name, args}
                           ↓
              Your code executes function
                           ↓
              FunctionResponse{name, result} → Gemini → Final answer
```

---

## Defining Tools

```go
// Define a function declaration (OpenAPI-style schema)
getWeatherTool := &genai.Tool{
    FunctionDeclarations: []*genai.FunctionDeclaration{
        {
            Name:        "get_weather",
            Description: "Get the current weather for a given location. Use when the user asks about weather conditions.",
            Parameters: &genai.Schema{
                Type: genai.TypeObject,
                Properties: map[string]*genai.Schema{
                    "location": {
                        Type:        genai.TypeString,
                        Description: "City name, e.g. 'Buenos Aires, Argentina'",
                    },
                    "unit": {
                        Type:        genai.TypeString,
                        Enum:        []string{"celsius", "fahrenheit"},
                        Description: "Temperature unit",
                    },
                },
                Required: []string{"location"},
            },
        },
    },
}
```

### Schema Types
| Type | Go Constant | JSON |
|---|---|---|
| String | `genai.TypeString` | `"string"` |
| Number | `genai.TypeNumber` | `"number"` |
| Integer | `genai.TypeInteger` | `"integer"` |
| Boolean | `genai.TypeBoolean` | `"boolean"` |
| Array | `genai.TypeArray` | `"array"` |
| Object | `genai.TypeObject` | `"object"` |

---

## Automatic Function Calling

The SDK can automatically call functions and feed results back to the model:

```go
func main() {
    // ... client setup ...

    // Define the actual Go function
    getWeather := func(location string, unit string) map[string]any {
        // Call your weather API here
        return map[string]any{
            "temperature": 22,
            "condition":   "sunny",
            "location":    location,
            "unit":        unit,
        }
    }

    config := &genai.GenerateContentConfig{
        Tools: []*genai.Tool{getWeatherTool},
    }

    result, err := client.Models.GenerateContent(ctx, "gemini-2.5-flash",
        genai.Text("What's the weather in Buenos Aires?"),
        config,
    )
    if err != nil {
        log.Fatal(err)
    }

    // Check if the model wants to call a function
    for _, candidate := range result.Candidates {
        for _, part := range candidate.Content.Parts {
            if fc, ok := part.(*genai.FunctionCall); ok {
                fmt.Printf("Function: %s, Args: %v\n", fc.Name, fc.Args)

                // Execute the function
                location, _ := fc.Args["location"].(string)
                unit, _ := fc.Args["unit"].(string)
                if unit == "" {
                    unit = "celsius"
                }
                weatherResult := getWeather(location, unit)

                // Send function result back to model
                finalResult, err := client.Models.GenerateContent(ctx, "gemini-2.5-flash",
                    &genai.FunctionResponse{
                        Name:     fc.Name,
                        Response: weatherResult,
                    },
                    config,
                )
                if err != nil {
                    log.Fatal(err)
                }
                fmt.Println(finalResult.Text())
            }
        }
    }
}
```

---

## Manual Function Calling

For more control over which functions get called:

```go
config := &genai.GenerateContentConfig{
    Tools: []*genai.Tool{getWeatherTool},
    ToolConfig: &genai.ToolConfig{
        FunctionCallingConfig: &genai.FunctionCallingConfig{
            // AUTO (default): model decides
            // ANY: model must call a function
            // NONE: disable function calling
            Mode: genai.FunctionCallingAuto,

            // Restrict to specific functions (optional)
            AllowedFunctionNames: []string{"get_weather"},
        },
    },
}
```

### Mode Selection
| Mode | Behavior | Use When |
|---|---|---|
| `AUTO` | Model decides whether to call a function | General assistant, mixed queries |
| `ANY` | Model must call one of the provided functions | Strict tool-use flows, form filling |
| `NONE` | Functions are described but never called | System instructions reference, testing |

---

## Multiple Tools

```go
tools := []*genai.Tool{
    {
        FunctionDeclarations: []*genai.FunctionDeclaration{
            {
                Name:        "search_database",
                Description: "Search the product database by query",
                Parameters: &genai.Schema{
                    Type: genai.TypeObject,
                    Properties: map[string]*genai.Schema{
                        "query":    {Type: genai.TypeString, Description: "Search query"},
                        "category": {Type: genai.TypeString, Enum: []string{"electronics", "books", "clothing"}},
                        "max_results": {Type: genai.TypeInteger, Description: "Max results to return"},
                    },
                    Required: []string{"query"},
                },
            },
            {
                Name:        "get_order_status",
                Description: "Get the current status of an order by its ID",
                Parameters: &genai.Schema{
                    Type: genai.TypeObject,
                    Properties: map[string]*genai.Schema{
                        "order_id": {Type: genai.TypeString, Description: "The order ID (format: ORD-XXXXX)"},
                    },
                    Required: []string{"order_id"},
                },
            },
        },
    },
}

// Dispatch function calls
func dispatch(fc *genai.FunctionCall) (map[string]any, error) {
    switch fc.Name {
    case "search_database":
        query := fc.Args["query"].(string)
        return searchDB(query, fc.Args)
    case "get_order_status":
        orderID := fc.Args["order_id"].(string)
        return getOrderStatus(orderID)
    default:
        return nil, fmt.Errorf("unknown function: %s", fc.Name)
    }
}
```

---

## Best Practices

### Function Descriptions
- Write clear, specific descriptions — the model uses them to decide when to call
- Include examples in descriptions: `"Get weather for a city, e.g. 'Buenos Aires'"`
- Use `Enum` for constrained values to prevent hallucination

### Argument Validation
```go
func validateArgs(fc *genai.FunctionCall) error {
    switch fc.Name {
    case "search_database":
        query, ok := fc.Args["query"].(string)
        if !ok || query == "" {
            return errors.New("query is required")
        }
        if len(query) > 500 {
            return errors.New("query too long")
        }
    }
    return nil
}
```

### Rules
- Always validate function call arguments before executing
- Never execute destructive operations (DELETE, DROP) without user confirmation
- Limit the number of tools per request (≤10) for better model accuracy
- Use `Required` fields to ensure critical arguments are provided
- Log all function calls and results for debugging and audit
- Set timeouts on the actual function execution, not just the API call
