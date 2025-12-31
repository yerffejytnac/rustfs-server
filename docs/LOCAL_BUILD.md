# Building RustFS from Source on macOS

This guide documents how to build RustFS Docker images from source on Apple Silicon (M1/M2/M3) Macs.

## Prerequisites

- Docker Desktop with buildx support
- Git
- ~10GB disk space for build cache

## Why Build from Source?

The official `rustfs/rustfs:latest` image may not include the latest features from the `main` branch. For example, FTPS/SFTP support (PR #1308) was merged but not yet released.

## Quick Build

```bash
# Clone the RustFS repository
git clone https://github.com/rustfs/rustfs.git rustfs-src
cd rustfs-src

# Checkout main branch (includes latest merged PRs)
git checkout main
git pull origin main

# Build for arm64 (Apple Silicon)
docker buildx build \
  --platform linux/arm64 \
  --build-arg TARGETPLATFORM=linux/arm64 \
  -t rustfs/rustfs:local \
  --load \
  -f Dockerfile.source .
```

Build time: ~5-10 minutes depending on your machine.

## Important: Platform Arguments

The `Dockerfile.source` uses `TARGETPLATFORM` to determine which Rust target to compile for. **You must pass this explicitly** or it defaults to `linux/amd64`:

```bash
# ❌ WRONG - defaults to x86_64, fails on ARM Mac
docker build -f Dockerfile.source .

# ❌ WRONG - platform flag alone doesn't set the ARG
docker buildx build --platform linux/arm64 -f Dockerfile.source .

# ✅ CORRECT - explicit build arg
docker buildx build \
  --platform linux/arm64 \
  --build-arg TARGETPLATFORM=linux/arm64 \
  -t rustfs/rustfs:local \
  --load \
  -f Dockerfile.source .
```

## Common Issues

### Error: "can't find crate for `core`"

```
error[E0463]: can't find crate for `core`
  = note: the `x86_64-unknown-linux-gnu` target may not be installed
```

**Cause**: The build is trying to cross-compile for x86_64 on an ARM Mac.

**Fix**: Add `--build-arg TARGETPLATFORM=linux/arm64` to your build command.

### Error: "ProcessFdQuotaExceeded"

**Cause**: macOS default file descriptor limit (256) is too low.

**Fix**: Increase the limit before building:
```bash
ulimit -n 4096
```

### Build Cache Issues

If you encounter cache corruption:
```bash
# Clear Docker build cache
docker builder prune -f

# Rebuild without cache
docker buildx build --no-cache \
  --platform linux/arm64 \
  --build-arg TARGETPLATFORM=linux/arm64 \
  -t rustfs/rustfs:local \
  --load \
  -f Dockerfile.source .
```

### Slow Builds

Rust compilation is CPU-intensive. Tips:
- Close other applications during build
- Ensure Docker Desktop has adequate CPU/memory allocation
- Use `--mount=type=cache` (already in Dockerfile) to cache dependencies

## Verifying the Build

Check FTPS/SFTP support is included:

```bash
docker run --rm rustfs/rustfs:local rustfs --help | grep -E "(ftps|sftp)"
```

Expected output:
```
--ftps-enable          Enable FTPS server
--ftps-address         FTPS server bind address
--sftp-enable          Enable SFTP server
--sftp-address         SFTP server bind address
```

## Using the Local Image

Update `.env` to use your local build:

```env
RUSTFS_IMAGE=rustfs/rustfs:local
```

Then restart:
```bash
docker compose down && docker compose up -d
```

## Building for x86_64 (Intel/AMD)

If you need an x86_64 image (e.g., for deployment):

```bash
docker buildx build \
  --platform linux/amd64 \
  --build-arg TARGETPLATFORM=linux/amd64 \
  -t rustfs/rustfs:local-amd64 \
  --load \
  -f Dockerfile.source .
```

Note: This cross-compiles on ARM Mac and takes significantly longer.

## Multi-Architecture Build

To build for both platforms (requires pushing to a registry):

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t your-registry/rustfs:latest \
  --push \
  -f Dockerfile.source .
```

## Alternative: Use Official Build Script

The repository includes `docker-buildx.sh` but it downloads **pre-built release binaries** rather than building from source:

```bash
# Downloads pre-built binary (may not have latest features)
./docker-buildx.sh --platforms linux/arm64

# To build from source, use Dockerfile.source directly
```

## Resources

- [RustFS GitHub](https://github.com/rustfs/rustfs)
- [PR #1308: FTPS/SFTP Support](https://github.com/rustfs/rustfs/pull/1308)
- [Docker Buildx Documentation](https://docs.docker.com/buildx/working-with-buildx/)
