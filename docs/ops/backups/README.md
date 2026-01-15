# Backup Strategy

## Overview

All services use **incremental forever** backups with block-level deduplication to minimize cloud storage costs and upload bandwidth while maintaining fast, point-in-time recovery.

## Key Principles

- **Daily incremental backups**: Only changed data is uploaded
- **Direct restoration**: Any backup point can be restored directly without reconstructing chains
- **Cloud-optimized**: Designed for services like Backblaze B2, AWS S3, etc.
- **Deduplication**: Content-defined chunking eliminates redundant data across backups

## Technology

We use [tool name - e.g., Restic/Duplicacy/Kopia] for all backups, providing:
- Encryption at rest
- Compression
- Verification and integrity checks
- Cross-platform compatibility

## Retention Policy

- **Daily backups**: Keep last 7 days
- **Weekly backups**: Keep last 4 weeks  
- **Monthly backups**: Keep last 12 months
- **Yearly backups**: Keep last 3 years

## Per-Service Documentation

See individual service backup configurations:
- [Service A](./service-a-backup.md)
- [Service B](./service-b-backup.md)
- [Database](./database-backup.md)

## Recovery Testing

All backup configurations must include recovery test procedures. Untested backups are not backups.
