# AWS Lambda Manager

## Overview

The `AwsLambdaManager` is a comprehensive Go library for managing AWS Lambda functions via the AWS Lambda API. It provides complete Lambda resource management including functions, layers, event source mappings, aliases, and function invocation with async support.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Function Management**: List, get, create, update, and delete Lambda functions
- **Code Management**: Update function code with ZIP files or S3 references
- **Function Invocation**: Invoke functions synchronously or asynchronously
- **Layer Management**: Publish and list Lambda layers
- **Event Source Mappings**: Create, list, and delete event source mappings
- **Alias Management**: Create, list, and delete function aliases
- **Permission Management**: Add and remove function permissions
- **Account Settings**: Retrieve Lambda account settings
- **Async Operations**: Async execution for all operations via worker pool
- **AWS Authentication**: Supports access key, secret key, and session tokens with AWS v4 signing
- **Retry Logic**: Automatic retries with exponential backoff via `go-retryablehttp`

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
    
    // Create AWS Lambda manager (configuration via viper)
    manager, err := lambda.NewAwsLambdaManager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // List Lambda functions
    functions, err := manager.ListFunctions(ctx)
    if err != nil {
        panic(err)
    }
    
    for _, fn := range functions {
        fmt.Printf("Function: %s (Runtime: %s)\n", fn.Name, fn.Runtime)
    }
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `AwsLambdaManager` | Main manager with HTTP client and worker pool |
| `LambdaFunction` | Lambda function with name, ARN, runtime, config |
| `LambdaCode` | Function code location (ZIP file, S3, or image) |
| `LambdaInvocationResult` | Result of function invocation |
| `LambdaLayer` | Lambda layer with ARN, name, and compatibility |
| `LambdaEventSourceMapping` | Event source mapping for triggers |
| `LambdaAlias` | Function alias with version and description |

### Concurrency Model

```
┌─────────────────────────────────────────────────────────────┐
│                   AwsLambdaManager                         │
├─────────────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │ retryablehttp  │  → Retry logic (max 3, 1-10s wait)│
│  │ Client         │  → HTTP timeout: 60s                   │
│  │                │  → AWS v4 request signing             │
│  └────────────────┘                                      │
│                                                         │
│  ┌─────────────┐      ┌──────────────┐                   │
│  │  WorkerPool │◄─────│ AsyncResult │                   │
│  │  (6 workers)│      │   Channel   │                   │
│  └─────────────┘      └──────────────┘                   │
│         ▲                      │                            │
│         │                      ▼                            │
│  ┌─────────────┐      ┌──────────────┐                  │
│  │  SubmitJob  │      │ ExecuteAsync │                  │
│  └─────────────┘      └──────────────┘                  │
│                                                         │&
│  Region: AWS region (default: us-east-1)                      │
│  BaseURL: https://lambda.{region}.amazonaws.com         │
└─────────────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewAwsLambdaManager()
    │
    ├── Check viper config: "aws_lambda.enabled"
    ├── Get region: "aws_lambda.region" (default: "us-east-1")
    ├── Get credentials: access_key_id, secret_access_key, session_token
    ├── Build baseURL: "https://lambda.{region}.amazonaws.com/2015-03-31"
    ├── Create retryablehttp.Client (max 3 retries, 1-10s wait, 60s timeout)
    ├── Test connection: testConnection() → ListFunctions()
    ├── Create WorkerPool(6)
    ├── Start worker pool
    └── Return AwsLambdaManager
```

### 2. API Request Flow

```
apiRequest(ctx, method, path, body)
    │
    ├── Build URL: baseURL + path
    ├── If body: JSON marshal → bytes.Reader
    ├── Create request with context
    ├── Set Content-Type: application/json
    ├── signRequest(req)
    │   ├── Set Host header
    │   ├── Set X-Amz-Date header
    │   ├── If session_token: Set X-Amz-Security-Token
    │   └── If credentials: Set Authorization (AWS4-HMAC-SHA256)
    ├── Execute request
    ├── Read response body
    ├── Check status code (200-299 = success)
    └── Return response bytes or error
```

### 3. List Functions Flow

```
ListFunctions(ctx)
    │
    ├── apiRequest(ctx, "GET", "/functions", nil)
    ├── JSON unmarshal to LambdaListResponse
    └── Return []LambdaFunction
```

### 4. Function Invocation Flow

```
InvokeFunction(ctx, functionName, payload)
    │
    ├── Marshal payload to JSON
    ├── Build URL: baseURL/functions/{name}/invocations
    ├── Create POST request with payload
    ├── Set X-Amz-Invocation-Type: "RequestResponse" (sync)
    ├── signRequest(req)
    ├── Execute request
    ├── Record duration
    ├── Read response
    └── Return LambdaInvocationResult
```

### 5. Async Invocation Flow

```
InvokeFunctionEvent(ctx, functionName, payload)
    │
    ├── Marshal payload to JSON
    ├── Build URL: baseURL/functions/{name}/invocations
    ├── Create POST request with payload
    ├── Set X-Amz-Invocation-Type: "Event" (async)
    ├── signRequest(req)
    ├── Execute request
    └── Return LambdaInvocationResult (status code only)
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `aws_lambda.enabled` | bool | false | Enable/disable AWS Lambda manager |
| `aws_lambda.region` | string | "us-east-1" | AWS region |
| `aws_lambda.access_key_id` | string | "" | AWS access key ID |
| `aws_lambda.secret_access_key` | string | "" | AWS secret access key |
| `aws_lambda.session_token` | string | "" | AWS session token (optional) |

### Environment Variables

The Lambda manager does not directly use environment variables. Configuration is read through Viper, which can load from:
- Configuration files (JSON, YAML, TOML)
- Environment variables (if Viper is configured for it)
- AWS credentials can also be set via standard AWS env vars (AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_SESSION_TOKEN)

## Usage Examples

### Function Operations

```go
ctx := context.Background()

// List all Lambda functions
functions, err := manager.ListFunctions(ctx)
if err != nil {
    panic(err)
}

for _, fn := range functions {
    fmt.Printf("Function: %s\n", fn.Name)
    fmt.Printf("  ARN: %s\n", fn.ARN)
    fmt.Printf("  Runtime: %s\n", fn.Runtime)
    fmt.Printf("  Handler: %s\n", fn.Handler)
    fmt.Printf("  Memory: %d MB\n", fn.MemorySize)
    fmt.Printf("  Timeout: %d seconds\n", fn.Timeout)
    fmt.Printf("  State: %s\n", fn.State)
    fmt.Printf("  Last Modified: %s\n", fn.LastModified)
}

// Get specific function
function, err := manager.GetFunction(ctx, "my-function")
if err != nil {
    panic(err)
}
fmt.Printf("Function: %s (Version: %s)\n", function.Name, function.Version)

// Create a function
newFn := lambda.LambdaFunction{
    Name:        "my-function",
    Runtime:     "go1.x",
    Role:        "arn:aws:iam::123456789012:role/lambda-role",
    Handler:     "main",
    Description: "My Lambda function",
    Timeout:     30,
    MemorySize:   128,
    Environment: map[string]string{
        "ENVIRONMENT": "production",
    },
}
created, err := manager.CreateFunction(ctx, newFn)
if err != nil {
    panic(err)
}
fmt.Printf("Created: %s\n", created.Name)

// Update function code
code := lambda.LambdaCode{
    ZipFile: zipBytes, // ZIP file content as byte array
}
_, err = manager.UpdateFunctionCode(ctx, "my-function", code)
if err != nil {
    panic(err)
}

// Update function configuration
updated, err := manager.UpdateFunctionConfiguration(ctx, "my-function", newFn)
if err != nil {
    panic(err)
}

// Delete function
err = manager.DeleteFunction(ctx, "my-function")
if err != nil {
    panic(err)
}
```

### Function Invocation

```go
// Invoke function synchronously
payload := map[string]interface{}{
    "key": "value",
}
result, err := manager.InvokeFunction(ctx, "my-function", payload)
if err != nil {
    panic(err)
}

fmt.Printf("Status: %d\n", result.StatusCode)
fmt.Printf("Log Result: %s\n", result.LogResult)
fmt.Printf("Payload: %s\n", result.Payload)
fmt.Printf("Duration: %v\n", result.Duration)

if result.FunctionError != "" {
    fmt.Printf("Function error: %s\n", result.FunctionError)
}

// Invoke function asynchronously
eventResult, err := manager.InvokeFunctionEvent(ctx, "my-function", payload)
if err != nil {
    panic(err)
}
fmt.Printf("Async invoke status: %d\n", eventResult.StatusCode)
```

### Layer Operations

```go
// List Lambda layers
layers, err := manager.ListLayers(ctx)
if err != nil {
    panic(err)
}

for _, layer := range layers {
    fmt.Printf("Layer: %s (Name: %s)\n", layer.ARN, layer.Name)
    fmt.Printf("  Compatible Runtimes: %v\n", layer.CompatibleRuntimes)
}

// Publish layer version
runtime := lambda.LambdaLayer{
    Name:             "my-layer",
    CompatibleRuntimes: []string{"go1.x", "nodejs18.x"},
}
newLayer, err := manager.PublishLayerVersion(ctx, "my-layer", zipBytes, []string{"go1.x"})
if err != nil {
    panic(err)
}
fmt.Printf("Published layer: %s\n", newLayer.ARN)
```

### Event Source Mapping Operations

```go
// List event source mappings
mappings, err := manager.ListEventSourceMappings(ctx, "my-function")
if err != nil {
    panic(err)
}

for _, mapping := range mappings {
    fmt.Printf("Mapping: %s\n", mapping.UUID)
    fmt.Printf("  Function: %s\n", mapping.FunctionARN)
    fmt.Printf("  Event Source: %s\n", mapping.EventSourceARN)
    fmt.Printf("  State: %s\n", mapping.State)
}

// Create event source mapping
newMapping, err := manager.CreateEventSourceMapping(ctx, lambda.LambdaEventSourceMapping{
    FunctionARN:          "arn:aws:lambda:us-east-1:123456789012:function:my-function",
    EventSourceARN:       "arn:aws:kinesis:us-east-1:123456789012:stream:my-stream",
    BatchSize:            100,
    MaximumRetryAttempts: 3,
})
if err != nil {
    panic(err)
}
fmt.Printf("Created mapping: %s\n", newMapping.UUID)

// Delete event source mapping
err = manager.DeleteEventSourceMapping(ctx, "uuid-1234-5678")
if err != nil {
    panic(err)
}
```

### Alias Operations

```go
// List function aliases
aliases, err := manager.ListAliases(ctx, "my-function")
if err != nil {
    panic(err)
}

for _, alias := range aliases {
    fmt.Printf("Alias: %s (Function: %s, Version: %s)\n", 
        alias.Name, alias.FunctionName, alias.FunctionVersion)
}

// Create alias
newAlias, err := manager.CreateAlias(ctx, "my-function", lambda.LambdaAlias{
    Name:            "prod",
    FunctionVersion: "1",
    Description:     "Production alias",
})
if err != nil {
    panic(err)
}
fmt.Printf("Created alias: %s\n", newAlias.Name)

// Delete alias
err = manager.DeleteAlias(ctx, "my-function", "prod")
if err != nil {
    panic(err)
}
```

### Permission Operations

```go
// Add permission to function
perm := map[string]string{
    "Action":              "lambda:InvokeFunction",
    "Principal":           "s3.amazonaws.com",
    "SourceArn":           "arn:aws:s3:::my-bucket",
}
err := manager.AddPermission(ctx, "my-function", perm)
if err != nil {
    panic(err)
}

// Remove permission from function
err = manager.RemovePermission(ctx, "my-function", "statement-id")
if err != nil {
    panic(err)
}
```

### Account Settings

```go
// Get account settings
settings, err := manager.GetAccountSettings(ctx)
if err != nil {
    panic(err)
}

fmt.Printf("Account Settings: %+v\n", settings)
```

### Async Operations

```go
// Async list functions
result := manager.ListFunctionsAsync(ctx)
select {
case functions := <-result.Ch:
    fmt.Printf("Found %d functions\n", len(functions))
case err := <-result.ErrCh:
    fmt.Printf("Error: %v\n", err)
}

// Async get function
result = manager.GetFunctionAsync(ctx, "my-function")
select {
case fn := <-result.Ch:
    fmt.Printf("Function: %s\n", fn.Name)
case err := <-result.ErrCh:
    fmt.Printf("Error: %v\n", err)
}

// Async invoke (sync)
result = manager.InvokeFunctionAsync(ctx, "my-function", payload)
select {
case invokeResult := <-result.Ch:
    fmt.Printf("Status: %d\n", invokeResult.StatusCode)
case err := <-result.ErrCh:
    fmt.Printf("Error: %v\n", err)
}

// Async invoke (event)
result = manager.InvokeFunctionEventAsync(ctx, "my-function", payload)
select {
case eventResult := <-result.Ch:
    fmt.Printf("Async status: %d\n", eventResult.StatusCode)
case err := <-result.ErrCh:
    fmt.Printf("Error: %v\n", err)
}

// Other async methods:
// - ListLayersAsync
// - ListEventSourceMappingsAsync
// - ListAliasesAsync
// - GetAccountSettingsAsync
```

### Get Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
fmt.Printf("Region: %s\n", status["region"])
fmt.Printf("Functions: %v\n", status["functions"])

if status["connected"] == false {
    if errMsg, ok := status["error"].(string); ok {
        fmt.Printf("Error: %s\n", errMsg)
    }
}
```

## Dependencies

| Dependency | Role |
|------------|------|
| `github.com/hashicorp/go-retryablehttp` | HTTP client with retry logic for API resilience |
| `github.com/spf13/viper` | Configuration management for AWS credentials |
| `stackyrd/config` | Internal configuration package |
| `stackyrd/pkg/logger` | Structured logging |
| Standard library | `context`, `encoding/json`, `bytes`, `io`, `time`, `fmt`, `crypto/hmac`, `crypto/sha256` |

## Error Handling

The library uses Go error handling patterns:

- **Connection failures**: `testConnection()` validates connectivity via `ListFunctions()`
- **API errors**: HTTP status codes are checked and errors returned with response body
- **AWS errors**: Response contains AWS error codes and messages
- **Context cancellation**: All operations respect `context.Context` for timeouts
- **Nil checks**: Public methods handle nil receiver gracefully in `GetStatus()`

Example error handling:

```go
functions, err := manager.ListFunctions(ctx)
if err != nil {
    if strings.Contains(err.Error(), "AuthFailure") {
        return fmt.Errorf("authentication failed: %w", err)
    }
    if strings.Contains(err.Error(), "ResourceNotFoundException") {
        return fmt.Errorf("function not found: %w", err)
    }
    return fmt.Errorf("failed to list functions: %w", err)
}
```

## Common Pitfalls

### 1. AWS Credentials Not Configured

**Problem**: `AuthFailure` or `SignatureDoesNotMatch` errors

**Solution**: 
- Configure credentials: `viper.Set("aws_lambda.access_key_id", "AKIA...")`
- Configure secret: `viper.Set("aws_lambda.secret_access_key", "secret...")`
- Or use AWS CLI: `aws configure`
- Or set environment variables: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`

### 2. Wrong Region

**Problem**: Resources not found or empty results

**Solution**:
- Check region: `viper.Set("aws_lambda.region", "us-west-2")`
- Verify resources exist in the specified region
- Common regions: us-east-1, us-west-2, eu-west-1

### 3. Function Not Found

**Problem**: `ResourceNotFoundException` error

**Solution**:
- Verify function name is correct
- Check function exists in the region
- Use `ListFunctions()` to list available functions

### 4. Context Timeout

**Problem**: Operations timing out

**Solution**:
```go
ctx, cancel := context.WithTimeout(context.Background(), 60*time.Second)
defer cancel()
// Operations will use this timeout (client has 60s default)
```

### 5. AWS v4 Signing Issues

**Problem**: Signature validation failures

**Solution**:
- Ensure system clock is synchronized (uses UTC time)
- Verify region in signing scope matches configured region
- Check that Host header matches the endpoint

### 6. Insufficient Permissions

**Problem**: `UnauthorizedOperation` error

**Solution**:
- Attach appropriate IAM policy to the user/role
- Required actions: `lambda:ListFunctions`, `lambda:InvokeFunction`, etc.
- Check IAM policy simulator

### 7. Payload Too Large

**Problem**: `RequestEntityTooLarge` error

**Solution**:
- Lambda payload limit is 6MB (ZIP file)
- For larger deployments, use S3 bucket reference in LambdaCode
- Optimize deployment package size

## Advanced Usage

### Function Monitoring

```go
func monitorFunctions(manager *lambda.AwsLambdaManager) {
    ticker := time.NewTicker(30 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:
            ctx := context.Background()
            functions, err := manager.ListFunctions(ctx)
            if err != nil {
                fmt.Printf("Error: %v\n", err)
                continue
            }
            
            fmt.Printf("\n=== Lambda Functions (%d) ===\n", len(functions))
            for _, fn := range functions {
                fmt.Printf("  %s: %s (%s)\n", fn.Name, fn.State, fn.Runtime)
            }
        }
    }
}
```

### Function Health Check

```go
func checkFunctionHealth(manager *lambda.AwsLambdaManager, functionName string) bool {
    ctx := context.Background()
    
    fn, err := manager.GetFunction(ctx, functionName)
    if err != nil {
        return false
    }
    
    // Check if function is in a good state
    if fn.State == "Active" {
        // Try a test invocation
        payload := map[string]interface{}{}
        result, err := manager.InvokeFunction(ctx, functionName, payload)
        if err != nil {
            return false
        }
        return result.StatusCode == 200
    }
    
    return false
}
```

### Batch Function Operations

```go
func batchDelete(manager *lambda.AwsLambdaManager, prefix string) error {
    ctx := context.Background()
    
    // Get all functions
    functions, err := manager.ListFunctions(ctx)
    if err != nil {
        return err
    }
    
    // Filter functions to delete
    var toDelete []string
    for _, fn := range functions {
        if strings.HasPrefix(fn.Name, prefix) {
            toDelete = append(toDelete, fn.Name)
        }
    }
    
    if len(toDelete) == 0 {
        fmt.Println("No functions to delete")
        return nil
    }
    
    fmt.Printf("Deleting %d functions...\n", len(toDelete))
    for _, name := range toDelete {
        err := manager.DeleteFunction(ctx, name)
        if err != nil {
            fmt.Printf("  Error deleting %s: %v\n", name, err)
        } else {
            fmt.Printf("  Deleted: %s\n", name)
        }
    }
    
    return nil
}
```

### Custom API Requests

While the manager provides high-level methods, you can extend it for custom API calls:

```go
// Example: Get function concurrency config (not directly exposed)
func getConcurrencyConfig(manager *lambda.AwsLambdaManager, ctx context.Context, functionName string) (string, error) {
    path := fmt.Sprintf("/functions/%s/concurrency", functionName)
    
    data, err := manager.ApiRequest(ctx, "GET", path, nil)
    if err != nil {
        return "", err
    }
    
    // Parse response
    var result map[string]interface{}
    if err := json.Unmarshal(data, &result); err != nil {
        return "", err
    }
    
    return fmt.Sprintf("%+v", result), nil
}
```

### Event-Driven Architecture

```go
// Example: Process Lambda events
type LambdaEventHandler struct {
    lambdaManager *lambda.AwsLambdaManager
}

func (h *LambdaEventHandler) HandleEvent(eventType string, payload interface{}) error {
    ctx := context.Background()
    
    switch eventType {
    case "invoke-sync":
        result, err := h.lambdaManager.InvokeFunction(ctx, "event-processor", payload)
        if err != nil {
            return err
        }
        fmt.Printf("Sync invoke result: %s\n", result.Payload)
        
    case "invoke-event":
        result, err := h.lambdaManager.InvokeFunctionEvent(ctx, "event-processor", payload)
        if err != nil {
            return err
        }
        fmt.Printf("Async invoke status: %d\n", result.StatusCode)
    }
    
    return nil
}
```

## Internal Algorithms

### AWS v4 Signing Algorithm

```
signRequest(req):
    │
    ├── dateStamp = now.Format("20060102")
    ├── amzDate = now.Format("20060102T150405Z")
    │
    ├── Set headers:
    │   ├── Host = req.URL.Host
    │   ├── X-Amz-Date = amzDate
    │   └── If sessionToken: X-Amz-Security-Token = sessionToken
    │
    └── If accessKey && secretKey:
            └── Authorization = "AWS4-HMAC-SHA256 Credential={accessKey}/{dateStamp}/{region}/lambda/aws4_request, SignedHeaders=host;x-amz-date, Signature={signature}"
```

### URL Construction

```
BaseURL Format: https://lambda.{region}.amazonaws.com/2015-03-31

Example:
  region: us-east-1
  URL: https://lambda.us-east-1.amazonaws.com/2015-03-31
```

### API Request Format

```
Method: GET/POST/PUT/DELETE
URL: baseURL + path
Headers:
  - Content-Type: application/json
  - Host: lambda.{region}.amazonaws.com
  - X-Amz-Date: {timestamp}
  - X-Amz-Security-Token: {sessionToken} (if present)
  - Authorization: AWS4-HMAC-SHA256 ... (if credentials present)
```

### Response Parsing

The manager uses JSON throughout:

**Function list** (returns object with Functions):
```json
{
  "Functions": [
    {
      "FunctionName": "my-function",
      "FunctionArn": "arn:aws:lambda:...",
      "Runtime": "go1.x",
      "State": "Active"
    }
  ]
}
→ Unmarshaled to []LambdaFunction
```

**Invocation result**:
```json
{
  "StatusCode": 200,
  "Payload": "...",
  "LogResult": "..."
}
→ Returned as LambdaInvocationResult
```

### Retry Logic

Using `go-retryablehttp`:
- Maximum retries: 3
- Minimum wait: 1 second
- Maximum wait: 10 seconds
- HTTP client timeout: 60 seconds

Retries are automatic for transient network errors.

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.