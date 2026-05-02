# OpenRouter Manager

## Overview

The `OpenRouterManager` is a Go library for interacting with the OpenRouter API. It provides chat completion functionality with support for listing available models and async operations. The library uses OpenRouter API with Bearer token authentication and includes retry logic for reliability.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Chat Completions**: Create chat completions with conversation history
- **Model Listing**: List all available models through OpenRouter
- **Simple Chat**: Send a single prompt and get a response
- **Model Configuration**: Configurable model, temperature, max tokens
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
    
    // Create OpenRouter manager (configuration via viper)
    manager, err := openrouter.NewOpenRouterManager(log)
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
        fmt.Printf("  %s (owned by: %s)\n", model.ID, model.OwnedBy)
    }
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `OpenRouterManager` | Main manager with HTTP client, API key, worker pool |
| `OpenRouterChatMessage` | Single message in a chat conversation (role, content) |
| `OpenRouterChatRequest` | Request structure for chat completion |
| `OpenRouterChatResponse` | Response structure with choices |
| `OpenRouterChatChoice` | Single completion choice |
| `OpenRouterModel` | Available model information |
| `OpenRouterModelList` | List of available models |

### Concurrency Model

```
┌─────────────────────────────────────────────────────┐
│                 OpenRouterManager                       │
├─────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │  retryablehttp │  → API requests with retry               │
│  │  .Client      │  → Bearer token authentication           │
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
│  BaseURL: https://openrouter.ai/api/v1           │
│  APIKey: OpenRouter API key (Bearer token)           │
│  Model: OpenRouter model (e.g., "openai/gpt-3.5-turbo") │
└─────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewOpenRouterManager(logger)
    │
    ├── Check viper config: "openrouter.enabled"
    ├── Get api_key: "openrouter.api_key"
    ├── Get model: "openrouter.model"
    ├── Get temperature: "openrouter.temperature"
    ├── Get max_tokens: "openrouter.max_tokens"
    │
    ├── Build baseURL: "https://openrouter.ai/api/v1"
    │
    ├── Create retryablehttp.Client:
    │   ├── RetryMax = 3
    │   ├── RetryWaitMin = 1s
    │   ├── RetryWaitMax = 10s
    │   └── HTTPClient.Timeout = 120s
    │
    ├── Test connection: testConnection()
    │   └── GET /models (with Bearer token)
    │
    ├── Create WorkerPool(4)
    ├── Start worker pool
    └── Return OpenRouterManager
```

### 2. List Models Flow

```
ListModels(ctx)
    │
    ├── GET /models
    ├── Set header: Authorization = Bearer <api_key>
    ├── Execute with retryablehttp.Client
    ├── Check status code (200 OK)
    ├── Decode JSON into OpenRouterModelList
    └── Return *OpenRouterModelList
```

### 3. Create Chat Completion Flow

```
CreateChatCompletion(ctx, request)
    │
    ├── If request.Model is empty, set to manager.Model
    ├── If request.Temperature is 0, set to manager.Temperature
    ├── If request.MaxTokens is 0, set to manager.MaxTokens
    │
    ├── Marshal request to JSON
    ├── POST /chat/completions
    ├── Set header: Content-Type = application/json
    ├── Set header: Authorization = Bearer <api_key>
    ├── Execute with retryablehttp.Client
    ├── Check status code (200 OK)
    ├── Decode JSON into OpenRouterChatResponse
    └── Return *OpenRouterChatResponse
```

### 4. Simple Chat Flow

```
SimpleChat(ctx, prompt)
    │
    ├── Build messages array with single user message
    ├── Call CreateChatCompletion(ctx, request)
    ├── Check response choices length
    └── Return response.Choices[0].Message.Content
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `openrouter.enabled` | bool | false | Enable/disable OpenRouter manager |
| `openrouter.api_key` | string | "" | OpenRouter API key |
| `openrouter.model` | string | "" | Model to use (e.g., "openai/gpt-3.5-turbo") |
| `openrouter.temperature` | float64 | 0.7 | Sampling temperature (0-2) |
| `openrouter.max_tokens` | int | 2048 | Max tokens in response |

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

request := openrouter.OpenRouterChatRequest{
    Messages: []openrouter.OpenRouterChatMessage{
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

### Simple Chat

```go
ctx := context.Background()

response, err := manager.SimpleChat(ctx, "Explain quantum computing in simple terms")
if err != nil {
    panic(err)
}
fmt.Printf("Response: %s\n", response)
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
request := openrouter.OpenRouterChatRequest{
    Messages: []openrouter.OpenRouterChatMessage{
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

// Async simple chat
simpleChan := manager.SimpleChatAsync(ctx, "What is the capital of France?")
simpleResp, err := simpleChan.Get(ctx)
if err != nil {
    panic(err)
}
fmt.Printf("Response: %s\n", simpleResp)
```

### Getting Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
fmt.Printf("Base URL: %s\n", status["base_url"])
fmt.Printf("Model: %s\n", status["model"])
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

- **Authentication failures**: Returned from API (invalid API key)
- **API errors**: Returned from OpenRouter API calls
- **Nil checks**: Public methods check manager state

Example error handling:

```go
models, err := manager.ListModels(ctx)
if err != nil {
    if strings.Contains(err.Error(), "401") {
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

**Problem**: `openrouter.enabled` is false

**Solution**:
- Set `openrouter.enabled = true` in configuration
- Check viper configuration is loaded

### 2. API Key Missing

**Problem**: `api_key` not set

**Solution**:
- Set `openrouter.api_key` to your OpenRouter API key
- Get API key from https://openrouter.ai/keys

### 3. Invalid Model

**Problem**: `model` not set or invalid

**Solution**:
- Set `openrouter.model` to a valid model (e.g., "openai/gpt-3.5-turbo", "anthropic/claude-3-sonnet")
- Check https://openrouter.ai/models for available models

### 4. Rate Limiting

**Problem**: `429` Too Many Requests

**Solution**:
- Library retries up to 3 times
- Consider using async operations to spread requests
- Respect OpenRouter rate limits

### 5. Worker Pool Not Available

**Problem**: Async operations falling back to direct execution

**Solution**:
- The pool is created during initialization
- Check that `NewOpenRouterManager()` succeeded
- Verify `openrouter.enabled` is true

## Advanced Usage

### Health Check

```go
func healthCheck(manager *openrouter.OpenRouterManager) error {
    status := manager.GetStatus()
    if status["connected"] == false {
        return fmt.Errorf("not connected to OpenRouter API")
    }
    fmt.Printf("OpenRouter API is healthy (Model: %s)\n", status["model"])
    return nil
}
```

### Batch Processing with Async

```go
func batchChatRequests(manager *openrouter.OpenRouterManager, prompts []string) ([]string, error) {
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

## Internal Algorithms

### API Request Flow

```
CreateChatCompletion():
    │
    ├── Build JSON payload with model, messages, temperature, max_tokens
    ├── POST to /chat/completions
    ├── Set Authorization header with Bearer token
    ├── Execute with retryablehttp (3 retries)
    ├── Parse response JSON
    └── Return completion with choices
```

### Connection Test

```
testConnection():
    │
    ├── GET /models
    ├── Set Authorization header
    ├── Check status code (200 OK)
    └── Return error if failed
```

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/models` | GET | List available models |
| `/chat/completions` | POST | Create chat completion |

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.