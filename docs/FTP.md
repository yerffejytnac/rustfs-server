# RustFS File Transfer Protocols

RustFS supports multiple file transfer protocols for accessing object storage beyond the standard S3 API:

- **FTPS** — File Transfer Protocol over TLS for traditional file transfers
- **SFTP** — SSH File Transfer Protocol for secure file transfers with key-based authentication

Both protocols share the same IAM-based authentication and authorization system as the S3 API.

---

## Quick Start

### Prerequisites

- RustFS server built with FTPS/SFTP support (see [LOCAL_BUILD.md](LOCAL_BUILD.md))
- TLS certificates for FTPS (in `./tls/`)
- SSH host keys for SFTP (in `./sftp/`)

### Generate Required Keys

```bash
cd /path/to/your/rustfs-project

# FTPS: Generate self-signed TLS certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls/key.pem \
  -out tls/cert.pem \
  -subj "/CN=localhost"

# SFTP: Generate SSH host key
ssh-keygen -t ed25519 -f sftp/host_key -N ""

# SFTP: Create authorized_keys (add your public key)
cat ~/.ssh/id_ed25519.pub > sftp/authorized_keys
```

### Environment Configuration

Add these settings to your `.env` file:

```env
# FTPS Configuration
RUSTFS_FTPS_ENABLE=true
RUSTFS_FTPS_ADDRESS=:8021
RUSTFS_FTPS_CERTS_FILE=/opt/tls/cert.pem
RUSTFS_FTPS_KEY_FILE=/opt/tls/key.pem
RUSTFS_FTPS_PASSIVE_PORTS=40000-41000

# SFTP Configuration
RUSTFS_SFTP_ENABLE=true
RUSTFS_SFTP_ADDRESS=:8022
RUSTFS_SFTP_HOST_KEY=/opt/sftp/host_key
RUSTFS_SFTP_AUTHORIZED_KEYS=/opt/sftp/authorized_keys
```

### Start the Server

```bash
docker compose down && docker compose up -d
```

---

## Protocol Details

### FTPS (FTP over TLS)

| Property | Value |
|----------|-------|
| **Default Port** | 8021 |
| **Protocol** | FTP with Explicit TLS |
| **Authentication** | Access Key / Secret Key (same as S3) |
| **Encryption** | TLS 1.2/1.3 for control and data connections |

**Supported Operations:**
- ✅ File upload/download
- ✅ Directory listing
- ✅ File deletion
- ✅ Passive mode data connections

**Limitations:**
- ❌ Cannot create/delete buckets (use S3 API)
- ❌ No file rename/copy operations
- ❌ No multipart upload

### SFTP (SSH File Transfer Protocol)

| Property | Value |
|----------|-------|
| **Default Port** | 8022 |
| **Protocol** | SSH2 File Transfer |
| **Authentication** | Password, SSH Public Key, or SSH Certificate |
| **Encryption** | SSH2 with modern cipher suites (Ed25519, RSA, ECDSA) |

**Supported Operations:**
- ✅ File upload/download
- ✅ Directory listing and manipulation
- ✅ File deletion
- ✅ Bucket creation/deletion via mkdir/rmdir

**Limitations:**
- ❌ No file rename/copy operations
- ❌ No multipart upload
- ❌ No symlinks or file attribute modification

---

## Testing & Verification

### Verify Server Status

```bash
# Check container is running and healthy
docker compose ps

# View protocol-specific logs
docker compose logs -f rustfs 2>&1 | grep -iE "(ftps|sftp)"
```

### Test FTPS Connection

```bash
# Install lftp if needed
brew install lftp

# Connect and list buckets (self-signed cert)
lftp -u rustfsadmin,rustfsadmin \
     -e "set ssl:verify-certificate no; ls /; quit" \
     ftps://localhost:8021

# Using curl
curl -k --ftp-ssl -u rustfsadmin:rustfsadmin \
     ftp://localhost:8021/ --list-only

# Verify TLS certificate
openssl s_client -connect localhost:8021 -starttls ftp
```

**Expected Output:**
```
drwxr-xr-x   1 ftp      ftp             0 Dec 31 00:00 mybucket
```

### Test SFTP Connection

```bash
# Password authentication
sftp -P 8022 -o StrictHostKeyChecking=no rustfsadmin@localhost

# SSH key authentication (if configured)
sftp -P 8022 -i ~/.ssh/id_ed25519 -o StrictHostKeyChecking=no rustfsadmin@localhost

# Verbose debug output
sftp -P 8022 -o StrictHostKeyChecking=no -o LogLevel=DEBUG3 rustfsadmin@localhost
```

**Expected Output:**
```
Connected to localhost.
sftp>
```

### Verify FTPS/SFTP Support in Binary

```bash
docker run --rm rustfs/rustfs:local rustfs --help | grep -E "(ftps|sftp)"

# Expected output:
# --ftps-enable          Enable FTPS server
# --sftp-enable          Enable SFTP server
```

---

## FTPS Usage Examples

```bash
# Login
lftp -u rustfsadmin,rustfsadmin \
     -e "set ssl:verify-certificate no" \
     ftps://localhost:8021

# List buckets (root directory)
lftp> ls /

# Change to a bucket
lftp> cd mybucket

# Upload a file
lftp> put localfile.txt

# Download a file
lftp> get remotefile.txt

# Delete a file
lftp> rm remotefile.txt

# Create a directory (prefix)
lftp> mkdir mydir

# Remove a directory
lftp> rmdir mydir

# Exit
lftp> bye
```

### Batch Upload with lftp

```bash
lftp -u rustfsadmin,rustfsadmin \
     -e "set ssl:verify-certificate no; \
         cd mybucket; \
         mput *.jpg; \
         bye" \
     ftps://localhost:8021
```

---

## SFTP Usage Examples

```bash
# Login (password)
sftp -P 8022 -o StrictHostKeyChecking=no rustfsadmin@localhost

# Interactive commands
sftp> ls                    # List current directory
sftp> cd mybucket           # Change to bucket
sftp> put localfile.txt     # Upload file
sftp> get remotefile.txt    # Download file
sftp> rm remotefile.txt     # Delete file
sftp> mkdir newbucket       # Create bucket
sftp> rmdir emptybucket     # Delete empty bucket
sftp> bye                   # Exit
```

### Batch Operations with SFTP

```bash
# Upload multiple files
sftp -P 8022 -o StrictHostKeyChecking=no rustfsadmin@localhost <<EOF
cd mybucket
put file1.txt
put file2.txt
put file3.txt
bye
EOF
```

### Using scp with SFTP

```bash
# Upload
scp -P 8022 -o StrictHostKeyChecking=no localfile.txt rustfsadmin@localhost:/mybucket/

# Download
scp -P 8022 -o StrictHostKeyChecking=no rustfsadmin@localhost:/mybucket/remotefile.txt ./
```

---

## Configuration Reference

### FTPS Options

| CLI Option | Environment Variable | Description | Default |
|------------|---------------------|-------------|---------|
| `--ftps-enable` | `RUSTFS_FTPS_ENABLE` | Enable FTPS server | `false` |
| `--ftps-address` | `RUSTFS_FTPS_ADDRESS` | FTPS bind address | `0.0.0.0:21` |
| `--ftps-certs-file` | `RUSTFS_FTPS_CERTS_FILE` | TLS certificate file (PEM) | — |
| `--ftps-key-file` | `RUSTFS_FTPS_KEY_FILE` | TLS private key file (PEM) | — |
| `--ftps-passive-ports` | `RUSTFS_FTPS_PASSIVE_PORTS` | Passive port range (e.g., `40000-41000`) | — |
| `--ftps-external-ip` | `RUSTFS_FTPS_EXTERNAL_IP` | External IP for NAT traversal | — |

### SFTP Options

| CLI Option | Environment Variable | Description | Default |
|------------|---------------------|-------------|---------|
| `--sftp-enable` | `RUSTFS_SFTP_ENABLE` | Enable SFTP server | `false` |
| `--sftp-address` | `RUSTFS_SFTP_ADDRESS` | SFTP bind address | `0.0.0.0:22` |
| `--sftp-host-key` | `RUSTFS_SFTP_HOST_KEY` | SSH host key file | — |
| `--sftp-authorized-keys` | `RUSTFS_SFTP_AUTHORIZED_KEYS` | Authorized keys file | — |

---

## Docker Compose Configuration

The `docker-compose.yml` should expose the necessary ports:

```yaml
services:
  rustfs:
    ports:
      - "8021:8021"           # FTPS control
      - "8022:8022"           # SFTP
      - "40000-41000:40000-41000"  # FTPS passive data
    volumes:
      - ./tls:/opt/tls:ro     # TLS certificates
      - ./sftp:/opt/sftp:ro   # SSH keys
    environment:
      # FTPS
      - RUSTFS_FTPS_ENABLE=${RUSTFS_FTPS_ENABLE}
      - RUSTFS_FTPS_ADDRESS=${RUSTFS_FTPS_ADDRESS}
      - RUSTFS_FTPS_CERTS_FILE=${RUSTFS_FTPS_CERTS_FILE}
      - RUSTFS_FTPS_KEY_FILE=${RUSTFS_FTPS_KEY_FILE}
      - RUSTFS_FTPS_PASSIVE_PORTS=${RUSTFS_FTPS_PASSIVE_PORTS}
      # SFTP
      - RUSTFS_SFTP_ENABLE=${RUSTFS_SFTP_ENABLE}
      - RUSTFS_SFTP_ADDRESS=${RUSTFS_SFTP_ADDRESS}
      - RUSTFS_SFTP_HOST_KEY=${RUSTFS_SFTP_HOST_KEY}
      - RUSTFS_SFTP_AUTHORIZED_KEYS=${RUSTFS_SFTP_AUTHORIZED_KEYS}
```

---

## Architecture

```
┌─────────────────────────────────────────┐
│            RustFS Clients               │
│    (FTPS Client, SFTP Client, S3)       │
└─────────────────────────────────────────┘
                    │
┌─────────────────────────────────────────┐
│       Protocol Gateway Layer            │
│   - Action Mapping                      │
│   - Authorization (IAM Policy)          │
│   - Operation Support Check             │
└─────────────────────────────────────────┘
                    │
┌─────────────────────────────────────────┐
│       Internal S3 Client                │
│       (ProtocolS3Client)                │
└─────────────────────────────────────────┘
                    │
┌─────────────────────────────────────────┐
│       Storage Layer (ECStore)           │
│   - Object Storage                      │
│   - Erasure Coding                      │
│   - Metadata Management                 │
└─────────────────────────────────────────┘
```

All protocols share the unified IAM authentication and authorization system:

- **Access Key**: Username identifier
- **Secret Key**: Password/API key
- **SSH Public Keys**: For SFTP key-based authentication
- **IAM Policies**: Fine-grained access control with `s3:*` action namespace

---

## Troubleshooting

### FTPS Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Connection timeout | Firewall blocking ports | Ensure ports 8021 and 40000-41000 are open |
| Certificate error | Self-signed cert rejected | Use `set ssl:verify-certificate no` in lftp |
| Authentication failed | Wrong credentials | Verify access key / secret key |
| Passive mode fails | Port range not exposed | Check Docker port mapping for 40000-41000 |
| Upload hangs | NAT traversal issue | Set `RUSTFS_FTPS_EXTERNAL_IP` to your host IP |

**Debug FTPS TLS:**
```bash
openssl s_client -connect localhost:8021 -starttls ftp
```

### SFTP Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Connection refused | SFTP not enabled | Set `RUSTFS_SFTP_ENABLE=true` |
| Host key verification failed | First connection | Use `-o StrictHostKeyChecking=no` |
| Permission denied (publickey) | Key not in authorized_keys | Add your public key to `sftp/authorized_keys` |
| Permission denied (password) | Wrong credentials | Verify access key / secret key |
| Host key file error | Missing or invalid key | Regenerate with `ssh-keygen -t ed25519` |

**Debug SFTP Connection:**
```bash
sftp -P 8022 -o LogLevel=DEBUG3 -o StrictHostKeyChecking=no rustfsadmin@localhost
```

**Verify SSH Host Key:**
```bash
ssh-keygen -l -f sftp/host_key
```

### General Issues

**Enable Debug Logging:**
```env
RUSTFS_OBS_LOGGER_LEVEL=debug
```

```bash
docker compose down && docker compose up -d
docker compose logs -f rustfs
```

**Check macOS Firewall:**
```bash
# Check status
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate

# Add Docker if needed
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --add /Applications/Docker.app
```

---

## Security Considerations

### TLS Certificates

For production, use certificates from a trusted CA. For development:

```bash
# Generate self-signed certificate (development only)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls/key.pem \
  -out tls/cert.pem \
  -subj "/CN=your-hostname"
```

### SSH Host Keys

Protect your host key; it identifies your server:

```bash
# Generate Ed25519 key (recommended)
ssh-keygen -t ed25519 -f sftp/host_key -N ""

# Or RSA 4096-bit
ssh-keygen -t rsa -b 4096 -f sftp/host_key -N ""
```

### Authorized Keys

For key-based SFTP authentication, add public keys to `sftp/authorized_keys`:

```bash
# Add a user's public key
cat user_public_key.pub >> sftp/authorized_keys

# Set proper permissions
chmod 600 sftp/authorized_keys
```

### Creating Dedicated Users

Create separate users for different clients instead of using admin credentials:

1. Open RustFS Console: http://localhost:9001
2. Navigate to **Identity** → **Users**
3. Click **Create User**
4. Assign appropriate policies (`readwrite`, `readonly`, or custom)

---

## See Also

- [LOCAL_BUILD.md](LOCAL_BUILD.md) — Building RustFS from source
- [RustFS GitHub](https://github.com/rustfs/rustfs)
- [RustFS Console GitHub](https://github.com/rustfs/console)
