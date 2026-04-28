# AWS EC2 Manager

## Overview

The `AWSEC2Manager` is a comprehensive Go library for managing AWS EC2 operations via the AWS EC2 API (Query API over HTTPS). It provides complete EC2 resource management including instances, volumes, snapshots, security groups, VPCs, subnets, key pairs, and AMIs with async support.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Instance Management**: Describe, run, terminate, start, stop, and reboot EC2 instances
- **Volume Management**: Create, delete, and describe EBS volumes
- **Snapshot Management**: Create and describe EBS snapshots
- **Security Groups**: Describe and manage security groups with inbound/outbound rules
- **VPC Management**: Describe VPCs and subnets
- **Key Pair Management**: Describe SSH key pairs
- **AMI Management**: Describe Amazon Machine Images
- **Async Operations**: Async execution for all describe operations via worker pool
- **AWS Authentication**: Supports access key, secret key, and session tokens with AWS v4 signing
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
    
    // Create AWS EC2 manager (configuration via viper)
    manager, err := ec2.NewAWSEC2Manager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // List EC2 instances
    instances, err := manager.DescribeInstances(ctx)
    if err != nil {
        panic(err)
    }
    
    for _, inst := range instances {
        fmt.Printf("Instance: %s (Type: %s, State: %s)\n", #inst.InstanceID, inst.InstanceType, inst.State)
    }
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `AWSEC2Manager` | Main manager with HTTP client and worker pool |
| `EC2Instance` | EC2 instance with ID, type, state, IPs, tags |
| `EC2Volume` | EBS volume with size, type, IOPS, encryption |
| `EC2Snapshot` | EBS snapshot with progress and state |
| `EC2SecurityGroup` | Security group with inbound/outbound rules |
| `EC2SecurityRule` | Individual security rule with protocol and port range |
| `EC2VPC` | VPC with CIDR block and state |
| `EC2Subnet` | Subnet with CIDR, AZ, and VPC ID |
| `EC2KeyPair` | SSH key pair with fingerprint |
| `EC2Image` | AMI with owner, architecture, and state |
| `EC2ListResponse` | Generic list wrapper for all EC2 resources |

### Concurrency Model

```
┌─────────────────────────────────────────────────────────────┐
│                   AWSEC2Manager                           │
├─────────────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │ retryablehttp  │  → Retry logic (max 3, 1-10s wait)│
│  │ Client         │  → HTTP timeout: 60s                   │
│  │                │  → AWS v4 request signing             │
│  └────────────────┘                                      │
│                                                         │
│  ┌─────────────┐      ┌──────────────┐                   │
│  │  WorkerPool │◄─────│ AsyncResult │                   │
│  │  (6 workers)│      │   Channel   │                   │
│  └─────────────┘      └──────────────┘                   │
│         ▲                      │                            │
│         │                      ▼                            │
│  ┌─────────────┐      ┌──────────────┐                  │
│  │  SubmitJob  │      │ ExecuteAsync │                  │
│  └─────────────┘      └──────────────┘                  │
│                                                         │
│  Region: AWS region (default: us-east-1)                      │
│  BaseURL: https://ec2.{region}.amazonaws.com         │
└─────────────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewAWSEC2Manager()
    │
    ├── Check viper config: "aws_ec2.enabled"
    ├── Get region: "aws_ec2.region" (default: "us-east-1")
    ├── Get credentials: access_key_id, secret_access_key, session_token
    ├── Build baseURL: "https://ec2.{region}.amazonaws.com"
    ├── Create retryablehttp.Client (max 3 retries, 1-10s wait, 60s timeout)
    ├── Create AWSEC2Manager with credentials
    ├── Create WorkerPool(6)
    ├── Start worker pool
    └── Return AWSEC2Manager
```

### 2. API Request Flow

```
apiRequest(ctx, action, params)
    │
    ├── Build URL: baseURL/?Action={action}&Version=2016-11-15&{params}
    ├── Create GET request with context
    ├── signRequest(req)
    │   ├── Set Host header
    │   ├── Set X-Amz-Date header
    │   ├── If session_token: Set X-Amz-Security-Token
    │   └── If credentials: Set Authorization (AWS4-HMAC-SHA256)
    ├── Execute request
    ├── Read response body
    ├── Check status code (200-299 = success)
    └── Return response bytes or error
```

### 3. Instance Description Flow

```
DescribeInstances(ctx)
    │
    ├── apiRequest(ctx, "DescribeInstances", nil)
    ├── JSON unmarshal to result with Reservations
    ├── Extract instances from all reservations
    └── Return []EC2Instance
```

### 4. Run Instances Flow

```
RunInstances(ctx, imageID, instanceType, minCount, maxCount)
    │
    ├── Build params: ImageId, InstanceType, MinCount, MaxCount
    ├── apiRequest(ctx, "RunInstances", params)
    ├── JSON unmarshal to result
    └── Return []EC2Instance
```

### 5. AWS v4 Signing Flow

```
signRequest(req)
    │
    ├── Get current time (UTC)
    ├── Format date stamp: "20060102"
    ├── Format AMZ date: "20060102T150405Z"
    ├── Set headers: Host, X-Amz-Date
    ├── If session token: Set X-Amz-Security-Token
    └── If credentials:
            └── Set Authorization: AWS4-HMAC-SHA256
                    Credential={accessKey}/{dateStamp}/{region}/ec2/aws4_request
                    SignedHeaders=host;x-amz-date
                    Signature={UNSIGNED-PAYLOAD}
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `aws_ec2.enabled` | bool | false | Enable/disable AWS EC2 manager |
| `aws_ec2.region` | string | "us-east-1" | AWS region |
| `aws_ec2.access_key_id` | string | "" | AWS access key ID |
| `aws_ec2.secret_access_key` | string | "" | AWS secret access key |
| `aws_ec2.session_token` | string | "" | AWS session token (optional) |

### Environment Variables

The EC2 manager does not directly use environment variables. Configuration is read through Viper, which can load from:
- Configuration files (JSON, YAML, TOML)
- Environment variables (if Viper is configured for it)
- AWS credentials can also be set via standard AWS env vars (AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_SESSION_TOKEN)

## Usage Examples

### Instance Operations

```go
ctx := context.Background()

// Describe instances
instances, err := manager.DescribeInstances(ctx)
if err != nil {
    panic(err)
}

for _, inst := range instances {
    fmt.Printf("Instance: %s\n", inst.InstanceID)
    fmt.Printf("  Type: %s\n", inst.InstanceType)
    fmt.Printf("  State: %s\n", inst.State)
    fmt.Printf("  Public IP: %s\n", inst.PublicIP)
    fmt.Printf("  Private IP: %s\n", inst.PrivateIP)
    fmt.Printf("  AZ: %s\n", inst.AvailabilityZone)
    if inst.Tags != nil {
        fmt.Printf("  Tags: %v\n", inst.Tags)
    }
}

// Run new instances
newInstances, err := manager.RunInstances(ctx, "ami-12345678", "t2.micro", 1, 1)
if err != nil {
    panic(err)
}

for _, inst := range newInstances {
    fmt.Printf("Launched: %s\n", inst.InstanceID)
}

// Terminate instances
err = manager.TerminateInstances(ctx, []string{"i-12345678", "i-87654321"})
if err != nil {
    panic(err)
}

// Start instances
err = manager.StartInstances(ctx, []string{"i-12345678"})
if err != nil {
    panic(err)
}

// Stop instances
err = manager.StopInstances(ctx, []string{"i-12345678"})
if err != nil {
    panic(err)
}

// Reboot instances
err = manager.RebootInstances(ctx, []string{"i-12345678"})
if err != nil {
    panic(err)
}
```

### Volume Operations

```go
// Describe volumes
volumes, err := manager.DescribeVolumes(ctx)
if err != nil {
    panic(err)
}

for _, vol := range volumes {
    fmt.Printf("Volume: %s (Size: %d GB, Type: %s)\n", #vol.VolumeID, vol.Size, vol.VolumeType)
    fmt.Printf("  State: %s, Encrypted: %v\n", vol.State, vol.Encrypted)
}

// Create volume
volume, err := manager.CreateVolume(ctx, 100, "gp3", "us-east-1a")
if err != nil {
    panic(err)
}
fmt.Printf("Created volume: %s\n", volume.VolumeID)

// Delete volume
err = manager.DeleteVolume(ctx, "vol-12345678")
if err != nil {
    panic(err)
}
```

### Snapshot Operations

```go
// Describe snapshots
snapshots, err := manager.DescribeSnapshots(ctx)
if err != nil {
    panic(err)
}

for _, snap := range snapshots {
    fmt.Printf("Snapshot: %s (Volume: %s, Size: %d GB)\n", #snap.SnapshotID, snap.VolumeID, snap.Size)
    fmt.Printf("  State: %s, Progress: %s\n", snap.State, snap.Progress)
}

// Create snapshot
snapshot, err := manager.CreateSnapshot(ctx, "vol-12345678", "Backup before update")
if err != nil {
    panic(err)
}
fmt.Printf("Created snapshot: %s\n", snapshot.SnapshotID)
```

### Security Group Operations

```go
// Describe security groups
groups, err := manager.DescribeSecurityGroups(ctx)
if err != nil {
    panic(err)
}

for _, sg := range groups {
    fmt.Printf("Security Group: %s (%s)\n", sg.GroupName, sg.GroupID)
    fmt.Printf("  Description: %s\n", sg.Description)
    fmt.Printf("  VPC: %s\n", sg.VPCID)
    if sg.InboundRules != nil {
        fmt.Println("  Inbound rules:")
        for _, rule := range sg.InboundRules {
            fmt.Printf("    Protocol: %s, Ports: %d-%d\n", #rule.Protocol, rule.FromPort, rule.ToPort)
        }
    }
}
```

### VPC and Subnet Operations

```go
// Describe VPCs
vpcs, err := manager.DescribeVPCs(ctx)
if err != nil {
    panic(err)
}

for _, vpc := range vpcs {
    fmt.Printf("VPC: %s (CIDR: %s, State: %s)\n", vpc.VPCID, vpc.CIDRBlock, vpc.State)
    fmt.Printf("  Default: %v\n", vpc.IsDefault)
}

// Describe subnets
subnets, err := manager.DescribeSubnets(ctx)
if err != nil {
    panic(err)
}

for _, subnet := range subnets {
    fmt.Printf("Subnet: %s (VPC: %s, AZ: %s)\n", #subnet.SubnetID, subnet.VPCID, subnet.AvailabilityZone)
    fmt.Printf("  CIDR: %s, Public: %v\n", subnet.CIDRBlock, subnet.IsPublic)
}
```

### Key Pair and AMI Operations

```go
// Describe key pairs
keyPairs, err := manager.DescribeKeyPairs(ctx)
if err != nil {
    panic(err)
}

for _, kp := range keyPairs {
    fmt.Printf("Key Pair: %s (Type: %s)\n", kp.KeyName, kp.KeyType)
}

// Describe images (owned by self)
images, err := manager.DescribeImages(ctx, []string{"self"})
if err != nil {
    panic(err)
}

for _, img := range images {
    fmt.Printf("AMI: %s (Name: %s)\n", img.ImageID, img.Name)
    fmt.Printf("  Description: %s\n", img.Description)
    fmt.Printf("  Architecture: %s\n", img.Architecture)
}
```

### Async Operations

```go
// Async describe instances
result := manager.DescribeInstancesAsync(ctx)
select {
case instances := <-result.Ch:#    fmt.Printf("Found %d instances\n", len(instances))
case err := <-result.ErrCh:#    fmt.Printf("Error: %v\n", err)
}

// Async describe volumes
result = manager.DescribeVolumesAsync(ctx)
select {
case volumes := <-result.Ch:#    fmt.Printf("Found %d volumes\n", len(volumes))
case err := <-result.ErrCh:#    fmt.Printf("Error: %v\n", err)
}

// Other async methods:
// - DescribeSnapshotsAsync
// - DescribeSecurityGroupsAsync
// - DescribeVPCsAsync
// - DescribeSubnetsAsync
// - DescribeKeyPairsAsync
// - DescribeImagesAsync
```

### Get Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
fmt.Printf("Region: %s\n", status["region"])
fmt.Printf("Instances: %v\n", status["instances"])

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
| `github.com/spf13/viper` | Configuration management for AWS credentials |
| `stackyrd/config` | Internal configuration package |
| `stackyrd/pkg/logger` | Structured logging |
| Standard library | `context`, `encoding/json`, `io`, `time`, `fmt`, `crypto/hmac`, `crypto/sha256` |

## Error Handling

The library uses Go error handling patterns:

- **Connection failures**: `GetStatus()` validates connectivity via `DescribeInstances()`
- **API errors**: HTTP status codes are checked and errors returned with response body
- **AWS errors**: Response contains AWS error codes and messages
- **Context cancellation**: All operations respect `context.Context` for timeouts
- **Nil checks**: Public methods handle nil receiver gracefully in `GetStatus()`

Example error handling:

```go
instances, err := manager.DescribeInstances(ctx)
if err != nil {
    if strings.Contains(err.Error(), "AuthFailure") {
        return fmt.Errorf("authentication failed: %w", err)
    }
    if strings.Contains(err.Error(), "InvalidInstanceID.NotFound") {
        return fmt.Errorf("instance not found: %w", err)
    }
    return fmt.Errorf("failed to describe instances: %w", err)
}
```

## Common Pitfalls

### 1. AWS Credentials Not Configured

**Problem**: `AuthFailure` or `SignatureDoesNotMatch` errors

**Solution**: 
- Configure credentials: `viper.Set("aws_ec2.access_key_id", "AKIA...")`
- Configure secret: `viper.Set("aws_ec2.secret_access_key", "secret...")`
- Or use AWS CLI: `aws configure`
- Or set environment variables: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`

### 2. Wrong Region

**Problem**: Resources not found or empty results

**Solution**:
- Check region: `viper.Set("aws_ec2.region", "us-west-2")`
- Verify resources exist in the specified region
- Common regions: us-east-1, us-west-2, eu-west-1

### 3. Instance Not Found

**Problem**: `InvalidInstanceID.NotFound` error

**Solution**:
- Verify instance ID format: `i-12345678abcdef01`
- Check instance exists in the region
- Use `DescribeInstances()` to list available instances

### 4. Context Timeout

**Problem**: Operations timing out

**Solution**:
```go
ctx, cancel := context.WithTimeout(context.Background(), 60*time.Second)
defer cancel()
// Operations will use this timeout (client has 60s default)
```

### 5. AWS v4 Signing Issues

**Problem**: Signature validation failures

**Solution**:
- Ensure system clock is synchronized (uses UTC time)
- Verify region in signing scope matches configured region
- Check that Host header matches the endpoint

### 6. Insufficient Permissions

**Problem**: `UnauthorizedOperation` error

**Solution**:
- Attach appropriate IAM policy to the user/role
- Required actions: `ec2:DescribeInstances`, `ec2:RunInstances`, etc.
- Check IAM policy simulator

## Advanced Usage

### Monitoring Instances

```go
func monitorInstances(manager *ec2.AWSEC2Manager) {
    ticker := time.NewTicker(30 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:#            instances, err := manager.DescribeInstances(context.Background())
            if err != nil {
                fmt.Printf("Error: %v\n", err)
                continue
            }
            
            fmt.Printf("\n=== EC2 Instances (%d) ===\n", len(instances))
            for _, inst := range instances {
                fmt.Printf("  %s: %s (%s)\n", inst.InstanceID, inst.State, inst.InstanceType)
            }
        }
    }
}
```

### Instance Health Check

```go
func checkInstanceHealth(manager *ec2.AWSEC2Manager, instanceID string) bool {
    ctx := context.Background()
    
    instances, err := manager.DescribeInstances(ctx)
    if err != nil {
        return false
    }
    
    for _, inst := range instances {
        if inst.InstanceID == instanceID {
            if inst.State == "running" {
                return inst.PublicIP != "" || inst.PrivateIP != ""
            }
        }
    }
    
    return false
}
```

### Batch Operations

```go
func batchTerminate(manager *ec2.AWSEC2Manager, filter string) error {
    ctx := context.Background()
    
    // Get all instances
    instances, err := manager.DescribeInstances(ctx)
    if err != nil {
        return err
    }
    
    // Filter instances to terminate
    var toTerminate []string
    for _, inst := range instances {
        if strings.Contains(inst.InstanceID, filter) || strings.Contains(inst.InstanceType, filter) {
            toTerminate = append(toTerminate, inst.InstanceID)
        }
    }
    
    if len(toTerminate) == 0 {
        fmt.Println("No instances to terminate")
        return nil
    }
    
    fmt.Printf("Terminating %d instances...\n", len(toTerminate))
    return manager.TerminateInstances(ctx, toTerminate)
}
```

### Custom API Requests

While the manager provides high-level methods, you can extend it for custom API calls:

```go
// Example: Get instance status (not directly exposed)
func getInstanceStatus(manager *ec2.AWSEC2Manager, ctx context.Context, instanceID string) (string, error) {
    params := map[string]string{
        "InstanceId.1": instanceID,
    }
    
    data, err := manager.ApiRequest(ctx, "DescribeInstanceStatus", params)
    if err != nil {
        return "", err
    }
    
    // Parse response
    // ... (implementation depends on AWS response structure)
    
    return "ok", nil
}
```

## Internal Algorithms

### AWS v4 Signing Algorithm

```
signRequest(req):
    │
    ├── dateStamp = now.Format("20060102")
    ├── amzDate = now.Format("20060102T150405Z")
    │
    ├── Set headers:
    │   ├── Host = req.URL.Host
    │   ├── X-Amz-Date = amzDate
    │   └── If sessionToken: X-Amz-Security-Token = sessionToken
    │
    └── If accessKey && secretKey:
            └── Authorization = "AWS4-HMAC-SHA256 Credential={accessKey}/{dateStamp}/{region}/ec2/aws4_request, SignedHeaders=host;x-amz-date, Signature={signature}"
```

### URL Construction

```
BaseURL Format: https://ec2.{region}.amazonaws.com

Example:
  region: us-east-1
  URL: https://ec2.us-east-1.amazonaws.com
```

### API Request Format

```
URL: baseURL/?Action={Action}&Version=2016-11-15&{params...}

Example:
  Action: DescribeInstances
  URL: https://ec2.us-east-1.amazonaws.com/?Action=DescribeInstances&Version=2016-11-15
```

### Response Parsing

The manager uses JSON throughout (AWS EC2 API returns XML by default, but this implementation uses JSON):

**Instance list** (returns object with Reservations):
```json
{
  "Reservations": [
    {
      "Instances": [
        {
          "InstanceId": "i-12345678",
          "InstanceType": "t2.micro",
          "State": "running"
        }
      ]
    }
  ]
}
→ Extracted to []EC2Instance
```

### Retry Logic

Using `go-retryablehttp`:
- Maximum retries: 3
- Minimum wait: 1 second
- Maximum wait: 10 seconds
- HTTP client timeout: 60 seconds

Retries are automatic for transient network errors.

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.