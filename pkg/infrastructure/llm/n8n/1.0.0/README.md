# n8n Workflow Automation Manager

## Overview

The `N8NManager` is a Go library for interacting with the n8n workflow automation platform. It provides comprehensive workflow management including creating, updating, activating workflows, managing executions, credentials, tags, users, and node types. The library uses n8n REST API with API key or Basic authentication and includes retry logic for reliability.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Workflow Management**: List, get, create, update, delete, activate, deactivate workflows
- **Execution Management**: List, get, delete, retry workflow executions with filtering
- **Credential Management**: List, get, create, update, delete credentials
- **Tag Management**: List, create, update, delete workflow tags
- **User Management**: List users, invite new users, delete users
- **Node Type Operations**: List and get registered node types
- **Health Monitoring**: Check system health and get owner information
- **Authentication**: Support for API key (X-N8N-API-KEY) and Basic Auth
- **Retry Logic**: HTTP client with retry support (3 retries)
- **Worker Pool**: Async job execution support (6 workers)
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
    
    // Create n8n manager (configuration via viper)
    manager, err := n8n.NewN8NManager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // List workflows
    workflows, err := manager.ListWorkflows(ctx)
    if err != nil {
        panic(err)
    }
    fmt.Printf("Found %d workflows\n", len(workflows))
    
    for _, wf := range workflows {
        fmt.Printf("Workflow: %s (ID: %s, Active: %v)\n", 
            wf.Name, wf.ID, wf.Active)
    }
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `N8NManager` | Main manager with HTTP client, API key, worker pool |
| `N8NWorkflow` | Workflow with nodes, connections, settings, tags |
| `N8NWorkflowNode` | Node in a workflow with parameters and credentials |
| `N8NWorkflowConnections` | Connections between workflow nodes |
| `N8NWorkflowSettings` | Workflow execution settings |
| `N8NExecution` | Workflow execution instance |
| `N8NExecutionResponse` | Execution creation response |
| `N8NWebhook` | Webhook configuration |
| `N8NCredential` | Stored credential for external services |
| `N8NTag` | Workflow tag |
| `N8NUser` | n8n user |
| `N8NNodeType` | Registered node type |
| `N8NHealthStatus` | System health status |
| `N8NWorkflowRunOptions` | Options for running a workflow |
| `N8NActivationResult` | Workflow activation result |
| `N8NActivationError` | Workflow activation error |

### Concurrency Model

```
┌─────────────────────────────────────────────────────┐
│                   N8NManager                            │
├─────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │  retryablehttp │  → API requests with retry               │
│  │  .Client      │  → API key or Basic Auth                │
│  └────────────────┘                                      │
│                                                          │
│  ┌─────────────┐      ┌──────────────┐                   │
│  │  WorkerPool │◄─────│ AsyncResult │                   │
│  │  (6 workers)│      │   Channel   │                   │
│  └─────────────┘      └──────────────┘                   │
│         ▲                      │                            │
│         │                      ▼                            │
│  ┌─────────────┐      ┌──────────────┐                  │
│  │  SubmitJob  │      │ ExecuteAsync │                  │
│  └─────────────┘      └──────────────┘                  │
│                                                          │
│  BaseURL: http://{host}:{port}/api/v1        │
│  APIKey: n8n API key (X-N8N-API-KEY header)      │
│  Username/Password: For Basic Auth                │
└─────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewN8NManager(logger)
    │
    ├── Check viper config: "n8n.enabled"
    ├── Get host: "n8n.host"
    ├── Get port: "n8n.port"
    ├── Get api_key: "n8n.api_key"
    ├── Get username: "n8n.username"
    ├── Get password: "n8n.password"
    │
    ├── Build baseURL: "http://{host}:{port}/api/v1"
    │
    ├── Create retryablehttp.Client:
    │   ├── RetryMax = 3
    │   ├── RetryWaitMin = 1s
    │   ├── RetryWaitMax = 10s
    │   └── HTTPClient.Timeout = 120s
    │
    ├── Test connection: testConnection()
    │   └── GET /health
    │
    ├── Create WorkerPool(6)
    ├── Start worker pool
    └── Return N8NManager
```

### 2. Workflow Operations Flow

```
ListWorkflows(ctx)
    │
    ├── Call doRequest(ctx, "GET", "/workflows", nil)
    ├── Set authentication headers (setAuthHeaders)
    ├── Parse response JSON
    └── Return []N8NWorkflow

GetWorkflow(ctx, workflowID)
    │
    ├── Call doRequest(ctx, "GET", "/workflows/{id}", nil)
    ├── Parse response JSON
    └── Return *N8NWorkflow

CreateWorkflow(ctx, workflow)
    │
    ├── Marshal workflow to JSON
    ├── Call doRequest(ctx, "POST", "/workflows", jsonData)
    ├── Parse response JSON
    └── Return *N8NWorkflow

UpdateWorkflow(ctx, workflowID, workflow)
    │
    ├── Marshal workflow to JSON
    ├── Call doRequest(ctx, "PATCH", "/workflows/{id}", jsonData)
    ├── Parse response JSON
    └── Return *N8NWorkflow

DeleteWorkflow(ctx, workflowID)
    │
    ├── Call doRequest(ctx, "DELETE", "/workflows/{id}", nil)
    └── Return error
```

### 3. Execution Operations Flow

```
ListExecutions(ctx, workflowID, status, limit)
    │
    ├── Build query parameters (workflowId, status, limit)
    ├── Call doRequest(ctx, "GET", "/executions?...", nil)
    ├── Parse response JSON
    └── Return []N8NExecution

RunWorkflow(ctx, workflowID, opts)
    │
    ├── Marshal options to JSON (if provided)
    ├── Call doRequest(ctx, "POST", "/workflows/{id}/execute", jsonData)
    ├── Parse response JSON
    └── Return *N8NExecutionResponse
```

### 4. Authentication Flow

```
setAuthHeaders(req)
    │
    ├── If APIKey is set:
    │   └── Set header: X-N8N-API-KEY = <api_key>
    │
    ├── Else if Username and Password are set:
    │   └── Set Basic Auth: username + ":" + password (base64)
    │
    └── Set header: Content-Type = application/json
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `n8n.enabled` | bool | false | Enable/disable n8n manager |
| `n8n.host` | string | "localhost" | n8n host |
| `n8n.port` | int | 5678 | n8n port |
| `n8n.api_key` | string | "" | n8n API key (X-N8N-API-KEY) |
| `n8n.username` | string | "" | Username for Basic Auth |
| `n8n.password` | string | "" | Password for Basic Auth |

## Usage Examples

### List Workflows

```go
ctx := context.Background()

workflows, err := manager.ListWorkflows(ctx)
if err != nil {
    panic(err)
}

fmt.Printf("Total Workflows: %d\n", len(workflows))
for _, wf := range workflows {
    fmt.Printf("  %s (ID: %s, Active: %v)\n", 
        wf.Name, wf.ID, wf.Active)
}
```

### Get a Workflow

```go
ctx := context.Background()

workflow, err := manager.GetWorkflow(ctx, "workflow-id-here")
if err != nil {
    panic(err)
}

fmt.Printf("Workflow: %s\n", workflow.Name)
fmt.Printf("Active: %v\n", workflow.Active)
fmt.Printf("Nodes: %d\n", len(workflow.Nodes))
fmt.Printf("Created: %v\n", workflow.CreatedAt)
```

### Create a Workflow

```go
ctx := context.Background()

workflow := &n8n.N8NWorkflow{
    Name: "My New Workflow",
    Nodes: []n8n.N8NWorkflowNode{
        {
            ID:   "1",
            Name: "Start",
            Type: "n8n-nodes-base.Start",
        },
    },
    Connections: n8n.N8NWorkflowConnections{
        Main: [][]n8n.N8NConnection{
            {
                Node: "1",
                Type: "main",
                Index: 0,
            },
        },
    },
}

created, err := manager.CreateWorkflow(ctx, workflow)
if err != nil {
    panic(err)
}

fmt.Printf("Created Workflow: %s (ID: %s)\n", created.Name, created.ID)
```

### Update a Workflow

```go
ctx := context.Background()

workflow, err := manager.GetWorkflow(ctx, "workflow-id-here")
if err != nil {
    panic(err)
}

// Modify workflow
workflow.Name = "Updated Workflow Name"

updated, err := manager.UpdateWorkflow(ctx, workflow.ID, workflow)
if err != nil {
    panic(err)
}

fmt.Printf("Updated Workflow: %s\n", updated.Name)
```

### Delete a Workflow

```go
ctx := context.Background()

err := manager.DeleteWorkflow(ctx, "workflow-id-here")
if err != nil {
    panic(err)
}
fmt.Println("Workflow deleted successfully")
```

### Activate a Workflow

```go
ctx := context.Background()

result, err := manager.ActivateWorkflow(ctx, "workflow-id-here")
if err != nil {
    panic(err)
}

fmt.Printf("Activated Workflow: %s (Active: %v)\n", result.Name, result.Active)
if len(result.Errors) > 0 {
    for _, e := range result.Errors {
        fmt.Printf("Error: %s (Node: %s)\n", e.Error, e.Node)
    }
}
```

### Run a Workflow

```go
ctx := context.Background()

opts := &n8n.N8NWorkflowRunOptions{
    Mode: "manual",
}

response, err := manager.RunWorkflow(ctx, "workflow-id-here", opts)
if err != nil {
    panic(err)
}

fmt.Printf("Execution ID: %s\n", response.ExecutionID)
fmt.Printf("Waiting for Webhook: %v\n", response.WaitingForWebhook)
```

### List Executions

```go
ctx := context.Background()

// List all executions for a workflow
executions, err := manager.ListExecutions(ctx, "workflow-id-here", "", 10)
if err != nil {
    panic(err)
}

fmt.Printf("Total Executions: %d\n", len(executions))
for _, exec := range executions {
    fmt.Printf("  ID: %s, Status: %s, Started: %v\n", 
        exec.ID, exec.Status, exec.StartedAt)
}
```

### Get Execution

```go
ctx := context.Background()

execution, err := manager.GetExecution(ctx, "execution-id-here")
if err != nil {
    panic(err)
}

fmt.Printf("Execution: %s\n", execution.ID)
fmt.Printf("Status: %s\n", execution.Status)
fmt.Printf("Workflow ID: %s\n", execution.WorkflowID)
```

### List Credentials

```go
ctx := context.Background()

credentials, err := manager.ListCredentials(ctx)
if err != nil {
    panic(err)
}

fmt.Printf("Total Credentials: %d\n", len(credentials))
for _, cred := range credentials {
    fmt.Printf("  %s (ID: %d, Type: %s)\n", 
        cred.Name, cred.ID, cred.Type)
}
```

### Create Credential

```go
ctx := context.Background()

credential := &n8n.N8NCredential{
    Name: "My API Key",
    Type: "httpBasicAuth",
    Data: map[string]interface{}{
        "username": "myuser",
        "password": "mypassword",
    },
}

created, err := manager.CreateCredential(ctx, credential)
if err != nil {
    panic(err)
}

fmt.Printf("Created Credential: %s (ID: %d)\n", created.Name, created.ID)
```

### List Tags

```go
ctx := context.Background()

tags, err := manager.ListTags(ctx)
if err != nil {
    panic(err)
}

fmt.Printf("Total Tags: %d\n", len(tags))
for _, tag := range tags {
    fmt.Printf("  %s (ID: %d, Usage: %d)\n", 
        tag.Name, tag.ID, tag.UsageCount)
}
```

### List Users

```go
ctx := context.Background()

users, err := manager.ListUsers(ctx)
if err != nil {
    panic(err)
}

fmt.Printf("Total Users: %d\n", len(users))
for _, user := range users {
    fmt.Printf("  %s %s (%s)\n", 
        user.FirstName, user.LastName, user.Email)
}
```

### Get Health Status

```go
ctx := context.Background()

health, err := manager.GetHealth(ctx)
if err != nil {
    panic(err)
}

fmt.Printf("Health Status: %s\n", health.Status)
```

### Get Owner Information

```go
ctx := context.Background()

owner, err := manager.GetOwnerName(ctx)
if err != nil {
    panic(err)
}

fmt.Printf("Owner: %s\n", owner)
```

### Async Operations

```go
ctx := context.Background()

// Async list workflows
workflowsChan := manager.ListWorkflowsAsync(ctx)
workflows, err := workflowsChan.Get(ctx)
if err != nil {
    panic(err)
}
fmt.Printf("Workflows: %d\n", len(workflows))

// Async create workflow
workflow := &n8n.N8NWorkflow{Name: "Async Workflow"}
createChan := manager.CreateWorkflowAsync(ctx, workflow)
created, err := createChan.Get(ctx)
if err != nil {
    panic(err)
}
fmt.Printf("Created: %s\n", created.Name)

// Async run workflow
runChan := manager.RunWorkflowAsync(ctx, "workflow-id", nil)
response, err := runChan.Get(ctx)
if err != nil {
    panic(err)
}
fmt.Printf("Execution: %s\n", response.ExecutionID)
```

### Getting Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
fmt.Printf("Base URL: %s\n", status["base_url"])

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
| Standard library | `encoding/json`, `net/http`, `bytes`, `io`, `time`, `strconv`, `strings` |

## Error Handling

The library uses Go error handling patterns:

- **Authentication failures**: Returned from API (invalid API key or credentials)
- **API errors**: Returned from n8n API calls
- **Nil checks**: Public methods check manager state

Example error handling:

```go
workflows, err := manager.ListWorkflows(ctx)
if err != nil {
    if strings.Contains(err.Error(), "401") {
        return fmt.Errorf("invalid API key: %w", err)
    }
    if strings.Contains(err.Error(), "403") {
        return fmt.Errorf("access forbidden: %w", err)
    }
    if strings.Contains(err.Error(), "404") {
        return fmt.Errorf("workflow not found: %w", err)
    }
    return fmt.Errorf("API request failed: %w", err)
}
```

## Common Pitfalls

### 1. Manager Not Enabled

**Problem**: `n8n.enabled` is false

**Solution**:
- Set `n8n.enabled = true` in configuration
- Check viper configuration is loaded

### 2. Authentication Missing

**Problem**: Neither API key nor username/password set

**Solution**:
- Set `n8n.api_key` to your n8n API key, OR
- Set both `n8n.username` and `n8n.password` for Basic Auth
- Get API key from n8n → Settings → API Keys

### 3. n8n Not Running

**Problem**: Cannot connect to n8n

**Solution**:
- Ensure n8n is running locally or at the specified host/port
- Check that the n8n API is accessible
- Verify host and port settings

### 4. Workflow Not Found

**Problem**: `404` when getting/updating/deleting workflow

**Solution**:
- Verify workflow ID is correct
- Use ListWorkflows() to get valid workflow IDs
- Check workflow exists and is accessible

### 5. Rate Limiting

**Problem**: `429` Too Many Requests

**Solution**:
- Library retries up to 3 times
- Consider using async operations to spread requests
- Respect n8n rate limits

### 6. Worker Pool Not Available

**Problem**: Async operations falling back to direct execution

**Solution**:
- The pool is created during initialization
- Check that `NewN8NManager()` succeeded
- Verify `n8n.enabled` is true

## Advanced Usage

### Health Check

```go
func healthCheck(manager *n8n.N8NManager) error {
    status := manager.GetStatus()
    if status["connected"] == false {
        return fmt.Errorf("not connected to n8n API")
    }
    fmt.Printf("n8n API is healthy (URL: %s)\n", status["base_url"])
    return nil
}
```

### Batch Processing with Async

```go
func batchCreateWorkflows(manager *n8n.N8NManager, workflows []*n8n.N8NWorkflow) ([]*n8n.N8NWorkflow, error) {
    ctx := context.Background()
    results := make([]*n8n.N8NWorkflow, len(workflows))
    errors := make([]error, len(workflows))
    
    // Submit all requests asynchronously
    for i, wf := range workflows {
        func(idx int, workflow *n8n.N8NWorkflow) {
            manager.SubmitAsyncJob(func() {
                created, err := manager.CreateWorkflow(ctx, workflow)
                results[idx] = created
                errors[idx] = err
            })
        }(i, wf)
    }
    
    // Collect results
    for i, err := range errors {
        if err != nil {
            return nil, fmt.Errorf("failed to create workflow %d: %w", i, err)
        }
    }
    return results, nil
}
```

### Workflow with Run Options

```go
func runWorkflowWithData(manager *n8n.N8NManager, workflowID string) error {
    ctx := context.Background()
    
    opts := &n8n.N8NWorkflowRunOptions{
        Mode: "manual",
        RunData: map[string]interface{}{
            "startTime": time.Now().Format(time.RFC3339),
        },
        PinData: map[string]interface{}{
            "input": "test data",
        },
    }
    
    response, err := manager.RunWorkflow(ctx, workflowID, opts)
    if err != nil {
        return err
    }
    
    fmt.Printf("Started execution: %s\n", response.ExecutionID)
    return nil
}
```

## Internal Algorithms

### API Request Flow

```
doRequest(ctx, method, endpoint, body):
    │
    ├── Create request with context
    ├── Set authentication headers (setAuthHeaders)
    ├── Execute with retryablehttp (3 retries)
    ├── Check status code (200-299 = success)
    ├── Read response body
    └── Return response body or error
```

### Authentication

```
setAuthHeaders(req):
    │
    ├── If APIKey != "":
    │   └── Set header: X-N8N-API-KEY = APIKey
    │
    ├── Else if Username != "" && Password != "":
    │   └── Set Basic Auth header
    │
    └── Set header: Content-Type = application/json
```

### Connection Test

```
testConnection():
    │
    ├── GET /health
    ├── Set authentication headers
    ├── Check status code (200 OK)
    └── Return error if failed
```

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/health` | GET | Get health status |
| `/owner` | GET | Get owner information |
| `/workflows` | GET | List workflows |
| `/workflows` | POST | Create workflow |
| `/workflows/{id}` | GET | Get workflow |
| `/workflows/{id}` | PATCH | Update workflow |
| `/workflows/{id}` | DELETE | Delete workflow |
| `/workflows/{id}/activate` | POST | Activate workflow |
| `/workflows/{id}/deactivate` | POST | Deactivate workflow |
| `/workflows/{id}/execute` | POST | Run workflow |
| `/executions` | GET | List executions |
| `/executions/{id}` | GET | Get execution |
| `/executions/{id}` | DELETE | Delete execution |
| `/executions/{id}/retry` | POST | Retry execution |
| `/credentials` | GET | List credentials |
| `/credentials/{id}` | GET | Get credential |
| `/credentials` | POST | Create credential |
| `/credentials/{id}` | PATCH | Update credential |
| `/credentials/{id}` | DELETE | Delete credential |
| `/tags` | GET | List tags |
| `/tags` | POST | Create tag |
| `/tags/{id}` | PATCH | Update tag |
| `/tags/{id}` | DELETE | Delete tag |
| `/users` | GET | List users |
| `/users` | POST | Invite user |
| `/users/{id}` | DELETE | Delete user |
| `/node-types` | GET | List node types |
| `/node-types/{name}` | GET | Get node type |

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.