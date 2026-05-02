# Claude Manager

## Overview

The `ClaudeManager` is a Go library for interacting with Anthropic's Claude API. It provides chat completion functionality with support for conversation history, streaming, and async operations. The library uses the Anthropic API v1 with API key authentication (x-api-key header) and includes retry logic for reliability.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Chat Completions**: Create chat completions with conversation history
- **Simple Chat**: Send a single prompt and get a response
- **Model Configuration**: Configurable model, temperature, max tokens
- **Retry Logic**: HTTP client with retry support (3 retries)
- **Worker Pool**: Async job execution support (4 workers)
- **Status Monitoring**: Get connection status and API health
- **Streaming Support**: Can be extended for streaming responses

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
    
    // Create Claude manager (configuration via viper)
    manager, err := claude.NewClaudeManager(log)
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
| `ClaudeManager` | Main manager with HTTP client, API key, worker pool |
| `ClaudeMessage` | Single message in a chat conversation (role, content) |
| `ClaudeCompletionRequest` | Request structure for chat completion |
| `ClaudeCompletionResponse` | Response structure with content and usage stats |

### Concurrency Model

```
┌─────────────────────────────────────────────────────┐
│                   ClaudeManager                         │
├─────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │  retryablehttp │  → API requests with retry               │
│  │  .Client      │  → x-api-key authentication              │
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
│  BaseURL: https://api.anthropic.com/v1           │
│  APIKey: Anthropic API key                         │
│  Version: 2023-06-01 (anthropic-version header)    │
└─────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewClaudeManager(logger)
    │
    ├── Check viper config: "claude.enabled"
    ├── Get model: "claude.model"
    ├── Get api_key: "claude.api_key"
    ├── Get temperature: "claude.temperature"
    ├── Get max_tokens: "claude.max_tokens"
    │
    ├── Create retryablehttp.Client:
    │   ├── RetryMax = 3
    │   ├── RetryWaitMin = 1s
    │   ├── RetryWaitMax = 10s
    │   └── HTTPClient.Timeout = 120s
    │
    ├── Test connection: testConnection()
    │   └── GET /models
    │
    ├── Create WorkerPool(4)
    ├── Start worker pool
    └── Return ClaudeManager
```

### 2. Create Chat Completion Flow

```
CreateChatCompletion(ctx, messages)
    │
    ├── Build ClaudeCompletionRequest:
    │   ├── Model = c.Model
    │   ├── Messages = messages
    │   ├── Temperature = c.Temperature
    │   └── MaxTokens = c.MaxTokens
    │
    ├── Marshal request to JSON
    ├── POST /messages
    ├── Set header: x-api-key = <api_key>
    ├── Set header: anthropic-version = "2023-06-01"
    ├── Set header: Content-Type = application/json
    ├── Execute with retryablehttp.Client
    ├── Check status code (200 OK)
    ├── Decode JSON into ClaudeCompletionResponse
    └── Return *ClaudeCompletionResponse
```

### 3. Simple Chat Flow

```
SimpleChat(ctx, prompt)
    │
    ├── Build messages array with single user message
    ├── Call CreateChatCompletion(ctx, messages)
    ├── Check response content length
    └── Return response.Content[0].Text
```

### 4. Get Status Flow

```
GetStatus()
    │
    ├── If nil: return connected=false
    ├── Create context with 5s timeout
    ├── GET /models
    ├── Set header: x-api-key = <api_key>
    ├── Set header: anthropic-version = "2023-06-01"
    ├── Return connected, model, temperature, max_tokens, pool_active
    └── Note: Tests actual API connectivity
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `claude.enabled` | bool | false | Enable/disable Claude manager |
| `claude.model` | string | "" | Claude model (e.g., "claude-3-opus-20240229", "claude-3-sonnet-20240229") |
| `claude.api_key` | string | "" | Anthropic API key |
| `claude.temperature` | float64 | 0.7 | Sampling temperature (0-2) |
| `claude.max_tokens` | int | 2048 | Max tokens in response |

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

### Chat with Conversation History

```go
ctx := context.Background()

messages := []claude.ClaudeMessage{
    {
        Role:    "system",
        Content: "You are a helpful assistant specialized in Go programming.",
    },
    {
        Role:    "user",
        Content: "How do I create a goroutine?",
    },
}

result, err := manager.CreateChatCompletion(ctx, messages)
if err != nil {
    panic(err)
}

if len(result.Content) > 0 {
    fmt.Printf("Response: %s\n", result.Content[0].Text)
    fmt.Printf("Usage: Input tokens: %d, Output tokens: %d\n", 
        result.Usage.InputTokens, result.Usage.OutputTokens)
}
```

### Async Chat Completion

```go
ctx := context.Background()

messages := []claude.ClaudeMessage{
    {
        Role:    "user",
        Content: "Write a Go function to reverse a string",
    },
}

// Async completion
resultChan := manager.CreateChatCompletionAsync(ctx, messages)
result, err := resultChan.Get(ctx)
if err != nil {
    panic(err)
}

if len(result.Content) > 0 {
    fmt.Printf("Response: %s\n", result.Content[0].Text)
}
```

### Async Simple Chat

```go
ctx := context.Background()

responseChan := manager.SimpleChatAsync(ctx, "What is the capital of France?")
response, err := responseChan.Get(ctx)
if err != nil {
    panic(err)
}
fmt.Printf("Response: %s\n", response)
```

### Getting Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
fmt.Printf("Model: %s\n", status["model"])
fmt.Printf("Temperature: %v\n", status["temperature"])
fmt.Printf("Max Tokens: %v\n", status["max_tokens"])
fmt.Printf("API Version: %s\n", status["api_version"])

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
- **API errors**: Returned from Anthropic API calls
- **Nil checks**: Public methods check manager state

Example error handling:

```go
result, err := manager.CreateChatCompletion(ctx, messages)
if err != nil {
    if strings.Contains(err.Error(), "401") {
        return fmt.Errorf("invalid API key: %w", err)
    }
    if strings.Contains(err.Error(), "429") {
        return fmt.Errorf("rate limit exceeded: %w", err)
    }
    if strings.Contains(err.Error(), "500") {
        return fmt.Errorf("Anthropic server error: %w", err)
    }
    return fmt.Errorf("API request failed: %w", err)
}
```

## Common Pitfalls

### 1. Manager Not Enabled

**Problem**: `claude.enabled` is false

**Solution**:
- Set `claude.enabled = true` in configuration
- Check viper configuration is loaded

### 2. API Key Missing

**Problem**: `api_key` not set

**Solution**:
- Set `claude.api_key` to your Anthropic API key
- Get API key from https://console.anthropic.com/settings/keys

### 3. Invalid Model

**Problem**: `model` not set or invalid

**Solution**:
- Set `claude.model` to a valid model (e.g., "claude-3-opus-20240229", "claude-3-sonnet-20240229")
- Check https://docs.anthropic.com/en/docs/about-claude/models for available models

### 4. Rate Limiting

**Problem**: `429` Too Many Requests

**Solution**:
- Library retries up to 3 times
- Consider using async operations to spread requests
- Respect Anthropic rate limits

### 5. Token Limits

**Problem**: Response truncated or error about token limits

**Solution**:
- Adjust `claude.max_tokens` to control response length
- Be mindful of total tokens (input + output)
- Claude 3 models support up to 200K tokens context window

### 6. Worker Pool Not Available

**Problem**: Async operations falling back to direct execution

**Solution**:
- The pool is created during initialization
- Check that `NewClaudeManager()` succeeded
- Verify `claude.enabled` is true

## Advanced Usage

### Multi-turn Conversation

```go
func multiTurnConversation(manager *claude.ClaudeManager) error {
    ctx := context.Background()
    
    messages := []claude.ClaudeMessage{
        {
            Role:    "system",
            Content: "You are a helpful programming tutor.",
        },
    }
    
    // Turn 1
    messages = append(messages, claude.ClaudeMessage{
        Role:    "user",
        Content: "What is a closure in Go?",
    })
    
    result, err := manager.CreateChatCompletion(ctx, messages)
    if err != nil {
        return err
    }
    
    if len(result.Content) > 0 {
        response := result.Content[0].Text
        fmt.Printf("Turn 1: %s\n", response)
        // Add assistant response to history
        messages = append(messages, claude.ClaudeMessage{
            Role:    "assistant",
            Content: response,
        })
    }
    
    // Turn 2
    messages = append(messages, claude.ClaudeMessage{
        Role:    "user",
        Content: "Can you show an example?",
    })
    
    result, err = manager.CreateChatCompletion(ctx, messages)
    if err != nil {
        return err
    }
    
    // ... process response
    return nil
}
```

### Custom Request with All Parameters

```go
func customChatRequest(manager *claude.ClaudeManager) error {
    ctx := context.Background()
    
    messages := []claude.ClaudeMessage{
        {
            Role:    "user",
            Content: "Explain recursion with an example",
        },
    }
    
    // Create custom request with specific parameters
    payload := claude.ClaudeCompletionRequest{
        Model:       "claude-3-opus-20240229",
        Messages:    messages,
        Temperature: 0.5,
        MaxTokens:   1024,
        Stream:      false,
    }
    
    jsonData, err := json.Marshal(payload)
    if err != nil {
        return err
    }
    
    req, err := retryablehttp.NewRequestWithContext(ctx, "POST", 
        manager.BaseURL+"/messages", bytes.NewReader(jsonData))
    if err != nil {
        return err
    }
    
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("x-api-key", manager.APIKey)
    req.Header.Set("anthropic-version", manager.Version)
    
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
func healthCheck(manager *claude.ClaudeManager) error {
    status := manager.GetStatus()
    if status["connected"] == false {
        return fmt.Errorf("not connected to Claude API")
    }
    fmt.Printf("Claude API is healthy (Model: %s)\n", status["model"])
    return nil
}
```

### Batch Processing with Async

```go
func batchChatRequests(manager *claude.ClaudeManager, prompts []string) ([]string, error) {
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
    ├── POST to /messages
    ├── Set x-api-key header with API key
    ├── Set anthropic-version header
    ├── Execute with retryablehttp (3 retries)
    ├── Parse response JSON
    └── Return completion with content and usage stats
```

### Token Counting

```
Usage stats in response:
{
  "usage": {
    "input_tokens": 50,
    "output_tokens": 100,
  }
}

Use these to monitor token usage and costs.
```

### Connection Test

```
testConnection():
    │
    ├── GET /models
    ├── Set x-api-key header
    ├── Set anthropic-version header
    ├── Check status code (200 OK)
    └── Return error if failed
```

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/models` | GET | List available models |
| `/messages` | POST | Create chat completion |

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.