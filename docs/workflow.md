# Homelab Media & Content Workflow

This document describes how media, projects, and blogs flow through the system.

## Goals

- Centralize **all photos and videos** on the NAS.
- Keep **Lightroom Classic** fast by using local catalogs + NAS media.
- Use **Immich** as the primary remote viewer of photos and videos.
- Use **MediaCMS** to host and stream large video files.
- Use **HedgeDoc** for project notes and blog drafts.
- Avoid unnecessary duplication of large media.
- Keep everything reproducible via `docker-compose` stacks managed by Dockge.

---

## Storage Overview

### Pools

- `/mnt/aegir` – HDD pool for media & bulk storage.
- `/mnt/verkamaedhr` – SSD pool for container data, databases, configs.

### Key datasets & purposes

On **`/mnt/aegir`**:

- `/mnt/aegir/media/photos-raw`  
  Long-term home for original RAW stills.

- `/mnt/aegir/media/photos-exports`  
  Finished JPEG/PNG/TIFF exports from Lightroom for sharing.

- `/mnt/aegir/media/videos`  
  Original video footage and DaVinci Resolve project media.

- `/mnt/aegir/media/plex`  
  Media libraries for Plex (movies, TV, home videos if desired).

- `/mnt/aegir/media/immich-library`  
  Immich originals and derived files (thumbnails, etc).

- `/mnt/aegir/import/photosync`  
  Target for PhotoSync auto-upload from phones/tablets.

- `/mnt/aegir/backups/lightroom`  
  Backups of Lightroom catalogs and presets.

- `/mnt/aegir/family-share`  
  General SMB share for family documents, exports, etc.

On **`/mnt/verkamaedhr`**:

- `/mnt/verkamaedhr/docker/*`  
  Each app has its own folder for configs and DBs:
  - `immich-db`, `immich-config`
  - `mediacms-db`
  - `hedgedoc`
  - `filebrowser`
  - `code-server`
  - `plex`
  - `spoolman`
  - `dockge`

---

## Photography Workflow

### 1. Ingest

**From cameras (SD / CFexpress):**

1. Copy new shoots to a **local NVMe drive** on the workstation:
   - Example path: `D:\Photos\_ingest\YYYY\YYYY-MM-DD_project-name`
2. Cull quickly in Lightroom or Photo Mechanic.

**From phones / tablets:**

Two options:

- **Immich app** – recommended going forward: automatic backup to Immich server. :contentReference[oaicite:0]{index=0}  
- **PhotoSync** – alternate path:
  - Configure PhotoSync to upload to the NAS via SMB/WebDAV, target:  
    `\\nas\photosync` → `/mnt/aegir/import/photosync` :contentReference[oaicite:1]{index=1}  

Immich can be configured to index both its own upload folder and possibly a “watched” library on `/mnt/aegir/media/immich-library`.

### 2. Organize & store RAWs

1. After culling, move RAWs from local NVMe to:
   - `\\nas\photos-raw\YYYY\YYYY-MM-DD_project-name`
   - Maps to `/mnt/aegir/media/photos-raw/…` on the NAS.
2. Lightroom Classic catalog keeps **references** to files over SMB.

**Catalog placement:**

- Primary Lightroom catalog lives on **local SSD** for performance.
- Lightroom’s built-in **catalog backup** is pointed at:
  - `\\nas\lightroom-backups` → `/mnt/aegir/backups/lightroom`.

This keeps editing snappy while the NAS acts as safe storage for RAWs and catalog backups.

### 3. Editing & exports

- Edit in Lightroom Classic on the workstation.
- Export finished JPEGs to:
  - `\\nas\photos-exports\YYYY\YYYY-MM-DD_project-name`
  - That maps to `/mnt/aegir/media/photos-exports/...`.

Optional: a “social” subfolder for downsized or watermarked versions.

### 4. Immich usage

- Immich stores its library in `/mnt/aegir/media/immich-library`.
- Immich DB & configs live in `/mnt/verkamaedhr/docker/immich-*`.

Suggested strategy:

- Let Immich manage its own upload structure internally.
- For historical photos already arranged in `photos-raw` or `photos-exports`, configure an **external library path** for Immich to index (read-only).

Result:

- Lightroom remains the **source of truth** for edits / ratings.
- Immich is the **viewer & sharing** layer, especially for mobile/remote access.

---

## Video Workflow (DaVinci Resolve + MediaCMS)

### 1. Ingest

- Copy footage from card → local NVMe for active editing:
  - Example: `D:\Video\Projects\YYYY\YYYY-MM-DD_project-name`.
- Mirror that structure onto NAS:

  - `\\nas\videos\YYYY\YYYY-MM-DD_project-name`
  - `/mnt/aegir/media/videos/…`

Use **Syncthing or rsync** between your local NVMe project folder and the NAS folder so you don’t manually copy things for every change.

### 2. Resolve project files

Options:

- Store Resolve project library on local SSD for maximum performance.
- Export project archives to:
  - `\\nas\videos\project-archives`
  - for long-term preservation.

### 3. MediaCMS

MediaCMS is used as a “YouTube-like” front end for finished or WIP videos. :contentReference[oaicite:2]{index=2}  

Workflow:

1. Render final or review versions from Resolve directly to:
   - `/mnt/aegir/media/videos/mediacms-uploads`.
2. Point MediaCMS to this path for ingest/processing.
3. MediaCMS stores its database on SSD:
   - `/mnt/verkamaedhr/docker/mediacms-db`.
4. Generated encoded assets stay on HDD with the rest of your video library.

---

## Blogging & Documentation (HedgeDoc + Git)

HedgeDoc gives you fast collaborative Markdown editing for:

- Project notes
- Blog post drafts
- Checklists/shot lists

### Layout

- HedgeDoc config & data:
  - `/mnt/verkamaedhr/docker/hedgedoc`

Within HedgeDoc:

- One “Notebook” per project or per year.
- Each **project note** links to:
  - NAS photo path (`/mnt/aegir/media/photos-raw/…`)
  - Resolve project folder (`/mnt/aegir/media/videos/…`)
  - Immich/MediaCMS share URLs.

Finished posts can be exported to Markdown and checked into a **separate static-blog repo** or into `Homelab` under `content/` if you want to keep everything together.

---

## Remote Access

- **Photos & videos:** Immich (primary), MediaCMS for long-form / project videos.
- **Generic files & family docs:** Filebrowser, rooted at `/mnt/aegir` but locked down via ACLs.
- **Development & Homelab config:** code-server, working directly in the `Homelab` repo and related config directories.

---

## Family & Data Management

- **Family SMB shares:**
  - `\\nas\family` → `/mnt/aegir/family-share`
  - `\\nas\dropbox` → `/mnt/aegir/import/photosync`

- **Spoolman:** manages 3D printer filaments; DB lives on SSD, no huge storage.

- **Backups:**
  - Critical photos (e.g., `/mnt/aegir/media/photos-raw`) and Lightroom backups (`/mnt/aegir/backups/lightroom`) are synced to **Backblaze B2** using TrueNAS Cloud Sync tasks.
  - Video may be selectively backed up depending on size & importance.

