# Docker Volumes & Storage

**Persistent Data in an Ephemeral World**

> Understanding how Docker manages data -- from ephemeral container layers to production-grade persistent storage strategies

`Container` -> `Volume` -> `Mount` -> `Persist` -> `Backup`

Volumes - Storage - Persistence - DevOps

---

## Table of Contents

1. [Topics](#topics)
2. [The Container Filesystem](#the-container-filesystem)
3. [Union Filesystems & Copy-on-Write](#union-filesystems--copy-on-write)
4. [Docker Storage Mount Types](#docker-storage-mount-types)
5. [Named Volumes vs Anonymous Volumes](#named-volumes-vs-anonymous-volumes)
6. [Bind Mounts](#bind-mounts)
7. [tmpfs Mounts](#tmpfs-mounts)
8. [Volume Drivers & Plugins](#volume-drivers--plugins)
9. [Docker Volume Commands Cheat Sheet](#docker-volume-commands-cheat-sheet)
10. [Sharing Data Between Containers](#sharing-data-between-containers)
11. [Volumes in Docker Compose](#volumes-in-docker-compose)
12. [Backup & Restore Strategies](#backup--restore-strategies)
13. [Storage Drivers](#storage-drivers)
14. [Performance Considerations](#performance-considerations)
15. [Database Containers & Persistence](#database-containers--persistence)
16. [Production Storage Patterns](#production-storage-patterns)
17. [Volume Security Considerations](#volume-security-considerations)
18. [Summary & Further Reading](#summary--further-reading)

---

## Topics

### Storage Fundamentals

- The container filesystem
- Union filesystems & layers
- Named vs anonymous volumes
- Bind mounts

### Mount Types & Commands

- tmpfs mounts
- Volume drivers & plugins
- Docker volume commands cheat sheet
- Sharing data between containers

### Compose & Operations

- Volumes in Docker Compose
- Backup & restore strategies
- Storage drivers (overlay2, btrfs)
- Performance considerations

### Production Patterns

- Database containers & persistence
- Production storage patterns
- Security considerations
- Summary & further reading

---

## The Container Filesystem

Containers use a **layered, copy-on-write** filesystem. Each image layer is read-only; the running container adds a thin writable layer on top.

### How Layers Work

- Each Dockerfile instruction creates a new layer
- Layers are **read-only** and shared between containers
- A thin **writable layer** is added when the container starts
- Writes use copy-on-write -- modified files are copied up
- When the container is removed, the writable layer is **deleted**

### The Ephemeral Problem

- Container data is **temporary by default**
- `docker rm` destroys all written data
- No way to share data between containers
- Writable layer is tightly coupled to the host
- Performance penalty from copy-on-write

### Layer Diagram

```
┌─────────────────────────────────────┐
│   Writable Container Layer (R/W)    │  ← Deleted on container removal
├─────────────────────────────────────┤
│   Layer 4: CMD / EXPOSE             │  ← Read-only
├─────────────────────────────────────┤
│   Layer 3: COPY app source          │  ← Read-only
├─────────────────────────────────────┤
│   Layer 2: RUN npm install          │  ← Read-only
├─────────────────────────────────────┤
│   Layer 1: FROM node:20-alpine      │  ← Read-only (base image)
└─────────────────────────────────────┘
```

---

## Union Filesystems & Copy-on-Write

Docker uses a **union filesystem** (UnionFS) to merge multiple read-only layers into a single coherent view. The default on modern Linux is **OverlayFS**.

### OverlayFS Mechanics

- **lowerdir** -- stacked read-only image layers
- **upperdir** -- writable container layer
- **merged** -- unified view presented to the container
- **workdir** -- internal scratch space for atomic operations

### Copy-on-Write (CoW)

- First write to a file copies it from lowerdir to upperdir
- Subsequent reads/writes use the upperdir copy
- Deletions create a *whiteout* file in the upperdir
- Efficient for many containers sharing the same base image

```bash
# Inspect the storage driver in use
docker info | grep "Storage Driver"
# Storage Driver: overlay2

# See the layers of an image
docker inspect node:20-alpine \
  --format='{{.GraphDriver.Data}}'

# View overlay mount for a running container
docker inspect myapp \
  --format='{{.GraphDriver.Data.MergedDir}}'

# Check disk usage by layer
docker system df -v
```

---

## Docker Storage Mount Types

Docker provides **three mount types** to persist data beyond the container lifecycle. Each serves a different purpose.

### Volumes

- Managed by Docker
- Stored in `/var/lib/docker/volumes/`
- Best for persistent application data
- Survive container removal
- Can use remote drivers (NFS, cloud)
- Works on Linux, Mac, Windows

### Bind Mounts

- Map any host path into container
- Full control over host location
- Great for dev (hot reload)
- Host changes visible instantly
- Can be read-only (`:ro`)
- Dependent on host file structure

### tmpfs Mounts

- Stored in host memory only
- Never written to disk
- Good for secrets, temp data
- Lost when container stops
- Linux only
- Configurable size limits

### Diagram

```
    Host Filesystem                     Docker Area                  Memory
   ┌──────────────┐              ┌──────────────────────┐       ┌──────────┐
   │  /home/user/ │──bind mount──│ /var/lib/docker/     │       │  tmpfs   │
   │  /data/app/  │              │   volumes/mydata/_data│      │  (RAM)   │
   └──────────────┘              └──────────────────────┘       └──────────┘
         ↕                               ↕                          ↕
   ┌─────────────────────────────────────────────────────────────────────┐
   │                     Container Filesystem                           │
   └─────────────────────────────────────────────────────────────────────┘
```

---

## Named Volumes vs Anonymous Volumes

### Named Volumes

- Explicit, user-defined name
- Easy to reference and manage
- Persist independently of containers
- Can be shared between containers
- **Recommended** for production use

```bash
# Create a named volume
docker volume create pgdata

# Use it with a container
docker run -d \
  -v pgdata:/var/lib/postgresql/data \
  --name db postgres:16

# Same volume, new container
docker run -d \
  -v pgdata:/var/lib/postgresql/data \
  --name db2 postgres:16
```

### Anonymous Volumes

- Auto-generated hash name
- Hard to identify and manage
- Created by `VOLUME` in Dockerfile or `-v /path`
- Accumulate as orphans over time
- **Avoid** in production -- use named volumes instead

```bash
# Creates an anonymous volume
docker run -d \
  -v /var/lib/postgresql/data \
  postgres:16

# List volumes — spot the anonymous ones
docker volume ls
# DRIVER    VOLUME NAME
# local     pgdata          ← named
# local     a1b2c3d4e5f6... ← anonymous

# Clean up dangling anonymous volumes
docker volume prune
```

---

## Bind Mounts

Bind mounts map a **specific host directory** into the container. Changes on either side are reflected instantly.

### Development Use Cases

- Hot-reload source code without rebuilding
- Share config files from host
- Access build output on host
- Mount test fixtures

### Cautions

- Container can modify host files
- Path must exist on host (not auto-created with --mount)
- Breaks portability (host-dependent paths)
- Permission issues between host/container UIDs

### Syntax Comparison

```bash
# Legacy -v syntax
docker run -d \
  -v $(pwd)/src:/app/src \
  -v $(pwd)/config:/app/config:ro \
  myapp:dev

# Modern --mount syntax (recommended)
docker run -d \
  --mount type=bind,src=$(pwd)/src,dst=/app/src \
  --mount type=bind,src=$(pwd)/config,dst=/app/config,readonly \
  myapp:dev

# Key difference: --mount errors if
# source doesn't exist (safer)
# -v silently creates an empty directory
```

### Read-Only Bind Mounts

Add `:ro` or `readonly` to prevent the container from modifying host files. Essential for config and secrets.

---

## tmpfs Mounts

tmpfs mounts store data in **host memory only**. Data is never written to the host filesystem and is lost when the container stops.

### When to Use tmpfs

- **Secrets & credentials** -- never touch disk
- **Session state** -- ephemeral by nature
- **Scratch/temp files** -- faster than disk I/O
- **Build caches** -- no need to persist
- **Security-sensitive data** -- no forensic trace on disk

### Limitations

- Linux hosts only (not on Docker Desktop for Mac/Windows)
- Cannot be shared between containers
- Consumes host RAM -- set size limits
- Data lost on container stop (not just removal)

```bash
# Using --tmpfs flag
docker run -d \
  --tmpfs /tmp:rw,size=64m \
  --tmpfs /run/secrets:rw,size=1m \
  myapp:1.0

# Using --mount syntax (more options)
docker run -d \
  --mount type=tmpfs,dst=/tmp,tmpfs-size=67108864 \
  --mount type=tmpfs,dst=/run/secrets,tmpfs-mode=0700 \
  myapp:1.0

# tmpfs-size: bytes (64MB = 67108864)
# tmpfs-mode: file permissions (octal)
```

---

## Volume Drivers & Plugins

Docker volumes use the **local driver** by default, but plugins enable storage on remote and cloud backends.

| Driver / Plugin | Backend | Use Case |
|-----------------|---------|----------|
| `local` | Host filesystem | Default; single-host workloads |
| `local` + NFS opts | NFS share | Shared storage across hosts |
| `rexray/ebs` | AWS EBS | Persistent block storage on AWS |
| `rexray/efs` | AWS EFS | Shared file storage on AWS |
| `azure_file` | Azure Files | Shared storage on Azure |
| `vieux/sshfs` | SSH/SFTP | Mount remote dir via SSH |
| `netapp/trident` | NetApp | Enterprise SAN/NAS |

```bash
# Create NFS-backed volume
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.100,rw \
  --opt device=:/exports/data \
  nfs_data

# Install and use a plugin
docker plugin install vieux/sshfs

docker volume create \
  --driver vieux/sshfs \
  -o sshcmd=user@host:/remote/path \
  -o password=secret \
  sshvol
```

---

## Docker Volume Commands Cheat Sheet

| Command | Description | Example |
|---------|-------------|---------|
| `docker volume create` | Create a named volume | `docker volume create mydata` |
| `docker volume ls` | List all volumes | `docker volume ls --filter dangling=true` |
| `docker volume inspect` | Show volume details | `docker volume inspect mydata` |
| `docker volume rm` | Remove a volume | `docker volume rm mydata` |
| `docker volume prune` | Remove all unused volumes | `docker volume prune -f` |
| `docker run -v` | Mount volume (legacy syntax) | `docker run -v mydata:/app/data img` |
| `docker run --mount` | Mount volume (modern syntax) | `docker run --mount src=mydata,dst=/data img` |

### -v vs --mount Syntax

`-v name:/path` -- concise, auto-creates volumes

`--mount type=volume,src=name,dst=/path` -- explicit, errors on missing source. **Prefer --mount** for clarity.

### Inspect Output

```json
{
  "Name": "pgdata",
  "Driver": "local",
  "Mountpoint": "/var/lib/docker/volumes/pgdata/_data",
  "Scope": "local"
}
```

---

## Sharing Data Between Containers

Named volumes can be mounted by **multiple containers simultaneously**, enabling shared storage and data pipelines.

### Pattern: Shared Volume

```bash
# Create a shared volume
docker volume create shared_data

# Writer container
docker run -d --name writer \
  -v shared_data:/data \
  alpine sh -c \
  'while true; do date >> /data/log.txt; sleep 5; done'

# Reader container
docker run -d --name reader \
  -v shared_data:/data:ro \
  alpine tail -f /data/log.txt
```

### Pattern: Sidecar / Init Container

```bash
# Init: populate config from git
docker run --rm \
  -v app_config:/config \
  alpine/git clone \
  https://github.com/org/config.git /config

# App: use the config
docker run -d \
  -v app_config:/app/config:ro \
  myapp:latest
```

### Concurrency Warning

Multiple writers to the same volume can cause **data corruption**. Use application-level locking or a database for concurrent access.

---

## Volumes in Docker Compose

```yaml
# docker-compose.yml
services:
  app:
    build: .
    ports: ["3000:3000"]
    volumes:
      - ./src:/app/src           # bind mount (dev)
      - app_uploads:/app/uploads # named volume
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready"]
      interval: 5s

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes

volumes:
  pgdata:                 # default local driver
  redis_data:
    driver: local
  app_uploads:
    driver_opts:
      type: none
      o: bind
      device: /mnt/uploads
```

### Top-Level `volumes:` Key

- Declares named volumes for the project
- Compose prefixes with project name: `myapp_pgdata`
- Supports driver, driver_opts, external, labels

### External Volumes

```yaml
volumes:
  pgdata:
    external: true  # must pre-exist
    name: production_pgdata
```

Use `external: true` to reference volumes managed outside Compose.

### Lifecycle

- `docker compose down` -- keeps volumes
- `docker compose down -v` -- **destroys volumes**
- Always be cautious with `-v` flag in production

---

## Backup & Restore Strategies

### Volume Backup with tar

```bash
# Backup: mount volume into temp container
docker run --rm \
  -v pgdata:/source:ro \
  -v $(pwd):/backup \
  alpine tar czf /backup/pgdata-backup.tar.gz \
    -C /source .

# Restore: extract into volume
docker run --rm \
  -v pgdata:/target \
  -v $(pwd):/backup \
  alpine tar xzf /backup/pgdata-backup.tar.gz \
    -C /target
```

### Database-Native Backup

```bash
# PostgreSQL — pg_dump
docker exec db \
  pg_dump -U postgres mydb \
  > backup.sql

# MySQL — mysqldump
docker exec mysql \
  mysqldump -u root -p mydb \
  > backup.sql

# Restore
cat backup.sql | docker exec -i db \
  psql -U postgres mydb
```

### Automated Backup Script

```bash
#!/bin/bash
# backup-volumes.sh
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR=/backups

for vol in pgdata redis_data uploads; do
  docker run --rm \
    -v ${vol}:/source:ro \
    -v ${BACKUP_DIR}:/backup \
    alpine tar czf \
      /backup/${vol}_${DATE}.tar.gz \
      -C /source .
done

# Retention: keep last 7 days
find ${BACKUP_DIR} -name "*.tar.gz" \
  -mtime +7 -delete
```

### Cloud Backup

- Pipe tar output to `aws s3 cp` or `gsutil cp`
- Use volume snapshots on cloud providers (EBS snapshots)
- Schedule with cron or container-based scheduler
- Always test your restore process

---

## Storage Drivers

Storage drivers control how **image layers and the writable container layer** are stored on disk. This is different from volume drivers.

| Driver | Backing Filesystem | Status | Notes |
|--------|-------------------|--------|-------|
| **overlay2** | xfs, ext4 | Default & recommended | Best all-round performance; uses kernel OverlayFS |
| **btrfs** | btrfs | Supported | Native snapshots; good for many builds |
| **zfs** | zfs | Supported | Snapshots, compression, dedup; resource-heavy |
| **devicemapper** | direct-lvm | Deprecated | Was default on CentOS/RHEL; avoid for new installs |
| **vfs** | Any | Testing only | No CoW; full copy per layer. Very slow, very large |

### Check / Configure

```bash
# Check current driver
docker info | grep -i storage

# Set in /etc/docker/daemon.json
{
  "storage-driver": "overlay2"
}
```

### Recommendation

Use **overlay2** unless you have a specific reason not to. It is the default on all major Linux distributions and offers the best balance of performance and stability.

---

## Performance Considerations

### I/O Performance Ranking

| Mount Type | Read | Write | Best For |
|------------|------|-------|----------|
| tmpfs | Fastest | Fastest | Temp/scratch data |
| Bind mount | Native | Native | Dev, host data |
| Named volume | Near-native | Near-native | Databases, state |
| Container layer | Slower | Slowest | Avoid for data |

### Docker Desktop (Mac/Windows)

- Bind mounts are **much slower** due to VM file sharing
- Use `:cached` or `:delegated` consistency modes
- VirtioFS (default on newer versions) is faster than gRPC-FUSE
- Named volumes bypass the VM sharing -- prefer for `node_modules`

### Optimization Tips

- Write-heavy apps: use named volumes, not the container layer
- Avoid storing large datasets in the writable layer
- Use `--mount type=tmpfs` for temp/scratch data
- On Mac: keep `node_modules` in a named volume
- Use `.dockerignore` to reduce build context
- Match filesystem (xfs with `d_type=true` for overlay2)

### Benchmarking

```bash
# Test write performance
docker run --rm -v mydata:/data alpine \
  dd if=/dev/zero of=/data/test \
  bs=1M count=1024 oflag=direct

# Compare with container layer
docker run --rm alpine \
  dd if=/dev/zero of=/tmp/test \
  bs=1M count=1024 oflag=direct
```

---

## Database Containers & Persistence

Running databases in Docker is common, but **losing data is the #1 beginner mistake**. Always use named volumes.

### PostgreSQL

```yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: admin
      POSTGRES_PASSWORD_FILE: /run/secrets/db_pass
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./init:/docker-entrypoint-initdb.d:ro
    secrets:
      - db_pass

volumes:
  pgdata:
```

### MySQL / MariaDB

```bash
docker run -d --name mysql \
  -e MYSQL_ROOT_PASSWORD=secret \
  -v mysql_data:/var/lib/mysql \
  mysql:8
```

### MongoDB

```bash
docker run -d --name mongo \
  -v mongo_data:/data/db \
  -v mongo_config:/data/configdb \
  mongo:7
```

### Redis (Persistent)

```bash
docker run -d --name redis \
  -v redis_data:/data \
  redis:7-alpine \
  redis-server --appendonly yes
```

### Critical Rules

- **Always** use named volumes for DB data dirs
- **Never** use `docker compose down -v` carelessly
- Test backup & restore **before** you need it
- In production, consider managed databases instead

---

## Production Storage Patterns

### Pattern: Named Volumes + Backups

- Named volumes for all persistent data
- Automated daily backups to S3/GCS
- Tested restore runbook
- Volume labels for management

```bash
docker volume create \
  --label env=production \
  --label service=api \
  api_uploads
```

### Pattern: NFS for Shared Storage

- Multiple containers across hosts sharing data
- NFS volume driver for cross-host access
- Suitable for static assets, uploads, shared config

### Pattern: Cloud Block Storage

- AWS EBS / GCP Persistent Disk / Azure Disk
- Snapshot-based backups
- IOPS provisioning for databases
- Tied to availability zone -- plan for failover

### Pattern: Object Storage for Assets

- S3 / GCS / Azure Blob for user uploads, media
- Don't store large files in volumes -- use object storage
- CDN integration for serving static assets
- Infinite scale, built-in redundancy

### Anti-Pattern: Data in Container Layer

Never store important data without a volume. One `docker rm` and it's gone forever.

---

## Volume Security Considerations

### Common Security Risks

- Bind-mounting `/` or `/etc` gives container full host access
- Mounting Docker socket (`/var/run/docker.sock`) = root on host
- Sensitive data persisted in volumes without encryption
- World-readable volume permissions
- Orphaned volumes containing secrets

### Best Practices

- Use **read-only** bind mounts where possible
- Run containers as non-root user
- Use Docker secrets or tmpfs for credentials
- Set `read_only: true` on the container root filesystem
- Regularly prune unused volumes

### Read-Only Root Filesystem

```bash
# Immutable container with targeted writable mounts
docker run -d \
  --read-only \
  --tmpfs /tmp:rw,size=64m \
  --tmpfs /run:rw,size=8m \
  -v app_data:/app/data \
  myapp:latest
```

### Docker Secrets (Swarm)

```bash
# Create a secret
echo "s3cr3t" | docker secret create db_pass -

# Use in service (mounted at /run/secrets/)
docker service create \
  --secret db_pass \
  --name db postgres:16
```

Secrets are stored encrypted at rest and mounted as tmpfs in the container.

---

## Summary & Further Reading

### Key Takeaways

- Container data is **ephemeral** -- volumes make it persist
- **Named volumes** for production; bind mounts for dev
- **tmpfs** for secrets and scratch data
- Always use volumes for **database containers**
- Backup your volumes -- and **test your restores**
- **overlay2** is the recommended storage driver
- Use `--mount` over `-v` for clarity
- Never store critical data in the container layer

### Decision Quick Reference

| Scenario | Use |
|----------|-----|
| Database data | Named volume |
| Source code (dev) | Bind mount |
| Secrets | tmpfs / Docker secrets |
| Shared config | Read-only bind mount |
| Cross-host sharing | NFS volume driver |
| Large file uploads | Object storage (S3) |

### Further Reading

- **Docker Docs** -- docs.docker.com/storage/
- **Volumes** -- docs.docker.com/storage/volumes/
- **Bind Mounts** -- docs.docker.com/storage/bind-mounts/
- **Storage Drivers** -- docs.docker.com/storage/storagedriver/
- **"Docker Deep Dive"** by Nigel Poulton
- **"Kubernetes Patterns"** by Ibryam & Huss

### Hands-On Practice

- Run PostgreSQL with a named volume, insert data, recreate the container
- Write a backup/restore script for a Docker volume
- Set up a Compose project with bind mounts + named volumes
- Benchmark volume vs container layer write performance
- Configure NFS-backed volumes across two hosts

### Tools Worth Knowing

`dive` (inspect layers) - `lazydocker` (TUI) - `ctop` (container top) - `duf` (disk usage) - `ncdu` (volume size analysis)
