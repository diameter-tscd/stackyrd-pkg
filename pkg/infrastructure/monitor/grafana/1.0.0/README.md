# Grafana Manager

## Overview

The `GrafanaManager` is a Go library for interacting with the Grafana API. It provides comprehensive dashboard management (create, update, get, delete, list), data source management, annotation creation, and health monitoring. The library supports both API key (Bearer token) and Basic authentication, with retry logic for reliability.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Dashboard Management**: Create, update, get, delete, and list Grafana dashboards
- **Data Source Management**: Create data sources (Prometheus, InfluxDB, etc.)
- **Annotations**: Create annotations on dashboards
- **Health Monitoring**: Check Grafana health status
- **Authentication**: Supports API key (Bearer) and Basic auth
- **Retry Logic**: HTTP client with retry support (3 retries)
- **Worker Pool**: Async job execution support (5 workers)
- **Status Monitoring**: Get connection status and API health
- **Async Operations**: Full async support for all operations

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
    
    // Create Grafana manager (configuration via config)
    cfg := config.GrafanaConfig{
        Enabled: true,
        URL:     "http://localhost:3000",
        APIKey:  "your-api-key",
    }
    manager, err := grafana.NewGrafanaManager(cfg, log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // List dashboards
    dashboards, err := manager.ListDashboards(ctx)
    if err != nil {
        panic(err)
    }
    fmt.Printf("Found %d dashboards\n", len(dashboards))
    for _, db := range dashboards {
        fmt.Printf("  %s (UID: %s)\n", db.Title, db.UID)
    }
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `GrafanaManager` | Main manager with HTTP client, credentials, worker pool |
| `GrafanaDashboard` | Dashboard with panels, time settings, templating |
| `GrafanaPanel` | Individual panel with queries, field config |
| `GrafanaTarget` | Query target (PromQL expression) |
| `GrafanaDataSource` | Data source configuration |
| `GrafanaAnnotation` | Annotation with time, tags, text |

### Concurrency Model

```
┌─────────────────────────────────────────────────────┐
│                   GrafanaManager                         │
├─────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │  retryablehttp │  → API requests with retry               │
│  │  .Client      │  → Bearer token or Basic auth           │
│  └────────────────┘                                      │
│                                                          │
│  ┌─────────────┐      ┌──────────────┐                   │
│  │  WorkerPool │◄─────│ AsyncResult │                   │
│  │  (5 workers)│      │   Channel   │                   │
│  └─────────────┘      └──────────────┘                   │
│         ▲                      │                            │
│         │                      ▼                            │
│  ┌─────────────┐      ┌──────────────┐                  │
│  │  SubmitJob  │      │ ExecuteAsync │                  │
│  └─────────────┘      └──────────────┘                  │
│                                                          │
│  BaseURL: http://localhost:3000 (default)         │
│  APIKey: Grafana API key (Bearer token)                │
│  Username/Password: For Basic auth                 │
└─────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewGrafanaManager(cfg, logger)
    │
    ├── Check cfg.Enabled
    ├── Get baseURL from cfg.URL (trim trailing slash)
    ├── Get apiKey, username, password from cfg
    │
    ├── Create retryablehttp.Client:
    │   ├── RetryMax = 3
    │   ├── RetryWaitMin = 1s
    │   ├── RetryWaitMax = 5s
    │   └── HTTPClient.Timeout = 30s
    │
    ├── Set authentication:
    │   ├── If APIKey: hook sets "Authorization: Bearer {apiKey}"
    │   └── If Username: use Basic auth
    │
    ├── Test connection: testConnection()
    │   └── GET /api/health
    │
    ├── Create WorkerPool(5)
    ├── Start worker pool
    └── Return GrafanaManager
```

### 2. Create Dashboard Flow

```
CreateDashboard(ctx, dashboard)
    │
    ├── Build payload: {"dashboard": dashboard, "overwrite": false}
    ├── Marshal to JSON
    ├── POST /api/dashboards/db
    ├── Set header: Content-Type = application/json
    ├── Execute with retryablehttp.Client
    ├── Check status code (200 OK)
    ├── Decode JSON response for ID, UID, URL
    └── Return *GrafanaDashboard
```

### 3. Get Dashboard Flow

```
GetDashboard(ctx, uid)
    │
    ├── GET /api/dashboards/uid/{uid}
    ├── Execute with retryablehttp.Client
    ├── Check status code (200 OK)
    ├── Decode JSON response
    └── Return *GrafanaDashboard
```

### 4. Health Check Flow

```
testConnection()
    │
    ├── GET /api/health
    ├── Execute with retryablehttp.Client
    ├── Check status code (200 OK)
    └── Return error if failed
```

## Configuration

### Via config.GrafanaConfig struct

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `Enabled` | bool | false | Enable/disable Grafana manager |
| `URL` | string | "http://localhost:3000" | Grafana base URL |
| `APIKey` | string | "" | Grafana API key (Bearer token) |
| `Username` | string | "" | Username for Basic auth |
| `Password` | string | "" | Password for Basic auth |

## Usage Examples

### Create Dashboard

```go
ctx := context.Background()

dashboard := grafana.GrafanaDashboard{
    Title:    "My Dashboard",
    Timezone: "browser",
    SchemaVersion: 36,
    Version:   1,
    Panels: []grafana.GrafanaPanel{
        {
            ID:    1,
            Title: "CPU Usage",
            Type:  "graph",
            GridPos: grafana.GrafanaGridPos{H: 8, W: 12, X: 0, Y: 0},
            Targets: []grafana.GrafanaTarget{
                {
                    Expr:      "cpu_usage",
                    LegendFormat: "{{instance}}",
                },
            },
        },
    },
}

result, err := manager.CreateDashboard(ctx, dashboard)
if err != nil {
    panic(err)
}
fmt.Printf("Created dashboard: %s (UID: %s)\n", result.Title, result.UID)
```

### Get Dashboard

```go
ctx := context.Background()

dashboard, err := manager.GetDashboard(ctx, "my-dashboard-uid")
if err != nil {
    panic(err)
}
fmt.Printf("Dashboard: %s\n", dashboard.Title)
fmt.Printf("Panels: %d\n", len(dashboard.Panels))
for _, panel := range dashboard.Panels {
    fmt.Printf("  Panel: %s (Type: %s)\n", panel.Title, panel.Type)
}
```

### Update Dashboard

```go
ctx := context.Background()

dashboard, err := manager.GetDashboard(ctx, "my-dashboard-uid")
if err != nil {
    panic(err)
}

// Modify dashboard
dashboard.Title = "Updated Dashboard Title"

updated, err := manager.UpdateDashboard(ctx, *dashboard)
if err != nil {
    panic(err)
}
fmt.Printf("Updated dashboard version: %d\n", updated.Version)
```

### Delete Dashboard

```go
ctx := context.Background()

err := manager.DeleteDashboard(ctx, "my-dashboard-uid")
if err != nil {
    panic(err)
}
fmt.Println("Dashboard deleted successfully")
```

### List Dashboards

```go
ctx := context.Background()

dashboards, err := manager.ListDashboards(ctx)
if err != nil {
    panic(err)
}

fmt.Printf("Total dashboards: %d\n", len(dashboards))
for _, db := range dashboards {
    fmt.Printf("  %s (UID: %s, Type: %s)\n", db.Title, db.UID, db.Type)
}
```

### Create Data Source

```go
ctx := context.Background()

ds := grafana.GrafanaDataSource{
    Name:     "Prometheus",
    Type:     "prometheus",
    URL:      "http://localhost:9090",
    Access:   "proxy",
    BasicAuth: true,
    BasicAuthUser: "admin",
    Database: "mydb",
}

result, err := manager.CreateDataSource(ctx, ds)
if err != nil {
    panic(err)
}
fmt.Printf("Created data source: %s (ID: %d)\n", result.Name, result.ID)
```

### Create Annotation

```go
ctx := context.Background()

annotation := grafana.GrafanaAnnotation{
    DashboardID: 1,
    Time:       time.Now().UnixNano(),
    Tags:      []string{"deploy", "production"},
    Text:      "Deployment completed",
}

result, err := manager.CreateAnnotation(ctx, annotation)
if err != nil {
    panic(err)
}
fmt.Printf("Created annotation ID: %d\n", result.ID)
```

### Async Operations

```go
ctx := context.Background()

// Async create dashboard
dashboard := grafana.GrafanaDashboard{Title: "Async Dashboard"}
createChan := manager.CreateDashboardAsync(ctx, dashboard)
created, err := createChan.Get(ctx)
if err != nil {
    panic(err)
}
fmt.Printf("Created: %s\n", created.Title)

// Async list dashboards
listChan := manager.ListDashboardsAsync(ctx)
dashboards, err := listChan.Get(ctx)
if err != nil {
    panic(err)
}
fmt.Printf("Total: %d\n", len(dashboards))
```

### Getting Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
fmt.Printf("URL: %s\n", status["url"])
fmt.Printf("Version: %v\n", status["version"])
fmt.Printf("Database: %v\n", status["database"])

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
| Standard library | `encoding/json`, `net/http`, `bytes`, `io`, `time`, `strings` |

## Error Handling

The library uses Go error handling patterns:

- **Authentication failures**: Returned from API (invalid API key)
- **API errors**: Returned from Grafana API calls
- **Nil checks**: Public methods check manager state

Example error handling:

```go
dashboard, err := manager.GetDashboard(ctx, uid)
if err != nil {
    if strings.Contains(err.Error(), "401") {
        return fmt.Errorf("invalid API key: %w", err)
    }
    if strings.Contains(err.Error(), "403") {
        return fmt.Errorf("access forbidden: %w", err)
    }
    if strings.Contains(err.Error(), "404") {
        return fmt.Errorf("dashboard not found: %w", err)
    }
    return fmt.Errorf("API request failed: %w", err)
}
```

## Common Pitfalls

### 1. Manager Not Enabled

**Problem**: Grafana manager not enabled

**Solution**:
- Set `cfg.Enabled = true` in GrafanaConfig
- Ensure configuration is passed correctly

### 2. API Key Missing

**Problem**: No authentication credentials

**Solution**:
- Set `cfg.APIKey` to your Grafana API key
- Or set `cfg.Username` and `cfg.Password` for Basic auth
- Generate API key in Grafana: Administration → API Keys → New API Key

### 3. Grafana Not Reachable

**Problem**: Cannot connect to Grafana

**Solution**:
- Verify `cfg.URL` is correct (e.g., "http://localhost:3000")
- Ensure Grafana is running and accessible
- Check firewall/network settings

### 4. Dashboard Not Found

**Problem**: `404` when getting dashboard

**Solution**:
- Verify dashboard UID is correct
- Use ListDashboards to find valid UIDs
- Check dashboard exists and is accessible

### 5. Worker Pool Not Available

**Problem**: Async operations falling back to direct execution

**Solution**:
- The pool is created during initialization
- Check that `NewGrafanaManager()` succeeded
- Verify `cfg.Enabled` is true

## Advanced Usage

### Health Check

```go
func healthCheck(manager *grafana.GrafanaManager) error {
    ctx := context.Background()
    health, err := manager.GetHealth(ctx)
    if err != nil {
        return fmt.Errorf("not connected to Grafana: %w", err)
    }
    fmt.Printf("Grafana is healthy: %v\n", health)
    return nil
}
```

### Batch Dashboard Operations with Async

```go
func batchCreateDashboards(manager *grafana.GrafanaManager, dashboards []grafana.GrafanaDashboard) ([]*grafana.GrafanaDashboard, error) {
    ctx := context.Background()
    results := make([]*grafana.GrafanaDashboard, len(dashboards))
    errors := make([]error, len(dashboards))
    
    for i, db := range dashboards {
        func(idx int, d grafana.GrafanaDashboard) {
            manager.SubmitAsyncJob(func() {
                created, err := manager.CreateDashboard(ctx, d)
                results[idx] = created
                errors[idx] = err
            })
        }(i, db)
    }
    
    for i, err := range errors {
        if err != nil {
            return nil, fmt.Errorf("failed to create dashboard %d: %w", i, err)
        }
    }
    return results, nil
}
```

## Internal Algorithms

### API Request Flow

```
CreateDashboard():
    │
    ├── Build JSON payload with dashboard object
    ├── POST to /api/dashboards/db
    ├── Set Authorization header or Basic auth
    ├── Execute with retryablehttp (3 retries)
    ├── Parse response JSON
    └── Return dashboard with UID and version
```

### Health Check

```
GetHealth():
    │
    ├── GET /api/health
    ├── Parse response for version, database status
    └── Return health map
```

### Connection Test

```
testConnection():
    │
    ├── GET /api/health
    ├── Check status code (200 OK)
    └── Return error if failed
```

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/health` | GET | Health check |
| `/api/dashboards/db` | POST | Create dashboard |
| `/api/dashboards/uid/{uid}` | GET | Get dashboard by UID |
| `/api/dashboards/uid/{uid}` | POST | Update dashboard |
| `/api/dashboards/uid/{uid}` | DELETE | Delete dashboard |
| `/api/search?type=dash-db` | GET | List dashboards |
| `/api/datasources` | POST | Create data source |
| `/api/annotations` | POST | Create annotation |

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.