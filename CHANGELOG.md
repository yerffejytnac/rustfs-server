# Changelog

## [Unreleased]

### Added
- Docker Compose setup for local RustFS development
- Environment-based configuration via `.env` file
- FTPS support with TLS certificate configuration (port 8021)
- SFTP support with SSH key authentication (port 8022)
- Volume permission helper service for non-root container
- Health checks for S3 API and console endpoints
- README with setup and configuration instructions
- `.gitignore` for secrets, keys, and runtime data
- `docs/LOCAL_BUILD.md` with macOS build instructions and troubleshooting
- `docs/FTP.md` and `docs/FUJI_FTP_SETUP.md` added

### Changed
- Default image changed to `rustfs/rustfs:local` (built from source with FTPS/SFTP)
- Updates `docs/LOCAL_BUILD.md` for building UI with image
