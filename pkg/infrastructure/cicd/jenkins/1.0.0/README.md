# Jenkins Manager

## Overview

The `JenkinsManager` is a comprehensive Go library for Jenkins CI/CD integration. It provides job management, pipeline execution, build monitoring, artifact retrieval, and notification support with worker pool-based async job processing.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Job Management**: Create, queue, execute, list, and delete Jenkins jobs
- **Pipeline Support**: Create and execute pipeline configurations with parameters and triggers
- **Build Monitoring**: Retrieve build information, logs, artifacts, and stage details
- **Queue System**: Asynchronous job processing with priority support
- **Notifications**: Email, Slack, and webhook notifications for job completion
- **Metrics Tracking**: Track job/build statistics, success/failure rates, and resource usage
- **Worker Pool**: Async job execution with configurable concurrency
- **Authentication**: Basic auth, API key support, and TLS configuration
- **Webhook Support**: Event-driven architecture for job status updates

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
    
    // Create Jenkins manager (configuration via viper)
    manager, err := jenkins.NewJenkinsManager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // Create a job
    job := &jenkins.JenkinsJob{
        Name:      "my-build",
        Type:      "freestyle",
        Parameters: map[string]interface{}{
            "BRANCH": "main",
        },
        Priority:  1,
    }
    
    err = manager.CreateJob(job)
    if err != nil {
        panic(err)
    }
    
    fmt.Printf("Job created: %s\n", job.ID)
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `JenkinsManager` | Main manager with HTTP client, worker pool, job queue, and metrics |
| `JenkinsConfig` | Configuration struct with host, auth, notifications, and pipeline templates |
| `JenkinsJob` | Represents a Jenkins job with parameters, status, and timing info |
| `JenkinsResult` | Result of job execution with status, output, artifacts |
| `JenkinsMetrics` | Metrics tracking for jobs, builds, and resource usage |
| `PipelineConfig` | Pipeline configuration with definition, parameters, triggers |
| `BuildInfo` | Build details with stages, artifacts, and logs |
| `BuildStage` | Individual build stage with timing and status |
| `Artifact` | Build artifact with path, size, and download URL |
| `WebhookEvent` | Webhook event for event-driven updates |
| `NotificationConfig` | Notification settings for email, Slack, and webhooks |
| `EmailConfig` | Email notification configuration |
| `SlackConfig` | Slack notification configuration |

### Concurrency Model

```
┌─────────────────────────────────────────────────────────────┐
│                   JenkinsManager                             │
├─────────────────────────────────────────────────────────────┤
│  ┌────────────────┐                                   │
│  │ retryablehttp  │  → Retry logic (max configurable)   │
│  │ Client         │  → TLS support (insecure skip option) │
│  └────────────────┘                                   │
│                                                          │
│  ┌─────────────┐      ┌──────────────┐                  │
│  │  WorkerPool │◄─────│ processJobs  │                  │
│  │  (configurable)│      │ goroutine    │                  │
│  └─────────────┘      └──────────────┘                  │
│         ▲                      │                           │
│         │                      ▼                           │
│  ┌─────────────┐      ┌──────────────┐                  │
│  │  JobQueue   │─────▶│ ExecuteJob  │                  │
│  │  (channel)  │      │ goroutine   │                  │
│  └─────────────┘      └──────────────┘                  │
│                                                          │
│  ┌─────────────┐      ┌──────────────┐                  │
│  │  Results    │─────▶│ monitorJob  │                  │
│  │  (channel)  │      │ Completion  │                  │
│  └─────────────┘      └──────────────┘                  │
│                                                          │
│  ┌─────────────┐                                   │
│  │ webhookChan │───▶ processWebhooks goroutine      │
│  └─────────────┘                                   │
└─────────────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewJenkinsManager()
    │
    ├── Check viper config: "jenkins.enabled"
    ├── loadJenkinsConfig()
    │   ├── Host, Port, UseHTTPS
    │   ├── Username, Password, APIKey
    │   ├── Timeout, MaxRetries
    │   ├── InsecureSkipTLS
    │   ├── DefaultQueueSize, MaxConcurrentJobs
    │   ├── Notification config (Email, Slack, Webhooks)
    │   └── Pipeline templates
    ├── Build baseURL: protocol://host:port
    ├── Create retryablehttp.Client (configurable retries)
    ├── Configure TLS if InsecureSkipTLS
    ├── Create channels: JobQueue, Results, webhookChan
    ├── Initialize maps: jobs, pipelines, builds
    ├── Create WorkerPool(MaxConcurrentJobs)
    ├── Start worker pool
    ├── Start processJobs() goroutine
    ├── Start processWebhooks() goroutine
    └── Return JenkinsManager
```

### 2. Job Creation Flow

```
CreateJob(job)
    │
    ├── Validate job (not nil, name not empty)
    ├── createJobConfig(job)
    │   ├── Check for template in config.PipelineTemplates
    │   └── If no template: create default freestyle config
    ├── POST to {baseURL}/createItem?name={job.Name}
    │   ├── Set Content-Type: application/xml
    │   ├── Add auth header (API key or Basic auth)
    │   └── Send XML configuration
    ├── If status != 200: return error
    ├── Generate job ID: "job_{name}_{timestamp}"
    ├── Set job status: "pending"
    ├── Store in jobs map
    ├── Queue job: JobQueue <- job
    └── Return nil
```

### 3. Job Execution Flow

```
processJobs() goroutine
    │
    └── for job := range JobQueue:
            │
            └── go func(job):
                    │
                    ├── ExecuteJob(job.ID)
                    │   ├── Build parameters from job.Parameters
                    │   ├── POST to {baseURL}/job/{name}/buildWithParameters?{params}
                    │   ├── Add auth header
                    │   └── If status != 201: return error
                    │
                    ├── Update job status: "running"
                    └── monitorJobCompletion(job)
                            │
                            └── Ticker (every 5s):
                                    │
                                    ├── GetLatestBuild(job.Name)
                                    │   └── GET {baseURL}/job/{name}/lastBuild/api/json
                                    │
                                    ├── If build status != BUILDING/RUNNING:
                                    │   ├── Calculate duration
                                    │   ├── Create JenkinsResult
                                    │   ├── Update job: status, completedAt, duration
                                    │   ├── Update metrics
                                    │   ├── Send result to Results channel
                                    │   ├── Send notification if enabled
                                    │   ├── Send webhook event
                                    │   └── return
                                    │
                                    └── Continue monitoring
```

### 4. Pipeline Execution Flow

```
ExecutePipeline(name, parameters)
    │
    ├── Create JenkinsJob from pipeline
    │   ├── ID: generateJobID(name)
    │   ├── Name: name
    │   ├── Type: "pipeline"
    │   ├── Parameters: parameters
    │   ├── Status: "pending"
    │   └── CreatedAt: now
    ├── Store in jobs map
    ├── Queue job: JobQueue <- job
    └── Return nil
```

### 5. Build Info Retrieval Flow

```
GetLatestBuild(jobName)
    │
    ├── GET {baseURL}/job/{jobName}/lastBuild/api/json
    ├── Add auth header
    ├── If status != 200: return error
    ├── JSON decode to BuildInfo
    └── Return BuildInfo
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `jenkins.enabled` | bool | false | Enable/disable Jenkins manager |
| `jenkins.host` | string | "" | Jenkins server hostname |
| `jenkins.port` | int | 8080 | Jenkins server port |
| `jenkins.use_https` | bool | false | Use HTTPS protocol |
| `jenkins.username` | string | "" | Username for Basic auth |
| `jenkins.password` | string | "" | Password for Basic auth |
| `jenkins.api_key` | string | "" | API key (Jenkins Crumb) |
| `jenkins.timeout` | int | 30 | HTTP client timeout (seconds) |
| `jenkins.max_retries` | int | 3 | Max retry attempts |
| `jenkins.insecure_skip_tls` | bool | false | Skip TLS verification |
| `jenkins.default_queue_size` | int | 100 | Default job queue size |
| `jenkins.max_concurrent_jobs` | int | 10 | Max concurrent job executions |
| `jenkins.notification.enabled` | bool | false | Enable notifications |
| `jenkins.notification.webhooks` | []string | [] | Webhook URLs |
| `jenkins.notification.email.enabled` | bool | false | Enable email notifications |
| `jenkins.notification.email.smtp_host` | string | "" | SMTP server host |
| `jenkins.notification.email.smtp_port` | int | 587 | SMTP server port |
| `jenkins.notification.email.from` | string | "" | Email sender |
| `jenkins.notification.email.to` | []string | [] | Email recipients |
| `jenkins.notification.slack.enabled` | bool | false | Enable Slack notifications |
| `jenkins.notification.slack.webhook` | string | "" | Slack webhook URL |
| `jenkins.notification.slack.channel` | string | "" | Slack channel |
| `jenkins.pipeline_templates` | map[string]string | {} | Pipeline template XMLs |

### Example Configuration (YAML)

```yaml
jenkins:
  enabled: true
  host: "jenkins.example.com"
  port: 8080
  use_https: true
  username: "admin"
  password: "secret"
  api_key: ""
  timeout: 60
  max_retries: 5
  insecure_skip_tls: false
  default_queue_size: 200
  max_concurrent_jobs: 20
  notification:
    enabled: true
    webhooks:
      - "https://my-webhook.com/jenkins"
    email:
      enabled: true
      smtp_host: "smtp.gmail.com"
      smtp_port: 587
      from: "ci@example.com"
      to:
        - "team@example.com"
    slack:
      enabled: true
      webhook: "https://hooks.slack.com/..."
      channel: "ci-cd"
  pipeline_templates:
    nodejs: |
      <?xml version='1.0' encoding='UTF-8'?>
      <project>
        <description>Node.js Pipeline</description>
        ...
```

## Usage Examples

### Basic Job Operations

```go
ctx := context.Background()

// Create a freestyle job
job := &jenkins.JenkinsJob{
    Name:      "build-app",
    Type:      "freestyle",
    Parameters: map[string]interface{}{
        "BRANCH": "main",
        "ENVIRONMENT": "production",
    },
    Priority:  1,
}

err := manager.CreateJob(job)
if err != nil {
    panic(err)
}
fmt.Printf("Created job: %s (ID: %s)\n", job.Name, job.ID)

// List all jobs
jobs := manager.ListJobs()
fmt.Printf("Total jobs: %d\n", len(jobs))
for _, j := range jobs {
    fmt.Printf("  - %s (Status: %s)\n", j.Name, j.Status)
}

// Get specific job
job, err := manager.GetJob(job.ID)
if err != nil {
    panic(err)
}
fmt.Printf("Job: %s, Created: %v\n", job.Name, job.CreatedAt)

// Queue a job for execution
err = manager.QueueJob(job.ID)
if err != nil {
    panic(err)
}

// Execute job immediately
err = manager.ExecuteJob(job.ID)
if err != nil {
    panic(err)
}

// Delete a job
err = manager.DeleteJob(job.ID)
if err != nil {
    panic(err)
}
```

### Pipeline Operations

```go
// Create a pipeline configuration
pipeline := &jenkins.PipelineConfig{
    Name:       "deploy-pipeline",
    Definition: `
        pipeline {
            agent any
            stages {
                stage('Build') {
                    steps {
                        sh 'npm install'
                        sh 'npm run build'
                    }
                }
                stage('Test') {
                    steps {
                        sh 'npm test'
                    }
                }
                stage('Deploy') {
                    steps {
                        sh 'npm run deploy'
                    }
                }
            }
        }
    `,
    Parameters: map[string]interface{}{
        "DEPLOY_ENV": "staging",
    },
    Environment: map[string]string{
        "NODE_ENV": "production",
    },
    Triggers: []jenkins.TriggerConfig{
        {Type: "poll", Enabled: true, Config: map[string]interface{}{"cron": "0 * * * *"}},
    },
    Timeout: 30 * time.Minute,
    Retries: 2,
}

err := manager.CreatePipeline(pipeline)
if err != nil {
    panic(err)
}

// Execute pipeline
err = manager.ExecutePipeline("deploy-pipeline", map[string]interface{}{
    "DEPLOY_ENV": "production",
})
if err != nil {
    panic(err)
}

// List all pipelines
pipelines := manager.ListPipelines()
fmt.Printf("Total pipelines: %d\n", len(pipelines))
```

### Build Information

```go
// Get latest build for a job
build, err := manager.GetLatestBuild("build-app")
if err != nil {
    panic(err)
}

fmt.Printf("Build #%d: %s (Result: %s)\n", 
    build.Number, build.Status, build.Result)
fmt.Printf("Duration: %v\n", build.Duration)
fmt.Printf("Timestamp: %v\n", build.Timestamp)

// Get specific build info
build, err = manager.GetBuild(123)
if err != nil {
    panic(err)
}

// Get build logs
logs, err := manager.GetBuildLogs("build-app", build.Number)
if err != nil {
    panic(err)
}
fmt.Printf("Build logs:\n%s\n", logs)

// Get build artifacts
artifacts, err := manager.GetBuildArtifacts("build-app", build.Number)
if err != nil {
    panic(err)
}

fmt.Printf("Artifacts (%d):\n", len(artifacts))
for _, art := range artifacts {
    fmt.Printf("  - %s (%s, %d bytes)\n", art.Name, art.Path, art.Size)
    fmt.Printf("    Download: %s\n", art.DownloadURL)
}

// Get build stages
if build.Stages != nil {
    fmt.Println("Build stages:")
    for _, stage := range build.Stages {
        fmt.Printf("  - %s: %s (%.1f%%)\n", 
            stage.Name, stage.Status, stage.Progress*100)
    }
}
```

### Job Monitoring

```go
// Get queued jobs
queued := manager.GetQueuedJobs()
fmt.Printf("Queued jobs: %d\n", len(queued))

// Get running jobs
running := manager.GetRunningJobs()
fmt.Printf("Running jobs: %d\n", len(running))

// Get completed jobs
completed := manager.GetCompletedJobs()
fmt.Printf("Completed jobs: %d\n", len(completed))
```

### Metrics and Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
fmt.Printf("Base URL: %s\n", status["base_url"])
fmt.Printf("Total jobs: %v\n", status["total_jobs"])
fmt.Printf("Total builds: %v\n", status["total_builds"])
fmt.Printf("Pending jobs: %v\n", status["pending_jobs"])

// Get metrics
metrics := manager.GetMetrics()
fmt.Printf("\n=== Jenkins Metrics ===\n")
fmt.Printf("Total Jobs: %d\n", metrics.TotalJobs)
fmt.Printf("Successful Jobs: %d\n", metrics.SuccessfulJobs)
fmt.Printf("Failed Jobs: %d\n", metrics.FailedJobs)
fmt.Printf("Total Builds: %d\n", metrics.TotalBuilds)
fmt.Printf("Successful Builds: %d\n", metrics.SuccessfulBuilds)
fmt.Printf("Failed Builds: %d\n", metrics.FailedBuilds)
fmt.Printf("Average Build Time: %v\n", metrics.AverageBuildTime)
fmt.Printf("Last Activity: %v\n", metrics.LastActivity)
fmt.Printf("CPU Usage: %.2f%%\n", metrics.CPUTimeUsage)
fmt.Printf("Memory Usage: %.2f MB\n", metrics.MemoryUsage)
fmt.Printf("Disk Usage: %.2f MB\n", metrics.DiskUsage)
```

### Async Job Processing

The manager processes jobs asynchronously using a worker pool:

```go
// Jobs are automatically processed by the worker pool
// You can monitor results via the Results channel (internal)

// The processJobs() goroutine:
//   - Listens on JobQueue channel
//   - Executes jobs concurrently (up to MaxConcurrentJobs)
//   - Monitors job completion
//   - Sends results to Results channel
//   - Triggers notifications and webhooks
```

## Dependencies

| Dependency | Role |
|------------|------|
| `github.com/hashicorp/go-retryablehttp` | HTTP client with retry logic for resilience |
| `github.com/spf13/viper` | Configuration management |
| `stackyrd/config` | Internal configuration package |
| `stackyrd/pkg/logger` | Structured logging |
| Standard library | `context`, `encoding/json`, `crypto/tls`, `net/http`, `sync`, `time`, `io`, `fmt`, `strings` |

## Error Handling

The library uses Go error handling patterns:

- **Connection failures**: `testConnection()` validates connectivity on status check
- **API errors**: HTTP status codes are checked and errors returned with response body
- **Job not found**: Returns `fmt.Errorf("job not found: %s", jobID)`
- **Nil checks**: Public methods handle nil receiver gracefully in `GetStatus()`
- **Auth errors**: Handled via `addAuthHeader()` with API key or Basic auth

Example error handling:

```go
job, err := manager.GetJob(jobID)
if err != nil {
    if strings.Contains(err.Error(), "not found") {
        // Job doesn't exist
        return nil
    }
    return fmt.Errorf("failed to get job: %w", err)
}
```

## Common Pitfalls

### 1. Jenkins Not Running

**Problem**: Connection test fails

**Solution**: 
- Verify Jenkins is running: `systemctl status jenkins` or check URL in browser
- Check host/port configuration
- Verify firewall allows connections

### 2. Authentication Failed

**Problem**: 401/403 errors

**Solution**:
- Configure correct username/password for Basic auth
- Or set API key (Jenkins Crumb): `jenkins.api_key`
- Verify user has necessary permissions

### 3. TLS/HTTPS Issues

**Problem**: Certificate validation errors

**Solution**:
- Set `jenkins.use_https: true` for HTTPS
- For self-signed certs: `jenkins.insecure_skip_tls: true` (development only!)
- Or configure proper CA certificates

### 4. Job Already Exists

**Problem**: CreateJob fails with conflict

**Solution**:
- Check if job exists before creating
- Use unique job names or delete existing job first
- Consider using timestamps or unique identifiers

### 5. Pipeline Definition Missing

**Problem**: CreatePipeline fails

**Solution**:
- Ensure `PipelineConfig.Definition` is not empty
- Provide valid Jenkins pipeline script (Groovy)
- Check pipeline syntax

### 6. Notification Failures

**Problem**: Notifications not sent

**Solution**:
- Enable notifications: `jenkins.notification.enabled: true`
- Configure email/Slack/webhook settings
- Check SMTP settings for email
- Verify webhook URLs are reachable

### 7. Worker Pool Exhaustion

**Problem**: Jobs not executing

**Solution**:
- Increase `jenkins.max_concurrent_jobs`
- Monitor pending jobs: `manager.GetQueuedJobs()`
- Check for stalled jobs

## Advanced Usage

### Custom Job Configuration

```go
// Create job with custom XML configuration
func createCustomJob(manager *jenkins.JenkinsManager) error {
    customConfig := `
        <?xml version='1.0' encoding='UTF-8'?>
        <project>
          <description>Custom Job</description>
          <builders>
            <hudson.tasks.Shell>
              <command>
                echo "Custom build"
                ./build.sh
              </command>
            </hudson.tasks.Shell>
          </builders>
          <publishers>
            <hudson.tasks.junit.JUnitResultArchiver>
              <testResultsPattern>**/target/surefire-reports/*.xml</testResultsPattern>
            </hudson.tasks.junit.JUnitResultArchiver>
          </publishers>
        </project>
    `
    
    // Use template or directly set in createJobConfig
    job := &jenkins.JenkinsJob{
        Name: "custom-build",
        Type: "custom",
    }
    
    // Store template for use in createJobConfig
    // manager.config.PipelineTemplates["custom"] = customConfig
    
    return manager.CreateJob(job)
}
```

### Build Monitoring with Stages

```go
func monitorBuild(manager *jenkins.JenkinsManager, jobName string) {
    // Get initial build info
    build, err := manager.GetLatestBuild(jobName)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    
    fmt.Printf("Monitoring build #%d\n", build.Number)
    
    // Poll for completion
    for {
        build, err = manager.GetBuild(build.Number)
        if err != nil {
            fmt.Printf("Error: %v\n", err)
            return
        }
        
        fmt.Printf("Status: %s, Result: %s\n", build.Status, build.Result)
        
        // Check stages
        if build.Stages != nil {
            for _, stage := range build.Stages {
                fmt.Printf("  Stage: %s (%s, %.1f%%)\n",
                    stage.Name, stage.Status, stage.Progress*100)
            }
        }
        
        if build.Status != "BUILDING" && build.Status != "RUNNING" {
            fmt.Printf("Build completed with result: %s\n", build.Result)
            break
        }
        
        time.Sleep(5 * time.Second)
    }
}
```

### Artifact Download

```go
func downloadArtifacts(manager *jenkins.JenkinsManager, jobName string, buildNumber int) {
    artifacts, err := manager.GetBuildArtifacts(jobName, buildNumber)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    
    for _, art := range artifacts {
        fmt.Printf("Downloading %s...\n", art.Name)
        
        // Download via URL
        resp, err := http.Get(art.DownloadURL)
        if err != nil {
            fmt.Printf("  Error: %v\n", err)
            continue
        }
        defer resp.Body.Close()
        
        // Save to file
        data, _ := io.ReadAll(resp.Body)
        os.WriteFile(art.Name, data, 0644)
        fmt.Printf("  Saved to %s\n", art.Name)
    }
}
```

### Metrics Dashboard

```go
func printMetricsDashboard(manager *jenkins.JenkinsManager) {
    metrics := manager.GetMetrics()
    
    fmt.Println("╔════════════════════════════════════════╗")
    fmt.Println("║       Jenkins Metrics Dashboard       ║")
    fmt.Println("╠════════════════════════════════════════╣")
    fmt.Printf("║ Total Jobs:        %-16d  ║\n", metrics.TotalJobs)
    fmt.Printf("║ Successful Jobs:   %-16d  ║\n", metrics.SuccessfulJobs)
    fmt.Printf("║ Failed Jobs:       %-16d  ║\n", metrics.FailedJobs)
    fmt.Printf("║ Total Builds:      %-16d  ║\n", metrics.TotalBuilds)
    fmt.Printf("║ Successful Builds: %-16d  ║\n", metrics.SuccessfulBuilds)
    fmt.Printf("║ Failed Builds:     %-16d  ║\n", metrics.FailedBuilds)
    fmt.Println("╠════════════════════════════════════════╣")
    fmt.Printf("║ Avg Build Time:   %-16v  ║\n", metrics.AverageBuildTime)
    fmt.Printf("║ CPU Usage:         %-16.2f%% ║\n", metrics.CPUTimeUsage)
    fmt.Printf("║ Memory Usage:      %-16.2fMB ║\n", metrics.MemoryUsage)
    fmt.Printf("║ Disk Usage:        %-16.2fMB ║\n", metrics.DiskUsage)
    fmt.Println("╚════════════════════════════════════════╝")
}
```

### Webhook Event Handling

```go
// The manager sends webhook events for:
//   - job_completed
//   - job_failed
//   - (extensible for more events)

// Webhook payload structure:
{
    "job_id": "job_build-app_1234567890",
    "job_name": "build-app",
    "status": "completed",
    "build_id": 123,
    "duration": "5m30s",
    "timestamp": "2026-04-29T00:01:23Z",
    "error": "",
    "artifacts": [...],
    "metadata": {...}
}
```

## Internal Algorithms

### Job ID Generation

```
generateJobID(name)
    │
    └── Return: fmt.Sprintf("job_%s_%d", name, time.Now().UnixNano())
            │
            Example: "job_my-build_1714387200000000000"
```

### Authentication Header Logic

```
addAuthHeader(req)
    │
    ├── If APIKey != "":
    │   └── req.Header.Set("Jenkins-Crumb", APIKey)
    │
    └── If Username != "" && Password != "":
        └── req.SetBasicAuth(Username, Password)
```

### Notification Dispatch

```
sendNotification(result)
    │
    ├── If Notification.Email.Enabled:
    │   └── go sendEmailNotification(job, result, message)
    │
    ├── If Notification.Slack.Enabled:
    │   └── go sendSlackNotification(job, result, message)
    │
    └── For each webhook in Notification.Webhooks:
        └── go sendWebhookNotification(webhook, job, result)
```

### Metrics Update on Job Completion

```
handleJobFailure() or monitorJobCompletion():
    │
    ├── Lock metrics mutex
    ├── Update counters (TotalJobs, SuccessfulJobs, FailedJobs)
    ├── Update build counters (TotalBuilds, SuccessfulBuilds, FailedBuilds)
    ├── Calculate average build time
    ├── Update LastActivity
    └── Unlock metrics mutex
```

### Job Queue Processing

```
processJobs() goroutine:
    │
    └── for job := range JobQueue:
            │
            └── go func(job):
                    │
                    ├── ExecuteJob(job.ID)
                    │   └── If error: handleJobFailure()
                    │
                    └── monitorJobCompletion(job)
                        │
                        └── Ticker loop (5s interval):
                                │
                                ├── GetLatestBuild(job.Name)
                                ├── If build complete:
                                │   ├── Create JenkinsResult
                                │   ├── Update job status
                                │   ├── Update metrics
                                │   ├── Send to Results channel
                                │   ├── Send notification
                                │   ├── Send webhook
                                │   └── return
                                │
                                └── Continue monitoring
```

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.