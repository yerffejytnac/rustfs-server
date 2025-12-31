# Fujifilm GFX100S II FTP Setup with RustFS

This guide covers connecting your Fujifilm GFX100S II camera to your local RustFS server for wireless photo transfers via FTPS.

## Prerequisites

- RustFS server running with FTPS enabled (see [FTP.md](FTP.md))
- TLS certificates generated (see [FTP.md → Generate Required Keys](FTP.md#generate-required-keys))
- Camera and Mac on the same network

---

## 1. RustFS Server Configuration

### Server Settings Overview

| Setting | Value | Description |
|---------|-------|-------------|
| **FTPS Port** | `8021` | Control connection port |
| **Passive Ports** | `40000-41000` | Data transfer port range |
| **TLS Certificate** | `/opt/tls/cert.pem` | Mounted from `./tls/cert.pem` |
| **TLS Key** | `/opt/tls/key.pem` | Mounted from `./tls/key.pem` |
| **Camera Credentials** | `FUJIFILM` / `12345678` | Dedicated user for camera |

> **Note**: For complete FTPS configuration options, see [FTP.md → Configuration Reference](FTP.md#configuration-reference).

### Get Your Mac's IP Address

The camera needs your Mac's local IP address:

```bash
# Wi-Fi
ipconfig getifaddr en0

# Ethernet
ipconfig getifaddr en1
```

### Create a Dedicated Camera User

Instead of using the admin credentials, create a separate user for the camera:

1. Open the **RustFS Console**: http://localhost:9001
2. Login with `rustfsadmin` / `rustfsadmin`
3. Navigate to **Identity** → **Users**
4. Click **Create User**
5. Enter credentials:
   - **Access Key**: `FUJIFILM`
   - **Secret Key**: `12345678` (must be 8-40 characters)
6. Under **Assign Policies**, select `readwrite` (or create a custom policy for specific buckets)
7. Click **Save**

> **Tip**: Using a dedicated user keeps your admin credentials separate and allows you to revoke camera access without affecting other services.

### Create a Bucket for Camera Uploads

```bash
# Using AWS CLI (with S3 endpoint)
aws --endpoint-url http://localhost:9000 \
    s3 mb s3://camera-uploads

# Or via console at http://localhost:9001
```

---

## 2. Camera FTP Configuration

Reference: [Fujifilm GFX100S II Owner's Manual](https://fujifilm-dsc.com/en-int/manual/gfx100s-ii/) — Network/USB Features and Settings section.

### Step 2.1: Connect Camera to Network

1. Navigate to **NETWORK/USB SETTING** → **CONNECTION SETTING**
2. Select **Wi-Fi** or **Wired LAN** (if using adapter)
3. Connect to the **same network** as your Mac running RustFS

### Step 2.2: Configure FTP Server Settings

Navigate to **NETWORK/USB SETTING** → **FTP TRANSFER SETTINGS** and enter:

| Camera Setting | Value |
|----------------|-------|
| **Server Type** | `FTPS` (FTP over SSL/TLS) |
| **Server Address** | Your Mac's IP (e.g., `192.168.1.100`) |
| **Port Number** | `8021` |
| **User Name** | `FUJIFILM` |
| **Password** | `12345678` |
| **Destination Folder** | `/camera-uploads` (or your bucket path) |
| **Passive Mode** | `ON` |
| **Encryption** | `Explicit TLS` (try this first) |

### Step 2.3: Certificate Validation

Since we're using a self-signed certificate:
- Accept the certificate warning on first connection
- Or set **Certificate Verification** to `OFF` if available

### Step 2.4: Auto Upload Settings (Optional)

For automatic transfer after each shot:

1. **NETWORK/USB SETTING** → **FTP TRANSFER SETTINGS** → **AUTO TRANSFER**
2. Set to `ON`

For manual batch transfer:

1. Use **PLAYBACK MENU** → **FTP TRANSFER** to select and upload images

---

## 3. Testing & Validation

### Step 3.1: Verify Server is Running

```bash
# Check container health
docker compose ps

# View FTPS logs in real-time
docker compose logs -f rustfs 2>&1 | grep -i ftps
```

### Step 3.2: Test FTP Connection from Mac

Before testing with the camera, verify FTPS works locally with the camera user:

```bash
# Using lftp (install with: brew install lftp)
lftp -u FUJIFILM,12345678 \
     -e "set ssl:verify-certificate no; ls; quit" \
     ftps://localhost:8021

# Or using curl
curl -k --ftp-ssl -u FUJIFILM:12345678 \
     ftp://localhost:8021/ --list-only
```

### Step 3.3: Camera Connection Test

1. On camera: **NETWORK/USB SETTING** → **FTP TRANSFER SETTINGS** → **CONNECTION TEST**
2. Should show "Connection OK" or similar success message

### Step 3.4: Test Upload from Camera

1. Take a test photo
2. Go to **PLAYBACK** → select image → **FTP TRANSFER**
3. Or if auto-transfer is enabled, it should upload automatically

### Step 3.5: Verify Upload on Server

```bash
# Check uploaded files via S3 API
aws --endpoint-url http://localhost:9000 \
    s3 ls s3://camera-uploads/ --recursive

# Or check via console UI
open http://localhost:9001
```

### Step 3.6: Monitor Real-Time Transfers

Watch the logs during upload:

```bash
docker compose logs -f rustfs 2>&1 | grep -E "(FTPS|STOR|transfer)"
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| **Connection timeout** | Check firewall; ensure ports 8021 and 40000-41000 are open |
| **Certificate error** | Camera may reject self-signed cert; look for "trust" option |
| **Authentication failed** | Verify credentials match the user created in console (`FUJIFILM` / `12345678`) |
| **Passive mode fails** | Ensure passive port range (40000-41000) is exposed in Docker |
| **Upload hangs** | Check if `RUSTFS_EXTERNAL_ADDRESS` is set to your Mac's IP |

> **Note**: For general FTPS troubleshooting, see [FTP.md → Troubleshooting](FTP.md#troubleshooting).

### Enable Debug Logging

For more verbose output, update `.env`:

```env
RUSTFS_OBS_LOGGER_LEVEL=debug
```

Then restart:

```bash
docker compose down && docker compose up -d
```

### macOS Firewall

If using macOS firewall, allow incoming connections:

```bash
# Check firewall status
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate

# Add Docker to allowed apps (if needed)
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --add /Applications/Docker.app
```

---

## Resources

- [RustFS File Transfer Protocols](FTP.md) — Complete FTPS/SFTP documentation
- [Building RustFS from Source](LOCAL_BUILD.md)
- [Fujifilm GFX100S II Owner's Manual](https://fujifilm-dsc.com/en-int/manual/gfx100s-ii/)
- [RustFS GitHub](https://github.com/rustfs/rustfs)
