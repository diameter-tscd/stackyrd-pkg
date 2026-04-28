# SMTP Manager

## Overview

The `SMTPManager` is a Go library for managing SMTP email sending operations. It supports sending plain text and HTML emails with attachments, custom headers, CC/BCC recipients, and async operations via a worker pool.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Email Sending**: Send plain text and HTML emails
- **Attachments**: Support for email attachments with custom content types
- **Recipients**: Support for To, CC, BCC recipients
- **Custom Headers**: Add custom email headers
- **TLS/STARTTLS**: Support for secure connections (TLS and STARTTLS)
- **Authentication**: Plain authentication with username/password
- **Async Operations**: Async email sending via worker pool
- **Worker Pool**: Async job execution support (4 workers)
- **Connection Test**: Validates connectivity via test connection

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
    
    // Create SMTP manager (configuration via viper)
    manager, err := smtp.NewSMTPManager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // Send a simple email
    err = manager.SendSimpleEmail(ctx, "recipient@example.com", "Hello", "This is a test email.")
    if err != nil {
        panic(err)
    }
    fmt.Println("Email sent!")
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `SMTPManager` | Main manager with SMTP client, credentials, worker pool |
| `EmailMessage` | Email with To, CC, BCC, subject, body, HTML flag, headers, attachments |
| `EmailAttachment` | Attachment with filename, content type, content bytes |

### Concurrency Model

```
┌─────────────────────────────────────────────────────────────┐
│                   SMTPManager                             │
├─────────────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │  SMTP       │  → smtp.Client (net/smtp)             │
│  │  Client     │  → TLS/STARTTLS, Auth                 │
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
│  Host: SMTP server address (e.g., smtp.gmail.com)          │
│  Port: SMTP server port (default: 587 for STARTTLS)         │
│  FromEmail: Sender email address                          │
│  FromName: Sender display name                            │
└─────────────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewSMTPManager(logger)
    │
    ├── Check viper config: "smtp.enabled"
    ├── Get host: "smtp.host"
    ├── Get port: "smtp.port"
    ├── Get username: "smtp.username"
    ├── Get password: "smtp.password"
    ├── Get fromEmail: "smtp.from_email"
    ├── Get fromName: "smtp.from_name"
    ├── Get useTLS: "smtp.use_tls"
    ├── Get useSTARTTLS: "smtp.use_starttls"
    │
    ├── Test connection: testConnection()
    │   ├── If useTLS: tls.Dial → smtp.NewClient
    │   ├── Else: smtp.Dial
    │   ├── If useSTARTTLS: client.StartTLS(tlsConfig)
    │   └── If username/password: client.Auth(PlainAuth)
    │
    ├── Create WorkerPool(4)
    ├── Start worker pool
    └── Return SMTPManager
```

### 2. Send Email Flow

```
SendEmail(ctx, message)
    │
    ├── Build headers map:
    │   ├── From: "FromName <FromEmail>"
    │   ├── To: first recipient
    │   ├── Subject
    │   ├── Content-Type: text/plain or text/html
    │   └── Custom headers
    │
    ├── Build message body: headers + "\r\n" + body
    ├── Get all recipients: To + CC + BCC
    ├── smtp.SendMail(addr, auth, FromEmail, recipients, msgBytes)
    └── Return error
```

### 3. Test Connection Flow

```
testConnection()
    │
    ├── addr = fmt.Sprintf("%s:%d", Host, Port)
    │
    ├── If UseTLS:
    │   ├── tls.Dial("tcp", addr, tlsConfig)
    │   └── smtp.NewClient(conn, Host)
    │
    ├── Else:
    │   └── smtp.Dial(addr)
    │
    ├── If UseSTARTTLS && !UseTLS:
    │   └── client.StartTLS(tlsConfig)
    │
    ├── If Username && Password:
    │   └── client.Auth(PlainAuth("", Username, Password, Host))
    │
    ├── client.Close()
    └── Return error
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `smtp.enabled` | bool | false | Enable/disable SMTP manager |
| `smtp.host` | string | "" | SMTP server hostname |
| `smtp.port` | int | 587 | SMTP server port |
| `smtp.username` | string | "" | SMTP username |
| `smtp.password` | string | "" | SMTP password |
| `smtp.from_email` | string | "" | Sender email address |
| `smtp.from_name` | string | "" | Sender display name |
| `smtp.use_tls` | bool | false | Use TLS (port 465 typically) |
| `smtp.use_starttls` | bool | true | Use STARTTLS (port 587 typically) |

### Environment Variables

The SMTP manager does not directly use environment variables. Configuration is read through Viper, which can load from:
- Configuration files (JSON, YAML, TOML)
- Environment variables (if Viper is configured for it)

You can set SMTP parameters via:
```go
viper.Set("smtp.enabled", true)
viper.Set("smtp.host", "smtp.gmail.com")
viper.Set("smtp.port", 587)
viper.Set("smtp.username", "your-email@gmail.com")
viper.Set("smtp.password", "your-app-password")
viper.Set("smtp.from_email", "your-email@gmail.com")
viper.Set("smtp.from_name", "Your Name")
viper.Set("smtp.use_tls", false)
viper.Set("smtp.use_starttls", true)
```

## Usage Examples

### Sending Simple Email

```go
ctx := context.Background()

// Send a simple text email
err := manager.SendSimpleEmail(ctx, "recipient@example.com", "Hello", "This is a test email.")
if err != nil {
    panic(err)
}
fmt.Println("Email sent!")
```

### Sending HTML Email

```go
ctx := context.Background()

htmlBody := `<html><body><h1>Hello!</h1><p>This is HTML email.</p></body></html>`

err := manager.SendHTMLEmail(ctx, "recipient@example.com", "HTML Email", htmlBody)
if err != nil {
    panic(err)
}
fmt.Println("HTML email sent!")
```

### Sending Email with Multiple Recipients

```go
ctx := context.Background()

message := smtp.EmailMessage{
    To:      []string{"user1@example.com", "user2@example.com"},
    CC:      []string{"manager@example.com"},
    BCC:     []string{"archive@example.com"},
    Subject: "Team Update",
    Body:    "Please find the latest updates attached.",
    IsHTML:  false,
}

err := manager.SendEmail(ctx, message)
if err != nil {
    panic(err)
}
fmt.Println("Email with multiple recipients sent!")
```

### Sending Email with Attachments

```go
ctx := context.Background()

// Read attachment file
attachmentData, err := os.ReadFile("document.pdf")
if err != nil {
    panic(err)
}

message := smtp.EmailMessage{
    To:      []string{"recipient@example.com"},
    Subject: "Document Attached",
    Body:    "Please find the document attached.",
    Attachments: []smtp.EmailAttachment{
        {
            Filename:    "document.pdf",
            ContentType: "application/pdf",
            Content:     attachmentData,
        },
    },
}

err = manager.SendEmail(ctx, message)
if err != nil {
    panic(err)
}
fmt.Println("Email with attachment sent!")
```

### Sending Email with Custom Headers

```go
ctx := context.Background()

message := smtp.EmailMessage{
    To:      []string{"recipient@example.com"},
    Subject: "Custom Headers",
    Body:    "Email with custom headers.",
    Headers: map[string]string{
        "X-Custom-Header": "CustomValue",
        "X-Priority":      "1",
    },
}

err := manager.SendEmail(ctx, message)
if err != nil {
    panic(err)
}
fmt.Println("Email with custom headers sent!")
```

### Async Operations

```go
ctx := context.Background()

// Async send simple email
result := manager.SendSimpleEmailAsync(ctx, "recipient@example.com", "Async Test", "Sent asynchronously!")
select {
case <-result.Ch:
    fmt.Println("Email sent asynchronously!")
case err := <-result.ErrCh:
    fmt.Printf("Error: %v\n", err)
}

// Async send HTML email
result2 := manager.SendHTMLEmailAsync(ctx, "recipient@example.com", "Async HTML", "<h1>Hello!</h1>")
select {
case <-result2.Ch:
    fmt.Println("HTML email sent asynchronously!")
case err := <-result2.ErrCh:
    fmt.Printf("Error: %v\n", err)
}

// Async send complex email
message := smtp.EmailMessage{
    To:     []string{"recipient@example.com"},
    Subject: "Async Complex",
    Body:   "Complex email sent async.",
}
result3 := manager.SendEmailAsync(ctx, message)
select {
case <-result3.Ch:
    fmt.Println("Complex email sent asynchronously!")
case err := <-result3.ErrCh:
    fmt.Printf("Error: %v\n", err)
}
```

### Get Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
fmt.Printf("Host: %s\n", status["host"])
fmt.Printf("Port: %v\n", status["port"])
fmt.Printf("Username: %s\n", status["username"])
fmt.Printf("From Email: %s\n", status["from_email"])
fmt.Printf("Use TLS: %v\n", status["use_tls"])
fmt.Printf("Use STARTTLS: %v\n", status["use_starttls"])

if status["connected"] == false {
    if errMsg, ok := status["error"].(string); ok {
        fmt.Printf("Error: %s\n", errMsg)
    }
}
```

## Dependencies

| Dependency | Role |
|------------|------|
| `github.com/spf13/viper` | Configuration management |
| `stackyrd/config` | Internal configuration package |
| `stackyrd/pkg/logger` | Structured logging |
| Standard library | `context`, `crypto/tls`, `fmt`, `net/smtp`, `time` |

## Error Handling

The library uses Go error handling patterns:

- **Connection failures**: `testConnection()` validates via SMTP dial/auth
- **Send failures**: Returned from `smtp.SendMail()`
- **TLS errors**: Handled during `testConnection()`
- **Authentication errors**: Handled during `testConnection()`
- **Nil checks**: Public methods handle nil receiver gracefully in `GetStatus()`

Example error handling:

```go
err := manager.SendSimpleEmail(ctx, to, subject, body)
if err != nil {
    if strings.Contains(err.Error(), "authentication failed") {
        return fmt.Errorf("invalid SMTP credentials: %w", err)
    }
    if strings.Contains(err.Error(), "connection refused") {
        return fmt.Errorf("SMTP server not reachable: %w", err)
    }
    if strings.Contains(err.Error(), "timeout") {
        return fmt.Errorf("SMTP timeout: %w", err)
    }
    return fmt.Errorf("failed to send email: %w", err)
}
```

## Common Pitfalls

### 1. SMTP Server Not Reachable

**Problem**: `connection refused` or `no route to host` errors

**Solution**: 
- Verify SMTP server address: `viper.Set("smtp.host", "smtp.gmail.com")`
- Check network connectivity (ping the server)
- Verify firewall allows access to SMTP port (587, 465, or 25)

### 2. Authentication Failed

**Problem**: `authentication failed` or `invalid credentials` errors

**Solution**:
- Verify username and password
- For Gmail, use App Password (not regular password)
- Check if 2FA is enabled (requires App Password)

### 3. TLS/STARTTLS Issues

**Problem**: TLS handshake errors or STARTTLS failures

**Solution**:
- For port 465: set `smtp.use_tls = true`, `smtp.use_starttls = false`
- For port 587: set `smtp.use_tls = false`, `smtp.use_starttls = true`
- For port 25: set both to false

### 4. From Email Mismatch

**Problem**: `From address not verified` or similar errors

**Solution**:
- Ensure `smtp.from_email` matches authenticated account
- For Gmail, From address must be the account email or verified alias

### 5. Timeout Issues

**Problem**: Operations timing out

**Solution**:
```go
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
```

### 6. Worker Pool Not Available

**Problem**: Async operations falling back to direct execution

**Solution**:
- The pool is created during initialization
- Check that `NewSMTPManager()` succeeded
- Verify `smtp.enabled` is true

## Advanced Usage

### Email with Multiple Attachments

```go
func sendWithMultipleAttachments(manager *smtp.SMTPManager, to string) error {
    ctx := context.Background()
    
    var attachments []smtp.EmailAttachment
    
    // Add multiple files
    files := []struct{
        name string
        path string
        contentType string
    }{
        {"report.pdf", "report.pdf", "application/pdf"},
        {"image.png", "image.png", "image/png"},
        {"data.csv", "data.csv", "text/csv"},
    }
    
    for _, f := range files {
        data, err := os.ReadFile(f.path)
        if err != nil {
            return fmt.Errorf("failed to read %s: %w", f.name, err)
        }
        attachments = append(attachments, smtp.EmailAttachment{
            Filename:    f.name,
            ContentType: f.contentType,
            Content:     data,
        })
    }
    
    message := smtp.EmailMessage{
        To:         []string{to},
        Subject:    "Multiple Attachments",
        Body:       "Please find the attached files.",
        Attachments: attachments,
    }
    
    return manager.SendEmail(ctx, message)
}
```

### Email Template with HTML

```go
func sendTemplatedEmail(manager *smtp.SMTPManager, to, name string) error {
    ctx := context.Background()
    
    htmlBody := fmt.Sprintf(`
        <html>
        <body>
            <h1>Hello %s!</h1>
            <p>Thank you for joining our service.</p>
            <p>Your account is now active.</p>
            <hr>
            <p><small>This is an automated email.</small></p>
        </body>
        </html>
    `, name)
    
    return manager.SendHTMLEmail(ctx, to, "Welcome!", htmlBody)
}
```

### Batch Email Sending

```go
func sendBatchEmails(manager *smtp.SMTPManager, recipients []string, subject, body string) {
    ctx := context.Background()
    
    for _, to := range recipients {
        result := manager.SendSimpleEmailAsync(ctx, to, subject, body)
        go func(to string, ch *AsyncResult[struct{}]) {
            select {
            case <-ch.Ch:
                fmt.Printf("Email sent to %s\n", to)
            case err := <-ch.ErrCh:
                fmt.Printf("Failed to send to %s: %v\n", to, err)
            }
        }(to, result)
    }
}
```

### Email with Reply-To

```go
func sendWithReplyTo(manager *smtp.SMTPManager, to, replyTo, subject, body string) error {
    ctx := context.Background()
    
    message := smtp.EmailMessage{
        To:      []string{to},
        Subject: subject,
        Body:    body,
        Headers: map[string]string{
            "Reply-To": replyTo,
        },
    }
    
    return manager.SendEmail(ctx, message)
}
```

## Internal Algorithms

### Email Message Building

```
SendEmail():
    │
    ├── Build headers map:
    │   ├── From: "FromName <FromEmail>"
    │   ├── To: first recipient
    │   ├── Subject
    │   ├── Content-Type: text/plain or text/html
    │   └── Custom headers from message.Headers
    │
    ├── Build message: headers + "\r\n" + body
    ├── Collect recipients: To + CC + BCC
    └── smtp.SendMail(addr, auth, FromEmail, recipients, msgBytes)
```

### Connection Test

```
testConnection():
    │
    ├── If UseTLS: tls.Dial → smtp.NewClient
    ├── Else: smtp.Dial
    ├── If UseSTARTTLS: client.StartTLS()
    ├── If Username/Password: client.Auth(PlainAuth)
    ├── client.Close()
    └── Return error
```

### Async Operation Pattern

```
SendEmailAsync(ctx, message):
    │
    └── ExecuteAsync(ctx, func):
        └── return SendEmail(ctx, message)
            └── Returns (struct{}, error)
```

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.