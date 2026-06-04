# Ngelmak Storage

## Overview

This repository contains storage-related infrastructure and tooling for the Ngelmak project. The main services used here are:

- **PostgreSQL** for relational database storage
- **Redis** for in-memory caching and fast state storage
- **SeaweedFS** for distributed file storage and object persistence

The repository also includes Docker Compose definitions to run these services locally for development and testing.

## Main Tools

### PostgreSQL

PostgreSQL provides a stable, ACID-compliant relational database. It is used for:

- storing structured application data
- managing transactional workloads
- supporting SQL queries and indexes

You can find PostgreSQL-related configuration and compose files under `databases/docker-compose.db.yaml`.

### Redis

Redis is used as a fast, in-memory key-value store. Common use cases include:

- caching frequently accessed data
- managing session or state data
- supporting pub/sub and fast lookup patterns

Redis is often used alongside PostgreSQL to reduce database load and improve response times.

### SeaweedFS

SeaweedFS is a scalable distributed file system for object storage. It is used here for:

- storing large files and binary objects
- providing a distributed filer interface
- supporting replication, snapshots, and scalable storage volumes

SeaweedFS configuration files are located in `seaweed-storage/`.

## Repository Structure

- `databases/`
  - `docker-compose.db.yaml` - PostgreSQL and Redis compose setup
  - `README.md` - database-specific notes and usage instructions

- `seaweed-storage/`
  - `docker-compose.seaweed.yaml` - main SeaweedFS compose definition
  - `docker-compose.volume2.yaml` - additional SeaweedFS volume configuration
  - `REPLICATION_BACKUP.md` - SeaweedFS replication and backup guidance
  - `STORAGE_README.md` - SeaweedFS storage usage notes
  - `data/` - runtime SeaweedFS data directories for filer, master, and volume servers

## Getting Started

1. Start the database stack:

   ```bash
   docker compose -f databases/docker-compose.db.yaml up -d
   ```

2. Start SeaweedFS:

   ```bash
   docker compose -f seaweed-storage/docker-compose.seaweed.yaml up -d
   ```

3. Verify services are running and consult the local compose logs if needed.

## Notes

- PostgreSQL is the primary relational store.
- Redis is intended for caching and fast state access.
- SeaweedFS is used for scalable file storage and object persistence.

For more details, consult the service-specific documentation files in each folder.



---
---
---
# Create the directory if it doesn't exist
mkdir -p ./data/filer_meta

# Change ownership to UID 1000 (the standard user inside the SeaweedFS image)
sudo chown -R 1000:1000 ./data/filer_meta

# Ensure read/write/execute permissions
chmod -R 755 ./data/filer_meta   