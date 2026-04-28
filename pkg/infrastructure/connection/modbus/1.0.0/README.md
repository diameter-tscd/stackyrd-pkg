# Modbus Manager

## Overview

The `ModbusManager` is a Go library for managing Modbus TCP client connections using the goburrow/modbus library. It provides reading and writing of coils, discrete inputs, holding registers, and input registers, along with support for 32-bit float values and async operations via a worker pool.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Coil Operations**: Read multiple coils, write single/multiple coils
- **Discrete Inputs**: Read multiple discrete inputs
- **Holding Registers**: Read/write single/multiple holding registers
- **Input Registers**: Read multiple input registers
- **Float32 Support**: Read/write 32-bit float values (IEEE 754) using two consecutive registers
- **Async Operations**: Async read/write operations via worker pool
- **Worker Pool**: Async job execution support (4 workers)
- **Connection Test**: Validates connectivity via reading input register
- **Configurable**: TCP address, port, unit ID, timeouts

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
    
    // Create Modbus manager (configuration via viper)
    manager, err := modbus.NewModbusManager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // Read holding registers
    values, err := manager.ReadHoldingRegisters(0, 10)
    if err != nil {
        panic(err)
    }
    fmt.Printf("Register values: %v\n", values)
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `ModbusManager` | Main manager with Modbus client, handler, address, port, unit ID |
| `ModbusRegister` | Register with address and 16-bit value |
| `ModbusCoil` | Coil with address and boolean value |

### Concurrency Model

```
┌─────────────────────────────────────────────────────────────┐
│                   ModbusManager                          │
├─────────────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │  Modbus    │  → goburrow/modbus TCP client        │
│  │  Client    │  → Coils, registers, floats          │
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
│  Address: Modbus device IP (e.g., 192.168.1.100)       │
│  Port: TCP port (default: 502)                          │
│  UnitID: Slave/unit ID (default: 1)                    │
└─────────────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewModbusManager(logger)
    │
    ├── Check viper config: "modbus.enabled"
    ├── Get address: "modbus.address"
    ├── Get port: "modbus.port"
    ├── Get unitID: "modbus.unit_id"
    │
    ├── Create TCPClientHandler: modbus.NewTCPClientHandler(address:port)
    ├── Set handler.SlaveId = byte(unitID)
    ├── Set handler.Timeout = 10s, IdleTimeout = 60s
    │
    ├── Create Client: modbus.NewClient(handler)
    ├── Test connection: client.ReadInputRegisters(0, 1)
    ├── Set connected = true
    ├── Create WorkerPool(4)
    ├── Start worker pool
    └── Return ModbusManager
```

### 2. Read Coils Flow

```
ReadCoils(address, quantity)
    │
    ├── client.ReadCoils(address, quantity) → results []byte
    ├── For i := 0 to quantity-1:
    │   ├── byteIndex = i / 8
    │   ├── bitIndex = i % 8
    │   └── values[i] = (results[byteIndex] & (1 << bitIndex)) != 0
    └── Return []bool
```

### 3. Read Holding Registers Flow

```
ReadHoldingRegisters(address, quantity)
    │
    ├── client.ReadHoldingRegisters(address, quantity) → results []byte
    ├── For i := 0 to quantity-1:
    │   └── values[i] = uint16(results[i*2])<<8 | uint16(results[i*2+1])
    └── Return []uint16
```

### 4. Write Single Coil Flow

```
WriteSingleCoil(address, value)
    │
    ├── val = 0xFF00 if value else 0x0000
    └── client.WriteSingleCoil(address, val)
```

### 5. Write Multiple Registers Flow

```
WriteMultipleRegisters(address, values)
    │
    ├── data = make([]byte, quantity*2)
    ├── For i, val := range values:
    │   ├── data[i*2] = byte(val >> 8)
    │   └── data[i*2+1] = byte(val & 0xFF)
    └── client.WriteMultipleRegisters(address, quantity, data)
```

### 6. Read Float32 Flow

```
ReadFloat32(address)
    │
    ├── ReadHoldingRegisters(address, 2) → regs []uint16
    ├── bits = uint32(regs[0])<<16 | uint32(regs[1])
    └── return *(*float32)(unsafe.Pointer(&bits))
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `modbus.enabled` | bool | false | Enable/disable Modbus manager |
| `modbus.address` | string | "" | Modbus device IP address |
| `modbus.port` | int | 502 | TCP port (default: 502) |
| `modbus.unit_id` | int | 1 | Slave/unit ID |

### Environment Variables

The Modbus manager does not directly use environment variables. Configuration is read through Viper, which can load from:
- Configuration files (JSON, YAML, TOML)
- Environment variables (if Viper is configured for it)

You can set Modbus parameters via:
```go
viper.Set("modbus.enabled", true)
viper.Set("modbus.address", "192.168.1.100")
viper.Set("modbus.port", 502)
viper.Set("modbus.unit_id", 1)
```

## Usage Examples

### Reading Coils

```go
ctx := context.Background()

// Read 10 coils starting at address 0
coils, err := manager.ReadCoils(0, 10)
if err != nil {
    panic(err)
}
for i, val := range coils {
    fmt.Printf("Coil %d: %v\n", i, val)
}
```

### Reading Discrete Inputs

```go
ctx := context.Background()

// Read 8 discrete inputs starting at address 0
inputs, err := manager.ReadDiscreteInputs(0, 8)
if err != nil {
    panic(err)
}
for i, val := range inputs {
    fmt.Printf("Input %d: %v\n", i, val)
}
```

### Reading Holding Registers

```go
ctx := context.Background()

// Read 5 holding registers starting at address 100
regs, err := manager.ReadHoldingRegisters(100, 5)
if err != nil {
    panic(err)
}
for i, val := range regs {
    fmt.Printf("Register %d: %d (0x%04X)\n", 100+i, val, val)
}
```

### Reading Input Registers

```go
ctx := context.Background()

// Read 4 input registers starting at address 300
inputs, err := manager.ReadInputRegisters(300, 4)
if err != nil {
    panic(err)
}
for i, val := range inputs {
    fmt.Printf("Input Register %d: %d\n", 300+i, val)
}
```

### Writing Single Coil

```go
ctx := context.Background()

// Write coil at address 0 to true (ON)
err := manager.WriteSingleCoil(0, true)
if err != nil {
    panic(err)
}
fmt.Println("Coil written!")
```

### Writing Single Register

```go
ctx := context.Background()

// Write value 12345 to holding register at address 10
err := manager.WriteSingleRegister(10, 12345)
if err != nil {
    panic(err)
}
fmt.Println("Register written!")
```

### Writing Multiple Coils

```go
ctx := context.Background()

// Write multiple coils
values := []bool{true, false, true, true, false}
err := manager.WriteMultipleCoils(0, values)
if err != nil {
    panic(err)
}
fmt.Println("Multiple coils written!")
```

### Writing Multiple Registers

```go
ctx := context.Background()

// Write multiple registers
values := []uint16{100, 200, 300, 400}
err := manager.WriteMultipleRegisters(0, values)
if err != nil {
    panic(err)
}
fmt.Println("Multiple registers written!")
```

### Reading Float32

```go
ctx := context.Background()

// Read a 32-bit float from registers 50 and 51
floatVal, err := manager.ReadFloat32(50)
if err != nil {
    panic(err)
}
fmt.Printf("Float value: %f\n", floatVal)
```

### Writing Float32

```go
ctx := context.Background()

// Write a 32-bit float to registers 50 and 51
err := manager.WriteFloat32(50, 3.14159)
if err != nil {
    panic(err)
}
fmt.Println("Float written!")
```

### Async Operations

```go
ctx := context.Background()

// Async read coils
result := manager.ReadCoilsAsync(ctx, 0, 10)
select {
case coils := <-result.Ch:
    fmt.Printf("Async coils: %v\n", coils)
case err := <-result.ErrCh:
    fmt.Printf("Error: %v\n", err)
}

// Async read holding registers
result2 := manager.ReadHoldingRegistersAsync(ctx, 100, 5)
select {
case regs := <-result2.Ch:
    fmt.Printf("Async registers: %v\n", regs)
case err := <-result2.ErrCh:
    fmt.Printf("Error: %v\n", err)
}

// Async read input registers
result3 := manager.ReadInputRegistersAsync(ctx, 300, 4)
select {
case regs := <-result3.Ch:
    fmt.Printf("Async input regs: %v\n", regs)
case err := <-result3.ErrCh:
    fmt.Printf("Error: %v\n", err)
}

// Async write single coil
result4 := manager.WriteSingleCoilAsync(ctx, 0, true)
select {
case <-result4.Ch:
    fmt.Println("Async coil write complete!")
case err := <-result4.ErrCh:
    fmt.Printf("Error: %v\n", err)
}

// Async write single register
result5 := manager.WriteSingleRegisterAsync(ctx, 10, 999)
select {
case <-result5.Ch:
    fmt.Println("Async register write complete!")
case err := <-result5.ErrCh:
    fmt.Printf("Error: %v\n", err)
}
```

### Get Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
fmt.Printf("Address: %s\n", status["address"])
fmt.Printf("Port: %v\n", status["port"])
fmt.Printf("Unit ID: %v\n", status["unit_id"])

if status["connected"] == false {
    if errMsg, ok := status["error"].(string); ok {
        fmt.Printf("Error: %s\n", errMsg)
    }
}
```

## Dependencies

| Dependency | Role |
|------------|------|
| `github.com/goburrow/modbus` | Modbus client library |
| `github.com/spf13/viper` | Configuration management |
| `stackyrd/config` | Internal configuration package |
| `stackyrd/pkg/logger` | Structured logging |
| Standard library | `context`, `fmt`, `unsafe`, `time` |

## Error Handling

The library uses Go error handling patterns:

- **Connection failures**: `testConnection()` validates via `ReadInputRegisters()`
- **Read/write errors**: Returned from Modbus client operations
- **Context cancellation**: Async operations respect `context.Context`
- **Nil checks**: Public methods handle nil receiver gracefully in `GetStatus()`

Example error handling:

```go
regs, err := manager.ReadHoldingRegisters(address, quantity)
if err != nil {
    if strings.Contains(err.Error(), "connection refused") {
        return fmt.Errorf("Modbus device not reachable: %w", err)
    }
    if strings.Contains(err.Error(), "timeout") {
        return fmt.Errorf("Modbus timeout: %w", err)
    }
    return fmt.Errorf("failed to read registers: %w", err)
}
```

## Common Pitfalls

### 1. Modbus Device Not Reachable

**Problem**: `connection refused` or `no route to host` errors

**Solution**: 
- Verify device address: `viper.Set("modbus.address", "192.168.1.100")`
- Check network connectivity (ping the device)
- Verify firewall allows access to port 502

### 2. Wrong Unit ID

**Problem**: `illegal function` or `no response` errors

**Solution**:
- Verify unit ID: `viper.Set("modbus.unit_id", 1)`
- Check device's slave ID configuration
- Try different unit IDs (common: 1, 2, 255)

### 3. Register Address Errors

**Problem**: `illegal data address` errors

**Solution**:
- Verify register addresses are within device's range
- Check if addressing is 0-based or 1-based (Modbus typically 0-based)
- Consult device documentation

### 4. Timeout Issues

**Problem**: Operations timing out

**Solution**:
```go
// The handler timeout is set to 10s by default
// For longer timeout, you'd need to modify the handler creation
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
```

### 5. Float Conversion Issues

**Problem**: Incorrect float values when using ReadFloat32/WriteFloat32

**Solution**:
- Ensure correct register order (some devices use swapped words)
- Verify IEEE 754 single-precision format
- Check endianness (this implementation uses big-endian register order)

### 6. Worker Pool Not Available

**Problem**: Async operations falling back to direct execution

**Solution**:
- The pool is created during initialization
- Check that `NewModbusManager()` succeeded
- Verify `modbus.enabled` is true

## Advanced Usage

### Monitoring Register Values

```go
func monitorRegisters(manager *modbus.ModbusManager, address uint16, count uint16) {
    ticker := time.NewTicker(5 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:
            regs, err := manager.ReadHoldingRegisters(address, count)
            if err != nil {
                fmt.Printf("Error reading registers: %v\n", err)
                continue
            }
            fmt.Printf("Registers %d-%d: %v\n", address, address+count-1, regs)
        }
    }
}
```

### Writing Multiple Values with Validation

```go
func writeAndVerify(manager *modbus.ModbusManager, address uint16, values []uint16) error {
    // Write values
    err := manager.WriteMultipleRegisters(address, values)
    if err != nil {
        return fmt.Errorf("write failed: %w", err)
    }
    
    // Read back and verify
    readValues, err := manager.ReadHoldingRegisters(address, uint16(len(values)))
    if err != nil {
        return fmt.Errorf("read back failed: %w", err)
    }
    
    for i, val := range values {
        if readValues[i] != val {
            return fmt.Errorf("verification failed at index %d: wrote %d, read %d", i, val, readValues[i])
        }
    }
    
    fmt.Println("Write verified successfully!")
    return nil
}
```

### Reading Device Status

```go
func checkDeviceStatus(manager *modbus.ModbusManager) {
    // Read some status registers (example addresses)
    statusRegs, err := manager.ReadInputRegisters(0, 4)
    if err != nil {
        fmt.Printf("Error reading status: %v\n", err)
        return
    }
    
    fmt.Println("Device Status:")
    fmt.Printf("  Status Register 0: %d\n", statusRegs[0])
    fmt.Printf("  Status Register 1: %d\n", statusRegs[1])
    fmt.Printf("  Status Register 2: %d\n", statusRegs[2])
    fmt.Printf("  Status Register 3: %d\n", statusRegs[3])
}
```

### Batch Reading Different Register Types

```go
func readAllData(manager *modbus.ModbusManager) {
    ctx := context.Background()
    
    // Async read coils
    coilsCh := manager.ReadCoilsAsync(ctx, 0, 8)
    // Async read holding registers
    regsCh := manager.ReadHoldingRegistersAsync(ctx, 0, 10)
    // Async read input registers
    inputsCh := manager.ReadInputRegistersAsync(ctx, 0, 5)
    
    // Collect results
    select {
    case coils := <-coilsCh.Ch:
        fmt.Printf("Coils: %v\n", coils)
    case err := <-coilsCh.ErrCh:
        fmt.Printf("Coils error: %v\n", err)
    }
    
    select {
    case regs := <-regsCh.Ch:
        fmt.Printf("Holding Registers: %v\n", regs)
    case err := <-regsCh.ErrCh:
        fmt.Printf("Registers error: %v\n", err)
    }
    
    select {
    case inputs := <-inputsCh.Ch:
        fmt.Printf("Input Registers: %v\n", inputs)
    case err := <-inputsCh.ErrCh:
        fmt.Printf("Inputs error: %v\n", err)
    }
}
```

## Internal Algorithms

### Modbus TCP Frame Format

```
ReadHoldingRegisters(address, quantity):
    │
    ├── TCP MBAP Header (7 bytes) + PDU
    ├── PDU: Function Code (0x03) + Start Address (2 bytes) + Quantity (2 bytes)
    ├── Response: Function Code + Byte Count + Register Values (quantity * 2 bytes)
    └── Parse: each register = 2 bytes, big-endian
```

### Float32 Conversion

```
ReadFloat32(address):
    │
    ├── ReadHoldingRegisters(address, 2) → regs[0], regs[1]
    ├── bits = (uint32(regs[0]) << 16) | uint32(regs[1])
    └── return *(*float32)(unsafe.Pointer(&bits))
```

### Coil Bit Unpacking

```
ReadCoils(address, quantity):
    │
    ├── Read coils → results []byte (1 byte per 8 coils)
    ├── For each coil i:
    │   ├── byteIndex = i / 8
    │   ├── bitIndex = i % 8
    │   └── value = (results[byteIndex] & (1 << bitIndex)) != 0
    └── Return []bool
```

### Async Operation Pattern

```
ReadCoilsAsync(ctx, address, quantity):
    │
    └── ExecuteAsync(ctx, func):
        └── return ReadCoils(address, quantity)
            └── Returns ([]bool, error)
```

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.