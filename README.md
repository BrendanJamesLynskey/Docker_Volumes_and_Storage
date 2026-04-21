# 💾 Docker Volumes and Storage

An interactive Reveal.js presentation on Docker storage — named volumes, bind mounts, tmpfs, volume drivers, backup strategies, storage drivers, and database persistence.

## ▶ [Open the Presentation](https://brendanjameslynskey.github.io/Docker_Volumes_and_Storage/)

## 📄 [Markdown Version](presentation.md)

---

## Contents

| # | Topic | Description |
|---|-------|-------------|
| 01 | Topics | Fundamentals, mounts, Compose, production patterns |
| 02 | The Container Filesystem | Layered read-only image plus writable layer |
| 03 | Union Filesystems & Copy-on-Write | OverlayFS mechanics and CoW semantics |
| 04 | Docker Storage Mount Types | Volumes, bind mounts, and tmpfs overview |
| 05 | Named Volumes vs Anonymous Volumes | Managed volumes versus auto-generated hashes |
| 06 | Bind Mounts | Host directories for dev and shared config |
| 07 | tmpfs Mounts | In-memory ephemeral storage for secrets |
| 08 | Volume Drivers & Plugins | Local, NFS, EBS, EFS, SSHFS backends |
| 09 | Docker Volume Commands Cheat Sheet | create, ls, inspect, rm, prune |
| 10 | Sharing Data Between Containers | Shared volumes and sidecar/init patterns |
| 11 | Volumes in Docker Compose | Top-level volumes key, external volumes |
| 12 | Backup & Restore Strategies | tar, database-native dumps, automation |
| 13 | Storage Drivers | overlay2, btrfs, zfs, devicemapper, vfs |
| 14 | Performance Considerations | Mount type I/O ranking and Docker Desktop |
| 15 | Database Containers & Persistence | Postgres, MySQL, Mongo, Redis with volumes |
| 16 | Production Storage Patterns | Named volumes, NFS, cloud block, object storage |
| 17 | Volume Security Considerations | Read-only roots, secrets, Docker socket risks |
| 18 | Summary & Further Reading | Decision reference and recommended tools |

---

## Slide Controls

| Action | Key |
|--------|-----|
| Next / Previous | `→` `←` or swipe |
| Overview | `Esc` |
| Fullscreen | `F` |
| Export to PDF | Append `?print-pdf` to URL, then print |

## Technology

[Reveal.js 4.6](https://revealjs.com) · [highlight.js](https://highlightjs.org) · Playfair Display + DM Sans + JetBrains Mono

Single self-contained `index.html` — no build step, no npm, no dependencies to install.

## References

[Docker Storage Documentation](https://docs.docker.com/storage/) · [Volumes](https://docs.docker.com/storage/volumes/) · [Bind Mounts](https://docs.docker.com/storage/bind-mounts/) · [Storage Drivers](https://docs.docker.com/storage/storagedriver/) · *Docker Deep Dive* by Nigel Poulton

## License

Educational use. Code examples provided as-is.
