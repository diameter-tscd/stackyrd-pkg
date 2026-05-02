# WSO2 API Manager

## Overview

The `WSO2Manager` is a Go library for managing WSO2 API Gateway resources. It provides comprehensive API management including creating, updating, deleting APIs, managing applications, subscriptions, throttling policies, and OAuth token generation. The library uses OAuth2 authentication with password grant type and supports async operations via worker pool.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **API Management**: List, get, create, update, delete APIs
- **Lifecycle Management**: Change API lifecycle state (publish, deploy, etc.)
- **Application Management**: List and create applications
- **Subscription Management**: List and create API subscriptions
- **Throttling Policies**: List available throttling policies
- **OAuth Token**: Generate tokens for applications
- **OAuth Authentication**: Automatic token management with refresh
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
    
    // Create WSO2 manager (configuration via viper)
    manager, err := wso2.NewWSO2Manager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // List all APIs
    apiList, err := manager.ListAPIs(ctx)
    if err != nil {
        panic(err)
    }
    fmt.Printf("Found %d APIs\n", apiList.Count)
    
    for _, api := range apiList.List {
        fmt.Printf("API: %s (ID: %s)\n", api.Name, api.ID)
    }
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `WSO2Manager` | Main manager with HTTP client, OAuth token, worker pool |
| `WSO2API` | API representation with metadata |
| `WSO2APICreateRequest` | Request to create a new API |
| `WSO2Application` | Application registered in WSO2 |
| `WSO2Subscription` | API subscription |
| `WSO2TokenResponse` | OAuth token response |
| `WSO2ThrottlePolicy` | Throttling policy |
| `WSO2TokenRequest` | DCR token request |
| `WSO2TokenInfo` | Generated token info |
| `WSO2LifecycleState` | API lifecycle state |
| `WSO2APIRevision` | API revision |

### Concurrency Model

```
┌─────────────────────────────────────────────────────┐
│                   WSO2Manager                           │
├─────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │  retryablehttp │  → API requests with retry               │
│  │  .Client      │  → OAuth Bearer token authentication    │
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
│  BaseURL: WSO2 API Gateway URL                      │
│  TokenURL: OAuth token endpoint                        │
│  AccessToken: Bearer token for authenticated requests    │
└─────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewWSO2Manager(logger)
    │
    ├── Check viper config: "wso2.enabled"
    ├── Get base_url: "wso2.base_url"
    ├── Get token_url: "wso2.token_url" (default: baseURL + "/oauth2/token")
    ├── Get username: "wso2.username"
    ├── Get password: "wso2.password"
    ├── Get client_id: "wso2.client_id"
    ├── Get client_secret: "wso2.client_secret"
    ├── Get tenant: "wso2.tenant" (default: "carbon.super")
    │
    ├── Create retryablehttp.Client:
    │   ├── RetryMax = 3
    │   ├── RetryWaitMin = 1s
    │   ├── RetryWaitMax = 10s
    │   └── HTTPClient.Timeout = 60s
    │
    ├── Authenticate with WSO2 (password grant)
    │   └── POST to tokenURL with credentials
    │
    ├── Create WorkerPool(4)
    ├── Start worker pool
    └── Return WSO2Manager
```

### 2. Authentication Flow

```
authenticate()
    │
    ├── Build form data:
    │   ├── grant_type = "password"
    │   ├── username = w.Username
    │   ├── password = w.Password
    │   └── scope = "apim:api_create apim:api_publish ..."
    │
    ├── Create POST request to w.TokenURL
    ├── Set header: Authorization = Basic base64(clientID:clientSecret)
    ├── Set header: Content-Type = application/x-www-form-urlencoded
    ├── Execute with retryablehttp.Client
    ├── Check status code (200 OK)
    ├── Decode JSON into WSO2TokenResponse
    ├── Store AccessToken and TokenExpiry
    └── Return error if failed
```

### 3. Ensure Authenticated Flow

```
ensureAuthenticated()
    │
    ├── Check if token expires in < 30 seconds
    ├── If yes: call authenticate() to refresh
    └── Return nil if valid
```

### 4. List APIs Flow

```
ListAPIs(ctx)
    │
    ├── Call ensureAuthenticated()
    ├── GET /api/am/publisher/v4/apis
    ├── Set header: Authorization = Bearer <access_token>
    ├── Execute with retryablehttp.Client
    ├── Check status code (200 OK)
    ├── Decode JSON into WSO2APIList
    └── Return *WSO2APIList
```

### 5. Create API Flow

```
CreateAPI(ctx, request)
    │
    ├── Call ensureAuthenticated()
    ├── Marshal request to JSON
    ├── POST /api/am/publisher/v4/apis
    ├── Set header: Authorization = Bearer <access_token>
    ├── Set header: Content-Type = application/json
    ├── Execute with retryablehttp.Client
    ├── Check status code (201 Created)
    ├── Decode JSON into WSO2API
    └── Return *WSO2API
```

### 6. Get Status Flow

```
GetStatus()
    │
    ├── If nil: return connected=false
    ├── Create context with 5s timeout
    ├── Call ensureAuthenticated()
    ├── Call ListAPIs(ctx)
    ├── Return connected, base_url, tenant, total_apis, token_valid, pool_active
    └── Note: Tests actual API connectivity
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `wso2.enabled` | bool | false | Enable/disable WSO2 manager |
| `wso2.base_url` | string | "" | WSO2 API Gateway base URL |
| `wso2.token_url` | string | baseURL + "/oauth2/token" | OAuth token endpoint |
| `wso2.username` | string | "" | Username for authentication |
| `wso2.password` | string | "" | Password for authentication |
| `wso2.client_id` | string | "" | OAuth client ID |
| `wso2.client_secret` | string | "" | OAuth client secret |
| `wso2.tenant` | string | "carbon.super" | Tenant domain |

## Usage Examples

### Listing APIs

```go
ctx := context.Background()

apiList, err := manager.ListAPIs(ctx)
if err != nil {
    panic(err)
}

fmt.Printf("Total APIs: %d\n", apiList.Count)
for _, api := range apiList.List {
    fmt.Printf("  %s (ID: %s, Context: %s)\n", 
        api.Name, api.ID, api.Context)
}
```

### Getting a Specific API

```go
ctx := context.Background()

api, err := manager.GetAPI(ctx, "api-id-here")
if err != nil {
    panic(err)
}

fmt.Printf("API: %s\n", api.Name)
fmt.Printf("Description: %s\n", api.Description)
fmt.Printf("Version: %s\n", api.Version)
fmt.Printf("Lifecycle: %s\n", api.LifeCycleStatus)
fmt.Printf("Provider: %s\n", api.Provider)
```

### Creating an API

```go
ctx := context.Background()

request := wso2.WSO2APICreateRequest{
    Name:        "MyAPI",
    Description: "My API Description",
    Context:     "/myapi",
    Version:     "1.0.0",
    Provider:    "admin",
    Type:        "HTTP",
    Transport:   []string{"https"},
}

api, err := manager.CreateAPI(ctx, request)
if err != nil {
    panic(err)
}

fmt.Printf("Created API: %s (ID: %s)\n", api.Name, api.ID)
```

### Updating an API

```go
ctx := context.Background()

request := wso2.WSO2APICreateRequest{
    Name:        "MyAPI Updated",
    Description: "Updated description",
    Context:     "/myapi",
    Version:     "1.0.1",
}

api, err := manager.UpdateAPI(ctx, "api-id-here", request)
if err != nil {
    panic(err)
}

fmt.Printf("Updated API: %s\n", api.Name)
```

### Deleting an API

```go
ctx := context.Background()

err := manager.DeleteAPI(ctx, "api-id-here")
if err != nil {
    panic(err)
}
fmt.Println("API deleted successfully")
```

### Changing API Lifecycle

```go
ctx := context.Background()

// Publish API
api, err := manager.ChangeAPILifecycle(ctx, "api-id-here", "Publish")
if err != nil {
    panic(err)
}
fmt.Printf("API lifecycle: %s\n", api.LifeCycleStatus)
```

### Listing Applications

```go
ctx := context.Background()

appList, err := manager.ListApplications(ctx)
if err != nil {
    panic(err)
}

fmt.Printf("Total Applications: %d\n", appList.Count)
for _, app := range appList.List {
    fmt.Printf("  %s (ID: %s, Throttling: %s)\n", 
        app.Name, app.ApplicationID, app.ThrottlingTier)
}
```

### Creating an Application

```go
ctx := context.Background()

app, err := manager.CreateApplication(ctx, "MyApp", "Unlimited", "My application")
if err != nil {
    panic(err)
}

fmt.Printf("Created Application: %s (ID: %s)\n", app.Name, app.ApplicationID)
```

### Listing Subscriptions

```go
ctx := context.Background()

// List all subscriptions for an API
subList, err := manager.ListSubscriptions(ctx, "api-id-here", "")
if err != nil {
    panic(err)
}

fmt.Printf("Total Subscriptions: %d\n", subList.Count)
for _, sub := range subList.List {
    fmt.Printf("  Sub ID: %s, App ID: %s, Status: %s\n", 
        sub.SubscriptionID, sub.ApplicationID, sub.Status)
}
```

### Creating a Subscription

```go
ctx := context.Background()

sub, err := manager.CreateSubscription(ctx, "api-id", "app-id", "Unlimited")
if err != nil {
    panic(err)
}

fmt.Printf("Created Subscription: %s\n", sub.SubscriptionID)
```

### Listing Throttling Policies

```go
ctx := context.Background()

policies, err := manager.ListThrottlingPolicies(ctx, "api")
if err != nil {
    panic(err)
}

fmt.Printf("Found %d policies\n", len(policies))
for _, policy := range policies {
    fmt.Printf("  %s (ID: %s)\n", policy.PolicyName, policy.PolicyID)
}
```

### Generating Keys (OAuth Tokens)

```go
ctx := context.Background()

tokenInfo, err := manager.GenerateKeys(ctx, "app-id", "PRODUCTION", 3600)
if err != nil {
    panic(err)
}

fmt.Printf("Client ID: %s\n", tokenInfo.ClientID)
fmt.Printf("Client Secret: %s\n", tokenInfo.ClientSecret)
fmt.Printf("Access Token: %s\n", tokenInfo.AccessToken)
```

### Async Operations

```go
ctx := context.Background()

// List APIs asynchronously
apiResult := manager.ListAPIsAsync(ctx)
apiList, err := apiResult.Get(ctx)
if err != nil {
    panic(err)
}
fmt.Printf("Got %d APIs asynchronously\n", apiList.Count)

// Create API asynchronously
createReq := wso2.WSO2APICreateRequest{
    Name:    "AsyncAPI",
    Context: "/asyncapi",
    Version: "1.0.0",
}
apiResult = manager.CreateAPIAsync(ctx, createReq)
api, err := apiResult.Get(ctx)
if err != nil {
    panic(err)
}
fmt.Printf("Created API asynchronously: %s\n", api.Name)
```

### Getting Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
fmt.Printf("Base URL: %s\n", status["base_url"])
fmt.Printf("Tenant: %s\n", status["tenant"])
fmt.Printf("Total APIs: %v\n", status["total_apis"])
fmt.Printf("Token Valid: %v\n", status["token_valid"])

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
| Standard library | `encoding/base64`, `encoding/json`, `net/http`, `net/url`, `strings`, `time` |

## Error Handling

The library uses Go error handling patterns:

- **Authentication failures**: Returned from token endpoint
- **API errors**: Returned from WSO2 API calls
- **Token expiration**: Automatically refreshed via ensureAuthenticated()
- **Nil checks**: Public methods check manager state

Example error handling:

```go
apiList, err := manager.ListAPIs(ctx)
if err != nil {
    if strings.Contains(err.Error(), "401") {
        return fmt.Errorf("authentication failed: %w", err)
    }
    if strings.Contains(err.Error(), "403") {
        return fmt.Errorf("access forbidden: %w", err)
    }
    if strings.Contains(err.Error(), "429") {
        return fmt.Errorf("rate limit exceeded: %w", err)
    }
    return fmt.Errorf("API request failed: %w", err)
}
```

## Common Pitfalls

### 1. Manager Not Enabled

**Problem**: `wso2.enabled` is false

**Solution**:
- Set `wso2.enabled = true` in configuration
- Check viper configuration is loaded

### 2. Missing Credentials

**Problem**: `username`, `password`, `client_id`, or `client_secret` not set

**Solution**:
- Set all required credentials in viper configuration
- Verify WSO2 OAuth app is properly configured

### 3. Token Expiration

**Problem**: API calls fail with 401 Unauthorized

**Solution**:
- Library automatically refreshes tokens via ensureAuthenticated()
- Check token validity in GetStatus()
- Verify token URL is correct

### 4. Invalid API ID

**Problem**: `API not found` or similar error

**Solution**:
- Verify API ID is correct
- Use ListAPIs() to get valid API IDs
- Check API lifecycle state

### 5. Rate Limiting

**Problem**: `429` Too Many Requests

**Solution**:
- Library retries up to 3 times
- Consider using async operations to spread requests
- Respect WSO2 rate limits

### 6. Worker Pool Not Available

**Problem**: Async operations falling back to direct execution

**Solution**:
- The pool is created during initialization
- Check that `NewWSO2Manager()` succeeded
- Verify `wso2.enabled` is true

### 7. Tenant Configuration

**Problem**: Authentication fails with tenant error

**Solution**:
- Default tenant is "carbon.super"
- Set `wso2.tenant` to your WSO2 tenant domain
- Verify tenant exists in your WSO2 deployment

## Advanced Usage

### Custom API Request

```go
func customRequest(manager *wso2.WSO2Manager) error {
    ctx := context.Background()
    
    // Ensure authenticated
    if err := manager.EnsureAuthenticated(); err != nil {
        return err
    }
    
    // Build custom request
    endpoint := manager.BaseURL + "/api/am/publisher/v4/apis"
    req, err := retryablehttp.NewRequestWithContext(ctx, "GET", endpoint, nil)
    if err != nil {
        return err
    }
    
    manager.SetAuthHeader(req)
    
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
func healthCheck(manager *wso2.WSO2Manager) error {
    status := manager.GetStatus()
    if status["connected"] == false {
        return fmt.Errorf("not connected to WSO2 API")
    }
    fmt.Printf("WSO2 API is healthy (URL: %s)\n", status["base_url"])
    return nil
}
```

### Using Different Grant Types

```go
// The default is password grant, but you can modify authenticate() for other grants
// For example, client_credentials:
func (w *WSO2Manager) authenticateClientCredentials() error {
    data := url.Values{}
    data.Set("grant_type", "client_credentials")
    data.Set("scope", "apim:api_view")
    
    // ... rest of authentication logic
}
```

### Batch Operations with Async

```go
func getMultipleAPIs(manager *wso2.WSO2Manager, apiIDs []string) ([]*wso2.WSO2API, error) {
    ctx := context.Background()
    results := make([]*wso2.WSO2API, len(apiIDs))
    errors := make([]error, len(apiIDs))
    
    // Submit all requests asynchronously
    for i, apiID := range apiIDs {
        func(idx int, id string) {
            manager.SubmitAsyncJob(func() {
                api, err := manager.GetAPI(ctx, id)
                results[idx] = api
                errors[idx] = err
            })
        }(i, apiID)
    }
    
    // Collect results
    for i, err := range errors {
        if err != nil {
            return nil, fmt.Errorf("failed to get %s: %w", apiIDs[i], err)
        }
    }
    return results, nil
}
```

### Token Introspection

```go
func checkTokenValidity(manager *wso2.WSO2Manager) {
    status := manager.GetStatus()
    if tokenValid, ok := status["token_valid"].(bool); ok {
        if tokenValid {
            fmt.Println("Token is still valid")
        } else {
            fmt.Println("Token has expired or is invalid")
        }
    }
}
```

## Internal Algorithms

### OAuth Authentication

```
authenticate():
    │
    ├── Prepare form data with credentials
    ├── Base64 encode clientID:clientSecret for Basic Auth
    ├── POST to token endpoint
    ├── Parse JSON response
    ├── Store AccessToken
    └── Calculate TokenExpiry = now + expires_in seconds
```

### Token Refresh

```
ensureAuthenticated():
    │
    ├── Check if token expires in < 30 seconds
    ├── If yes: call authenticate()
    └── Return nil (token is still valid)
```

### Auth Header Setting

```
setAuthHeader(req):
    │
    └── Set header: Authorization = "Bearer " + AccessToken
```

### API Response Parsing

```
All API responses are JSON:
{
  "count": 10,
  "list": [...]
}

Check HTTP status code for success.
```

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/am/publisher/v4/apis` | GET | List all APIs |
| `/api/am/publisher/v4/apis/{apiId}` | GET | Get specific API |
| `/api/am/publisher/v4/apis` | POST | Create new API |
| `/api/am/publisher/v4/apis/{apiId}` | PUT | Update API |
| `/api/am/publisher/v4/apis/{apiId}` | DELETE | Delete API |
| `/api/am/publisher/v4/apis/{apiId}/lifecycle-state` | POST | Change lifecycle |
| `/api/am/store/v1/applications` | GET | List applications |
| `/api/am/store/v1/applications` | POST | Create application |
| `/api/am/store/v1/subscriptions` | GET | List subscriptions |
| `/api/am/store/v1/subscriptions` | POST | Create subscription |
| `/api/am/store/v1/subscriptions/{subscriptionId}` | DELETE | Delete subscription |
| `/api/am/admin/v4/throttling/policies/{level}` | GET | List throttling policies |
| `/api/am/store/v1/applications/{appId}/generate-keys` | POST | Generate keys |
| `/oauth2/token` | POST | OAuth token endpoint |

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.