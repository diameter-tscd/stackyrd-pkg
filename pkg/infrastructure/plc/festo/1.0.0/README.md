# Festo Automation Manager

## Overview

The `FestoManager` is a Go library for managing Festo industrial automation devices. It provides device management (list, get, health), module monitoring, I/O control, motion control, process data retrieval, job management, program management, and telemetry configuration. The library uses Bearer token authentication and includes retry logic for reliability.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Device Management**: List all devices, get device details, check health status
- **Module Monitoring**: Retrieve module status and diagnostics
- **I/O Control**: Get digital/analog I/O state, set outputs
- **Motion Control**: Get axis status, send motion commands (move, stop, home)
- **Process Data**: Retrieve real-time process data (cycles, pressure, temperature, etc.)
- **Job Management**: List, start, stop automation jobs
- **Program Management**: List and retrieve automation programs
- **Telemetry Configuration**: Configure edge telemetry settings
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
    
    // Create Festo manager (configuration via viper)
    manager, err := festo.NewFestoManager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // List devices
    devices, err := manager.ListDevices(ctx)
    if err != nil {
        panic(err)
    }
    fmt.Printf("Found %d devices\n", devices.Total)
    for _, dev := range devices.Devices {
        fmt.Printf("  %s: %s (Type: %s, Status: %s)\n", dev.ID, dev.Name, dev.Type, dev.Status)
    }
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `FestoManager` | Main manager with HTTP client, credentials, worker pool |
| `FestoDevice` | Device with ID, name, type, status, IP, MAC, firmware |
| `FestoDeviceList` | List of devices with total count |
| `FestoModule` | Module within a device (slot, type, status, diagnostics) |
| `FestoDiagnostics` | Diagnostic info (error/warning codes, over-current, etc.) |
| `FestoIOState` | Digital/analog I/O state for a device |
| `FestoMotionAxis` | Motion axis with position, velocity, torque |
| `FestoMotionCommand` | Motion command (move, stop, home) |
| `FestoProcessData` | Real-time process data (cycles, pressure, etc.) |
| `FestoJob` | Automation job (status, progress, timing) |
| `FestoProgram` | Automation program with steps |
| `FestoProgramStep` | Single step in a program |
| `FestoTelemetryConfig` | Telemetry configuration |

### Concurrency Model

```
┌─────────────────────────────────────────────────────┐
│                   FestoManager                          │
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
│  BaseURL: https://{host}:{port}/api/v1           │
│  APIKey: Festo API key (Bearer token)                   │
│  TenantID: Optional tenant ID header (X-Tenant-ID)    │
└─────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewFestoManager(logger)
    │
    ├── Check viper config: "festo.enabled"
    ├── Get base_url: "festo.base_url"
    ├── Get api_key: "festo.api_key"
    ├── Get tenant_id: "festo.tenant_id"
    │
    ├── Create retryablehttp.Client:
    │   ├── RetryMax = 3
    │   ├── RetryWaitMin = 1s
    │   ├── RetryWaitMax = 10s
    │   └── HTTPClient.Timeout = 30s
    │
    ├── Test connection: testConnection()
    │   └── GET /api/v1/health
    │
    ├── Create WorkerPool(4)
    ├── Start worker pool
    └── Return FestoManager
```

### 2. List Devices Flow

```
ListDevices(ctx)
    │
    ├── GET /api/v1/devices
    ├── Set headers: Authorization (Bearer), X-Tenant-ID (optional)
    ├── Execute with retryablehttp.Client
    ├── Check status code (200 OK)
    ├── Decode JSON into FestoDeviceList
    └── Return *FestoDeviceList
```

### 3. Get Device Flow

```
GetDevice(ctx, deviceID)
    │
    ├── GET /api/v1/devices/{deviceID}
    ├── Set headers
    ├── Execute with retryablehttp.Client
    ├── Check status code (200 OK)
    ├── Decode JSON into FestoDevice
    └── Return *FestoDevice
```

### 4. Control Output Flow

```
SetDigitalOutput(ctx, deviceID, outputName, value)
    │
    ├── Build payload: {outputName: value}
    ├── POST /api/v1/devices/{deviceID}/io/outputs/digital
    ├── Set headers
    ├── Execute with retryablehttp.Client
    ├── Check status code (200/204 OK)
    └── Return error if any
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `festo.enabled` | bool | false | Enable/disable Festo manager |
| `festo.base_url` | string | "" | Festo API base URL (e.g., "https://festo-host:8080") |
| `festo.api_key` | string | "" | Festo API key (Bearer token) |
| `festo.tenant_id` | string | "" | Tenant ID (optional, sent as X-Tenant-ID header) |

## Usage Examples

### List Devices

```go
ctx := context.Background()

devices, err := manager.ListDevices(ctx)
if err != nil {
    panic(err)
}

fmt.Printf("Total devices: %d\n", devices.Total)
for _, dev := range devices.Devices {
    fmt.Printf("  %s: %s (Connected: %v)\n", dev.ID, dev.Name, dev.Connected)
}
```

### Get Device Details

```go
ctx := context.Background()

device, err := manager.GetDevice(ctx, "device-123")
if err != nil {
    panic(err)
}

fmt.Printf("Device: %s\n", device.Name)
fmt.Printf("Type: %s\n", device.Type)
fmt.Printf("Status: %s\n", device.Status)
fmt.Printf("IP: %s, MAC: %s\n", device.IPAddress, device.MACAddress)
fmt.Printf("Firmware: %s\n", device.FirmwareVersion)
```

### Get I/O State

```go
ctx := context.Background()

ioState, err := manager.GetIOState(ctx, "device-123")
if err != nil {
    panic(err)
}

fmt.Printf("Digital Inputs: %v\n", ioState.DigitalInputs)
fmt.Printf("Digital Outputs: %v\n", ioState.DigitalOutputs)
fmt.Printf("Analog Inputs: %v\n", ioState.AnalogInputs)
fmt.Printf("Analog Outputs: %v\n", ioState.AnalogOutputs)
```

### Set Digital Output

```go
ctx := context.Background()

err := manager.SetDigitalOutput(ctx, "device-123", "output1", true)
if err != nil {
    panic(err)
}
fmt.Println("Digital output set successfully")
```

### Set Analog Output

```go
ctx := context.Background()

err := manager.SetAnalogOutput(ctx, "device-123", "analogOut1", 50.5)
if err != nil {
    panic(err)
}
fmt.Println("Analog output set successfully")
```

### Get Motion Axes

```go
ctx := context.Background()

axes, err := manager.GetMotionAxes(ctx, "device-123")
if err != nil {
    panic(err)
}

fmt.Printf("Axes: %d\n", len(axes))
for _, axis := range axes {
    fmt.Printf("  Axis %s: %s (Position: %.2f, Velocity: %.2f)\n", 
        axis.AxisID, axis.Name, axis.Position, axis.Velocity)
}
```

### Send Motion Command

```go
ctx := context.Background()

cmd := festo.FestoMotionCommand{
    AxisID:    "axis1",
    Command:   "moveAbsolute",
    Position:  100.5,
    Velocity:  50.0,
}
err := manager.SendMotionCommand(ctx, "device-123", cmd)
if err != nil {
    panic(err)
}
fmt.Println("Motion command sent")
```

### Home Axis

```go
ctx := context.Background()

err := manager.HomeAxis(ctx, "device-123", "axis1")
if err != nil {
    panic(err)
}
fmt.Println("Axis homing started")
```

### Stop Axis

```go
ctx := context.Background()

err := manager.StopAxis(ctx, "device-123", "axis1")
if err != nil {
    panic(err)
}
fmt.Println("Axis stopped")
```

### Get Process Data

```go
ctx := context.Background()

processData, err := manager.GetProcessData(ctx, "device-123")
if err != nil {
    panic(err)
}

fmt.Printf("Cycle Count: %d\n", processData.CycleCount)
fmt.Printf("Cycle Time: %.2f ms\n", processData.CycleTime)
fmt.Printf("Pressure: %.2f bar\n", processData.Pressure)
fmt.Printf("Temperature: %.2f °C\n", processData.Temperature)
```

### List Jobs

```go
ctx := context.Background()

jobs, err := manager.ListJobs(ctx, "device-123")
if err != nil {
    panic(err)
}

fmt.Printf("Jobs: %d\n", len(jobs))
for _, job := range jobs {
    fmt.Printf("  %s: %s (Status: %s, Progress: %d%%)\n", 
        job.JobID, job.Name, job.Status, job.Progress)
}
```

### Start/Stop Job

```go
ctx := context.Background()

// Start job
err := manager.StartJob(ctx, "device-123", "job-456")
if err != nil {
    panic(err)
}

// Stop job
err = manager.StopJob(ctx, "device-123", "job-456")
if err != nil {
    panic(err)
}
```

### List Programs

```go
ctx := context.Background()

programs, err := manager.ListPrograms(ctx)
if err != nil {
    panic(err)
}

fmt.Printf("Programs: %d\n", len(programs))
for _, prog := range programs {
    fmt.Printf("  %s: %s (Version: %s)\n", prog.ProgramID, prog.Name, prog.Version)
}
```

### Get Program

```go
ctx := context.Background()

program, err := manager.GetProgram(ctx, "program-789")
if err != nil {
    panic(err)
}

fmt.Printf("Program: %s\n", program.Name)
fmt.Printf("Steps: %d\n", len(program.Steps))
```

### Get Device Health

```go
ctx := context.Background()

health, err := manager.GetDeviceHealth(ctx, "device-123")
if err != nil {
    panic(err)
}

fmt.Printf("Overall Health: %s\n", health.Overall)
fmt.Printf("Uptime: %.2f hours\n", health.UptimeHours)
fmt.Printf("Modules: %d\n", len(health.Modules))
```

### Configure Telemetry

```go
ctx := context.Background()

config := festo.FestoTelemetryConfig{
    DeviceID:    "device-123",
    IntervalSec: 60,
    Enabled:     true,
    Metrics:     []string{"pressure", "temperature", "cycleCount"},
    DestinationURL: "https://telemetry-server.com/ingest",
}
err := manager.ConfigureTelemetry(ctx, config)
if err != nil {
    panic(err)
}
fmt.Println("Telemetry configured")
```

### Async Operations

```go
ctx := context.Background()

// Async list devices
devicesChan := manager.ListDevicesAsync(ctx)
devices, err := devicesChan.Get(ctx)
if err != nil {
    panic(err)
}
fmt.Printf("Devices: %d\n", devices.Total)

// Async get device
deviceChan := manager.GetDeviceAsync(ctx, "device-123")
device, err := deviceChan.Get(ctx)
if err != nil {
    panic(err)
}
fmt.Printf("Device: %s\n", device.Name)

// Async get I/O state
ioChan := manager.GetIOStateAsync(ctx, "device-123")
ioState, err := ioChan.Get(ctx)
if err != nil {
    panic(err)
}
fmt.Printf("Digital Inputs: %v\n", ioState.DigitalInputs)
```

### Getting Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
fmt.Printf("Base URL: %s\n", status["base_url"])
fmt.Printf("Total Devices: %v\n", status["total_devices"])
fmt.Printf("Connected Devices: %v\n", status["connected_devices"])

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
- **API errors**: Returned from Festo API calls
- **Nil checks**: Public methods check manager state

Example error handling:

```go
devices, err := manager.ListDevices(ctx)
if err != nil {
    if strings.Contains(err.Error(), "401") {
        return fmt.Errorf("invalid API key: %w", err)
    }
    if strings.Contains(err.Error(), "403") {
        return fmt.Errorf("access forbidden: %w", err)
    }
    return fmt.Errorf("API request failed: %w", err)
}
```

## Common Pitfalls

### 1. Manager Not Enabled

**Problem**: `festo.enabled` is false

**Solution**:
- Set `festo.enabled = true` in configuration
- Check viper configuration is loaded

### 2. API Key Missing

**Problem**: `api_key` not set

**Solution**:
- Set `festo.api_key` to your Festo API key
- Get API key from Festo API management console

### 3. Festo Not Reachable

**Problem**: Cannot connect to Festo device/API

**Solution**:
- Verify `festo.base_url` is correct
- Ensure network can reach the Festo device
- Check firewall settings

### 4. Device Not Found

**Problem**: `404` when getting device

**Solution**:
- Verify device ID is correct
- Check device is registered and accessible

### 5. Worker Pool Not Available

**Problem**: Async operations falling back to direct execution

**Solution**:
- The pool is created during initialization
- Check that `NewFestoManager()` succeeded
- Verify `festo.enabled` is true

## Advanced Usage

### Health Check

```go
func healthCheck(manager *festo.FestoManager) error {
    ctx := context.Background()
    devices, err := manager.ListDevices(ctx)
    if err != nil {
        return fmt.Errorf("not connected to Festo API: %w", err)
    }
    fmt.Printf("Festo API is healthy (%d devices)\n", devices.Total)
    return nil
}
```

### Batch Operations with Async

```go
func batchGetDevices(manager *festo.FestoManager, deviceIDs []string) ([]*festo.FestoDevice, error) {
    ctx := context.Background()
    results := make([]*festo.FestoDevice, len(deviceIDs))
    errors := make([]error, len(deviceIDs))
    
    for i, id := range deviceIDs {
        func(idx int, devID string) {
            manager.SubmitAsyncJob(func() {
                dev, err := manager.GetDevice(ctx, devID)
                results[idx] = dev
                errors[idx] = err
            })
        }(i, id)
    }
    
    for i, err := range errors {
        if err != nil {
            return nil, fmt.Errorf("failed to get device %d: %w", i, err)
        }
    }
    return results, nil
}
```

## Internal Algorithms

### API Request Flow

```
ListDevices():
    │
    ├── GET /api/v1/devices
    ├── Set Authorization header (Bearer token)
    ├── Optionally set X-Tenant-ID header
    ├── Execute with retryablehttp (3 retries)
    ├── Parse response JSON
    └── Return device list
```

### Connection Test

```
testConnection():
    │
    ├── GET /api/v1/health
    ├── Set Authorization header
    ├── Check status code (200 OK)
    └── Return error if failed
```

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/v1/health` | GET | Health check |
| `/api/v1/devices` | GET | List devices |
| `/api/v1/devices/{deviceID}` | GET | Get device details |
| `/api/v1/devices/{deviceID}/modules` | GET | Get device modules |
| `/api/v1/devices/{deviceID}/io` | GET | Get I/O state |
| `/api/v1/devices/{deviceID}/io/outputs/digital` | POST | Set digital output |
| `/api/v1/devices/{deviceID}/io/outputs/analog` | POST | Set analog output |
| `/api/v1/devices/{deviceID}/motion/axes` | GET | Get motion axes |
| `/api/v1/devices/{deviceID}/motion/command` | POST | Send motion command |
| `/api/v1/devices/{deviceID}/process-data` | GET | Get process data |
| `/api/v1/devices/{deviceID}/jobs` | GET | List jobs |
| `/api/v1/devices/{deviceID}/jobs/{jobID}/start` | POST | Start job |
| `/api/v1/devices/{deviceID}/jobs/{jobID}/stop` | POST | Stop job |
| `/api/v1/programs` | GET | List programs |
| `/api/v1/programs/{programID}` | GET | Get program |
| `/api/v1/devices/{deviceID}/health` | GET | Get device health |
| `/api/v1/devices/{deviceID}/telemetry` | PUT | Configure telemetry |

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.