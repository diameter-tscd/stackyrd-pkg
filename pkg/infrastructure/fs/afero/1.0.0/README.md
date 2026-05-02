# Afero Manager#

## Overview#

The `aferoManager` is a Go library that provides a singleton filesystem abstraction using the Afero library. It wraps `embed.FS` (Go's embedded filesystem) and provides a unified interface for both development and production environments. In development mode, it uses `CopyOnWriteFs` to allow local file overrides while keeping embedded files as base. In production mode, it uses `ReadOnlyFs` for security. The library supports alias-based file access, making it easy to reference embedded files by friendly names.#

**Import Path:** `stackyrd/pkg/infrastructure/fs/afero`#

## Features#

- **Singleton Pattern**: Thread-safe initialization using `sync.Once`#
- **Dual Mode Operation**: Development (writable overlay) and Production (read-only)#
- **Alias Mapping**: Map friendly aliases to physical filesystem paths#
- **Embed.FS Support**: Wraps Go's `//go:embed` filesystem#
- **CopyOnWriteFs**: In dev mode, allows local overrides of embedded files#
- **ReadOnlyFs**: In prod mode, ensures embedded files cannot be modified#
- **Streaming Support**: `Stream()` function returns `io.ReadCloser` for efficient file reading#
- **Alias Resolution**: Automatic handling of `all:` prefix for embedded paths#
- **Thread Safety**: Uses `sync.RWMutex` for concurrent access to aliases and filesystem#
- **Testing Support**: `ResetForTesting()` function for test isolation#

## Quick Start#

```go
package main

import (
    "embed"
    "fmt"
    "stackyrd/pkg/infrastructure/fs/afero"
)

//go:embed all:assets/*
var embeddedAssets embed.FS

func main() {
    // Define alias map
    aliases := map[string]string{
        "config": "all:config/app.json",
        "static": "all:static/",
        "data":   "all:data/",
    }
    
    // Initialize in development mode (allows local overrides)
    afero.Init(embeddedAssets, aliases, true)
    
    // Read a file by alias
    data, err := afero.Read("config")
    if err != nil {
        panic(err)
    }
    fmt.Printf("Config: %s\n", string(data))
    
    // Check if alias exists
    if afero.Exists("static") {
        fmt.Println("Static files available")
    }
}
```

## Architecture#

### Core Structs#

| Struct | Description |
|--------|-------------|
| `aferoManager` | Singleton manager holding Afero filesystem and alias map |
| `embedFSWrapper` | Wraps `embed.FS` to implement `afero.Fs` interface |
| `embedFile` | Wraps `fs.File` to implement `afero.File` interface |

### Concurrency Model#

```
┌─────────────────────────────────────────────────────┐
│                   aferoManager                          │
├─────────────────────────────────────────────────────┤
│  ┌────────────────┐                                     │
│  │  afero.Fs  │  → Filesystem (ReadOnly or CopyOnWrite) │
│  └────────────────┘                                     │
│                                                          │
│  ┌─────────────┐                                      │
│  │  aliases    │  → map[string]string (alias→path)       │
│  └─────────────┘                                      │
│         │                                             │
│  ┌─────────────┐                                      │
│  │  sync.RWMutex│  → Thread-safe access                │
│  └─────────────┘                                      │
│                                                          │
│  Initialization: sync.Once (thread-safe singleton)       │
└─────────────────────────────────────────────────────┘
```

## How It Works#

### 1. Initialization Flow#

```
Init(embedFS, aliasMap, isDev)
    │
    ├── sync.Once ensures single initialization
    │
    ├── Create aferoManager with alias map copy
    │
    ├── If isDev == true:
    │   ├── Create embedFSWrapper from embed.FS
    │   ├── Create OS filesystem (writable)
    │   └── Create CopyOnWriteFs(baseFS, writableFS)
    │
    ├── If isDev == false:
    │   ├── Create embedFSWrapper from embed.FS
    │   └── Create ReadOnlyFs(baseFS)
    │
    └── Store as singleton instance
```

### 2. Read Flow#

```
Read(alias)
    │
    ├── Check instance != nil
    ├── Acquire read lock
    ├── Resolve alias to physical path
    │   └── Handle "all:" prefix (strip prefix)
    ├── Call afero.ReadFile(instance.fs, physicalPath)
    └── Return []byte, error
```

### 3. Stream Flow#

```
Stream(alias)
    │
    ├── Check instance != nil
    ├── Acquire read lock
    ├── Resolve alias to physical path
    ├── Call instance.fs.Open(physicalPath)
    └── Return io.ReadCloser, error
```

### 4. Exists Flow#

```
Exists(alias)
    │
    ├── Check instance != nil → return false
    ├── Acquire read lock
    ├── Check alias exists in map → return false if not
    ├── Resolve physical path (strip "all:" prefix)
    ├── Call instance.fs.Stat(physicalPath)
    └── Return err == nil
```

### 5. Alias Resolution#

```
resolveAlias(alias)
    │
    ├── Lookup alias in instance.aliases
    ├── If not found → return error
    ├── If path starts with "all:":
    │   └── Strip "all:" prefix
    └── Return physical path
```

## Configuration#

The Afero manager is configured via the `Init()` function parameters:

| Parameter | Type | Description |
|-----------|------|-------------|
| `embedFS` | `embed.FS` | The embedded filesystem from `//go:embed` directive |
| `aliasMap` | `map[string]string` | Maps aliases to physical paths (supports `all:` prefix) |
| `isDev` | `bool` | If true, use CopyOnWriteFs; if false, use ReadOnlyFs |

### Alias Map Format#

- Regular path: `"config": "config/app.json"` → direct path#
- Embedded path: `"static": "all:static/"` → path within `embed.FS`#

Example:

```go
aliases := map[string]string{
    "config": "all:config/production.json",  // Embedded file
    "data":   "data/local.json",              // Local file (dev mode override)
    "assets": "all:assets/",                 // Embedded directory
}
```

## Usage Examples#

### Basic File Reading#

```go
//go:embed all:files/*
var embeddedFS embed.FS

func main() {
    aliases := map[string]string{
        "readme": "all:README.md",
        "config": "all:config.json",
    }
    
    afero.Init(embeddedFS, aliases, false) // Production mode
    
    // Read file by alias
    content, err := afero.Read("readme")
    if err != nil {
        panic(err)
    }
    fmt.Printf("README has %d bytes\n", len(content))
}
```

### Development Mode with Overrides#

```go
//go:embed all:assets/*
var embeddedAssets embed.FS

func main() {
    aliases := map[string]string{
        "config": "all:config/",
        "uploads": "uploads/",  // Local directory for uploads
    }
    
    // Development mode: allows local overrides
    afero.Init(embeddedAssets, aliases, true)
    
    // In dev mode, you can create local files that shadow embedded ones
    // For example, creating "config/app.json" locally will override the embedded version
    
    data, err := afero.Read("config")
    if err != nil {
        panic(err)
    }
    fmt.Println(string(data))
}
```

### Streaming Large Files#

```go
func streamFile(alias string) error {
    // Get streaming reader
    reader, err := afero.Stream(alias)
    if err != nil {
        return err
    }
    defer reader.Close()
    
    // Stream to destination
    _, err = io.Copy(os.Stdout, reader)
    return err
}
```

### Checking File Existence#

```go
func checkFiles() {
    aliases := []string{"config", "static", "data"}
    
    for _, alias := range aliases {
        if afero.Exists(alias) {
            fmt.Printf("%s exists\n", alias)
        } else {
            fmt.Printf("%s not found\n", alias)
        }
    }
}
```

### Getting All Aliases#

```go
func listAliases() {
    aliases := afero.GetAliases()
    
    fmt.Println("Configured aliases:")
    for alias, path := range aliases {
        fmt.Printf("  %s → %s\n", alias, path)
    }
}
```

### Advanced Filesystem Access#

```go
func inspectFilesystem() {
    fs := afero.GetFileSystem()
    if fs == nil {
        fmt.Println("Filesystem not initialized")
        return
    }
    
    // Use Afero's full API
    // For example, list directory contents
    if dir, err := fs.Open("all:static/"); err == nil {
        defer dir.Close()
        // ... work with directory
    }
}
```

### Testing with Reset#

```go
func TestSomething(t *testing.T) {
    // Reset singleton for test isolation
    afero.ResetForTesting()
    
    //go:embed all:testdata/*
    var testFS embed.FS
    
    afero.Init(testFS, map[string]string{
        "test": "all:testdata/sample.txt",
    }, false)
    
    // Run test...
}
```

## Dependencies#

| Dependency | Role |
|------------|------|
| `github.com/spf13/afero` | Filesystem abstraction library |
| `embed` (standard library) | Go's embedded filesystem |
| `io`, `os`, `path/filepath`, `time` | Standard library filesystem operations |
| `sync` | Thread synchronization (Once, RWMutex) |

## Error Handling#

The library uses Go error handling patterns:

- **Not Initialized**: Returned when `Init()` hasn't been called#
- **Alias Not Found**: Returned when alias doesn't exist in map#
- **File Not Found**: Returned when physical file doesn't exist#
- **Unsupported Operations**: Returned for write operations on read-only filesystem#

Example error handling:

```go
data, err := afero.Read("config")
if err != nil {
    if strings.Contains(err.Error(), "not initialized") {
        return fmt.Errorf("afero manager not initialized: %w", err)
    }
    if strings.Contains(err.Error(), "not found in alias map") {
        return fmt.Errorf("invalid alias: %w", err)
    }
    if strings.Contains(err.Error(), "not supported") {
        return fmt.Errorf("operation not supported: %w", err)
    }
    return fmt.Errorf("failed to read file: %w", err)
}
```

## Common Pitfalls#

### 1. Manager Not Initialized#

**Problem**: Calling `Read()`, `Stream()`, etc. before `Init()`#

**Solution**:
- Always call `afero.Init()` before using other functions#
- Use `sync.Once` ensures initialization happens only once#

### 2. Alias Not Found#

**Problem**: `alias not found in alias map` error#

**Solution**:
- Check alias spelling in `aliasMap`#
- Verify alias was added during `Init()`#
- Use `afero.Exists()` to check before reading#

### 3. Embedded Path Prefix#

**Problem**: Paths not resolving correctly#

**Solution**:
- For embedded files, use `"all:"` prefix in alias map#
- The `all:` prefix tells the resolver to look in `embed.FS`#
- Example: `"config": "all:config/app.json"`#

### 4. Write Operations in Production#

**Problem**: `not supported for embedded filesystem` errors#

**Solution**:
- Production mode uses `ReadOnlyFs` - no writes allowed#
- Use development mode (`isDev=true`) for writable operations#
- Or use `CopyOnWriteFs` which allows writes to overlay#

### 5. Thread Safety#

**Problem**: Concurrent access issues#

**Solution**:
- The manager uses `sync.RWMutex` for thread safety#
- Multiple goroutines can safely call `Read()`, `Stream()`, etc.#
- `Init()` is protected by `sync.Once`#

### 6. Testing Interference#

**Problem**: Tests interfere with each other#

**Solution**:
- Call `afero.ResetForTesting()` at the start of each test#
- This resets the singleton and `sync.Once`#

## Advanced Usage#

### Custom Embed Directives#

```go
package main

import (
    "embed"
    "stackyrd/pkg/infrastructure/fs/afero"
)

// Embed single file
//go:embed config/production.json
var configFile embed.FS

// Embed directory recursively
//go:embed all:static/**
var staticFiles embed.FS

// Embed multiple patterns
//go:embed all:templates/*.html all:assets/**
var webAssets embed.FS

func main() {
    aliases := map[string]string{
        "config": "all:config/production.json",
        "static": "all:static/",
        "templates": "all:templates/",
        "assets": "all:assets/",
    }
    
    afero.Init(webAssets, aliases, false)
}
```

### Development Override Workflow#

```go
// In development, you want to edit config without recompiling
// Use CopyOnWriteFs to allow local overrides

//go:embed all:config/*
var embeddedConfig embed.FS

func main() {
    aliases := map[string]string{
        "app-config": "all:config/app.json",
        "local-config": "config/app.json",  // Local override
    }
    
    // Dev mode: local changes take precedence
    afero.Init(embeddedConfig, aliases, true)
    
    // Now you can edit config/app.json locally and it will be used
    // instead of the embedded version
}
```

### Inspecting Filesystem#

```go
func listFiles(alias string) error {
    fs := afero.GetFileSystem()
    if fs == nil {
        return fmt.Errorf("filesystem not initialized")
    }
    
    // Open directory
    dir, err := fs.Open("all:static/")
    if err != nil {
        return err
    }
    defer dir.Close()
    
    // Read directory entries
    entries, err := dir.Readdir(-1)
    if err != nil {
        return err
    }
    
    fmt.Println("Files in static/:")
    for _, entry := range entries {
        fmt.Printf("  %s (%d bytes)\n", entry.Name(), entry.Size())
    }
    return nil
}
```

### Error Recovery#

```go
func safeRead(alias string) ([]byte, error) {
    if !afero.Exists(alias) {
        return nil, fmt.Errorf("alias %s does not exist", alias)
    }
    
    data, err := afero.Read(alias)
    if err != nil {
        // Try fallback alias
        fallback := alias + "-fallback"
        if afero.Exists(fallback) {
            return afero.Read(fallback)
        }
        return nil, err
    }
    return data, nil
}
```

## Internal Algorithms#

### Singleton Initialization#

```
Init(embedFS, aliasMap, isDev):
    │
    ├── sync.Once.Do():
    │   ├── Create aferoManager{}
    │   ├── Copy aliasMap to instance.aliases
    │   ├── If isDev:
    │   │   ├── baseFS = embedFSWrapper{fs: embedFS}
    │   │   ├── writableFS = afero.NewOsFs()
    │   │   └── instance.fs = afero.NewCopyOnWriteFs(baseFS, writableFS)
    │   ├── Else:
    │   │   ├── baseFS = embedFSWrapper{fs: embedFS}
    │   │   └── instance.fs = afero.NewReadOnlyFs(baseFS)
    │   └── Return
    └── Subsequent calls are ignored
```

### Alias Resolution#

```
resolveAlias(alias):
    │
    ├── Lookup alias in instance.aliases
    ├── If not found → return error "alias not found"
    ├── If path starts with "all:":
    │   ├── Strip "all:" prefix (4 characters)
    │   └── Return cleaned path
    └── Return path as-is
```

### embedFSWrapper Operations#

```
embedFSWrapper implements afero.Fs:
    │
    ├── Open(name): Opens file from embed.FS
    ├── Stat(name): Returns file info
    ├── ReadFile(name): Reads entire file (via afero.ReadFile)
    └── Most operations return "not supported" error:
        ├── Chtimes, Create, Mkdir, MkdirAll
        ├── Remove, RemoveAll, Rename
        └── Chmod, Chown, WriteString
```

### embedFile Operations#

```
embedFile implements afero.File:
    │
    ├── Read(b): Reads from embedded file
    ├── ReadAt(b, off): Reads at offset (if supported)
    ├── Seek(offset, whence): Seeks to position (if supported)
    ├── Readdir(count): Reads directory entries (if directory)
    ├── Readdirnames(n): Reads directory names (if directory)
    ├── Close(): Closes the file
    └── Most operations return "not supported" error:
        ├── Write, WriteAt, Truncate
        └── Sync
```

## API Reference#

### Functions#

| Function | Signature | Description |
|----------|-----------|-------------|
| `Init` | `Init(embedFS embed.FS, aliasMap map[string]string, isDev bool)` | Initializes the singleton manager |
| `Read` | `Read(alias string) ([]byte, error)` | Reads file content by alias |
| `Stream` | `Stream(alias string) (io.ReadCloser, error)` | Returns streaming reader for alias |
| `Exists` | `Exists(alias string) bool` | Checks if alias exists and file is accessible |
| `GetAliases` | `GetAliases() map[string]string` | Returns copy of all aliases |
| `GetFileSystem` | `GetFileSystem() afero.Fs` | Returns underlying Afero filesystem |
| `ResetForTesting` | `ResetForTesting()` | Resets singleton for testing |

### Filesystem Modes#

| Mode | Afero Type | Description |
|------|-------------|-------------|
| Development (`isDev=true`) | `CopyOnWriteFs` | Base: embed.FS, Overlay: OS filesystem. Allows local overrides |
| Production (`isDev=false`) | `ReadOnlyFs` | Wraps embed.FS. Prevents any modifications |

## License#

This code is part of the Stackyrd project. See the main project LICENSE file for details.#