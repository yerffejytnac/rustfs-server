# Building RustFS from Source on macOS

This guide documents how to build RustFS Docker images from source on Apple Silicon (M1/M2/M3) Macs, including the web console UI.

## Prerequisites

- Docker Desktop with buildx support
- Git
- Node.js >= 22 and pnpm >= 10 (for console UI)
- ~10GB disk space for build cache

## Why Build from Source?

The official `rustfs/rustfs:latest` image may not include the latest features from the `main` branch. For example, FTPS/SFTP support (PR #1308) was merged but not yet released in a tagged release.

---

## Complete Build with Console UI

This is the recommended approach to get a fully-featured local image.

### Step 1: Clone Both Repositories

```bash
# Create a working directory
mkdir -p ~/Development
cd ~/Development

# Clone the main RustFS repository
git clone https://github.com/rustfs/rustfs.git rustfs-src
cd rustfs-src
git checkout main
git pull origin main

# Clone the console UI repository (sibling directory)
cd ..
git clone https://github.com/rustfs/console.git rustfs-console
```

### Step 2: Build the Console UI

```bash
cd ~/Development/rustfs-console

# Ensure correct Node.js version (if using asdf/nvm)
# Node.js 22+ required, pnpm 10+ required
node --version  # Should be v22.x or higher
pnpm --version  # Should be 10.x or higher

# Install dependencies
pnpm install

# Prepare Nuxt (generates types and configs)
pnpm nuxt prepare

# Generate static files for production
pnpm generate
```

The built files will be in `.output/public/`.

### Step 3: Copy Console Assets to RustFS

```bash
# Copy generated static files to the RustFS static folder
cp -r ~/Development/rustfs-console/.output/public/* ~/Development/rustfs-src/rustfs/static/

# Verify files were copied
ls ~/Development/rustfs-src/rustfs/static/
# Should show: _nuxt/, index.html, favicon.ico, etc.
```

### Step 4: Build the Docker Image

```bash
cd ~/Development/rustfs-src

# Build for Apple Silicon (arm64)
docker buildx build \
  --platform linux/arm64 \
  --build-arg TARGETPLATFORM=linux/arm64 \
  -t rustfs/rustfs:local \
  --load \
  -f Dockerfile.source .
```

Build time: ~5-10 minutes depending on your machine.

### Step 5: Verify the Build

```bash
# Check the image was created
docker images | grep rustfs/rustfs

# Verify FTPS/SFTP support
docker run --rm rustfs/rustfs:local rustfs --help | grep -E "(ftps|sftp)"

# Expected output:
# --ftps-enable          Enable FTPS server
# --sftp-enable          Enable SFTP server
```

### Step 6: Update Your Configuration

Update `.env` to use the local build:

```env
RUSTFS_IMAGE=rustfs/rustfs:local
```

Restart the container:

```bash
cd ~/Development/rustfs  # Your docker-compose project
docker compose down && docker compose up -d
```

### Step 7: Verify Console is Working

```bash
# Check console returns HTTP 200
curl -sI http://localhost:9001/rustfs/console/index.html | head -5

# Expected:
# HTTP/1.1 200 OK
# content-type: text/html
```

Open http://localhost:9001 in your browser.

---

## Quick Build (Without Console UI)

If you only need the server without the web UI:

```bash
git clone https://github.com/rustfs/rustfs.git rustfs-src
cd rustfs-src
git checkout main

docker buildx build \
  --platform linux/arm64 \
  --build-arg TARGETPLATFORM=linux/arm64 \
  -t rustfs/rustfs:local \
  --load \
  -f Dockerfile.source .
```

> **Note**: This build won't include the web console UI. You'll get a 404 at `localhost:9001`.

---

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

---

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

If you encounter cache corruption or `File exists (os error 17)`:
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

### Console Not Loading (404)

**Cause**: The `rustfs/static/` folder is empty — console assets weren't included in the build.

**Fix**: Build the console UI and copy static files before building RustFS (see "Complete Build with Console UI" above).

### Console Build: pnpm install fails

**Cause**: Wrong Node.js or pnpm version.

**Fix**: Ensure Node.js >= 22 and pnpm >= 10:
```bash
# If using asdf
asdf install nodejs 24.6.0
asdf set nodejs 24.6.0

# If using nvm
nvm install 22
nvm use 22

# Install/update pnpm
npm install -g pnpm@latest
```

### Slow Builds

Rust compilation is CPU-intensive. Tips:
- Close other applications during build
- Ensure Docker Desktop has adequate CPU/memory allocation (Settings → Resources)
- Use `--mount=type=cache` (already in Dockerfile) to cache dependencies
- First build is slowest; subsequent builds use cache

---

## Building for x86_64 (Intel/AMD)

If you need an x86_64 image (e.g., for deployment to x86 servers):

```bash
docker buildx build \
  --platform linux/amd64 \
  --build-arg TARGETPLATFORM=linux/amd64 \
  -t rustfs/rustfs:local-amd64 \
  --load \
  -f Dockerfile.source .
```

Note: This cross-compiles on ARM Mac and takes significantly longer.

---

## Multi-Architecture Build

To build for both platforms (requires pushing to a registry):

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t your-registry/rustfs:latest \
  --push \
  -f Dockerfile.source .
```

---

## Alternative: Use Official Build Script

The repository includes `docker-buildx.sh` but it downloads **pre-built release binaries** rather than building from source:

```bash
# Downloads pre-built binary (may not have latest features)
./docker-buildx.sh --platforms linux/arm64

# To build from source with latest changes, use Dockerfile.source directly
```

---

## Directory Structure Reference

After following this guide, your directory structure should look like:

```
~/Development/
├── rustfs/                    # Your docker-compose project
│   ├── docker-compose.yml
│   ├── .env
│   ├── tls/                   # TLS certificates for FTPS
│   ├── sftp/                  # SSH keys for SFTP
│   └── docs/
│       └── LOCAL_BUILD.md     # This file
├── rustfs-src/                # Cloned RustFS source
│   ├── Dockerfile.source
│   ├── rustfs/
│   │   └── static/            # Console UI assets (copied here)
│   └── ...
└── rustfs-console/            # Cloned console UI source
    ├── .output/
    │   └── public/            # Built static files
    └── ...
```

---

## Resources

- [RustFS GitHub](https://github.com/rustfs/rustfs)
- [RustFS Console GitHub](https://github.com/rustfs/console)
- [RustFS Docker Docs](https://docs.rustfs.com/installation/docker/)
- [PR #1308: FTPS/SFTP Support](https://github.com/rustfs/rustfs/pull/1308)
- [Docker Buildx Documentation](https://docs.docker.com/buildx/working-with-buildx/)
