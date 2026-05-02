# Google Magica AI Manager

## Overview

The `GoogleMagicaManager` is a Go library for interacting with Google's Magica AI image generation and editing services. It provides text-to-image generation, image editing, upscaling, style transfer, image variations, and batch processing. The library uses Google Cloud AI Platform API with Bearer token authentication and includes retry logic for reliability.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Text-to-Image**: Generate images from text prompts
- **Image Editing**: Edit images with prompts and optional masks
- **Image Upscaling**: Upscale images with scale factor and detail enhancement
- **Style Transfer**: Apply artistic styles to images
- **Image Variations**: Generate variations of existing images
- **Batch Generation**: Generate multiple images from multiple prompts
- **Image Upload**: Helper to encode images to base64
- **Quota Information**: Retrieve quota and rate limit info
- **Retry Logic**: HTTP client with retry support (3 retries)
- **Worker Pool**: Async job execution support (6 workers)
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
    
    // Create Google Magica manager (configuration via viper)
    manager, err := magica.NewGoogleMagicaManager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // Generate a simple image
    response, err := manager.GenerateImageSimple(ctx, "A beautiful sunset over mountains")
    if err != nil {
        panic(err)
    }
    
    for _, img := range response.Images {
        fmt.Printf("Generated image: %s (MIME: %s, Size: %dx%d)\n", 
            img.Base64Data[:50]+"...", img.MIMEType, img.Width, img.Height)
    }
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `GoogleMagicaManager` | Main manager with HTTP client, API key, worker pool |
| `MagicaImagePrompt` | Text prompt with negative prompt, aspect ratio, guidance scale, seed, style preset, safety filter, output format |
| `MagicaEditRequest` | Image editing request with prompt, mask prompt, edit mode, number of images, guidance scale |
| `MagicaUpscaleRequest` | Image upscaling request with scale factor and detail enhancement |
| `MagicaImageResponse` | Response containing generated images and metadata |
| `MagicaImage` | Single generated image with base64 data, URL, MIME type, dimensions, NSFW filter |
| `MagicaStyleTransferRequest` | Style transfer request with reference image, style intensity, preserve content |
| `MagicaVariationRequest` | Image variation request with reference image, number of variations, diversity scale |
| `MagicaBatchJob` | Batch generation job with status, prompts, results |
| `MagicaQuotaInfo` | Quota and rate limit information |

### Concurrency Model

```
┌─────────────────────────────────────────────────────┐
│                 GoogleMagicaManager                   │
├─────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │  retryablehttp │  → API requests with retry               │
│  │  .Client      │  → Bearer token authentication           │
│  └────────────────┘                                      │
│                                                          │
│  ┌─────────────┐      ┌──────────────┐                   │
│  │  WorkerPool │◄─────│ AsyncResult │                   │
│  │  (6 workers)│      │   Channel   │                   │
│  └─────────────┘      └──────────────┘                   │
│         ▲                      │                            │
│         │                      ▼                            │
│  ┌─────────────┐      ┌──────────────┐                  │
│  │  SubmitJob  │      │ ExecuteAsync │                  │
│  └─────────────┘      └──────────────┘                  │
│                                                          │
│  BaseURL: https://{location}-aiplatform.googleapis.com/v1 │
│  APIKey: Google Cloud API key (Bearer token)              │
│  Model: imagen-3.0-generate-002                      │
│  ProjectID: Google Cloud project ID                       │
│  Location: us-central1 (default)                        │
└─────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewGoogleMagicaManager(logger)
    │
    ├── Check viper config: "google_magica.enabled"
    ├── Get api_key: "google_magica.api_key"
    ├── Get project_id: "google_magica.project_id"
    ├── Get location: "google_magica.location" (default: "us-central1")
    ├── Get model: "google_magica.model" (default: "imagen-3.0-generate-002")
    ├── Get max_images: "google_magica.max_images" (default: 4)
    │
    ├── Create retryablehttp.Client:
    │   ├── RetryMax = 3
    │   ├── RetryWaitMin = 1s
    │   ├── RetryWaitMax = 15s
    │   └── HTTPClient.Timeout = 180s
    │
    ├── Build baseURL: "https://{location}-aiplatform.googleapis.com/v1/projects/{projectID}/locations/{location}/publishers/google/models/{model}"
    │
    ├── Test connection: testConnection()
    │   └── GET /projects/{projectID}/locations/{location}/publishers/google/models
    │
    ├── Create WorkerPool(6)
    ├── Start worker pool
    └── Return GoogleMagicaManager
```

### 2. Generate Image Flow

```
GenerateImage(ctx, prompt)
    │
    ├── Build payload with instances and parameters:
    │   ├── instances: [{prompt: prompt.Text}]
    │   └── parameters: sampleCount, aspectRatio, guidanceScale, seed, stylePreset, safetyFilterLevel, outputFormat
    │
    ├── Marshal request to JSON
    ├── POST {baseURL}:predict
    ├── Set header: Authorization = Bearer <api_key>
    ├── Set header: Content-Type = application/json
    ├── Execute with retryablehttp.Client
    ├── Check status code (200 OK)
    ├── Decode JSON into MagicaImageResponse
    └── Return *MagicaImageResponse
```

### 3. Edit Image Flow

```
EditImage(ctx, base64Image, request)
    │
    ├── Build payload with instances and parameters:
    │   ├── instances: [{prompt: request.Prompt, image: {bytesBase64Encoded: base64Image}, mask_prompt: request.MaskPrompt}]
    │   └── parameters: sampleCount, guidanceScale, editMode
    │
    ├── Marshal request to JSON
    ├── POST {baseURL}:predict
    ├── Set headers (Authorization, Content-Type)
    ├── Execute with retryablehttp.Client
    ├── Decode JSON into MagicaImageResponse
    └── Return *MagicaImageResponse
```

### 4. Upscale Image Flow

```
UpscaleImage(ctx, base64Image, request)
    │
    ├── Build payload with instances and parameters:
    │   ├── instances: [{image: {bytesBase64Encoded: base64Image}}]
    │   └── parameters: scaleFactor, enhanceDetails
    │
    ├── POST {baseURL}:predict
    ├── Execute with retryablehttp.Client
    ├── Decode JSON into MagicaImageResponse
    └── Return first image as *MagicaImage
```

### 5. Style Transfer Flow

```
StyleTransfer(ctx, base64ContentImage, request)
    │
    ├── Build payload with instances and parameters:
    │   ├── instances: [{contentImage: {bytesBase64Encoded: base64ContentImage}, styleImage: {bytesBase64Encoded: request.ReferenceImage}}]
    │   └── parameters: styleIntensity, preserveContent
    │
    ├── POST {baseURL}:predict
    ├── Execute with retryablehttp.Client
    ├── Decode JSON into MagicaImageResponse
    └── Return *MagicaImageResponse
```

### 6. Generate Variations Flow

```
GenerateVariations(ctx, base64Image, request)
    │
    ├── Build payload with instances and parameters:
    │   ├── instances: [{image: {bytesBase64Encoded: base64Image}}]
    │   └── parameters: sampleCount (numVariations), diversityScale
    │
    ├── POST {baseURL}:predict
    ├── Execute with retryablehttp.Client
    ├── Decode JSON into MagicaImageResponse
    └── Return *MagicaImageResponse
```

### 7. Batch Generation Flow

```
GenerateBatch(ctx, prompts)
    │
    ├── For each prompt in prompts:
    │   └── Call GenerateImage(ctx, prompt)
    │
    ├── Collect results (continue on error)
    └── Return slice of *MagicaImageResponse
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `google_magica.enabled` | bool | false | Enable/disable Google Magica manager |
| `google_magica.api_key` | string | "" | Google Cloud API key |
| `google_magica.project_id` | string | "" | Google Cloud project ID |
| `google_magica.location` | string | "us-central1" | Google Cloud location |
| `google_magica.model` | string | "imagen-3.0-generate-002" | Magica model |
| `google_magica.max_images` | int | 4 | Maximum images per generation |

## Usage Examples

### Generate Image from Text Prompt

```go
ctx := context.Background()

prompt := magica.MagicaImagePrompt{
    Text:           "A futuristic city with flying cars",
    NegativePrompt: "blurry, low quality",
    AspectRatio:    "16:9",
    GuidanceScale:  7.5,
    Seed:           12345,
    StylePreset:    "photorealistic",
    SafetyFilter:   "block_only_high",
    OutputFormat:   "png",
}

response, err := manager.GenerateImage(ctx, prompt)
if err != nil {
    panic(err)
}

for i, img := range response.Images {
    fmt.Printf("Image %d: %dx%d, NSFW filtered: %v\n", 
        i, img.Width, img.Height, img.NSFWFiltered)
    // Save or process base64 data
}
```

### Simple Image Generation

```go
ctx := context.Background()

response, err := manager.GenerateImageSimple(ctx, "A cute cat playing with yarn")
if err != nil {
    panic(err)
}

if len(response.Images) > 0 {
    img := response.Images[0]
    fmt.Printf("Generated image: %s\n", img.Base64Data[:50]+"...")
}
```

### Edit an Image

```go
ctx := context.Background()

// Read and encode image
imageData := base64.StdEncoding.EncodeToString(imageBytes)

request := magica.MagicaEditRequest{
    Prompt:        "Make the sky more dramatic",
    MaskPrompt:    "sky",
    EditMode:      "inpainting",
    NumImages:     2,
    GuidanceScale: 8.0,
}

response, err := manager.EditImage(ctx, imageData, request)
if err != nil {
    panic(err)
}

fmt.Printf("Generated %d edited images\n", len(response.Images))
```

### Upscale an Image

```go
ctx := context.Background()

imageData := base64.StdEncoding.EncodeToString(imageBytes)

request := magica.MagicaUpscaleRequest{
    ScaleFactor:    2,
    EnhanceDetails: true,
}

image, err := manager.UpscaleImage(ctx, imageData, request)
if err != nil {
    panic(err)
}

fmt.Printf("Upscaled image: %dx%d\n", image.Width, image.Height)
```

### Style Transfer

```go
ctx := context.Background()

contentImageData := base64.StdEncoding.EncodeToString(contentBytes)
styleImageData := base64.StdEncoding.EncodeToString(styleBytes)

request := magica.MagicaStyleTransferRequest{
    ReferenceImage:  styleImageData,
    StyleIntensity:  0.8,
    PreserveContent: true,
}

response, err := manager.StyleTransfer(ctx, contentImageData, request)
if err != nil {
    panic(err)
}

fmt.Printf("Style transfer complete, generated %d images\n", len(response.Images))
```

### Generate Variations

```go
ctx := context.Background()

imageData := base64.StdEncoding.EncodeToString(imageBytes)

request := magica.MagicaVariationRequest{
    ReferenceImage: imageData,
    NumVariations:  4,
    DiversityScale:  0.7,
}

response, err := manager.GenerateVariations(ctx, imageData, request)
if err != nil {
    panic(err)
}

fmt.Printf("Generated %d variations\n", len(response.Images))
```

### Batch Generation

```go
ctx := context.Background()

prompts := []magica.MagicaImagePrompt{
    {Text: "A mountain landscape"},
    {Text: "A beach sunset"},
    {Text: "A forest path"},
}

results, err := manager.GenerateBatch(ctx, prompts)
if err != nil {
    panic(err)
}

for i, result := range results {
    if result != nil {
        fmt.Printf("Prompt %d generated %d images\n", i, len(result.Images))
    }
}
```

### Upload Image Helper

```go
// Upload image and get base64 string
imageData, err := manager.UploadImage(imageBytes, "image/png")
if err != nil {
    panic(err)
}
fmt.Printf("Image encoded: %s...\n", imageData[:50])
```

### Get Quota Information

```go
ctx := context.Background()

quota, err := manager.GetQuotaInfo(ctx)
if err != nil {
    panic(err)
}

fmt.Printf("Daily quota: %d, Used: %d, Remaining: %d\n", 
    quota.DailyQuota, quota.DailyUsed, quota.Remaining)
fmt.Printf("Rate limit: %d per minute\n", quota.RateLimitPerMin)
```

### Async Operations

```go
ctx := context.Background()

// Async image generation
responseChan := manager.GenerateImageSimpleAsync(ctx, "A beautiful landscape")
response, err := responseChan.Get(ctx)
if err != nil {
    panic(err)
}

// Async image editing
imageData := base64.StdEncoding.EncodeToString(imageBytes)
editReq := magica.MagicaEditRequest{Prompt: "Add a rainbow"}
editChan := manager.EditImageAsync(ctx, imageData, editReq)
editResp, err := editChan.Get(ctx)
```

### Getting Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
fmt.Printf("Model: %s\n", status["model"])
fmt.Printf("Location: %s\n", status["location"])
fmt.Printf("Project ID: %s\n", status["project_id"])
fmt.Printf("Max Images: %v\n", status["max_images"])

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
| Standard library | `encoding/json`, `net/http`, `bytes`, `io`, `time`, `encoding/base64`, `mime/multipart` |

## Error Handling

The library uses Go error handling patterns:

- **Authentication failures**: Returned from API (invalid API key)
- **API errors**: Returned from Google Magica API calls
- **Nil checks**: Public methods check manager state

Example error handling:

```go
response, err := manager.GenerateImage(ctx, prompt)
if err != nil {
    if strings.Contains(err.Error(), "401") {
        return fmt.Errorf("invalid API key: %w", err)
    }
    if strings.Contains(err.Error(), "403") {
        return fmt.Errorf("permission denied: %w", err)
    }
    if strings.Contains(err.Error(), "429") {
        return fmt.Errorf("rate limit exceeded: %w", err)
    }
    return fmt.Errorf("API request failed: %w", err)
}
```

## Common Pitfalls

### 1. Manager Not Enabled

**Problem**: `google_magica.enabled` is false

**Solution**:
- Set `google_magica.enabled = true` in configuration
- Check viper configuration is loaded

### 2. API Key Missing

**Problem**: `api_key` not set

**Solution**:
- Set `google_magica.api_key` to your Google Cloud API key
- Ensure the key has access to AI Platform

### 3. Project ID Missing

**Problem**: `project_id` not set

**Solution**:
- Set `google_magica.project_id` to your Google Cloud project ID

### 4. Invalid Model

**Problem**: `model` not set or invalid

**Solution**:
- Set `google_magica.model` to a valid model (e.g., "imagen-3.0-generate-002")
- Check Google AI Platform documentation for available models

### 5. Rate Limiting

**Problem**: `429` Too Many Requests

**Solution**:
- Library retries up to 3 times
- Consider using async operations to spread requests
- Respect Google Cloud rate limits

### 6. Image Size Limits

**Problem**: Image too large or wrong format

**Solution**:
- Ensure images are properly base64 encoded
- Use supported MIME types (image/jpeg, image/png)
- Check output format settings

### 7. Worker Pool Not Available

**Problem**: Async operations falling back to direct execution

**Solution**:
- The pool is created during initialization
- Check that `NewGoogleMagicaManager()` succeeded
- Verify `google_magica.enabled` is true

## Advanced Usage

### Custom Generation with All Parameters

```go
func customImageGeneration(manager *magica.GoogleMagicaManager) error {
    ctx := context.Background()
    
    prompt := magica.MagicaImagePrompt{
        Text:           "A futuristic cityscape at night",
        NegativePrompt: "blurry, distorted",
        AspectRatio:    "16:9",
        GuidanceScale:  8.0,
        Seed:           12345,
        StylePreset:    "digital-art",
        SafetyFilter:   "block_medium_and_above",
        OutputFormat:   "png",
    }
    
    response, err := manager.GenerateImage(ctx, prompt)
    if err != nil {
        return err
    }
    
    // Process images
    for _, img := range response.Images {
        // Save or upload image
        data, _ := base64.StdEncoding.DecodeString(img.Base64Data)
        // ... save to file
    }
    return nil
}
```

### Health Check

```go
func healthCheck(manager *magica.GoogleMagicaManager) error {
    status := manager.GetStatus()
    if status["connected"] == false {
        return fmt.Errorf("not connected to Google Magica API")
    }
    fmt.Printf("Google Magica API is healthy (Model: %s)\n", status["model"])
    return nil
}
```

### Batch Processing with Async

```go
func batchImageGeneration(manager *magica.GoogleMagicaManager, prompts []string) ([]*magica.MagicaImageResponse, error) {
    ctx := context.Background()
    results := make([]*magica.MagicaImageResponse, len(prompts))
    errors := make([]error, len(prompts))
    
    // Submit all requests asynchronously
    for i, prompt := range prompts {
        func(idx int, p string) {
            manager.SubmitAsyncJob(func() {
                response, err := manager.GenerateImageSimple(ctx, p)
                results[idx] = response
                errors[idx] = err
            })
        }(i, prompt)
    }
    
    // Collect results
    for i, err := range errors {
        if err != nil {
            return nil, fmt.Errorf("failed to process prompt %d: %w", i, err)
        }
    }
    return results, nil
}
```

### Image from File Upload

```go
func generateFromFile(manager *magica.GoogleMagicaManager, filePath string) error {
    ctx := context.Background()
    
    // Read file
    data, err := os.ReadFile(filePath)
    if err != nil {
        return err
    }
    
    // Generate variations from file
    response, err := manager.GenerateImageFromFile(ctx, data, "A variation of this image")
    if err != nil {
        return err
    }
    
    fmt.Printf("Generated %d variations\n", len(response.Images))
    return nil
}
```

## Internal Algorithms

### API Request Flow

```
GenerateImage():
    │
    ├── Build JSON payload with instances and parameters
    ├── POST to {baseURL}:predict
    ├── Set Authorization header with Bearer token
    ├── Execute with retryablehttp (3 retries)
    ├── Parse response JSON
    └── Return images with metadata
```

### Image Encoding

```
UploadImage():
    │
    ├── Take image bytes and MIME type
    ├── Base64 encode the bytes
    └── Return base64 string
```

### Connection Test

```
testConnection():
    │
    ├── GET /projects/{projectID}/locations/{location}/publishers/google/models
    ├── Set Authorization header
    ├── Check status code (200 OK)
    └── Return error if failed
```

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/projects/{project}/locations/{location}/publishers/google/models` | GET | List available models |
| `{model}:predict` | POST | Generate images, edit, upscale, style transfer, variations |

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.