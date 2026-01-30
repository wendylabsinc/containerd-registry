# containerd-registry

A lightweight, production-ready OCI container registry server that uses containerd as its storage backend. This provides a Docker-compatible registry without requiring a separate storage solution.

## Overview

`containerd-registry` implements the [OCI Distribution Specification](https://github.com/opencontainers/distribution-spec), making it compatible with standard container tools like Docker, containerd, and Kubernetes. Instead of using traditional filesystem or cloud storage backends, it leverages containerd's content store directly.

### Why containerd-registry?

- **Zero External Dependencies**: Uses containerd's built-in content store (no S3, filesystem, or database required)
- **Lightweight**: Single binary with minimal resource footprint
- **Production Ready**: Structured logging, health checks, graceful shutdown, and configurable timeouts
- **OCI Compliant**: Fully implements the OCI Distribution Specification
- **Simple Deployment**: Works anywhere containerd runs

## Features

### Core Functionality
- ✅ **OCI Distribution API** - Full implementation of push/pull/list operations
- ✅ **Containerd Backend** - Direct integration with containerd content store
- ✅ **Multi-Architecture** - Builds for AMD64, ARM64, and ARM (v6, v7)
- ✅ **Structured Logging** - JSON and text formats with request tracing
- ✅ **Health Checks** - `/readyz` endpoint for Kubernetes probes
- ✅ **Graceful Shutdown** - Clean termination on SIGTERM/SIGINT

### Configuration
- 🔧 **Configurable Timeouts** - Read, write, idle, and shutdown timeouts
- 🔧 **Resource Limits** - Manifest size limits and blob lease expiration
- 🔧 **Safety Controls** - Optional DELETE operations (disabled by default)
- 🔧 **Flexible Binding** - Configurable listen address and port

## Installation

### From Container Image

Pre-built multi-architecture images are available on GitHub Container Registry:

```bash
docker pull ghcr.io/wendylabsinc/containerd-registry:latest
```

Architectures:
- `linux/amd64`
- `linux/arm64`
- `linux/arm/v6`
- `linux/arm/v7`

### From Source

Requirements:
- Go 1.25 or later
- containerd running locally

```bash
git clone https://github.com/wendylabsinc/containerd-registry.git
cd containerd-registry
go build -o containerd-registry
```

## Usage

### Basic Usage

The registry connects to containerd via the default socket and listens on port 8080:

```bash
./containerd-registry
```

Access the registry:
```bash
# Tag an image
docker tag myimage:latest localhost:8080/myimage:latest

# Push to the registry
docker push localhost:8080/myimage:latest

# Pull from the registry
docker pull localhost:8080/myimage:latest
```

### Docker Compose

Run with containerd in Docker Compose:

```yaml
version: '3.8'

services:
  containerd:
    image: containerd/containerd:latest
    privileged: true
    volumes:
      - containerd-data:/var/lib/containerd
    command: containerd --log-level debug

  registry:
    image: ghcr.io/wendylabsinc/containerd-registry:latest
    depends_on:
      - containerd
    ports:
      - "5000:8080"
    volumes:
      - /var/run/containerd:/var/run/containerd
    environment:
      - LOG_FORMAT=json
      - LISTEN_ADDRESS=:8080

volumes:
  containerd-data:
```

### Kubernetes Deployment

Deploy as a sidecar to containerd:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: containerd-registry
spec:
  containers:
  - name: registry
    image: ghcr.io/wendylabsinc/containerd-registry:latest
    ports:
    - containerPort: 8080
      name: registry
    env:
    - name: LOG_FORMAT
      value: "json"
    - name: LISTEN_ADDRESS
      value: ":8080"
    volumeMounts:
    - name: containerd-socket
      mountPath: /run/containerd
    livenessProbe:
      httpGet:
        path: /readyz
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /readyz
        port: 8080
      initialDelaySeconds: 3
      periodSeconds: 5
  volumes:
  - name: containerd-socket
    hostPath:
      path: /run/containerd
      type: Directory
```

## Configuration

All configuration is done via environment variables:

### Server Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `LISTEN_ADDRESS` | `:8080` | Address and port to listen on (e.g., `0.0.0.0:5000`, `:8080`) |
| `LOG_FORMAT` | `text` | Log format: `text` (human-readable) or `json` (structured) |

### Timeout Configuration

Use Go duration format (`5m`, `30s`, `2h30m`):

| Variable | Default | Description |
|----------|---------|-------------|
| `READ_TIMEOUT` | `5m` | Maximum time to read request headers and body |
| `WRITE_TIMEOUT` | `5m` | Maximum time to write the response |
| `IDLE_TIMEOUT` | `120s` | Maximum time to wait for the next request (keep-alive) |
| `SHUTDOWN_TIMEOUT` | `30s` | Maximum time to wait for graceful shutdown |

### Registry Limits

| Variable | Default | Description |
|----------|---------|-------------|
| `BLOB_LEASE_EXPIRATION` | `15m` | How long blob upload leases last before expiring |
| `MAX_MANIFEST_SIZE` | `4194304` | Maximum manifest size in bytes (4 MiB default) |

### Safety Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `ALLOW_DELETE` | disabled | Set to `1` to enable DELETE operations (blobs, manifests, tags) |

**⚠️ Warning**: Enabling DELETE can corrupt registry state if referenced content is deleted. Only enable if you understand the risks.

## API Endpoints

### Registry API

Implements the [OCI Distribution Specification](https://github.com/opencontainers/distribution-spec/blob/main/spec.md):

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/v2/` | API version check |
| `GET` | `/v2/_catalog` | List repositories |
| `GET` | `/v2/<name>/tags/list` | List tags for repository |
| `HEAD` | `/v2/<name>/manifests/<reference>` | Check if manifest exists |
| `GET` | `/v2/<name>/manifests/<reference>` | Get manifest |
| `PUT` | `/v2/<name>/manifests/<reference>` | Upload manifest |
| `DELETE` | `/v2/<name>/manifests/<reference>` | Delete manifest (if ALLOW_DELETE=1) |
| `HEAD` | `/v2/<name>/blobs/<digest>` | Check if blob exists |
| `GET` | `/v2/<name>/blobs/<digest>` | Download blob |
| `POST` | `/v2/<name>/blobs/uploads/` | Start blob upload |
| `PATCH` | `/v2/<name>/blobs/uploads/<uuid>` | Upload blob chunk |
| `PUT` | `/v2/<name>/blobs/uploads/<uuid>` | Complete blob upload |
| `DELETE` | `/v2/<name>/blobs/<digest>` | Delete blob (if ALLOW_DELETE=1) |

### Health Checks

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/readyz` | Health check - returns 200 if containerd is accessible |

Example:
```bash
curl http://localhost:8080/readyz
# OK

# If containerd is down:
# HTTP/1.1 503 Service Unavailable
# containerd not ready: ...
```

## Logging

### Text Format (Human-Readable)

```
2026/01/29 15:30:45 INFO HTTP request method=GET path=/v2/ remote=192.168.1.10:54321 status=200 duration_ms=2 bytes=23
2026/01/29 15:30:46 INFO HTTP request method=GET path=/v2/_catalog remote=192.168.1.10:54322 status=200 duration_ms=15 bytes=456
```

### JSON Format (Structured)

```json
{
  "time": "2026-01-29T15:30:45Z",
  "level": "INFO",
  "msg": "HTTP request",
  "method": "GET",
  "path": "/v2/",
  "remote": "192.168.1.10:54321",
  "status": 200,
  "duration_ms": 2,
  "bytes": 23
}
```

Log levels:
- **INFO**: 2xx and 3xx responses
- **WARN**: 4xx responses (client errors)
- **ERROR**: 5xx responses (server errors)

## Use Cases

### 1. Edge Device Registry

Deploy alongside containerd on edge devices for local image caching:

```bash
# On edge device with containerd
docker run -d \
  --name registry \
  -p 5000:8080 \
  -v /var/run/containerd:/var/run/containerd:ro \
  ghcr.io/wendylabsinc/containerd-registry:latest

# Configure containerd to use local registry as mirror
# Edit /etc/containerd/config.toml
```

### 2. CI/CD Build Cache

Use as a build cache in CI/CD pipelines:

```yaml
# GitHub Actions example
- name: Start registry
  run: |
    docker run -d --name registry \
      -p 5000:8080 \
      -v /var/run/containerd:/var/run/containerd \
      ghcr.io/wendylabsinc/containerd-registry:latest

- name: Build with cache
  run: |
    docker build \
      --cache-from localhost:5000/myapp:cache \
      --cache-to type=registry,ref=localhost:5000/myapp:cache \
      -t myapp:latest .
```

### 3. Development Environment

Local registry for development without external dependencies:

```bash
# Start registry
./containerd-registry

# Use with docker-compose
# docker-compose.yml
version: '3.8'
services:
  app:
    build: .
    image: localhost:8080/myapp:dev
```

### 4. Airgapped Environments

Run a fully offline registry using containerd's local storage:

```bash
# Pre-load images into containerd
ctr images pull docker.io/library/alpine:latest
ctr images pull docker.io/library/nginx:latest

# Start registry
./containerd-registry

# Images are immediately available at localhost:8080
```

## Architecture

```
┌─────────────────────────────────────┐
│   Docker / containerd / kubectl     │
│   (OCI Distribution API Client)     │
└─────────────────┬───────────────────┘
                  │ HTTP
                  │ (OCI Distribution Spec)
                  ↓
┌─────────────────────────────────────┐
│      containerd-registry Server     │
│  ┌──────────────────────────────┐   │
│  │   HTTP Server                │   │
│  │   - Request routing          │   │
│  │   - Logging middleware       │   │
│  │   - Health checks            │   │
│  └──────────────┬───────────────┘   │
│                 │                    │
│  ┌──────────────▼───────────────┐   │
│  │   OCI Registry Logic         │   │
│  │   - Manifest handling        │   │
│  │   - Blob uploads             │   │
│  │   - Repository listing       │   │
│  └──────────────┬───────────────┘   │
└─────────────────┼───────────────────┘
                  │ gRPC
                  ↓
┌─────────────────────────────────────┐
│          containerd Daemon          │
│  ┌──────────────────────────────┐   │
│  │   Content Store              │   │
│  │   - Blob storage (CAS)       │   │
│  │   - Manifest storage         │   │
│  │   - Image metadata           │   │
│  └──────────────────────────────┘   │
│  ┌──────────────────────────────┐   │
│  │   Lease Manager              │   │
│  │   - Upload session tracking  │   │
│  └──────────────────────────────┘   │
└─────────────────────────────────────┘
```

### How It Works

1. **Client Request**: Docker/containerd client sends OCI Distribution API request
2. **HTTP Server**: Receives request, logs it, routes to appropriate handler
3. **Registry Logic**: Validates request, interacts with containerd via gRPC
4. **Content Store**: containerd stores/retrieves blobs and manifests
5. **Response**: Registry formats response according to OCI spec

### Storage Model

- **Blobs**: Stored in containerd's content-addressable store (CAS)
- **Manifests**: Also stored in CAS, referenced by digest
- **Tags**: Managed via containerd's image service
- **Metadata**: Tracked by containerd's metadata database

## Building

### Local Build

```bash
go build -o containerd-registry
```

### Multi-Architecture Build

```bash
# Build for all supported platforms
docker buildx build \
  --platform linux/amd64,linux/arm64,linux/arm/v6,linux/arm/v7 \
  -t ghcr.io/wendylabsinc/containerd-registry:latest \
  --push \
  .
```

### GitHub Actions

The repository includes automated builds:
- `.github/workflows/build-registry.yml` - Builds and publishes multi-arch images
- `.github/workflows/ci.yml` - Runs tests and validation

## Troubleshooting

### Registry Won't Start

**Error**: `failed to create containerd client`

**Solution**: Ensure containerd is running and socket is accessible:
```bash
# Check containerd status
systemctl status containerd

# Verify socket exists
ls -l /run/containerd/containerd.sock

# Check permissions
groups  # Your user should be in 'docker' or have access to socket
```

### Push/Pull Fails

**Error**: `manifest unknown` or `blob unknown`

**Solution**: Images must exist in containerd's namespace:
```bash
# List images in containerd
ctr images list

# Tag image for registry
ctr images tag source:tag localhost:8080/dest:tag
```

### Health Check Fails

**Error**: `/readyz` returns 503

**Solution**: Check containerd connection:
```bash
# Test containerd directly
ctr version

# Check registry logs
docker logs registry-container

# Verify socket mount
docker inspect registry-container | grep containerd
```

### Permission Denied

**Error**: `permission denied` when accessing containerd socket

**Solution**: Run with proper permissions:
```bash
# Add user to docker group
sudo usermod -aG docker $USER

# Or run with elevated privileges
sudo ./containerd-registry

# In containers, mount socket with proper permissions
docker run -v /var/run/containerd:/var/run/containerd:ro ...
```

## Security Considerations

### Socket Access

The registry requires access to containerd's Unix socket, which provides full container management privileges. Ensure:
- Socket is only accessible to trusted users/containers
- Use read-only mounts when possible
- Apply appropriate file permissions

### DELETE Operations

DELETE is disabled by default because:
- Deleting blobs can corrupt images that reference them
- Deleting manifests can break tags
- containerd doesn't track references between registry and runtime

Only enable `ALLOW_DELETE=1` if:
- You understand the risks
- You have proper garbage collection in place
- You're using this in a development/testing environment

### Network Exposure

For production:
- Use TLS termination (nginx, Traefik, etc.)
- Implement authentication (basic auth, token auth)
- Restrict network access
- Don't expose directly to the internet

## Limitations

### Current Limitations

- **No Authentication**: Registry doesn't implement authentication (use reverse proxy)
- **No TLS**: Requires external TLS termination for HTTPS
- **Single Namespace**: Uses containerd's default namespace
- **Limited DELETE**: DELETE operations are discouraged (can corrupt state)

### Planned Features

- [ ] Authentication support (basic auth, token auth)
- [ ] TLS support
- [ ] Multi-namespace support
- [ ] Metrics endpoint (Prometheus)
- [ ] Garbage collection integration

## Contributing

This project is part of WendyOS infrastructure. For contributions:

1. Test changes locally with containerd
2. Ensure OCI spec compliance
3. Add appropriate logging
4. Update this README if adding features

## License

Proprietary - Wendy Labs Inc.

## Related Projects

- [containerd](https://github.com/containerd/containerd) - Container runtime
- [OCI Distribution Spec](https://github.com/opencontainers/distribution-spec) - Registry API specification
- [cuelabs.dev/go/oci](https://pkg.go.dev/cuelabs.dev/go/oci) - OCI registry library
