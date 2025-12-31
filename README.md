# RustFS Local Development

Local Docker Compose setup for [RustFS](https://github.com/rustfs/rustfs) — a high-performance, S3-compatible object storage system.

## Quick Start

```bash
# Copy environment template
cp .env.example .env

# Start services
docker compose up -d
```

## Access

| Service | URL | Credentials |
|---------|-----|-------------|
| Console | http://localhost:9001 | `rustfsadmin` / `rustfsadmin` |
| S3 API | http://localhost:9000 | Access/Secret keys from `.env` |
| FTPS | localhost:8021 | Same credentials |
| SFTP | localhost:8022 | SSH key authentication |

## Configuration

All configuration is managed via `.env`. Copy `.env.example` to get started.

### Core Settings

| Variable | Default | Description |
|----------|---------|-------------|
| `RUSTFS_ACCESS_KEY` | `rustfsadmin` | S3 access key |
| `RUSTFS_SECRET_KEY` | `rustfsadmin` | S3 secret key |
| `RUSTFS_S3_PORT` | `9000` | S3 API port |
| `RUSTFS_CONSOLE_PORT` | `9001` | Web console port |
| `RUSTFS_OBS_LOGGER_LEVEL` | `info` | Log level (trace/debug/info/warn/error) |

### FTPS Settings

| Variable | Default | Description |
|----------|---------|-------------|
| `RUSTFS_FTPS_ENABLE` | `true` | Enable FTPS |
| `RUSTFS_FTPS_PORT` | `8021` | FTPS control port |
| `RUSTFS_FTPS_CERTS_FILE` | `/opt/tls/cert.pem` | TLS certificate path |
| `RUSTFS_FTPS_KEY_FILE` | `/opt/tls/key.pem` | TLS private key path |
| `RUSTFS_FTPS_PASSIVE_PORTS` | `40000-41000` | Passive mode port range |

### SFTP Settings

| Variable | Default | Description |
|----------|---------|-------------|
| `RUSTFS_SFTP_ENABLE` | `true` | Enable SFTP |
| `RUSTFS_SFTP_PORT` | `8022` | SFTP port |
| `RUSTFS_SFTP_HOST_KEY` | `/opt/sftp/host_key` | SSH host key path |
| `RUSTFS_SFTP_AUTHORIZED_KEYS` | `/opt/sftp/authorized_keys` | Authorized public keys |

## Setup FTPS/SFTP Keys

Generate TLS certificate for FTPS:

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls/key.pem \
  -out tls/cert.pem \
  -subj "/CN=localhost"
```

Generate SSH keys for SFTP:

```bash
# Host key
ssh-keygen -t ed25519 -f sftp/host_key -N ""

# Authorize your public key
cat ~/.ssh/id_ed25519.pub > sftp/authorized_keys
```

## Directory Structure

```
.
├── .env                 # Local configuration (git-ignored)
├── .env.example         # Configuration template
├── docker-compose.yml   # Service definitions
├── tls/                 # TLS certificates for FTPS
│   ├── cert.pem
│   └── key.pem
└── sftp/                # SSH keys for SFTP
    ├── host_key
    └── authorized_keys
```

## Commands

```bash
# Start
docker compose up -d

# Stop
docker compose down

# View logs
docker compose logs -f rustfs

# Restart with new config
docker compose down && docker compose up -d

# Check health
curl http://localhost:9000/health
curl http://localhost:9001/rustfs/console/health
```

## License

Apache 2.0
