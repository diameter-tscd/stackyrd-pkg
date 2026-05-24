# OpenWrt Router Manager

## Overview

The `OpenWrtManager` provides a Go client for interacting with OpenWrt‑based routers. It supports configuration management, package installation, firewall rule handling, and system monitoring via the OpenWrt Ubus JSON‑RPC API.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **System Info** – Retrieve hostname, model, firmware version.
- **Network Interfaces** – List, enable/disable, and configure interfaces.
- **Firewall** – Add, remove, and list firewall rules.
- **Package Management** – Install, remove, and query opkg packages.
- **Service Control** – Start, stop, and restart system services.
- **Metrics** – CPU, memory, and storage usage statistics.
- **Retry Logic** – HTTP client with configurable retries.
- **Async Operations** – Optional worker pool for concurrent tasks.

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
    log := logger.NewLogger()
    mgr, err := router.NewOpenWrtManager(log)
    if err != nil {
        panic(err)
    }
    defer mgr.Close()

    ctx := context.Background()
    sysInfo, err := mgr.GetSystemInfo(ctx)
    if err != nil {
        panic(err)
    }
    fmt.Printf("Router: %s (%s)\n", sysInfo.Hostname, sysInfo.Model)
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `OpenWrtManager` | Main client holding HTTP client, auth token, and optional worker pool |
| `SystemInfo` | Hostname, model, firmware version |
| `Interface` | Network interface details |
| `FirewallRule` | Firewall rule definition |
| `PackageInfo` | Opkg package metadata |

### Concurrency Model

```
OpenWrtManager
├─ retryablehttp.Client (retry logic)
└─ WorkerPool (optional, for async calls)
```

## Configuration

Configuration is driven by **Viper**. Relevant keys:

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `openwrt.enabled` | bool | false | Enable the manager |
| `openwrt.base_url` | string | "" | Base URL of the router (e.g., `http://192.168.1.1`) |
| `openwrt.username` | string | "root" | RPC username |
| `openwrt.password` | string | "" | RPC password |
| `openwrt.retry_max` | int | 3 | Max retry attempts |

## Usage Examples

### List Interfaces

```go
ifs, err := mgr.ListInterfaces(ctx)
if err != nil { panic(err) }
for _, i := range ifs { fmt.Println(i.Name, i.Address) }
```

### Add Firewall Rule

```go
rule := router.FirewallRule{Action: "accept", Src: "*", DestPort: "22"}
if err := mgr.AddFirewallRule(ctx, rule); err != nil { panic(err) }
```

### Install Package

```go
if err := mgr.InstallPackage(ctx, "luci-app-wol"); err != nil { panic(err) }
```

## Dependencies

| Dependency | Role |
|------------|------|
| `github.com/hashicorp/go-retryablehttp` | HTTP client with retries |
| `github.com/spf13/viper` | Configuration |
| `stackyrd/pkg/logger` | Structured logging |

## Error Handling

All methods return Go `error`. Common patterns include checking for authentication failures (HTTP 401) and network timeouts.

## License

Part of the Stackyrd project. See the top‑level LICENSE file.
