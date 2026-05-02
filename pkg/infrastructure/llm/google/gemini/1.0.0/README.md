# Google Gemini Manager

## Overview

The `GoogleGeminiManager` is a Go library for interacting with Google's Gemini API (Generative Language API). It provides content generation, multi-modal chat (text + images), embeddings, token counting, and model listing. The library supports streaming, safety settings, and async operations with worker pool.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Content Generation**: Generate text content from prompts or conversation history
- **Multi-Modal Chat**: Send text and inline data (images, audio) in a single request
- **Embeddings**: Generate vector embeddings for text
- **Token Counting**: Count tokens in content before generation
- **Model Listing**: List available Gemini models with capabilities
- **Batch Chat**: Send multiple prompts in batch
- **Streaming**: Stream generated content via Server-Sent Events (SSE)
- **Safety Settings**: Configurable safety filters for harmful content
- **Generation Config**: Control temperature, topP, topK, max output tokens
- **Retry Logic**: HTTP client with retry support (3 retries)
- **Worker Pool**: Async job execution support (4 workers)
- **Status Monitoring**: Get connection status and API health

## Quick Start

```go
package main

import (
    "context"
    "fmt"
    "stackyrd/pkg/infrastructure"
    "stackyrd/pkg/logger"
)

func main() {
    // Initialize logger
    log := logger.NewLogger()
    
    // Create Google Gemini manager (configuration via viper)
    manager, err := gemini.NewGoogleGeminiManager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // Simple chat
    response, err := manager.SimpleChat(ctx, "What is Go programming language?")
    if err != nil {
        panic(err)
    }
    fmt.Printf("Response: %s\n", response)
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `GoogleGeminiManager` | Main manager with HTTP client, API key, worker pool |
| `GeminiContent` | Content with role and parts (text or inline data) |
| `GeminiPart` | Single part of content (text or inline data) |
| `GeminiInlineData` | Base64-encoded inline data (image, audio, etc.) |
| `GeminiGenerateRequest` | Request structure for content generation |
| `GeminiGenerateResponse` | Response with candidates and usage metadata |
| `GeminiCandidate` | Single generated response candidate |
| `GeminiUsageMetadata` | Token usage information |
| `GeminiEmbeddingRequest` | Request for embedding generation |
| `GeminiEmbeddingResponse` | Response containing embedding values |
| `GeminiCountTokensRequest` | Request to count tokens |
| `GeminiCountTokensResponse` | Response with total token count |
| `GeminiModel` | Available model information |
| `GeminiModelList` | List of available models |
| `GeminiGenerationConfig` | Generation parameters (temperature, topP, topK, etc.) |
| `GeminiSafetySetting` | Safety filter configuration |
| `GeminiStreamChunk` | Single chunk from streaming response |

### Concurrency Model

```
┌─────────────────────────────────────────────────────┐
│                 GoogleGeminiManager                    │
├─────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │  retryablehttp │  → API requests with retry               │
│  │  .Client      │  → API key in URL query parameter       │
│  └────────────────┘                                      │
│                                                          │
│  ┌─────────────┐      ┌──────────────┐                   │
│  │  WorkerPool │◄─────│ AsyncResult │                   │
│  │  (4 workers)│      │   Channel   │                   │
│  └─────────────┘      └──────────────┘                   │
│         ▲                      │                            │
│         │                      ▼                            │
│  ┌─────────────┐      ┌──────────────┐                  │
│  │  SubmitJob  │      │ ExecuteAsync │                  │
│  └─────────────┘      └──────────────┘                  │
│                                                          │
│  BaseURL: https://generativelanguage.googleapis.com/v1beta │
│  APIKey: Google API key (passed as URL query param `key`) │
│  Model: Gemini model (e.g., gemini-1.5-flash)          │
└─────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewGoogleGeminiManager(logger)
    │
    ├── Check viper config: "google_gemini.enabled"
    ├── Get api_key: "google_gemini.api_key"
    ├── Get model: "google_gemini.model" (default: "gemini-1.5-flash")
    ├── Get max_tokens: "google_gemini.max_tokens" (default: 8192)
    ├── Get temperature: "google_gemini.temperature"
    ├── Get top_p: "google_gemini.top_p"
    ├── Get top_k: "google_gemini.top_k"
    │
    ├── Create retryablehttp.Client:
    │   ├── RetryMax = 3
    │   ├── RetryWaitMin = 1s
    │   ├── RetryWaitMax = 10s
    │   └── HTTPClient.Timeout = 120s
    │
    ├── Test connection: testConnection()
    │   └── GET /models/{model}?key={api_key}
    │
    ├── Create WorkerPool(4)
    ├── Start worker pool
    └── Return GoogleGeminiManager
```

### 2. Generate Content Flow

```
GenerateContent(ctx, contents)
    │
    ├── Build GeminiGenerateRequest:
    │   ├── Contents = contents
    │   ├── GenerationConfig (temperature, topP, topK, maxOutputTokens)
    │   └── SafetySettings (harassment, hate speech, etc.)
    │
    ├── Marshal request to JSON
    ├── POST /models/{model}:generateContent?key={api_key}
    ├── Set header: Content-Type = application/json
    ├── Execute with retryablehttp.Client
    ├── Check status code (200 OK)
    ├── Decode JSON into GeminiGenerateResponse
    └── Return *GeminiGenerateResponse
```

### 3. Simple Chat Flow

```
SimpleChat(ctx, prompt)
    │
    ├── Build contents array with single user message (role: "user", text: prompt)
    ├── Call GenerateContent(ctx, contents)
    ├── Check response candidates length
    └── Return candidate.Content.Parts[0].Text
```

### 4. Multi-Modal Chat Flow

```
MultiModalChat(ctx, prompt, mimeType, base64Data)
    │
    ├── Build contents array with user message containing:
    │   ├── Text part with prompt
    │   └── InlineData part with mimeType and base64 data
    ├── Call GenerateContent(ctx, contents)
    ├── Check response candidates length
    └── Return candidate.Content.Parts[0].Text
```

### 5. Generate Embedding Flow

```
GenerateEmbedding(ctx, content)
    │
    ├── Build GeminiEmbeddingRequest with text content
    ├── Marshal request to JSON
    ├── POST /models/embedding-001:embedContent?key={api_key}
    ├── Set header: Content-Type = application/json
    ├── Execute with retryablehttp.Client
    ├── Check status code (200 OK)
    ├── Decode JSON into GeminiEmbeddingResponse
    └── Return embedding values
```

### 6. Streaming Flow

```
StreamContent(ctx, contents, onChunk)
    │
    ├── Build GeminiGenerateRequest with generation config
    ├── Marshal request to JSON
    ├── POST /models/{model}:streamGenerateContent?alt=sse&key={api_key}
    ├── Use standard http.Client (for streaming)
    ├── Parse SSE events
    ├── For each chunk, call onChunk(text, isDone)
    └── Return error if any
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `google_gemini.enabled` | bool | false | Enable/disable Google Gemini manager |
| `google_gemini.api_key` | string | "" | Google API key |
| `google_gemini.model` | string | "gemini-1.5-flash" | Gemini model |
| `google_gemini.max_tokens` | int | 8192 | Max output tokens |
| `google_gemini.temperature` | float64 | 0.7 | Sampling temperature (0-2) |
| `google_gemini.top_p` | float64 | 0.95 | Nucleus sampling probability mass |
| `google_gemini.top_k` | int | 40 | Top-K sampling |

## Usage Examples

### Simple Chat

```go
ctx := context.Background()

response, err := manager.SimpleChat(ctx, "Explain quantum computing in simple terms")
if err != nil {
    panic(err)
}
fmt.Printf("Response: %s\n", response)
```

### Multi-Modal Chat (Text + Image)

```go
ctx := context.Background()

// Read image file and encode to base64
imageData := base64.StdEncoding.EncodeToString(imageBytes)

response, err := manager.MultiModalChat(ctx, "Describe this image", "image/jpeg", imageData)
if err != nil {
    panic(err)
}
fmt.Printf("Response: %s\n", response)
```

### Chat with History

```go
ctx := context.Background()

history := []gemini.GeminiContent{
    {
        Role: "user",
        Parts: []gemini.GeminiPart{
            {Text: "What is a closure in Go?"},
        },
    },
    {
        Role: "model",
        Parts: []gemini.GeminiPart{
            {Text: "A closure is a function that captures variables from its surrounding scope..."},
        },
    },
    {
        Role: "user",
        Parts: []gemini.GeminiPart{
            {Text: "Can you show an example?"},
        },
    },
}

result, err := manager.ChatWithHistory(ctx, history)
if err != nil {
    panic(err)
}

if len(result.Candidates) > 0 {
    for _, part := range result.Candidates[0].Content.Parts {
        fmt.Printf("Response: %s\n", part.Text)
    }
}
```

### Generate Embeddings

```go
ctx := context.Background()

embedding, err := manager.GenerateEmbedding(ctx, "This is a sample text for embedding")
if err != nil {
    panic(err)
}

fmt.Printf("Embedding dimensions: %d\n", len(embedding.Values))
fmt.Printf("First 5 values: %v\n", embedding.Values[:5])
```

### Count Tokens

```go
ctx := context.Background()

contents := []gemini.GeminiContent{
    {
        Role: "user",
        Parts: []gemini.GeminiPart{
            {Text: "This is a long prompt that I want to count tokens for"},
        },
    },
}

tokenCount, err := manager.CountTokens(ctx, contents)
if err != nil {
    panic(err)
}
fmt.Printf("Token count: %d\n", tokenCount)
```

### List Available Models

```go
ctx := context.Background()

models, err := manager.ListModels(ctx)
if err != nil {
    panic(err)
}

fmt.Printf("Available models: %d\n", len(models.Models))
for _, model := range models.Models {
    fmt.Printf("  %s (v%s) - Input: %d, Output: %d\n", 
        model.DisplayName, model.Version, model.InputTokenLimit, model.OutputTokenLimit)
}
```

### Batch Chat

```go
ctx := context.Background()

prompts := []string{
    "What is Go?",
    "What is Python?",
    "What is Rust?",
}

results, errors := manager.BatchChat(ctx, prompts)
for i, result := range results {
    if errors[i] != nil {
        fmt.Printf("Prompt %d failed: %v\n", i, errors[i])
    } else {
        fmt.Printf("Prompt %d: %s\n", i, result)
    }
}
```

### Streaming Content

```go
ctx := context.Background()

contents := []gemini.GeminiContent{
    {
        Role: "user",
        Parts: []gemini.GeminiPart{
            {Text: "Write a short story about a robot"},
        },
    },
}

err := manager.StreamContent(ctx, contents, func(text string, isDone bool) {
    fmt.Print(text)
    if isDone {
        fmt.Println("\n[Stream complete]")
    }
})
if err != nil {
    panic(err)
}
```

### Async Operations

```go
ctx := context.Background()

// Async simple chat
responseChan := manager.SimpleChatAsync(ctx, "What is the capital of France?")
response, err := responseChan.Get(ctx)
if err != nil {
    panic(err)
}
fmt.Printf("Response: %s\n", response)

// Async embedding
embeddingChan := manager.GenerateEmbeddingAsync(ctx, "Sample text")
embedding, err := embeddingChan.Get(ctx)
if err != nil {
    panic(err)
}
fmt.Printf("Embedding dimensions: %d\n", len(embedding.Values))
```

### Getting Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
fmt.Printf("Model: %s\n", status["model"])
fmt.Printf("Base URL: %s\n", status["base_url"])
fmt.Printf("Available Models: %v\n", status["available_models"])
fmt.Printf("Max Tokens: %v\n", status["max_tokens"])
fmt.Printf("Temperature: %v\n", status["temperature"])

if status["connected"] == false {
    if errMsg, ok := status["error"].(string); ok {
        fmt.Printf("Error: %s\n", errMsg)
    }
}
```

## Dependencies

| Dependency | Role |
|------------|------|
| `github.com/hashicorp/go-retryablehttp` | HTTP client with retry logic |
| `github.com/spf13/viper` | Configuration management |
| `stackyrd/config` | Internal configuration package |
| `stackyrd/pkg/logger` | Structured logging |
| Standard library | `encoding/json`, `net/http`, `bytes`, `io`, `time`, `encoding/base64` |

## Error Handling

The library uses Go error handling patterns:

- **Authentication failures**: Returned from API (invalid API key)
- **API errors**: Returned from Google Gemini API calls
- **Nil checks**: Public methods check manager state

Example error handling:

```go
result, err := manager.GenerateContent(ctx, contents)
if err != nil {
    if strings.Contains(err.Error(), "400") {
        return fmt.Errorf("bad request: %w", err)
    }
    if strings.Contains(err.Error(), "403") {
        return fmt.Errorf("invalid API key: %w", err)
    }
    if strings.Contains(err.Error(), "429") {
        return fmt.Errorf("rate limit exceeded: %w", err)
    }
    return fmt.Errorf("API request failed: %w", err)
}
```

## Common Pitfalls

### 1. Manager Not Enabled

**Problem**: `google_gemini.enabled` is false

**Solution**:
- Set `google_gemini.enabled = true` in configuration
- Check viper configuration is loaded

### 2. API Key Missing

**Problem**: `api_key` not set

**Solution**:
- Set `google_gemini.api_key` to your Google API key
- Get API key from https://aistudio.google.com/app/apikey

### 3. Invalid Model

**Problem**: `model` not set or invalid

**Solution**:
- Set `google_gemini.model` to a valid model (e.g., "gemini-1.5-flash", "gemini-1.5-pro")
- Check https://ai.google.dev/models for available models

### 4. Rate Limiting

**Problem**: `429` Too Many Requests

**Solution**:
- Library retries up to 3 times
- Consider using async operations to spread requests
- Respect Google API rate limits

### 5. Token Limits

**Problem**: Response truncated or error about token limits

**Solution**:
- Adjust `google_gemini.max_tokens` to control response length
- Use `CountTokens` to check token count before generation
- Gemini 1.5 models support up to 1M tokens context window

### 6. Safety Blocks

**Problem**: Content blocked by safety filters

**Solution**:
- Check `PromptFeedback.BlockReason` in response
- Adjust safety settings in `GenerateContent` or use default
- Review safety ratings in candidates

### 7. Worker Pool Not Available

**Problem**: Async operations falling back to direct execution

**Solution**:
- The pool is created during initialization
- Check that `NewGoogleGeminiManager()` succeeded
- Verify `google_gemini.enabled` is true

## Advanced Usage

### Custom Generation Config

```go
func customGeneration(manager *gemini.GoogleGeminiManager) error {
    ctx := context.Background()
    
    contents := []gemini.GeminiContent{
        {
            Role: "user",
            Parts: []gemini.GeminiPart{
                {Text: "Explain recursion with an example"},
            },
        },
    }
    
    // Build custom request with specific parameters
    payload := gemini.GeminiGenerateRequest{
        Contents: contents,
        GenerationConfig: &gemini.GeminiGenerationConfig{
            Temperature:     0.3,
            TopP:            0.8,
            TopK:            20,
            MaxOutputTokens: 2048,
            StopSequences:   []string{"END"},
            CandidateCount:  1,
        },
        SafetySettings: []gemini.GeminiSafetySetting{
            {Category: "HARM_CATEGORY_HARASSMENT", Threshold: "BLOCK_ONLY_HIGH"},
        },
    }
    
    jsonData, err := json.Marshal(payload)
    if err != nil {
        return err
    }
    
    endpoint := fmt.Sprintf("%s/models/%s:generateContent?key=%s", 
        manager.BaseURL, manager.Model, manager.APIKey)
    req, err := retryablehttp.NewRequestWithContext(ctx, "POST", endpoint, bytes.NewReader(jsonData))
    if err != nil {
        return err
    }
    
    req.Header.Set("Content-Type", "application/json")
    resp, err := manager.Client.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    
    // Process response...
    return nil
}
```

### Health Check

```go
func healthCheck(manager *gemini.GoogleGeminiManager) error {
    status := manager.GetStatus()
    if status["connected"] == false {
        return fmt.Errorf("not connected to Google Gemini API")
    }
    fmt.Printf("Google Gemini API is healthy (Model: %s)\n", status["model"])
    return nil
}
```

### Batch Processing with Async

```go
func batchChatRequests(manager *gemini.GoogleGeminiManager, prompts []string) ([]string, error) {
    ctx := context.Background()
    results := make([]string, len(prompts))
    errors := make([]error, len(prompts))
    
    // Submit all requests asynchronously
    for i, prompt := range prompts {
        func(idx int, p string) {
            manager.SubmitAsyncJob(func() {
                response, err := manager.SimpleChat(ctx, p)
                results[idx] = response
                errors[idx] = err
            })
        }(i, prompt)
    }
    
    // Collect results
    for i, err := range errors {
        if err != nil {
            return nil, fmt.Errorf("failed to process prompt %d: %w", i, err)
        }
    }
    return results, nil
}
```

### Multi-Modal with Multiple Images

```go
func multiImageChat(manager *gemini.GoogleGeminiManager) error {
    ctx := context.Background()
    
    contents := []gemini.GeminiContent{
        {
            Role: "user",
            Parts: []gemini.GeminiPart{
                {Text: "Compare these two images:"},
                {
                    InlineData: &gemini.GeminiInlineData{
                        MIMEType: "image/jpeg",
                        Data:     base64Image1,
                    },
                },
                {
                    InlineData: &gemini.GeminiInlineData{
                        MIMEType: "image/jpeg",
                        Data:     base64Image2,
                    },
                },
            },
        },
    }
    
    result, err := manager.GenerateContent(ctx, contents)
    if err != nil {
        return err
    }
    
    if len(result.Candidates) > 0 {
        for _, part := range result.Candidates[0].Content.Parts {
            fmt.Printf("Response: %s\n", part.Text)
        }
    }
    return nil
}
```

## Internal Algorithms

### API Request Flow

```
GenerateContent():
    │
    ├── Build JSON payload with contents, generationConfig, safetySettings
    ├── POST to /models/{model}:generateContent?key={api_key}
    ├── Set Content-Type header
    ├── Execute with retryablehttp (3 retries)
    ├── Parse response JSON
    └── Return candidates with text and usage stats
```

### Token Counting

```
CountTokens():
    │
    ├── Build JSON payload with contents
    ├── POST to /models/{model}:countTokens?key={api_key}
    ├── Parse response for totalTokens
    └── Return token count
```

### Embedding Generation

```
GenerateEmbedding():
    │
    ├── Build JSON payload with text content
    ├── POST to /models/embedding-001:embedContent?key={api_key}
    ├── Parse response for embedding values
    └── Return embedding vector
```

### Connection Test

```
testConnection():
    │
    ├── GET /models/{model}?key={api_key}
    ├── Check status code (200 OK)
    └── Return error if failed
```

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/models?key={key}&pageSize=50` | GET | List available models |
| `/models/{model}?key={key}` | GET | Get specific model info |
| `/models/{model}:generateContent?key={key}` | POST | Generate content |
| `/models/{model}:streamGenerateContent?alt=sse&key={key}` | POST | Stream content (SSE) |
| `/models/{model}:countTokens?key={key}` | POST | Count tokens |
| `/models/embedding-001:embedContent?key={key}` | POST | Generate embedding |

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.