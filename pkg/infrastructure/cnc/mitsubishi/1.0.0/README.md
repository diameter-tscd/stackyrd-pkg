# Mitsubishi CNC Manager

## Overview

The `MitsubishiCNCManager` is a comprehensive Go library for managing Mitsubishi CNC (Computer Numerical Control) controllers. It supports both HTTP API and direct TCP binary protocol (MELSEC/CNC protocol) to interact with Mitsubishi CNC machines. The library provides machine status monitoring, axis position tracking, spindle status, tool management, program control, alarm monitoring, production data collection, and more with async support.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Machine Status**: Get power state, running state, alarm status, emergency stop, mode (AUTO, MDI, JOG, EDIT, HANDLE)
- **Axis Positions**: Monitor machine coordinates, absolute coordinates, relative coordinates, feedrate, and servo load for all axes
- **Spindle Status**: Monitor spindle speed, command speed, load percentage, gear, temperature
- **Tool Management**: Get current tool info, tool offset, tool life count, tool wear
- **Program Control**: Get active program info, start/stop programs, list stored NC programs
- **Alarm Monitoring**: Retrieve active alarms with type, message, severity
- **Production Data**: Get power-on time, operating time, cutting time, cycle time, parts counters
- **Overrides**: Monitor feed override, rapid override, spindle override
- **Work Offsets**: Get work coordinate system offsets (X, Y, Z, A, B, C)
- **Tool Magazine**: Get tool magazine status and pot information
- **Maintenance Info**: Retrieve maintenance counters and limits
- **Macro Variables**: Read macro/common variables
- **Diagnostic Data**: Get raw diagnostic data by category
- **Machine Control**: Start/stop programs, reset CNC controller
- **Dual Protocol**: Supports both HTTP REST API and raw TCP binary protocol
- **Async Operations**: Async execution for all operations via worker pool
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
    
    // Create Mitsubishi CNC manager (configuration via viper)
    manager, err := mitsubishi.NewMitsubishiCNCManager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // Get machine status
    status, err := manager.GetMachineStatus(ctx)
    if err != nil {
        panic(err)
    }
    
    fmt.Printf("Machine Power: %v\n", status.PowerOn)
    fmt.Printf("Running: %v\n", status.Running)
    fmt.Printf("Mode: %s\n", status.Mode)
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `MitsubishiCNCManager` | Main manager with HTTP client, TCP connection, and worker pool |
| `CNCMachineStatus` | Overall machine status (power, running, alarm, mode) |
| `CNCAxisPosition` | Axis position data (machine/absolute/relative coords, feedrate, load) |
| `CNCSpindleStatus` | Spindle status (speed, load, gear, temperature) |
| `CNCToolInfo` | Current tool information (number, offset, life count, wear) |
| `CNCProgramInfo` | Active program information (program number, sequence, block) |
| `CNCAlarm` | Machine alarm (number, type, message, severity) |
| `CNCProductionData` | Production metrics (times, parts counters) |
| `CNCOverride` | Override values (feed, rapid, spindle) |
| `CNCWorkOffset` | Work coordinate offset (X, Y, Z, A, B, C) |
| `CNCMacManTable` | Macro variable (number, value, comment) |
| `CNCToolMagazine` | Tool magazine status (number of pots, pot details) |
| `CNCToolPot` | Single tool pot (pot number, tool number, status) |
| `CNCMaintenanceInfo` | Maintenance counter (item, current/limit values) |
| `CNCDiagnosticData` | Raw diagnostic data (category, data map) |
| `CNCGCodeFile` | Stored NC program (program number, size, comment) |

### Concurrency Model

```
┌─────────────────────────────────────────────────────────────┐
│                   MitsubishiCNCManager                    │
├─────────────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │ retryablehttp │  → Retry logic (max 2, 1-5s wait)  │
│  │ Client        │  → HTTP timeout: configurable          │
│  └────────────────┘                                      │
│                                                         │
│  ┌─────────────┐      ┌──────────────┐                   │
│  │  WorkerPool │◄─────│ AsyncResult │                   │
│  │  (4 workers)│      │   Channel   │                   │
│  └─────────────┘      └──────────────┘                   │
│         ▲                      │                            │
│         │                      ▼                            │
│  ┌─────────────┐      ┌──────────────┐                  │
│  │  SubmitJob  │      │ ExecuteAsync │                  │
│  └─────────────┘      └──────────────┘                  │
│                                                         │
│  Protocol: HTTP API or TCP (port 683 default)            │
│  BaseURL: For HTTP mode                              │
│  Host/Port: For TCP mode                             │
└─────────────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewMitsubishiCNCManager()
    │
    ├── Check viper config: "mitsubishi_cnc.enabled"
    ├── Get host: "mitsubishi_cnc.host"
    ├── Get port: "mitsubishi_cnc.port" (default: 683)
    ├── Get baseURL: "mitsubishi_cnc.base_url"
    ├── Get useTCP: "mitsubishi_cnc.use_tcp"
    ├── Get timeout: "mitsubishi_cnc.timeout_sec" (default: 10)
    │
    ├── Create retryablehttp.Client (max 2 retries, 1-5s wait)
    ├── Set HTTP timeout
    │
    ├── If useTCP:
    │   └── connectTCP() → Establish TCP connection to host:port
    │
    ├── Else (HTTP mode):
    │   └── testConnection() → GET baseURL + "/api/v1/status"
    │
    ├── Create WorkerPool(4)
    ├── Start worker pool
    └── Return MitsubishiCNCManager
```

### 2. HTTP Request Flow

```
httpGet(ctx, endpoint)
    │
    ├── Build URL: baseURL + endpoint
    ├── Create GET request with context
    ├── Execute via retryablehttp.Client
    ├── Check status code (200 OK)
    ├── Read response body
    └── Return bytes or error
```

### 3. TCP Command Flow

```
sendTCPCommand(cmd)
    │
    ├── Check TCP connection exists
    ├── Set deadline: now + timeout
    ├── Write command bytes to connection
    ├── Read response (up to 4096 bytes)
    └── Return response bytes or error
```

### 4. Get Machine Status Flow

```
GetMachineStatus(ctx)
    │
    ├── If UseTCP:
    │   └── getMachineStatusTCP()
    │       ├── Send binary command: {0x01, 0xFF, 0x0A, 0x00, 0x10, 0x00, 0x00, 0x00}
    │       ├── Parse response byte 0 for status flags
    │       └── Parse byte 1 for mode
    │
    ├── Else (HTTP):
    │   ├── httpGet(ctx, "/api/v1/status")
    │   ├── JSON unmarshal to CNCMachineStatus
    │   └── Return status
    │
    └── Return CNCMachineStatus
```

### 5. Get Axis Positions Flow

```
GetAxisPositions(ctx)
    │
    ├── If UseTCP:
    │   └── getAxisPositionsTCP()
    │       ├── Send binary command
    │       ├── Parse response for multiple axes
    │       └── Return []CNCAxisPosition
    │
    ├── Else (HTTP):
    │   ├── httpGet(ctx, "/api/v1/axis/positions")
    │   ├── JSON unmarshal to struct with Axes field
    │   └── Return axes
    │
    └── Return []CNCAxisPosition
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `mitsubishi_cnc.enabled` | bool | false | Enable/disable Mitsubishi CNC manager |
| `mitsubishi_cnc.host` | string | "" | CNC controller host address |
| `mitsubishi_cnc.port` | int | 683 | TCP port (default: 683 for Mitsubishi CNC) |
| `mitsubishi_cnc.base_url` | string | "" | Base URL for HTTP API mode |
| `mitsubishi_cnc.use_tcp` | bool | false | Use direct TCP protocol instead of HTTP |
| `mitsubishi_cnc.timeout_sec` | int | 10 | Connection/request timeout in seconds |

### Environment Variables

The CNC manager does not directly use environment variables. Configuration is read through Viper, which can load from:
- Configuration files (JSON, YAML, TOML)
- Environment variables (if Viper is configured for it)

You can set CNC parameters via:
```go
viper.Set("mitsubishi_cnc.enabled", true)
viper.Set("mitsubishi_cnc.host", "192.168.1.100")
viper.Set("mitsubishi_cnc.port", 683)
viper.Set("mitsubishi_cnc.use_tcp", true)
```

## Usage Examples

### Machine Status

```go
ctx := context.Background()

// Get machine status
status, err := manager.GetMachineStatus(ctx)
if err != nil {
    panic(err)
}

fmt.Printf("Power On: %v\n", status.PowerOn)
fmt.Printf("Running: %v\n", status.Running)
fmt.Printf("Alarm: %v\n", status.Alarm)
fmt.Printf("Emergency Stop: %v\n", status.EmergencyStop)
fmt.Printf("Ready: %v\n", status.Ready)
fmt.Printf("Mode: %s\n", status.Mode)
fmt.Printf("SubMode: %s\n", status.SubMode)
fmt.Printf("Status Code: %d\n", status.StatusCode)
```

### Axis Positions

```go
ctx := context.Background()

// Get axis positions
axes, err := manager.GetAxisPositions(ctx)
if err != nil {
    panic(err)
}

for _, axis := range axes {
    fmt.Printf("Axis: %s\n", axis.AxisName)
    fmt.Printf("  Machine Coord: %.3f\n", axis.MachineCoord)
    fmt.Printf("  Absolute Coord: %.3f\n", axis.AbsCoord)
    fmt.Printf("  Relative Coord: %.3f\n", axis.RelCoord)
    fmt.Printf("  Feedrate: %.1f\n", axis.Feedrate)
    fmt.Printf("  Load: %.1f%%\n", axis.LoadPercent)
    fmt.Printf("  Servo Alarm: %v\n", axis.ServoAlarm)
}
```

### Spindle Status

```go
ctx := context.Background()

// Get spindle status (spindle number 1)
spindle, err := manager.GetSpindleStatus(ctx, 1)
if err != nil {
    panic(err)
}

fmt.Printf("Spindle %d:\n", spindle.SpindleNo)
fmt.Printf("  Speed: %.1f RPM\n", spindle.Speed)
fmt.Printf("  Command Speed: %.1f RPM\n", spindle.CommandSpeed)
fmt.Printf("  Load: %.1f%%\n", spindle.LoadPercent)
fmt.Printf("  Gear: %d\n", spindle.Gear)
fmt.Printf("  Temperature: %.1f°C\n", spindle.TempCelsius)
fmt.Printf("  Orientation: %v\n", spindle.Orientation)
```

### Tool Information

```go
ctx := context.Background()

// Get current tool info
tool, err := manager.GetToolInfo(ctx)
if err != nil {
    panic(err)
}

fmt.Printf("Current Tool: %d\n", tool.ToolNo)
fmt.Printf("  Offset: %d\n", tool.ToolOffset)
fmt.Printf("  Group: %d\n", tool.ToolGroup)
fmt.Printf("  Life Count: %d / %d (%s)\n", tool.LifeCount, tool.LifeMax, tool.LifeType)
fmt.Printf("  Diameter: %.3f\n", tool.Diameter)
fmt.Printf("  Length: %.3f\n", tool.Length)
fmt.Printf("  Wear X: %.3f, Z: %.3f\n", tool.WearX, tool.WearZ)
```

### Program Information

```go
ctx := context.Background()

// Get active program info
progInfo, err := manager.GetProgramInfo(ctx)
if err != nil {
    panic(err)
}

fmt.Printf("Program No: %d\n", progInfo.ProgramNo)
fmt.Printf("Main Program: %d\n", progInfo.MainProgram)
fmt.Printf("Sequence No: %d\n", progInfo.SequenceNo)
fmt.Printf("Block No: %d\n", progInfo.BlockNo)
fmt.Printf("Running Block: %s\n", progInfo.RunningBlock)
fmt.Printf("MDI Buffer: %s\n", progInfo.MDIBuffer)
```

### Alarms

```go
ctx := context.Background()

// Get active alarms
alarms, err := manager.GetAlarms(ctx)
if err != nil {
    panic(err)
}

fmt.Printf("Active Alarms: %d\n", len(alarms))
for _, alarm := range alarms {
    fmt.Printf("  Alarm %d [%s]: %s\n", alarm.AlarmNo, alarm.Type, alarm.Message)
    fmt.Printf("    Severity: %s, Axis: %s\n", alarm.Severity, alarm.Axis)
    fmt.Printf("    Time: %v, Active: %v\n", alarm.Timestamp, alarm.Active)
}
```

### Production Data

```go
ctx := context.Background()

// Get production data
prod, err := manager.GetProductionData(ctx)
if err != nil {
    panic(err)
}

fmt.Printf("Production Data:\n")
fmt.Printf("  Power-On Time: %d minutes\n", prod.PowerOnTime)
fmt.Printf("  Operating Time: %d minutes\n", prod.OperatingTime)
fmt.Printf("  Cutting Time: %d minutes\n", prod.CuttingTime)
fmt.Printf("  Cycle Time: %.1f seconds\n", prod.CycleTime)
fmt.Printf("  Parts Total: %d\n", prod.PartsTotal)
fmt.Printf("  Parts Required: %d\n", prod.PartsRequired)
fmt.Printf("  Parts Count: %d\n", prod.PartsCount)
fmt.Printf("  Parts Remaining: %d\n", prod.PartsRemaining)
```

### Overrides

```go
ctx := context.Background()

// Get override values
overrides, err := manager.GetOverrides(ctx)
if err != nil {
    panic(err)
}

fmt.Printf("Overrides:\n")
fmt.Printf("  Feed Override: %d%%\n", overrides.FeedOverride)
fmt.Printf("  Rapid Override: %d%%\n", overrides.RapidOverride)
fmt.Printf("  Spindle Override: %d%%\n", overrides.SpindleOverride)
```

### Work Offsets

```go
ctx := context.Background()

// Get work offsets
offsets, err := manager.GetWorkOffsets(ctx)
if err != nil {
    panic(err)
}

for _, offset := range offsets {
    fmt.Printf("Offset %d:\n", offset.OffsetNo)
    fmt.Printf("  X: %.3f, Y: %.3f, Z: %.3f\n", offset.X, offset.Y, offset.Z)
    fmt.Printf("  A: %.3f, B: %.3f, C: %.3f\n", offset.A, offset.B, offset.C)
}
```

### Tool Magazine

```go
ctx := context.Background()

// Get tool magazine status
magazine, err := manager.GetToolMagazine(ctx)
if err != nil {
    panic(err)
}

fmt.Printf("Tool Magazine %d:\n", magazine.MagazineNo)
fmt.Printf("  Number of Pots: %d\n", magazine.NumPots)

for _, pot := range magazine.Pots {
    fmt.Printf("  Pot %d: Tool %d (Offset: %d, Status: %s)\n", 
        pot.PotNo, pot.ToolNo, pot.ToolOffset, pot.Status)
}
```

### Maintenance Information

```go
ctx := context.Background()

// Get maintenance info
items, err := manager.GetMaintenanceInfo(ctx)
if err != nil {
    panic(err)
}

for _, item := range items {
    fmt.Printf("Maintenance Item %d: %s\n", item.ItemNo, item.Name)
    fmt.Printf("  Current: %.2f / Limit: %.2f %s\n", 
        item.CurrentValue, item.LimitValue, item.Unit)
    fmt.Printf("  Remaining: %.2f\n", item.Remaining)
}
```

### Macro Variables

```go
ctx := context.Background()

// Get macro variables (start from 1, get 10 variables)
variables, err := manager.GetMacroVariables(ctx, 1, 10)
if err != nil {
    panic(err)
}

for _, v := range variables {
    fmt.Printf("Variable #%d: %.3f", v.VariableNo, v.Value)
    if v.Comment != "" {
        fmt.Printf(" (%s)", v.Comment)
    }
    fmt.Println()
}
```

### Program List

```go
ctx := context.Background()

// Get stored NC programs
programs, err := manager.GetProgramList(ctx)
if err != nil {
    panic(err)
}

fmt.Printf("Stored Programs: %d\n", len(programs))
for _, prog := range programs {
    fmt.Printf("  Program %d: %s (Size: %d bytes)\n", 
        prog.ProgramNo, prog.Comment, prog.Size)
    fmt.Printf("    Modified: %v, Protected: %v\n", prog.ModifiedTime, prog.Protected)
}
```

### Diagnostic Data

```go
ctx := context.Background()

// Get diagnostic data for a category
diag, err := manager.GetDiagnosticData(ctx, "servo")
if err != nil {
    panic(err)
}

fmt.Printf("Diagnostic Data (%s):\n", diag.Category)
fmt.Printf("  Timestamp: %v\n", diag.Timestamp)
for key, value := range diag.Data {
    fmt.Printf("  %s: %v\n", key, value)
}
```

### Program Control

```go
ctx := context.Background()

// Start a program
err := manager.StartProgram(ctx, 100)
if err != nil {
    panic(err)
}
fmt.Println("Program 100 started")

// Stop program execution
err = manager.StopProgram(ctx)
if err != nil {
    panic(err)
}
fmt.Println("Program stopped")

// Reset CNC controller
err = manager.ResetCNC(ctx)
if err != nil {
    panic(err)
}
fmt.Println("CNC reset")
```

### Async Operations

```go
ctx := context.Background()

// Async get machine status
result := manager.GetMachineStatusAsync(ctx)
select {
case status := <-result.Ch:
    fmt.Printf("Status: Power=%v, Running=%v\n", status.PowerOn, status.Running)
case err := <-result.ErrCh:
    fmt.Printf("Error: %v\n", err)
}

// Async get axis positions
result2 := manager.GetAxisPositionsAsync(ctx)
select {
case axes := <-result2.Ch:
    fmt.Printf("Found %d axes\n", len(axes))
case err := <-result2.ErrCh:
    fmt.Printf("Error: %v\n", err)
}

// Async get spindle status
result3 := manager.GetSpindleStatusAsync(ctx, 1)
select {
case spindle := <-result3.Ch:
    fmt.Printf("Spindle speed: %.1f\n", spindle.Speed)
case err := <-result3.ErrCh:
    fmt.Printf("Error: %v\n", err)
}

// Async start program
result4 := manager.StartProgramAsync(ctx, 100)
select {
case <-result4.Ch:
    fmt.Println("Program started")
case err := <-result4.ErrCh:
    fmt.Printf("Error: %v\n", err)
}

// Async stop program
result5 := manager.StopProgramAsync(ctx)
select {
case <-result5.Ch:
    fmt.Println("Program stopped")
case err := <-result5.ErrCh:
    fmt.Printf("Error: %v\n", err)
}

// Other async methods:
// - GetToolInfoAsync
// - GetProgramInfoAsync
// - GetAlarmsAsync
// - GetProductionDataAsync
// - GetOverridesAsync
```

### Get Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
fmt.Printf("Host: %s\n", status["host"])
fmt.Printf("Port: %v\n", status["port"])
fmt.Printf("Mode: %s\n", status["mode"])
fmt.Printf("Running: %v\n", status["running"])
fmt.Printf("Alarm: %v\n", status["alarm"])
fmt.Printf("Emergency Stop: %v\n", status["emergency_stop"])
fmt.Printf("Power On: %v\n", status["power_on"])
fmt.Printf("Number of Axes: %v\n", status["num_axes"])
fmt.Printf("Current Tool: %v\n", status["current_tool"])
fmt.Printf("Parts Total: %v\n", status["parts_total"])
fmt.Printf("Parts Count: %v\n", status["parts_count"])

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
| `github.com/spf13/viper` | Configuration management for CNC parameters |
| `stackyrd/config` | Internal configuration package |
| `stackyrd/pkg/logger` | Structured logging |
| Standard library | `context`, `encoding/binary`, `encoding/json`, `fmt`, `io`, `net`, `net/http`, `time`, `bytes` |

## Error Handling

The library uses Go error handling patterns:

- **Connection failures**: `testConnection()` validates HTTP connectivity; `connectTCP()` validates TCP connection
- **API errors**: HTTP status codes are checked and errors returned with response body
- **TCP errors**: Connection and read/write errors returned with context
- **Context cancellation**: All operations respect `context.Context` for timeouts
- **Nil checks**: Public methods handle nil receiver gracefully in `GetStatus()`

Example error handling:

```go
status, err := manager.GetMachineStatus(ctx)
if err != nil {
    if strings.Contains(err.Error(), "connection refused") {
        return fmt.Errorf("CNC controller not reachable: %w", err)
    }
    if strings.Contains(err.Error(), "timeout") {
        return fmt.Errorf("CNC controller timeout: %w", err)
    }
    return fmt.Errorf("failed to get machine status: %w", err)
}
```

## Common Pitfalls

### 1. CNC Controller Not Reachable

**Problem**: `connection refused` or `no route to host` errors

**Solution**: 
- Verify host address: `viper.Set("mitsubishi_cnc.host", "192.168.1.100")`
- Check network connectivity (ping the CNC controller)
- Verify firewall allows access to port 683 (TCP) or HTTP port

### 2. Wrong Protocol Mode

**Problem**: Operations failing because of protocol mismatch

**Solution**:
- For HTTP API: set `viper.Set("mitsubishi_cnc.use_tcp", false)` and configure `base_url`
- For TCP protocol: set `viper.Set("mitsubishi_cnc.use_tcp", true)` and configure `host`/`port`

### 3. Timeout Issues

**Problem**: Operations timing out

**Solution**:
```go
viper.Set("mitsubishi_cnc.timeout_sec", 30) // Increase timeout
```

Or use context:
```go
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
```

### 4. TCP Binary Protocol

**Problem**: TCP mode not working as expected

**Solution**:
- The TCP implementation uses placeholder binary commands
- Real Mitsubishi MELSEC/CNC protocol requires proper frame formatting
- Consult Mitsubishi CNC communication protocol documentation
- Verify byte ordering (little-endian used in this implementation)

### 5. HTTP API Endpoints

**Problem**: HTTP mode returning 404 or 500 errors

**Solution**:
- Verify `base_url` is correct (e.g., `http://192.168.1.100:8080`)
- Check if CNC controller has HTTP API enabled
- Verify API endpoint paths match controller's implementation

### 6. Alarm Codes

**Problem**: Understanding alarm types and messages

**Solution**:
- Alarm types: PS (Parameter), SV (Servo), SP (Spindle), EX (External), etc.
- Severity levels: WARNING, ERROR, FATAL
- Use `GetAlarms()` to monitor and log alarms appropriately

## Advanced Usage

### Continuous Monitoring

```go
func monitorMachine(manager *mitsubishi.MitsubishiCNCManager) {
    ticker := time.NewTicker(2 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:
            ctx := context.Background()
            
            // Get machine status
            status, err := manager.GetMachineStatus(ctx)
            if err != nil {
                fmt.Printf("Error getting status: %v\n", err)
                continue
            }
            
            fmt.Printf("\n=== Machine Status ===\n")
            fmt.Printf("Power: %v, Running: %v, Mode: %s\n", 
                status.PowerOn, status.Running, status.Mode)
            
            if status.Alarm {
                alarms, _ := manager.GetAlarms(ctx)
                fmt.Printf("ALARM! Active alarms: %d\n", len(alarms))
            }
            
            // Get axis positions
            axes, err := manager.GetAxisPositions(ctx)
            if err == nil {
                for _, axis := range axes {
                    fmt.Printf("  %s: %.3f (%.1f%% load)\n", 
                        axis.AxisName, axis.MachineCoord, axis.LoadPercent)
                }
            }
        }
    }
}
```

### Production Tracking

```go
func trackProduction(manager *mitsubishi.MitsubishiCNCManager) {
    ctx := context.Background()
    
    for {
        prod, err := manager.GetProductionData(ctx)
        if err != nil {
            fmt.Printf("Error: %v\n", err)
            time.Sleep(10 * time.Second)
            continue
        }
        
        fmt.Printf("Parts: %d/%d (Remaining: %d)\n", 
            prod.PartsCount, prod.PartsTotal, prod.PartsRemaining)
        fmt.Printf("Cutting Time: %d min, Cycle Time: %.1f sec\n", 
            prod.CuttingTime, prod.CycleTime)
        
        if prod.PartsCount >= prod.PartsRequired {
            fmt.Println("Production target reached!")
            break
        }
        
        time.Sleep(30 * time.Second)
    }
}
```

### Tool Life Management

```go
func checkToolLife(manager *mitsubishi.MitsubishiCNCManager) {
    ctx := context.Background()
    
    tool, err := manager.GetToolInfo(ctx)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    
    fmt.Printf("Tool %d Life: %d/%d (%s)\n", 
        tool.ToolNo, tool.LifeCount, tool.LifeMax, tool.LifeType)
    
    // Calculate remaining life percentage
    if tool.LifeMax > 0 {
        remaining := float64(tool.LifeMax - tool.LifeCount) / float64(tool.LifeMax) * 100
        fmt.Printf("Remaining Life: %.1f%%\n", remaining)
        
        if remaining < 10 {
            fmt.Println("WARNING: Tool life nearly exhausted!")
        }
    }
    
    // Check wear
    if tool.WearX > 0.1 || tool.WearZ > 0.1 {
        fmt.Println("WARNING: Tool wear exceeds threshold!")
    }
}
```

### Maintenance Scheduling

```go
func checkMaintenance(manager *mitsubishi.MitsubishiCNCManager) {
    ctx := context.Background()
    
    items, err := manager.GetMaintenanceInfo(ctx)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    
    fmt.Println("=== Maintenance Status ===")
    for _, item := range items {
        remaining := item.Remaining
        if remaining < 0 {
            remaining = 0
        }
        
        fmt.Printf("Item %d (%s): %.1f/%.1f %s (Remaining: %.1f)\n", 
            item.ItemNo, item.Name, item.CurrentValue, item.LimitValue, 
            item.Unit, remaining)
        
        if remaining < (item.LimitValue * 0.1) {
            fmt.Printf("  -> MAINTENANCE REQUIRED SOON!\n")
        }
    }
}
```

## Internal Algorithms

### TCP Binary Protocol

The TCP mode uses a simplified binary protocol (placeholder implementation):

```
Command Format (8 bytes):
  Byte 0: Command code (0x01 = status, 0x02 = axis, etc.)
  Byte 1: Sub-command or parameter
  Byte 2: 0x0A (fixed)
  Byte 3: 0x00
  Byte 4: Data/address low byte
  Byte 5: Data/address high byte
  Byte 6: 0x00
  Byte 7: 0x00

Response Parsing:
  - Status: Byte 0 contains bit flags for power, running, alarm, etc.
  - Axis data: Multiple 32-byte blocks, each containing:
      - Axis name (4 bytes)
      - Machine coord (4 bytes, little-endian int32 / 1000)
      - Absolute coord (4 bytes)
      - Relative coord (4 bytes)
      - ... (additional fields)
```

### HTTP API Request Flow

```
httpGet(ctx, endpoint):
    │
    ├── Build URL: baseURL + endpoint
    ├── Create retryablehttp.Request with context
    ├── Execute client.Do(req)
    ├── Check status code == 200
    ├── Read body: io.ReadAll(resp.Body)
    └── Return bytes
```

### Worker Pool Integration

```
Async Operations:
    │
    ├── ExecuteAsync(ctx, func)
    │   └── Returns *AsyncResult[T]
    │
    ├── Result channels:
    │   ├── Ch: Receives result value
    │   └── ErrCh: Receives error
    │
    └── Usage: select on Ch and ErrCh
```

### Status Collection

```
GetStatus():
    │
    ├── Check if manager is nil
    ├── GetMachineStatus() for basic status
    ├── GetAxisPositions() for axis count
    ├── GetToolInfo() for current tool
    ├── GetProductionData() for parts info
    └── Return map with all collected data
```

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.