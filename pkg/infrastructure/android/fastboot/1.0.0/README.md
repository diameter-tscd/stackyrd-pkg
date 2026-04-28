# Fastboot Manager - Android Bootloader Interface

## Overview

The `FastbootManager` is a comprehensive Go library for managing Android Fastboot protocol interactions, providing rich device flashing and bootloader features. It enables programmatic control of Android devices in fastboot mode, supporting partition flashing, bootloader unlock/lock, A/B slot management, image validation, and OTA sideloading.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Device Management**: List and monitor devices in fastboot mode
- **Partition Flashing**: Flash boot, system, recovery, vendor, and custom partitions
- **A/B Slot Management**: Query and switch between A/B slots
- **Bootloader Operations**: Unlock/lock bootloader with OEM commands
- **Image Validation**: Validate sparse and raw images with CRC32 checks
- **Flash Progress Tracking**: Real-time progress monitoring for flash operations
- **Logical Partitions**: Create, delete, resize, and map dynamic partitions
- **OTA Sideloading**: Flash OTA zip packages and update super partition
- **Async Operations**: Worker pool-based asynchronous execution
- **Image Conversion**: Convert sparse images to raw format

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
    
    // Create Fastboot manager (configuration via viper)
    manager, err := fastboot.NewFastbootManager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // List devices in fastboot mode
    devices, err := manager.ListDevices(ctx)
    if err != nil {
        panic(err)
    }
    
    for _, device := range devices {
        fmt.Printf("Device: %s - Product: %s\n", device.Serial, device.Product)
    }
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `FastbootManager` | Main manager handling fastboot interactions with worker pool |
| `FastbootDevice` | Represents a device in fastboot mode with bootloader variables |
| `FastbootPartition` | Represents a partition with slot and logical partition info |
| `FlashProgress` | Tracks progress of flash operations (bytes, speed, status) |
| `FastbootSlot` | Represents an A/B slot with bootable/successful status |
| `FastbootFlashRequest` | Flash operation request with partition and image details |
| `FastbootFormatRequest` | Format operation request with filesystem type |
| `FastbootOEMCommand` | OEM-specific command with arguments |
| `FastbootImageInfo` | Metadata about flashable images (sparse, CRC32, size) |
| `FastbootBootConfig` | Boot image configuration (addresses, page size) |
| `SparseHeader` | Header information for Android sparse images |

### Concurrency Model

```
┌─────────────────────────────────────────────────────────────┐
│                   FastbootManager                           │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐      ┌──────────────┐                   │
│  │  WorkerPool │◄─────│ AsyncResult │                   │
│  │  (4 workers)│      │   Channel   │                   │
│  └─────────────┘      └──────────────┘                   │
│         ▲                      │                           │
│         │                      ▼                           │
│  ┌─────────────┐      ┌──────────────┐                  │
│  │  SubmitJob  │      │ ExecuteAsync │                  │
│  └─────────────┘      └──────────────┘                  │
│                                                         │
│  ┌─────────────────────────────────────────────┐          │
│  │       flashProgress (sync.Map)              │          │
│  │  Tracks: serial:partition → FlashProgress │          │
│  └─────────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────────┘
```

The manager uses a worker pool with 4 goroutines for asynchronous operations. Flash progress is tracked using a `sync.RWMutex`-protected map for concurrent access.

## How It Works

### 1. Initialization Flow

```
NewFastbootManager()
    │
    ├── Check viper config: "fastboot.enabled"
    ├── Get fastboot path from: "fastboot.path" (default: "fastboot")
    ├── Verify fastboot: exec "fastboot --version"
    ├── Create WorkerPool(4)
    ├── Initialize flashProgress map
    └── Return FastbootManager
```

### 2. Command Execution Flow

```
runFastboot(ctx, serial, args...)
    │
    ├── Build command: [fastboot, -s, serial, args...]
    ├── Create exec.CommandContext(ctx, fastbootPath, cmdArgs...)
    ├── Execute cmd.CombinedOutput()
    ├── If error: return formatted error with output
    └── Return output bytes
```

### 3. Flashing Flow with Progress Tracking

```
FlashPartition(ctx, serial, partition, imagePath)
    │
    ├── ValidateImage(path) → FastbootImageInfo
    │   ├── os.Stat() for size
    │   ├── Read magic bytes (0xED26FF3A = sparse)
    │   ├── Calculate CRC32
    │   └── Return FastbootImageInfo
    │
    ├── Create FlashProgress struct
    │   ├── Status: "flashing"
    │   ├── StartTime: now
    │   └── BytesTotal: image size
    │
    ├── Store in flashProgress map (with mutex)
    │
    ├── Execute: fastboot flash <partition> <imagePath>
    │
    ├── Update FlashProgress
    │   ├── Status: "complete" or "error"
    │   ├── EndTime: now
    │   ├── BytesWritten, Percent, SpeedMBps
    │   └── Store updated progress
    │
    └── Return error or nil
```

### 4. A/B Slot Management Flow

```
GetSlots(ctx, serial)
    │
    ├── GetDeviceInfo() → check HasSlotAB
    ├── If no A/B: return empty slice
    │
    ├── For each slot (a, b, ...):
    │   ├── Query: getvar slot-bootable:<suffix>
    │   ├── Query: getvar slot-successful:<suffix>
    │   ├── Query: getvar slot-retry-count:<suffix>
    │   ├── Query: getvar slot-unbootable:<suffix>
    │   ├── Check if current-slot matches
    │   └── Append FastbootSlot to result
    │
    └── Return []FastbootSlot
```

### 5. Async Operation Pattern

```
FlashPartitionAsync(ctx, serial, partition, imagePath)
    │
    └── ExecuteAsync(ctx, func(ctx) {
            err := f.FlashPartition(ctx, serial, partition, imagePath)
            return struct{}{}, err
        })
            │
            ├── Submit to WorkerPool
            └── Return *AsyncResult[struct{}]
                    │
                    └── Result via channel: <-result.Ch
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `fastboot.enabled` | bool | false | Enable/disable Fastboot manager |
| `fastboot.path` | string | "fastboot" | Path to fastboot executable |

### Environment Variables

The Fastboot manager does not directly use environment variables. Configuration is read through Viper, which can load from:
- Configuration files (JSON, YAML, TOML)
- Environment variables (if Viper is configured for it)
- Command-line flags

## Usage Examples

### Basic Device Operations

```go
ctx := context.Background()

// List devices in fastboot mode
devices, err := manager.ListDevices(ctx)

// Get detailed device info (all bootloader variables)
device, err := manager.GetDeviceInfo(ctx, "1234567890")
fmt.Printf("Product: %s, Bootloader: %s\n", device.Product, device.Bootloader)
fmt.Printf("Unlocked: %v, Has A/B: %v\n", device.IsUnlocked, device.HasSlotAB)

// Get a single variable
value, err := manager.GetVariable(ctx, serial, "current-slot")

// Wait for device to appear
err := manager.WaitForDevice(ctx, serial)
```

### Flashing Operations

```go
// Flash boot partition
err := manager.FlashBoot(ctx, serial, "/path/to/boot.img")

// Flash recovery
err := manager.FlashRecovery(ctx, serial, "/path/to/recovery.img")

// Flash system
err := manager.FlashSystem(ctx, serial, "/path/to/system.img")

// Flash vendor
err := manager.FlashVendor(ctx, serial, "/path/to/vendor.img")

// Flash DTBO
err := manager.FlashDTBO(ctx, serial, "/path/to/dtbo.img")

// Flash VBMeta with verification disabled
err := manager.FlashVBMeta(ctx, serial, "/path/to/vbmeta.img", true)

// Flash to specific slot (A/B devices)
err := manager.FlashPartitionWithSlot(ctx, serial, "system", "a", "/path/to/system.img")

// Flash to both slots
err := manager.FlashToBothSlots(ctx, serial, "boot", "/path/to/boot.img")

// Flash arbitrary partition
err := manager.FlashPartition(ctx, serial, "modem", "/path/to/modem.img")
```

### Flash All (Factory Image)

```go
// Flash complete factory image with options
err := manager.FlashAll(ctx, serial, "/path/to/factory/images", 
    false,  // skipBootloader
    false,  // skipRadio
    true,   // wipeData
)

// This will flash (if present):
// - bootloader.img
// - radio.img
// - boot.img
// - recovery.img
// - system.img
// - vendor.img
// - dtbo.img
// - vbmeta.img
// And optionally wipe userdata
```

### Bootloader Unlock/Lock

```go
// Check if unlock is allowed
allowed, err := manager.GetUnlockAbility(ctx, serial)

// Unlock bootloader (requires confirmation on device)
err := manager.OEMUnlock(ctx, serial)

// Unlock critical partitions (for bootloader/radio flashing)
err := manager.OEMUnlockCritical(ctx, serial)

// Lock bootloader
err := manager.OEMLock(ctx, serial)

// Get OEM device info
info, err := manager.GetDeviceInfoOEM(ctx, serial)
// Returns map with: Device is unlocked, Device is critical unlocked, etc.

// Execute custom OEM command
output, err := manager.ExecuteOEMCommand(ctx, serial, fastboot.FastbootOEMCommand{
    Command: "poweroff",
})
```

### A/B Slot Management

```go
// Get all slots information
slots, err := manager.GetSlots(ctx, serial)
for _, slot := range slots {
    fmt.Printf("Slot %s: bootable=%v, successful=%v, active=%v\n",
        slot.Suffix, slot.IsBootable, slot.IsSuccessful, slot.IsActive)
}

// Set active slot
err := manager.SetActiveSlot(ctx, serial, "a")

// Alternative method
err := manager.SetActiveSlotDirect(ctx, serial, "b")
```

### Boot Operations

```go
// Boot image temporarily (without flashing)
err := manager.BootImage(ctx, serial, "/path/to/boot.img")

// Boot and continue to normal boot
err := manager.ContinueBoot(ctx, serial)

// Reboot device
err := manager.Reboot(ctx, serial)

// Reboot to bootloader
err := manager.RebootToBootloader(ctx, serial)

// Reboot to recovery
err := manager.RebootToRecovery(ctx, serial)

// Power off device
err := manager.PowerOff(ctx, serial)
```

### Partition Management

```go
// List available partitions
partitions, err := manager.ListPartitions(ctx, serial)
for _, part := range partitions {
    fmt.Printf("Partition: %s (logical=%v, eraseable=%v)\n",
        part.Name, part.IsLogical, part.IsEraseable)
}

// Get partition size
size, err := manager.GetPartitionSize(ctx, serial, "system")

// Erase partition
err := manager.ErasePartition(ctx, serial, "userdata")

// Format partition with filesystem
err := manager.FormatPartition(ctx, serial, "userdata", "ext4")

// Format userdata (factory reset)
err := manager.FormatUserData(ctx, serial)

// Format cache
err := manager.FormatCache(ctx, serial)
```

### Logical Partition Operations (Dynamic Partitions)

```go
// Create a new logical partition in super
err := manager.CreateLogicalPartition(ctx, serial, "my_partition", 100*1024*1024) // 100MB

// Resize logical partition
err := manager.ResizeLogicalPartition(ctx, serial, "my_partition", 200*1024*1024) // 200MB

// Map partition for direct access
path, err := manager.MapPartition(ctx, serial, "my_partition")

// Unmap partition
err := manager.UnmapPartition(ctx, serial, "my_partition")

// Delete logical partition
err := manager.DeleteLogicalPartition(ctx, serial, "my_partition")
```

### Super Partition (Dynamic Partitions)

```go
// Update super partition
err := manager.UpdateSuper(ctx, serial, "/path/to/super.img")

// Wipe and recreate super partition
err := manager.WipeSuper(ctx, serial)
```

### OTA Sideloading

```go
// Sideload OTA zip package
err := manager.Sideload(ctx, serial, "/path/to/ota.zip")

// Sideload and reboot automatically
err := manager.SideloadWithReboot(ctx, serial, "/path/to/ota.zip")
```

### Image Validation

```go
// Validate image file
info, err := manager.ValidateImage("/path/to/image.img")
if err != nil {
    fmt.Printf("Invalid image: %v\n", err)
} else {
    fmt.Printf("Size: %d bytes\n", info.Size)
    fmt.Printf("Is sparse: %v\n", info.IsSparse)
    fmt.Printf("CRC32: %08x\n", info.CRC32)
    fmt.Printf("Valid: %v\n", info.IsValid)
}

// Check if image is sparse
isSparse, err := manager.IsSparseImage("/path/to/image.img")

// Convert sparse to raw
err := manager.ConvertSparseToRaw(ctx, "/path/to/sparse.img", "/path/to/raw.img")
```

### Flash Progress Tracking

```go
// Flash a partition
go func() {
    err := manager.FlashPartition(ctx, serial, "system", "/path/to/system.img")
    if err != nil {
        fmt.Printf("Flash failed: %v\n", err)
    }
}()

// Monitor progress (in another goroutine)
for {
    progress, exists := manager.GetFlashProgress(serial, "system")
    if !exists {
        break
    }
    
    fmt.Printf("Progress: %.1f%% (%d/%d bytes) - %s\n",
        progress.Percent, progress.BytesWritten, progress.BytesTotal, progress.Status)
    
    if progress.Status == "complete" || progress.Status == "error" {
        if progress.Status == "error" {
            fmt.Printf("Error: %s\n", progress.Error)
        }
        if progress.SpeedMBps > 0 {
            fmt.Printf("Speed: %.2f MB/s\n", progress.SpeedMBps)
        }
        break
    }
    
    time.Sleep(500 * time.Millisecond)
}

// Get all active flash operations
allProgress := manager.GetAllFlashProgress()

// Clear completed/errored progress entries
manager.ClearFlashProgress()
```

### Fetch and Flash from Network

```go
// Download image from URL and flash it
err := manager.FetchAndFlash(ctx, serial, "boot", "https://example.com/boot.img")
```

### Backup Partitions

```go
// Dump a single partition to file
err := manager.DumpPartition(ctx, serial, "boot", "/backup/boot.img")

// Backup multiple partitions
partitions := []string{"boot", "recovery", "system", "vendor"}
err := manager.BackupPartitions(ctx, serial, partitions, "/backup/")
```

### Async Operations

```go
// Async flash partition
result := manager.FlashPartitionAsync(ctx, serial, "boot", "/path/to/boot.img")
select {
case <-result.Ctx.Done():
    fmt.Println("Operation cancelled")
case <-result.Ch:
    fmt.Println("Flash complete")
}

// Async flash-all
result := manager.FlashAllAsync(ctx, serial, "/path/to/images", false, false, true)
select {
case <-result.Ch:
    fmt.Println("Flash-all complete")
}

// Async sideload
result := manager.SideloadAsync(ctx, serial, "/path/to/ota.zip")
select {
case <-result.Ch:
    fmt.Println("Sideload complete")
}

// Async slot information
result := manager.GetSlotsAsync(ctx, serial)
select {
case slots := <-result.Ch:
    fmt.Printf("Found %d slots\n", len(slots))
}

// Async image validation
result := manager.ValidateImageAsync(ctx, "/path/to/image.img")
select {
case info := <-result.Ch:
    fmt.Printf("Valid: %v\n", info.IsValid)
}
```

### Reboot Operations (Async)

```go
// Async reboot
result := manager.RebootAsync(ctx, serial)
select {
case <-result.Ch:
    fmt.Println("Reboot complete")
}

// Async reboot to bootloader
result := manager.RebootToBootloaderAsync(ctx, serial)
select {
case <-result.Ch:
    fmt.Println("Rebooted to bootloader")
}

// Async reboot to recovery
result := manager.RebootToRecoveryAsync(ctx, serial)
select {
case <-result.Ch:
    fmt.Println("Rebooted to recovery")
}
```

## Dependencies

| Dependency | Role |
|------------|------|
| `github.com/spf13/viper` | Configuration management for fastboot path and enable/disable |
| `stackyrd/config` | Internal configuration package |
| `stackyrd/pkg/logger` | Structured logging with Error, Info, Debug, Warn levels |
| Standard library | `context`, `os/exec`, `bufio`, `regexp`, `strconv`, `time`, `sync`, `hash/crc32`, `os` |

## Error Handling

The library uses Go error handling patterns:

- **Command failures**: `runFastboot` returns errors with combined stdout/stderr output
- **Context cancellation**: All operations respect `context.Context` for timeouts
- **Nil checks**: Public methods handle nil receiver gracefully in `GetStatus()`
- **Image validation**: Paths and image formats are validated before operations

Example error handling:

```go
err := manager.FlashPartition(ctx, serial, "boot", "/path/to/boot.img")
if err != nil {
    // err contains: "flash failed: fastboot command failed: <original error> (output: <stderr>)"
    return fmt.Errorf("failed to flash boot: %w", err)
}
```

## Common Pitfalls

### 1. Fastboot Not Found

**Problem**: `fastboot not found at fastboot: exec: "fastboot": executable file not found`

**Solution**: 
- Ensure fastboot is installed and in PATH (part of Android SDK Platform Tools)
- Or configure custom path: `viper.Set("fastboot.path", "/path/to/fastboot")`

### 2. Device Not in Fastboot Mode

**Problem**: No devices listed or "no devices" error

**Solution**: 
- Reboot device to bootloader: `adb reboot bootloader`
- Check connection: `fastboot devices` in terminal
- Try different USB port or cable

### 3. Bootloader Unlock Required

**Problem**: Flashing fails with "partition flashing is not allowed"

**Solution**:
- Unlock bootloader: `manager.OEMUnlock(ctx, serial)`
- Confirm on device screen
- May void warranty - check manufacturer policies

### 4. Sparse Image Issues

**Problem**: Flashing sparse image fails

**Solution**:
- Validate image: `info, _ := manager.ValidateImage(path)`
- Convert to raw: `manager.ConvertSparseToRaw(ctx, sparse, raw)`
- Or use fastboot's built-in sparse handling (automatic in modern fastboot)

### 5. A/B Slot Confusion

**Problem**: Flashing to wrong slot, device won't boot

**Solution**:
- Check current slot: `device, _ := manager.GetDeviceInfo(ctx, serial)`
- Flash to both slots: `manager.FlashToBothSlots(ctx, serial, "boot", path)`
- Set active slot: `manager.SetActiveSlot(ctx, serial, "a")`

### 6. Flash Progress Not Updating

**Problem**: `GetFlashProgress()` returns false

**Solution**:
- Ensure flash operation is in progress
- Check if using correct serial:partition key
- Progress is only available during/after `FlashPartition()` call

### 7. Context Timeout

**Problem**: Long flash operations timeout

**Solution**:
```go
ctx, cancel := context.WithTimeout(context.Background(), 10*time.Minute)
defer cancel()
// Flash operations on large partitions can take several minutes
```

### 8. OEM Commands Not Working

**Problem**: OEM commands return errors or unexpected output

**Solution**:
- Check device-specific OEM commands (varies by manufacturer)
- Some commands require unlocked bootloader
- Use `manager.GetDeviceInfoOEM()` to see available options

## Advanced Usage

### Custom Flash Script Execution

```go
// Flash-all with custom script detection
err := manager.FlashAll(ctx, serial, "/path/to/factory/images", false, false, true)
// Automatically detects and executes flash-all.sh or flash-all.bat
```

### Batch Operations with Worker Pool

```go
// Submit custom jobs to the manager's worker pool
manager.SubmitAsyncJob(func() {
    ctx := context.Background()
    manager.FlashPartition(ctx, serial, "boot", "/path/to/boot.img")
})

manager.SubmitAsyncJob(func() {
    ctx := context.Background()
    manager.FlashPartition(ctx, serial, "system", "/path/to/system.img")
})
```

### Device Monitoring with Flash Progress

```go
func monitorFlashProgress(manager *fastboot.FastbootManager, serial, partition string) {
    ticker := time.NewTicker(500 * time.Millisecond)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:
            progress, exists := manager.GetFlashProgress(serial, partition)
            if !exists {
                fmt.Println("Flash operation not found")
                return
            }
            
            fmt.Printf("\rProgress: %.1f%% (%.2f MB/s)", 
                progress.Percent, progress.SpeedMBps)
            
            if progress.Status == "complete" {
                fmt.Println("\nFlash complete!")
                return
            } else if progress.Status == "error" {
                fmt.Printf("\nFlash failed: %s\n", progress.Error)
                return
            }
        }
    }
}
```

### Conditional Flashing Based on Slot Status

```go
func smartFlash(manager *fastboot.FastbootManager, ctx context.Context, serial, partition, imagePath string) error {
    // Get slot information
    slots, err := manager.GetSlots(ctx, serial)
    if err != nil {
        return err
    }
    
    // Flash only active/healthy slots
    for _, slot := range slots {
        if slot.IsActive && slot.IsBootable && slot.IsSuccessful {
            target := partition + "_" + slot.Suffix
            fmt.Printf("Flashing %s (active, healthy)\n", target)
            
            if err := manager.FlashPartition(ctx, serial, target, imagePath); err != nil {
                return err
            }
        } else {
            fmt.Printf("Skipping slot %s (not optimal state)\n", slot.Suffix)
        }
    }
    
    return nil
}
```

### Image Validation Pipeline

```go
func validateBeforeFlash(manager *fastboot.FastbootManager, imagePath string) error {
    info, err := manager.ValidateImage(imagePath)
    if err != nil {
        return fmt.Errorf("validation failed: %w", err)
    }
    
    if !info.IsValid {
        return fmt.Errorf("image is not valid")
    }
    
    if info.IsSparse {
        fmt.Printf("Sparse image detected with %d chunks\n", info.Partitions)
    }
    
    fmt.Printf("Image OK: %d bytes, CRC32: %08x\n", info.Size, info.CRC32)
    return nil
}

// Usage
if err := validateBeforeFlash(manager, "/path/to/image.img"); err != nil {
    panic(err)
}
manager.FlashPartition(ctx, serial, "system", "/path/to/image.img")
```

## Internal Algorithms

### Sparse Image Detection

The sparse image validator reads the first 28 bytes of the image file:

```go
magic := uint32(header[0]) | uint32(header[1])<<8 | uint32(header[2])<<16 | uint32(header[3])<<24
if magic == 0xED26FF3A {
    // Sparse image format
    // Parse: major, minor, header_size, chunk_size, block_size, etc.
}
```

### Flash Progress Calculation

```
Progress Tracking:
    BytesWritten = image file size (set after successful flash)
    Percent = (BytesWritten / BytesTotal) * 100.0
    SpeedMBps = BytesTotal / 1024 / 1024 / duration_seconds
    Status transitions: "flashing" → "complete" or "error"
```

### Bootloader Variable Parsing

The getvar output parser uses regex:

```regex
^\(bootloader\)\s+(\S+):\s*(.*)$
```

**Example input**:
```
(bootloader) product: pixel
(bootloader) version-bootloader: 1234.56
```

**Parsed output**:
```go
map[string]string{
    "product": "pixel",
    "version-bootloader": "1234.56",
}
```

### A/B Slot Query Pattern

```
For each slot suffix (a, b, ...):
    getvar slot-bootable:<suffix>  → "yes" or "no"
    getvar slot-successful:<suffix> → "yes" or "no"
    getvar slot-retry-count:<suffix> → retry count (int)
    getvar slot-unbootable:<suffix> → "yes" or "no"
```

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.