# ADB Manager - Android Debug Bridge

## Overview

The `ADBManager` is a comprehensive Go library for managing Android Debug Bridge (ADB) interactions, providing rich device automation features. It enables programmatic control of Android devices through ADB commands, supporting device management, screen interaction, package management, file operations, system monitoring, and logcat streaming.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Device Management**: List, connect, disconnect, and monitor Android devices
- **Screen Interaction**: Capture screenshots, simulate taps, swipes, and key events
- **Package Management**: Install, uninstall, and query Android packages
- **File Operations**: Push, pull, and manage files on device
- **System Monitoring**: Battery, memory, network, and process information
- **Logcat Integration**: Stream and filter system logs
- **Async Operations**: Worker pool-based asynchronous execution
- **Context Support**: Full `context.Context` support for cancellation and timeouts

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
    
    // Create ADB manager (configuration via viper)
    manager, err := adb.NewADBManager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // List connected devices
    devices, err := manager.ListDevices(ctx)
    if err != nil {
        panic(err)
    }
    
    for _, device := range devices {
        fmt.Printf("Device: %s - State: %s\n", device.Serial, device.State)
    }
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `ADBManager` | Main manager handling ADB interactions with worker pool |
| `ADBDevice` | Represents a connected Android device with properties |
| `ADBPackage` | Represents an installed Android package |
| `ADBScreenInfo` | Device screen resolution and density information |
| `ADBLogEntry` | Single logcat entry with timestamp and metadata |
| `ADBMemoryInfo` | Device memory (RAM and storage) information |
| `ADBBatteryInfo` | Battery status, health, and temperature |
| `ADBAppInfo` | Detailed application information |
| `ADBFileEntry` | File or directory entry in device filesystem |
| `ADBProcess` | Running process information |
| `ADBNetworkInfo` | Network interface information |
| `LogcatOptions` | Configuration for logcat streaming |

### Concurrency Model

```
┌─────────────────────────────────────────────────────────────┐
│                      ADBManager                            │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐      ┌──────────────┐                    │
│  │  WorkerPool │◄─────│ AsyncResult │                    │
│  │  (6 workers)│      │   Channel   │                    │
│  └─────────────┘      └──────────────┘                    │
│         ▲                      │                            │
│         │                      ▼                            │
│  ┌─────────────┐      ┌──────────────┐                   │
│  │  SubmitJob  │      │ ExecuteAsync │                   │
│  └─────────────┘      └──────────────┘                   │
└─────────────────────────────────────────────────────────────┘
```

The manager uses a worker pool with 6 goroutines for asynchronous operations. Async methods return `*AsyncResult[T]` which provides a channel-based API for receiving results.

## How It Works

### 1. Initialization Flow

```
NewADBManager()
    │
    ├── Check viper config: "adb.enabled"
    ├── Get ADB path from: "adb.path" (default: "adb")
    ├── Verify ADB: exec "adb version"
    ├── Create WorkerPool(6)
    └── Return ADBManager
```

### 2. Command Execution Flow

```
runADB(ctx, serial, args...)
    │
    ├── Build command: [adb, -s, serial, args...]
    ├── Create exec.CommandContext(ctx, adbPath, cmdArgs...)
    ├── Execute cmd.CombinedOutput()
    ├── If error: return formatted error with output
    └── Return output bytes
```

### 3. Device Listing Example

```
ListDevices(ctx)
    │
    ├── runADB(ctx, "", "devices", "-l")
    ├── Parse output with bufio.Scanner
    ├── Skip header line
    ├── For each line:
    │   ├── Split by fields
    │   ├── Extract serial, state
    │   ├── Check if emulator (prefix "emulator-")
    │   └── Parse properties: product, model, device, transport_id
    └── Return []ADBDevice
```

### 4. Async Operation Pattern

```
ListDevicesAsync(ctx)
    │
    └── ExecuteAsync(ctx, func(ctx) {
            return a.ListDevices(ctx)
        })
            │
            ├── Submit to WorkerPool
            └── Return *AsyncResult[[]ADBDevice]
                    │
                    └── Result via channel: <-result.Ch
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `adb.enabled` | bool | false | Enable/disable ADB manager |
| `adb.path` | string | "adb" | Path to ADB executable |

### Environment Variables

The ADB manager does not directly use environment variables. Configuration is read through Viper, which can load from:
- Configuration files (JSON, YAML, TOML)
- Environment variables (if Viper is configured for it)
- Command-line flags

## Usage Examples

### Basic Device Operations

```go
ctx := context.Background()

// Get device state
state, err := manager.GetDeviceState(ctx, "emulator-5554")

// Get detailed device info with all properties
device, err := manager.GetDetailedDevice(ctx, "emulator-5554")

// Reboot device
err := manager.RebootDevice(ctx, "emulator-5554")

// Reboot to bootloader
err := manager.RebootToBootloader(ctx, "emulator-5554")

// Wait for device to be available
err := manager.WaitForDevice(ctx, "emulator-5554")
```

### Screen Interaction

```go
// Get screen information
screen, err := manager.GetScreenInfo(ctx, serial)
// screen.Width, screen.Height, screen.Density

// Capture screenshot (returns PNG bytes)
pngBytes, err := manager.Screenshot(ctx, serial)

// Save screenshot to file
err := manager.SaveScreenshot(ctx, serial, "/path/to/screenshot.png")

// Simulate tap
err := manager.Tap(ctx, serial, 500, 300)

// Swipe gesture
err := manager.Swipe(ctx, serial, 100, 500, 100, 200, 300) // 300ms duration

// Long press (1000ms default)
err := manager.LongPress(ctx, serial, 500, 300, 1000)

// Send text input
err := manager.SendText(ctx, serial, "Hello World")

// Key events
err := manager.PressHome(ctx, serial)    // Home button
err := manager.PressBack(ctx, serial)    // Back button
err := manager.PressPower(ctx, serial)   // Power button
```

### Package Management

```go
// List all packages
packages, err := manager.ListPackages(ctx, serial, false)

// List only system packages
systemPkgs, err := manager.ListPackages(ctx, serial, true)

// Get detailed package info
appInfo, err := manager.GetPackageInfo(ctx, serial, "com.example.app")
// appInfo.VersionName, appInfo.VersionCode, appInfo.APKPath

// Install APK
err := manager.InstallAPK(ctx, serial, "/path/to/app.apk")

// Uninstall package
err := manager.UninstallPackage(ctx, serial, "com.example.app")

// Clear app data
err := manager.ClearAppData(ctx, serial, "com.example.app")

// Force stop app
err := manager.ForceStop(ctx, serial, "com.example.app")

// Start app
err := manager.StartApp(ctx, serial, "com.example.app", ".MainActivity")
// Or start by package (launches default activity)
err := manager.StartAppByPackage(ctx, serial, "com.example.app")
```

### File Operations

```go
// Push file to device
err := manager.PushFile(ctx, serial, "/local/path/file.txt", "/sdcard/file.txt")

// Pull file from device
err := manager.PullFile(ctx, serial, "/sdcard/file.txt", "/local/path/file.txt")

// List remote directory
entries, err := manager.ListRemoteDirectory(ctx, serial, "/sdcard")
for _, entry := range entries {
    fmt.Printf("%s %s (size: %d)\n", entry.Permissions, entry.Name, entry.Size)
}

// Create directory
err := manager.MakeRemoteDirectory(ctx, serial, "/sdcard/newdir", true) // parents=true

// Remove file/directory
err := manager.RemoveRemoteFile(ctx, serial, "/sdcard/oldfile.txt", false) // recursive=false
```

### System Information

```go
// Memory information
memInfo, err := manager.GetMemoryInfo(ctx, serial)
fmt.Printf("RAM: %d KB total, %d KB free\n", memInfo.TotalRAM, memInfo.FreeRAM)
fmt.Printf("Storage: %d KB total, %d KB free\n", memInfo.TotalStorage, memInfo.FreeStorage)

// Battery information
battery, err := manager.GetBatteryInfo(ctx, serial)
fmt.Printf("Level: %d%%, Temperature: %.1f°C\n", battery.Level, battery.Temperature)

// Network interfaces
networks, err := manager.GetNetworkInfo(ctx, serial)
for _, net := range networks {
    fmt.Printf("Interface: %s, IPs: %v\n", net.Interface, net.IPAddresses)
}

// Running processes
processes, err := manager.GetRunningProcesses(ctx, serial)
for _, proc := range processes {
    fmt.Printf("PID: %d, Name: %s, User: %s\n", proc.PID, proc.Name, proc.User)
}
```

### Logcat Streaming

```go
// Get logcat entries (dump)
opts := adb.LogcatOptions{
    Priority: "W",       // Warning and above
    Tag:      "MyApp",   // Filter by tag
    MaxLines: 100,       // Limit output
    Buffer:   "main",    // main, events, radio, system
}
entries, err := manager.GetLogcat(ctx, serial, opts)

// Stream logcat (real-time)
logChan := make(chan adb.ADBLogEntry)
go func() {
    err := manager.StreamLogcat(ctx, serial, opts, logChan)
    if err != nil {
        fmt.Printf("Stream error: %v\n", err)
    }
}()

for entry := range logChan {
    fmt.Printf("[%s] %s: %s\n", entry.Timestamp, entry.Tag, entry.Message)
}
```

### Async Operations

```go
// Async device listing
result := manager.ListDevicesAsync(ctx)
// Result type: *AsyncResult[[]ADBDevice]

select {
case <-result.Ctx.Done():
    fmt.Println("Operation cancelled")
case devices := <-result.Ch:
    fmt.Printf("Found %d devices\n", len(devices))
}

// Async screenshot
screenResult := manager.ScreenshotAsync(ctx, serial)
select {
case pngBytes := <-screenResult.Ch:
    // Save to file
    os.WriteFile("screenshot.png", pngBytes, 0644)
}
```

### Port Forwarding

```go
// Forward local port to device port
err := manager.ForwardPort(ctx, serial, 8080, 80) // localhost:8080 -> device:80

// Remove port forward
err := manager.RemovePortForward(ctx, serial, 8080)

// Reverse port (device to host)
err := manager.ReversePort(ctx, serial, 8080, 8080) // device:8080 -> host:8080
```

## Dependencies

| Dependency | Role |
|------------|------|
| `github.com/spf13/viper` | Configuration management for ADB path and enable/disable |
| `stackyrd/config` | Internal configuration package |
| `stackyrd/pkg/logger` | Structured logging with Error, Info, Debug, Warn levels |
| Standard library | `context`, `os/exec`, `bufio`, `regexp`, `strconv`, `time` |

## Error Handling

The library uses Go error handling patterns:

- **Command failures**: `runADB` returns errors with combined stdout/stderr output
- **Context cancellation**: All operations respect `context.Context` for timeouts
- **Nil checks**: Public methods handle nil receiver gracefully in `GetStatus()`
- **Validation**: Image paths and inputs are validated before operations

Example error handling:

```go
devices, err := manager.ListDevices(ctx)
if err != nil {
    // err contains: "adb command failed: <original error> (output: <stderr>)"
    return fmt.Errorf("failed to list devices: %w", err)
}
```

## Common Pitfalls

### 1. ADB Not Found

**Problem**: `adb not found at adb: exec: "adb": executable file not found`

**Solution**: 
- Ensure ADB is installed and in PATH
- Or configure custom path: `viper.Set("adb.path", "/path/to/adb")`

### 2. Device Not Authorized

**Problem**: Device shows as "unauthorized" in `ListDevices()`

**Solution**: 
- Check device screen for authorization dialog
- Revoke USB debugging authorizations and reconnect
- Ensure USB debugging is enabled

### 3. Context Timeout

**Problem**: Operations hang or timeout

**Solution**:
```go
ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()
// Now operations will timeout after 10 seconds
```

### 4. Screenshot Returns Empty

**Problem**: `Screenshot()` returns nil or empty bytes

**Solution**:
- Ensure device screen is on: `manager.IsScreenOn(ctx, serial)`
- Try `manager.PressPower(ctx, serial)` to wake device
- Some devices require `exec-out` which may not work on older Android versions

### 5. Async Operations Not Completing

**Problem**: Async methods seem to hang

**Solution**:
- Check if worker pool is started (happens in `NewADBManager`)
- Use `select` with context or timeout when reading from result channels
- Call `manager.Close()` to clean up worker pool

### 6. Logcat Parsing Failures

**Problem**: Some logcat lines not parsed correctly

**Solution**:
- The parser expects format: `MM-DD HH:MM:SS.mmm PID TID P TAG: message`
- Use `LogcatOptions.Clear = true` to clear buffer before reading
- For streaming, handle partial reads in your processing loop

## Advanced Usage

### Integration with Custom Worker Pools

```go
// Submit custom jobs to the manager's worker pool
manager.SubmitAsyncJob(func() {
    // Custom background work
    ctx := context.Background()
    manager.InstallAPK(ctx, serial, "/path/to/app.apk")
})
```

### Batch Operations

```go
// Install multiple APKs concurrently
apks := []string{"app1.apk", "app2.apk", "app3.apk"}
for _, apk := range apks {
    apk := apk // capture loop variable
    manager.SubmitAsyncJob(func() {
        ctx := context.Background()
        manager.InstallAPK(ctx, serial, apk)
    })
}
```

### Device Monitoring Loop

```go
func monitorDevice(ctx context.Context, manager *adb.ADBManager, serial string) {
    ticker := time.NewTicker(5 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            battery, err := manager.GetBatteryInfo(ctx, serial)
            if err != nil {
                fmt.Printf("Error: %v\n", err)
                continue
            }
            fmt.Printf("Battery: %d%%\n", battery.Level)
        }
    }
}
```

## Internal Algorithms

### Logcat Parsing

The logcat parser uses regex to extract structured data:

```regex
^(\d{2}-\d{2}\s+\d{2}:\d{2}:\d{2}\.\d{3})\s+(\d+)\s+(\d+)\s+([VDIWEF])\s+(.+?)\s*:\s*(.*)$
```

**Groups**:
1. Timestamp (MM-DD HH:MM:SS.mmm)
2. PID
3. TID
4. Priority (V/D/I/W/E/F)
5. Tag
6. Message

### Device Property Parsing

Uses regex `\[(.+?)\]: \[(.+?)\]` to parse `getprop` output:
```
[ro.product.model]: [Pixel 6]
```

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.