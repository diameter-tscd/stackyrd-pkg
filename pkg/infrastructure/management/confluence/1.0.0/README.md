# Confluence Manager

## Overview

The `ConfluenceManager` is a Go library for interacting with the Atlassian Confluence API. It provides page retrieval with storage body expansion, async operations, and connection testing. The library uses HTTP Basic authentication (username + API token) and includes retry logic for reliability.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Page Retrieval**: Fetch Confluence pages by ID with body storage expansion
- **Basic Authentication**: Secure API access with username and API token
- **Retry Logic**: HTTP client with retry support (3 retries)
- **Worker Pool**: Async job execution support (4 workers)
- **Status Monitoring**: Get connection status and API health
- **Async Operations**: Asynchronous page retrieval

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
    
    // Create Confluence manager (configuration via viper)
    manager, err := confluence.NewConfluenceManager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // Get a page by ID
    page, err := manager.GetPage(ctx, "123456")
    if err != nil {
        panic(err)
    }
    fmt.Printf("Page Title: %s\n", page.Title)
    fmt.Printf("Page ID: %s\n", page.ID)
    fmt.Printf("Content: %s\n", page.Body.Storage.Value)
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `ConfluenceManager` | Main manager with HTTP client, credentials, worker pool |
| `ConfluencePage` | Confluence page with ID, title, type, body storage |

### Concurrency Model

```
┌─────────────────────────────────────────────────────┐
│                   ConfluenceManager                       │
├─────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │  retryablehttp │  → API requests with retry               │
│  │  .Client      │  → Basic auth (username + API token)    │
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
│  BaseURL: Confluence REST API base URL              │
│  Username: Confluence username                         │
│  APIToken: Confluence API token (password for basic auth) │
└─────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewConfluenceManager(logger)
    │
    ├── Check viper config: "confluence.enabled"
    ├── Get base_url: "confluence.base_url"
    ├── Get username: "confluence.username"
    ├── Get api_token: "confluence.api_token"
    │
    ├── Create retryablehttp.Client:
    │   ├── RetryMax = 3
    │   ├── RetryWaitMin = 1s
    │   ├── RetryWaitMax = 10s
    │   └── HTTPClient.Timeout = 30s
    │
    ├── Test connection: testConnection()
    │   └── GET /rest/api/space?limit=1
    │
    ├── Create WorkerPool(4)
    ├── Start worker pool
    └── Return ConfluenceManager
```

### 2. Get Page Flow

```
GetPage(ctx, pageID)
    │
    ├── Build request URL: {baseURL}/rest/api/content/{pageID}?expand=body.storage
    ├── Set Basic Auth header (username, apiToken)
    ├── Execute with retryablehttp.Client
    ├── Check status code (200 OK)
    ├── Decode JSON into ConfluencePage
    └── Return *ConfluencePage
```

### 3. Get Status Flow

```
GetStatus()
    │
    ├── If nil: return connected=false
    ├── Create context with 5s timeout
    ├── GET /rest/api/space?limit=1
    ├── Set Basic Auth header
    ├── Return connected, base_url, username, pool_active
    └── Note: Tests actual API connectivity
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `confluence.enabled` | bool | false | Enable/disable Confluence manager |
| `confluence.base_url` | string | "" | Confluence REST API base URL (e.g., "https://your-domain.atlassian.net/wiki") |
| `confluence.username` | string | "" | Confluence username (email) |
| `confluence.api_token` | string | "" | Confluence API token (generated from profile) |

## Usage Examples

### Get Page by ID

```go
ctx := context.Background()

page, err := manager.GetPage(ctx, "123456")
if err != nil {
    panic(err)
}

fmt.Printf("Page Title: %s\n", page.Title)
fmt.Printf("Page ID: %s\n", page.ID)
fmt.Printf("Page Type: %s\n", page.Type)
fmt.Printf("Content: %s\n", page.Body.Storage.Value)
fmt.Printf("Representation: %s\n", page.Body.Storage.Representation)
```

### Async Page Retrieval

```go
ctx := context.Background()

pageChan := manager.GetPageAsync(ctx, "123456")
page, err := pageChan.Get(ctx)
if err != nil {
    panic(err)
}

fmt.Printf("Page Title: %s\n", page.Title)
```

### Getting Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
fmt.Printf("Base URL: %s\n", status["base_url"])
fmt.Printf("Username: %s\n", status["username"])

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
| Standard library | `encoding/json`, `net/http`, `io`, `time` |

## Error Handling

The library uses Go error handling patterns:

- **Authentication failures**: Returned from API (invalid credentials)
- **API errors**: Returned from Confluence API calls
- **Nil checks**: Public methods check manager state

Example error handling:

```go
page, err := manager.GetPage(ctx, pageID)
if err != nil {
    if strings.Contains(err.Error(), "401") {
        return fmt.Errorf("invalid credentials: %w", err)
    }
    if strings.Contains(err.Error(), "403") {
        return fmt.Errorf("access forbidden: %w", err)
    }
    if strings.Contains(err.Error(), "404") {
        return fmt.Errorf("page not found: %w", err)
    }
    return fmt.Errorf("API request failed: %w", err)
}
```

## Common Pitfalls

### 1. Manager Not Enabled

**Problem**: `confluence.enabled` is false

**Solution**:
- Set `confluence.enabled = true` in configuration
- Check viper configuration is loaded

### 2. Missing Credentials

**Problem**: `base_url`, `username`, or `api_token` not set

**Solution**:
- Set `confluence.base_url` to your Confluence instance URL
- Set `confluence.username` to your Atlassian account email
- Set `confluence.api_token` to your API token (generate from https://id.atlassian.com/manage-profile/security/api-tokens)

### 3. Confluence Not Reachable

**Problem**: Cannot connect to Confluence

**Solution**:
- Verify the `confluence.base_url` is correct
- Ensure your network can reach the Confluence instance
- Check if VPN or firewall is blocking access

### 4. Page Not Found

**Problem**: `404` when getting page

**Solution**:
- Verify the page ID is correct
- Check that the page exists and is accessible
- Confirm the user has permission to view the page

### 5. Worker Pool Not Available

**Problem**: Async operations falling back to direct execution

**Solution**:
- The pool is created during initialization
- Check that `NewConfluenceManager()` succeeded
- Verify `confluence.enabled` is true

## Advanced Usage

### Health Check

```go
func healthCheck(manager *confluence.ConfluenceManager) error {
    status := manager.GetStatus()
    if status["connected"] == false {
        return fmt.Errorf("not connected to Confluence API")
    }
    fmt.Printf("Confluence API is healthy (URL: %s)\n", status["base_url"])
    return nil
}
```

### Batch Page Retrieval with Async

```go
func batchGetPages(manager *confluence.ConfluenceManager, pageIDs []string) ([]*confluence.ConfluencePage, error) {
    ctx := context.Background()
    results := make([]*confluence.ConfluencePage, len(pageIDs))
    errors := make([]error, len(pageIDs))
    
    // Submit all requests asynchronously
    for i, pageID := range pageIDs {
        func(idx int, id string) {
            manager.SubmitAsyncJob(func() {
                page, err := manager.GetPage(ctx, id)
                results[idx] = page
                errors[idx] = err
            })
        }(i, pageID)
    }
    
    // Collect results
    for i, err := range errors {
        if err != nil {
            return nil, fmt.Errorf("failed to get page %s: %w", pageIDs[i], err)
        }
    }
    return results, nil
}
```

## Internal Algorithms

### API Request Flow

```
GetPage():
    │
    ├── Build URL with expand=body.storage
    ├── Set Basic Auth header
    ├── Execute with retryablehttp (3 retries)
    ├── Parse response JSON
    └── Return page with storage content
```

### Connection Test

```
testConnection():
    │
    ├── GET /rest/api/space?limit=1
    ├── Set Basic Auth header
    ├── Check status code (200 OK)
    └── Return error if failed
```

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/rest/api/space` | GET | List spaces (used for health check) |
| `/rest/api/content/{pageID}` | GET | Get page by ID with expand options |

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.