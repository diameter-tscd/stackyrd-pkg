# LM Studio Manager

## Overview

The `LMStudioManager` is a Go library for interacting with LM Studio's local LLM API. It provides chat completion functionality with support for listing local models and async operations. The library connects to a locally running LM Studio instance via HTTP API and includes retry logic for reliability.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Chat Completions**: Create chat completions with conversation history
- **Model Listing**: List all available local models in LM Studio
- **Local LLM**: Connect to locally running LLM models via LM Studio
- **Retry Logic**: HTTP client with retry support (2 retries)
- **Worker Pool**: Async job execution support (2 workers)
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
    
    // Create LM Studio manager (configuration via viper)
    manager, err := lmstudio.NewLMStudioManager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // List available models
    models, err := manager.ListModels(ctx)
    if err != nil {
        panic(err)
    }
    fmt.Printf("Available models: %d\n", len(models.Data))
    for _, model := range models.Data {
        fmt.Printf("  %s\n", model.ID)
    }
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `LMStudioManager` | Main manager with HTTP client, worker pool |
| `LMStudioChatMessage` | Single message in a chat conversation (role, content) |
| `LMStudioChatRequest` | Request structure for chat completion |
| `LMStudioChatResponse` | Response structure with choices |
| `LMStudioChatChoice` | Single completion choice |
| `LMStudioModel` | Available model information |
| `LMStudioModelList` | List of available models |

### Concurrency Model

```
┌─────────────────────────────────────────────────────┐
│                   LMStudioManager                         │
├─────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │  retryablehttp │  → API requests with retry               │
│  │  .Client      │  → HTTP to local LM Studio server        │
│  └────────────────┘                                      │
│                                                          │
│  ┌─────────────┐      ┌──────────────┐                   │
│  │  WorkerPool │◄─────│ AsyncResult │                   │
│  │  (2 workers)│      │   Channel   │                   │
│  └─────────────┘      └──────────────┘                   │
│         ▲                      │                            │
│         │                      ▼                            │
│  ┌─────────────┐      ┌──────────────┐                  │
│  │  SubmitJob  │      │ ExecuteAsync │                  │
│  └─────────────┘      └──────────────┘                  │
│                                                          │
│  BaseURL: http://{host}:{port}/v1           │
│  Host: LM Studio host (default from config)              │
│  Port: LM Studio port (default from config)              │
└─────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewLMStudioManager(logger)
    │
    ├── Check viper config: "lmstudio.enabled"
    ├── Get host: "lmstudio.host"
    ├── Get port: "lmstudio.port"
    │
    ├── Build baseURL: "http://{host}:{port}/v1"
    │
    ├── Create retryablehttp.Client:
    │   ├── RetryMax = 2
    │   ├── RetryWaitMin = 500ms
    │   ├── RetryWaitMax = 3s
    │   └── HTTPClient.Timeout = 120s
    │
    ├── Test connection: testConnection()
    │   └── GET /models
    │
    ├── Create WorkerPool(2) - small pool for LLM operations
    ├── Start worker pool
    └── Return LMStudioManager
```

### 2. List Models Flow

```
ListModels(ctx)
    │
    ├── GET /models
    ├── Execute with retryablehttp.Client
    ├── Check status code (200 OK)
    ├── Decode JSON into LMStudioModelList
    └── Return *LMStudioModelList
```

### 3. Create Chat Completion Flow

```
CreateChatCompletion(ctx, request)
    │
    ├── Marshal request to JSON
    ├── POST /chat/completions
    ├── Set header: Content-Type = application/json
    ├── Execute with retryablehttp.Client
    ├── Check status code (200 OK)
    ├── Decode JSON into LMStudioChatResponse
    └── Return *LMStudioChatResponse
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `lmstudio.enabled` | bool | false | Enable/disable LM Studio manager |
| `lmstudio.host` | string | "localhost" | LM Studio host |
| `lmstudio.port` | int | 1234 | LM Studio port |

## Usage Examples

### List Available Models

```go
ctx := context.Background()

models, err := manager.ListModels(ctx)
if err != nil {
    panic(err)
}

fmt.Printf("Available models: %d\n", len(models.Data))
for _, model := range models.Data {
    fmt.Printf("  %s (owned by: %s)\n", model.ID, model.OwnedBy)
}
```

### Create Chat Completion

```go
ctx := context.Background()

request := lmstudio.LMStudioChatRequest{
    Messages: []lmstudio.LMStudioChatMessage{
        {
            Role:    "user",
            Content: "What is Go programming language?",
        },
    },
    Temperature: 0.7,
    MaxTokens:   2048,
}

response, err := manager.CreateChatCompletion(ctx, request)
if err != nil {
    panic(err)
}

if len(response.Choices) > 0 {
    fmt.Printf("Response: %s\n", response.Choices[0].Message.Content)
}
```

### Async Operations

```go
ctx := context.Background()

// Async list models
modelsChan := manager.ListModelsAsync(ctx)
models, err := modelsChan.Get(ctx)
if err != nil {
    panic(err)
}
fmt.Printf("Models: %d\n", len(models.Data))

// Async chat completion
request := lmstudio.LMStudioChatRequest{
    Messages: []lmstudio.LMStudioChatMessage{
        {
            Role:    "user",
            Content: "Explain recursion",
        },
    },
}
chatChan := manager.CreateChatCompletionAsync(ctx, request)
response, err := chatChan.Get(ctx)
if err != nil {
    panic(err)
}
```

### Getting Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
fmt.Printf("Base URL: %s\n", status["base_url"])
fmt.Printf("Available Models: %v\n", status["available_models"])

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
| Standard library | `encoding/json`, `net/http`, `bytes`, `io`, `time` |

## Error Handling

The library uses Go error handling patterns:

- **Connection failures**: LM Studio not running or not reachable
- **API errors**: Returned from LM Studio API calls
- **Nil checks**: Public methods check manager state

Example error handling:

```go
models, err := manager.ListModels(ctx)
if err != nil {
    if strings.Contains(err.Error(), "connection refused") {
        return fmt.Errorf("LM Studio is not running: %w", err)
    }
    return fmt.Errorf("API request failed: %w", err)
}
```

## Common Pitfalls

### 1. Manager Not Enabled

**Problem**: `lmstudio.enabled` is false

**Solution**:
- Set `lmstudio.enabled = true` in configuration
- Check viper configuration is loaded

### 2. LM Studio Not Running

**Problem**: Cannot connect to LM Studio

**Solution**:
- Ensure LM Studio is running locally
- Check that API server is enabled in LM Studio settings
- Verify host and port settings

### 3. No Models Loaded

**Problem**: ListModels returns empty or fails

**Solution**:
- Load a model in LM Studio
- Check that model is loaded and ready

### 4. Worker Pool Not Available

**Problem**: Async operations falling back to direct execution

**Solution**:
- The pool is created during initialization
- Check that `NewLMStudioManager()` succeeded
- Verify `lmstudio.enabled` is true

## Advanced Usage

### Health Check

```go
func healthCheck(manager *lmstudio.LMStudioManager) error {
    status := manager.GetStatus()
    if status["connected"] == false {
        return fmt.Errorf("not connected to LM Studio API")
    }
    fmt.Printf("LM Studio API is healthy (Models: %v)\n", status["available_models"])
    return nil
}
```

### Batch Processing with Async

```go
func batchChatRequests(manager *lmstudio.LMStudioManager, prompts []string) ([]string, error) {
    ctx := context.Background()
    results := make([]string, len(prompts))
    errors := make([]error, len(prompts))
    
    // Submit all requests asynchronously
    for i, prompt := range prompts {
        func(idx int, p string) {
            manager.SubmitAsyncJob(func() {
                request := lmstudio.LMStudioChatRequest{
                    Messages: []lmstudio.LMStudioChatMessage{
                        {
                            Role:    "user",
                            Content: p,
                        },
                    },
                }
                response, err := manager.CreateChatCompletion(ctx, request)
                if err == nil && len(response.Choices) > 0 {
                    results[idx] = response.Choices[0].Message.Content
                }
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

## Internal Algorithms

### API Request Flow

```
CreateChatCompletion():
    │
    ├── Build JSON payload with model, messages, temperature, max_tokens
    ├── POST to /chat/completions
    ├── Set Content-Type header
    ├── Execute with retryablehttp (2 retries)
    ├── Parse response JSON
    └── Return completion with choices
```

### Connection Test

```
testConnection():
    │
    ├── GET /models
    ├── Check status code (200 OK)
    └── Return error if failed
```

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/models` | GET | List available local models |
| `/chat/completions` | POST | Create chat completion |

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.