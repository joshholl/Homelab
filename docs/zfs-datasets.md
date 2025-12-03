# ZFS Dataset & Configuration Strategy

This document describes how ZFS datasets are structured and which properties are used for media vs application data.

---

## Pools

- **`aegir`** (HDD)  
  Large capacity pool with metadata special vdevs on SSDs (already configured).

- **`verkamaedhr`** (SSD)  
  Smaller, low-latency pool intended for container configs and databases.

---

## Dataset Tree

### On `aegir` (HDD, media)

Suggested datasets:

- `aegir/media`  
- `aegir/media/photos-raw`  
- `aegir/media/photos-exports`  
- `aegir/media/videos`  
- `aegir/media/plex`  
- `aegir/media/immich-library`  

- `aegir/import`  
- `aegir/import/photosync`  

- `aegir/backups`  
- `aegir/backups/lightroom`  

- `aegir/family-share`  

Mountpoints:

- `/mnt/aegir/media/photos-raw`
- `/mnt/aegir/media/photos-exports`
- `/mnt/aegir/media/videos`
- `/mnt/aegir/media/plex`
- `/mnt/aegir/media/immich-library`
- `/mnt/aegir/import/photosync`
- `/mnt/aegir/backups/lightroom`
- `/mnt/aegir/family-share`

### On `verkamaedhr` (SSD, apps)

Datasets:

- `verkamaedhr/docker`
  - `verkamaedhr/docker/immich-db`
  - `verkamaedhr/docker/immich-config`
  - `verkamaedhr/docker/mediacms-db`
  - `verkamaedhr/docker/hedgedoc`
  - `verkamaedhr/docker/filebrowser`
  - `verkamaedhr/docker/code-server`
  - `verkamaedhr/docker/plex`
  - `verkamaedhr/docker/spoolman`
  - `verkamaedhr/docker/dockge`

Mountpoints under `/mnt/verkamaedhr/docker/*`.

---

## ZFS Properties & Rationale

Below are recommended defaults; you can override per-dataset where needed from the TrueNAS UI.

### Common defaults

For almost all datasets:

- `compression = lz4`  
  - Lightweight, safe default; compresses text, JPEG sidecars, DB pages well, minimal CPU.

- `xattr = sa`  
  - Stores extended attributes in the inode, improving performance for many small files.

- `acltype = nfsv4` (or whatever TrueNAS sets by default)  
  - Keeps SMB permissions happy.

### Media datasets (large files)

**Datasets:**

- `aegir/media/photos-raw`
- `aegir/media/photos-exports`
- `aegir/media/videos`
- `aegir/media/plex`
- `aegir/media/immich-library`

Recommended:

- `atime = off`  
  - Avoids writes every time a photo/video is read.

- `recordsize = 1M`  
  - Ideal for large, mostly sequential I/O (RAW stills, video, streams).  
  - Reduces metadata overhead and improves throughput.

- `logbias = throughput`  
  - Favors sequential throughput over latency, which suits streaming and large reads.

### Import / “inbox” dataset

**Dataset:**

- `aegir/import/photosync`

Recommended:

- `atime = off` or `relatime`  
- `recordsize = 128K`  
  - Mixed workloads, a reasonable compromise between large media and smaller files.

This dataset is just a landing zone for PhotoSync/Immich. Files are either indexed by Immich or moved into `photos-raw`.

### Backups dataset

**Dataset:**

- `aegir/backups/lightroom`

Lightroom catalog backups are SQLite DBs and small side files.

Recommended:

- `compression = lz4`
- `recordsize = 16K`  
  - More aligned with SQLite/DB workloads and many small files.

- `atime = off`

### Family share

**Dataset:**

- `aegir/family-share`

Mixed workload (documents, photos, etc).

Recommended:

- `compression = lz4`
- `recordsize = 128K`
- `atime = relatime` (if you care about access tracking) or `off` for less write amplification.

---

## Application datasets on `verkamaedhr`

App data are mostly databases and configs, so latency matters more than raw throughput.

**Dataset: `verkamaedhr/docker` and children**

Recommended:

- `compression = lz4`
- `recordsize = 16K`  
  - Good fit for PostgreSQL, SQLite, config files.
- `atime = off`
- `logbias = latency`

For especially busy DBs (Immich, MediaCMS), you can create dedicated datasets:

- `verkamaedhr/docker/immich-db` with `recordsize = 8K`  
- `verkamaedhr/docker/mediacms-db` with `recordsize = 8K`

This matches typical DB page sizes, but 16K is a fine starting point if you want simplicity.

---

## Snapshots

### Principles

- **Media datasets** (mostly append-only) don’t need aggressive schedules.
- **App/DB datasets** need more frequent snapshots for safety.
- Keep a mix of **short-term, medium-term, and long-term** retention.

### Suggested snapshot tasks (TrueNAS)

#### For media (recursive on `aegir/media`)

1. **Hourly**, kept for **24 hours**  
   - Protects from accidental same-day deletions/overwrites.

2. **Daily**, kept for **30 days**  
   - Short-term rollback for projects.

3. **Weekly**, kept for **52 weeks**  
   - Long-term safety for slow-moving media.

You can do this with three **Periodic Snapshot Tasks** in TrueNAS, each with different schedules and lifetimes.

#### For backups & family data

**`aegir/backups` and `aegir/family-share`** (recursive):

- **Daily**, kept for **90 days**.

#### For app data (`verkamaedhr/docker`)

These change fast (DBs, config updates):

1. **Every 30 minutes**, kept for **48 hours** (recursive on `verkamaedhr/docker`)
2. **Daily**, kept for **30 days**

This combination protects from “oops I upgraded and broke it” as well as silent config mistakes.

---

## Backups to Backblaze B2

Not everything is worth sending to the cloud. The idea:

- **Must-keep:**
  - `aegir/media/photos-raw`
  - `aegir/backups/lightroom`
- **Nice-to-have, optional:**
  - Select directories from `aegir/media/videos` (important projects)
  - `aegir/family-share` (documents)

### Implementation (TrueNAS Cloud Sync)

Use TrueNAS **Cloud Sync Tasks** to Backblaze B2:

1. Create a B2 Credential in TrueNAS.
2. Create tasks:
   - `aegir/media/photos-raw` → `b2://<bucket>/photos-raw`
   - `aegir/backups/lightroom` → `b2://<bucket>/lightroom-backups`
3. Set them to run **daily** or **nightly**.
4. Use **PULL** from cloud if needed to restore; normal mode will be **PUSH** from NAS.

---

## SMB Shares

Recommended SMB shares:

- `Media` → `/mnt/aegir/media`  
  - Read-only for most users, read/write for you.

- `PhotosRaw` → `/mnt/aegir/media/photos-raw`  
  - You and any workstation editing photos.

- `LightroomBackups` → `/mnt/aegir/backups/lightroom`  
  - Only your user needs write.

- `Photosync` → `/mnt/aegir/import/photosync`  
  - Used as PhotoSync target.

- `Family` → `/mnt/aegir/family-share`  
  - ACLs for each family member.

These SMB shares back the workflows for Lightroom, PhotoSync, and general family usage.

---

## Why not put DBs on `aegir`?

Databases (Immich, MediaCMS, HedgeDoc, etc.) are latency-sensitive and don’t need huge capacity.

- Putting them on **SSD (`verkamaedhr`)**:
  - Speeds up all DB operations.
  - Keeps ZFS intent log and metadata on fast media.
- The actual large assets (photos/videos) remain on HDD where capacity is cheap.

This is a classic “fast metadata, big data” design for home media servers.

