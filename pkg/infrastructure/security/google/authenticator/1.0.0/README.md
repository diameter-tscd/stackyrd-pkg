# Google Authenticator OTP Library#

## Overview#

The `google_authenticator` package implements one-time password algorithms supported by Google Authenticator. It supports HMAC-Based One-time Password (HOTP) algorithm specified in RFC 4226 and Time-based One-time Password (TOTP) algorithm specified in RFC 6238. This is a pure Go implementation for generating and validating OTP codes.

**Import Path:** `stackyrd/pkg/infrastructure/security/google/authenticator`

## Features#

- **HOTP Support**: Generate and validate HMAC-based one-time passwords#
- **TOTP Support**: Generate and validate time-based one-time passwords#
- **Scratch Codes**: Support for 8-digit scratch codes#
- **Provisioning URI**: Generate URIs for QR code provisioning#
- **Window Validation**: Configurable window size for TOTP validation#
- **Replay Protection**: Prevent reuse of TOTP codes within window#
- **UTC Support**: Option to use UTC for TOTP timestamps#

## Quick Start#

```go
package main#

import (
    "fmt"
    "stackyrd/pkg/infrastructure/security/google/authenticator"
)

func main() {
    // Create OTP configuration
    config := authenticator.OTPConfig{
        Secret:      "JBSWY3DPEHPK3PXP", // 80-bit base32 encoded secret
        HotpCounter: 0,                     // 0 for TOTP, >0 for HOTP
        WindowSize:  3,                     // Validation window
        ScratchCodes: []int{12345678, 87654321}, // Scratch codes
        UTC:         false,                  // Use local time for TOTP
    }
    
    // Generate a TOTP code (current time step)
    t0 := int(time.Now().Unix() / 30)
    code := authenticator.ComputeCode(config.Secret, int64(t0))
    fmt.Printf("Current TOTP code: %06d\n", code)
    
    // Authenticate a user-provided code
    password := fmt.Sprintf("%06d", code)
    valid, err := config.Authenticate(password)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
    } else if valid {
        fmt.Println("Authentication successful!")
    } else {
        fmt.Println("Authentication failed!")
    }
}
```

## Architecture#

### Core Structs#

| Struct | Description |
|--------|-------------|
| `OTPConfig` | One-time password configuration (secret, counter, window, scratch codes) |
| `OTPConfig` Methods | `Authenticate()`, `ProvisionURI()`, `ProvisionURIWithIssuer()` |

### OTP Flow#

```
┌─────────────────────────────────────────────────────┐
│              Google Authenticator                     │
├─────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │  ComputeCode  │  → HMAC-SHA1, dynamic truncation     │
│  └────────────────┘                                      │
│                                                          │
│  ┌────────────────┐                                      │
│  │  OTPConfig    │  → Manage secret, counters, windows    │
│  └────────────────┘                                      │
│         │                                               │
│         ▼                                               │
│  ┌────────────────┐                                      │
│  │  Authenticate │  → Validate HOTP/TOTP/Scratch codes   │
│  └────────────────┘                                      │
│                                                          │
│  ┌────────────────┐                                      │
│  │ ProvisionURI │  → Generate otpauth:// URIs           │
│  └────────────────┘                                      │
└─────────────────────────────────────────────────────┘
```

## How It Works#

### 1. Compute Code (HOTP/TOTP)#

```
ComputeCode(secret, value)
    │
    ├── Decode base32 secret to bytes
    ├── Create HMAC-SHA1 with key
    ├── Write value as big-endian int64
    ├── Compute HMAC digest
    ├── Get offset from last byte (0x0f mask)
    ├── Extract 4 bytes at offset
    ├── Convert to int32 (big-endian)
    ├── Mask with 0x7fffffff
    └── Return code % 1000000 (6 digits)
```

### 2. Authenticate Flow#

```
config.Authenticate(password)
    │
    ├── Check length: 6 digits (HOTP/TOTP) or 8 digits (scratch)
    ├── Parse password to int
    │
    ├── If 8 digits: check scratch codes
    │   └── Remove used scratch code
    │
    ├── If HOTP (counter > 0):
    │   ├── Check codes in window [counter, counter+windowSize)
    │   ├── If valid: update counter += i+1
    │   └── Always increment counter
    │
    └── If TOTP (counter == 0):
        ├── Compute t0 = time.Now().Unix() / 30
        ├── Check codes in window [t0-window/2, t0+window/2]
        ├── If valid: add to disallowReuse
        └── Remove expired disallowed codes
```

### 3. Provisioning URI#

```
config.ProvisionURIWithIssuer(user, issuer)
    │
    ├── Determine auth type: "totp/" or "hotp/"
    ├── Build query string:
    │   ├── secret=<secret>
    │   ├── issuer=<issuer> (if provided)
    │   └── counter=<counter> (if HOTP)
    │
    └── Return "otpauth://<auth><issuer>:<user>?<query>"
```

## Configuration#

### OTPConfig Options#

| Field | Type | Description |
|-------|------|-------------|
| `Secret` | string | 80-bit base32 encoded secret key |
| `HotpCounter` | int | HOTP counter (0 for TOTP, >0 for HOTP) |
| `WindowSize` | int | Validation window size (typically 1-5) |
| `ScratchCodes` | []int | Array of 8-digit backup codes |
| `DisallowReuse` | []int | Timestamps of used TOTP codes (internal) |
| `UTC` | bool | Use UTC for TOTP timestamp (default: local time) |

## Usage Examples#

### Generate TOTP Code#

```go
// Generate current TOTP code
config := authenticator.OTPConfig{
    Secret: "JBSWY3DPEHPK3PXP",
}

t0 := int(time.Now().Unix() / 30)
code := authenticator.ComputeCode(config.Secret, int64(t0))
fmt.Printf("TOTP Code: %06d\n", code)
```

### Authenticate TOTP#

```go
config := authenticator.OTPConfig{
    Secret:      "JBSWY3DPEHPK3PXP",
    WindowSize:  3,
    UTC:         false,
}

// Simulate user entering code
userCode := "123456"
valid, err := config.Authenticate(userCode)
if err != nil {
    fmt.Printf("Error: %v\n", err)
} else if valid {
    fmt.Println("Authentication successful!")
} else {
    fmt.Println("Invalid code")
}
```

### Authenticate HOTP#

```go
config := authenticator.OTPConfig{
    Secret:      "JBSWY3DPEHPK3PXP",
    HotpCounter: 5, // Current HOTP counter
    WindowSize:  3,
}

// Simulate user entering HOTP code
userCode := "987654"
valid, err := config.Authenticate(userCode)
if err != nil {
    fmt.Printf("Error: %v\n", err)
} else if valid {
    fmt.Println("HOTP authentication successful!")
    fmt.Printf("New counter: %d\n", config.HotpCounter)
}
```

### Use Scratch Codes#

```go
config := authenticator.OTPConfig{
    Secret:       "JBSWY3DPEHPK3PXP",
    ScratchCodes: []int{12345678, 87654321, 11112222},
}

// Try scratch code
valid, err := config.Authenticate("12345678")
if valid {
    fmt.Println("Scratch code accepted!")
    fmt.Printf("Remaining scratch codes: %v\n", config.ScratchCodes)
}
```

### Generate Provisioning URI#

```go
config := authenticator.OTPConfig{
    Secret:      "JBSWY3DPEHPK3PXP",
    HotpCounter: 0, // 0 for TOTP
}

// Generate URI for QR code (no issuer)
uri := config.ProvisionURI("user@example.com")
fmt.Printf("Provisioning URI: %s\n", uri)
// Output: otpauth://totp/user@example.com?secret=JBSWY3DPEHPK3PXP

// With issuer
uri = config.ProvisionURIWithIssuer("user@example.com", "MyApp")
fmt.Printf("URI with issuer: %s\n", uri)
// Output: otpauth://totp/MyApp:user@example.com?secret=...&issuer=MyApp
```

### HOTP Provisioning URI#

```go
config := authenticator.OTPConfig{
    Secret:      "JBSWY3DPEHPK3PXP",
    HotpCounter: 10, // Current HOTP counter
}

uri := config.ProvisionURIWithIssuer("user@example.com", "MyApp")
fmt.Printf("HOTP URI: %s\n", uri)
// Output: otpauth://hotp/MyApp:user@example.com?secret=...&counter=10&issuer=MyApp
```

## Dependencies#

| Dependency | Role |
|------------|------|
| Standard library | `crypto/hmac`, `crypto/sha1`, `encoding/base32`, `encoding/binary`, `errors`, `net/url`, `sort`, `strconv`, `time` |

## Error Handling#

The library returns errors for invalid codes:

- **ErrInvalidCode**: Returned when password is not 6 or 8 digits, or doesn't start with valid digit#

Example error handling:

```go
valid, err := config.Authenticate(password)
if err != nil {
    if errors.Is(err, authenticator.ErrInvalidCode) {
        return fmt.Errorf("invalid code format: %w", err)
    }
    return fmt.Errorf("authentication error: %w", err)
}
if !valid {
    return fmt.Errorf("authentication failed")
}
```

## Common Pitfalls#

### 1. Invalid Secret Format#

**Problem**: Secret is not valid base32 encoded string#

**Solution**:
- Generate a valid 80-bit (16 character) base32 encoded secret#
- Use only characters: A-Z2-7#
- Example: `"JBSWY3DPEHPK3PXP"`#

### 2. Wrong Code Length#

**Problem**: Authenticating with wrong length code#

**Solution**:
- TOTP/HOTP codes are 6 digits#
- Scratch codes are 8 digits#
- The `Authenticate()` method checks length automatically#

### 3. HOTP Counter Not Advancing#

**Problem**: HOTP counter not updated after successful auth#

**Solution**:
- The `Authenticate()` method updates `HotpCounter` automatically#
- Save the updated `OTPConfig` after authentication#
- Counter should only increment on successful auth#

### 4. TOTP Time Drift#

**Problem**: TOTP codes not validating due to time drift#

**Solution**:
- Increase `WindowSize` to allow more time steps#
- Ensure system clock is synchronized#
- Use UTC with `UTC: true` for consistency#

### 5. Scratch Code Reuse#

**Problem**: Scratch codes being reused#

**Solution**:
- Scratch codes are removed after successful use#
- Save updated `ScratchCodes` array#
- Generate new scratch codes if needed#

## Advanced Usage#

### Custom TOTP Validation#

```go
func validateTOTP(secret string, userCode int, windowSize int) bool {
    t0 := int(time.Now().Unix() / 30)
    
    for t := t0 - windowSize; t <= t0+windowSize; t++ {
        if authenticator.ComputeCode(secret, int64(t)) == userCode {
            return true
        }
    }
    return false
}
```

### Batch Scratch Code Generation#

```go
func generateScratchCodes(count int) []int {
    codes := make([]int, count)
    for i := 0; i < count; i++ {
        // Generate random 8-digit code
        code := rand.Intn(90000000) + 10000000
        codes[i] = code
    }
    return codes
}
```

### TOTP with Custom Time Step#

```go
// Use 60-second time step instead of 30
func computeTOTP60(secret string) int {
    t0 := int(time.Now().Unix() / 60)
    return authenticator.ComputeCode(secret, int64(t0))
}
```

## Internal Algorithms#

### HMAC-SHA1 OTP (RFC 4226)#

```
ComputeCode(secret, value):
    │
    ├── Decode base32 secret → key bytes
    ├── Create HMAC-SHA1 with key
    ├── Write value as big-endian int64
    ├── Compute digest (20 bytes)
    ├── offset = digest[19] & 0x0f
    ├── Extract bytes: digest[offset..offset+4]
    ├── Convert to int32 (big-endian)
    ├── Mask: truncated &= 0x7fffffff
    └── Return: truncated % 1000000
```

### TOTP (RFC 6238)#

```
TOTP validation:
    │
    ├── t0 = current Unix time / time_step (default 30s)
    ├── for t in [t0-window, t0+window]:
    │   └── if ComputeCode(secret, t) == user_code → valid
    └── Check disallowReuse list
```

## Security Considerations#

1. **Secret Storage**: Store secrets securely (encrypted at rest)#
2. **Window Size**: Smaller windows are more secure but less tolerant to time drift#
3. **Scratch Codes**: Treat as passwords - store hashed if possible#
4. **Replay Protection**: TOTP codes should not be reused within window#
5. **HOTP Counter**: Must be synchronized and stored securely#

## License#

This code is part of the Stackyrd project. See the main project LICENSE file for details.#

## Original Source#

This implementation is based on the original source: https://github.com/dgryski/dgoogauth#

The package has been integrated into the Stackyrd infrastructure security module.#