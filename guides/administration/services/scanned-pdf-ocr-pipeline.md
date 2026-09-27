---
type: guide
title: Scanned PDF to searchable document pipeline with a folder watcher
description: Build an unattended pipeline that OCRs, cleans and compresses scanned PDFs and delivers them into a document-management import folder.
tags: [ocr, ocrmypdf, pdf, document-management, docker, samba]
status: draft
resource:
created: 2026-09-27T19:49:16Z
updated: 2026-09-27T19:49:16Z
generated:
  by: claude/sonnet-5
  at: 2026-09-27T19:49:16Z
verified: []
stale_after: 2027-09-27T19:49:16Z
sources:
- id: 2026-09-27-pdf-scan-ocr-pipeline
  resource: 'private:/sources/administration/services/2026-09-27-pdf-scan-ocr-pipeline.md'
relations: []
superseded_by:
---

# Scanned PDF to searchable document pipeline with a folder watcher

A physical scanner produces image-only PDFs that aren't searchable and are much larger than they
need to be. This guide builds an unattended pipeline: drop a scan in, walk away, and it shows up
in your document-management system already OCR'd, cleaned up, and compressed — with the original
kept as a fallback.

## Prerequisites

- A network scanner (or any process) that can write PDFs to a folder reachable over the network
  (e.g. an SMB/Samba share).
- Docker, to run an OCR watcher container.
- A document-management system (or any destination) with its own watched import folder(s).

## The requirement, independent of implementation

State the requirement before picking tools — the specific pieces (OCR engine, watch mechanism,
delivery target) can all be swapped as long as this holds:

1. Scanned, image-only PDFs land somewhere reachable over the network.
2. They go through OCR (get a text layer) and cleanup (deskew, rotate upright, remove scan noise).
3. They get optimized/compressed — scanner output is large and redundant.
4. OCR'd, optimized output lands automatically in the destination's watched import folder(s), with
   no manual step — split by category if the destination supports categories.
5. The original, un-OCR'd scan is kept as a fallback/archive.
6. It all runs unattended.

## Steps

### 1. Set up the intake folder

Expose a folder over the network (e.g. an SMB share) that the scanner writes to. If you have
multiple destination categories, give each one its own input subfolder
(`input/<category>/`) so routing is just "which folder did it land in."

### 2. Run an OCR watcher container per category

Using [`jbarlow83/ocrmypdf`](https://hub.docker.com/r/jbarlow83/ocrmypdf) (a Docker image wrapping
[`ocrmypdf`](https://ocrmypdf.readthedocs.io/) with a filesystem-watcher wrapper):

```yaml
services:
  ocr-watcher-<category>:
    image: jbarlow83/ocrmypdf
    restart: unless-stopped
    environment:
      OCR_ON_SUCCESS_ARCHIVE: "1"
      OCR_JSON_SETTINGS: >-
        {"clean": true, "mode": "skip", "rotate_pages": true, "deskew": true, "languages": ["<lang>"], "optimize": "1", "jobs": "5"}
    volumes:
      - "<input-dir>/<category>:/input"
      - "<destination-import-dir>/<category>:/output"
      - "<archive-dir>/<category>:/processed"
    entrypoint: python3
    command: /app/watcher.py
```

| Setting | Effect |
|---|---|
| `clean` | runs `unpaper` to strip scan noise before OCR |
| `mode: skip` | skip OCR on pages that already have a text layer (avoids double-OCR) |
| `rotate_pages` / `deskew` | auto-straighten pages |
| `languages` | OCR language(s), e.g. `["eng"]`, `["deu"]` |
| `optimize: 1` | the actual compression step — image recompression, JBIG2 for B/W scans, etc. |
| `jobs` | parallel OCR workers per file |
| `OCR_ON_SUCCESS_ARCHIVE=1` | after success, move the *original* input file into `/processed` — the fallback archive |

`/output` lands directly in the destination's own watched import folder — satisfies the
"no manual step" requirement.

### 3. Fix folder permissions before first use

The container runs as a non-root user (commonly uid 1000). Create `/input` and `/processed` with
permissions that user can write to (e.g. `777`, matching what the image itself creates for
`/output`) — **do this before the first run**, not after the first OCR success (see
Troubleshooting).

## Verify

```bash
docker ps -a --filter name=ocr-watcher                    # should show "Up", not "Restarting"
docker logs --tail 30 ocr-watcher-<category>               # recent "OCR is done" lines, no tracebacks
ls <input-dir>/<category>                                  # should be empty/near-empty if the watcher is keeping up
```

Drop a real scanned PDF into the input folder and confirm it appears, OCR'd, in the destination
import folder within a few seconds to a couple of minutes depending on page count.

## Troubleshooting

- **The watcher image's internal script path and settings field names can change between
  versions.** A `latest`-tagged image auto-updated by something like Watchtower can silently break
  your compose file — the watcher's on-disk entrypoint path, or field names in the JSON settings,
  may differ between releases. If the container starts crash-looping right after an image update
  with a "file not found" error on the entrypoint script, or the logs show a validation error
  about an unrecognized settings field, check the image's current internals directly (e.g. exec
  into the container and inspect the installed package's expected fields) against your compose
  file's settings, rather than assuming your own config regressed.
- **The watcher typically only reacts to filesystem "created" events, not a startup scan.** A
  backlog of files already sitting in the input folder when the container (re)starts may not get
  picked up automatically, and a moved/renamed file may not trigger processing either. If files
  aren't moving, check whether they're genuinely new since the container last started, or "touch"
  a fresh copy in to force a new create event.
- **A single unhandled exception can silently kill the watcher's processing thread without killing
  the container.** `docker ps` keeps showing "healthy" while the watcher has actually stopped
  picking up new files — one bad/corrupt input file, or a permission error (see below), is enough.
  If files stop moving but the container looks fine, restart it rather than trusting the status
  display; if a specific input file reliably triggers this, remove it and check it separately.
- **Wrong ownership on the input/archive folders causes a delayed failure, not an immediate one.**
  If those folders are owned by root with restrictive permissions (e.g. freshly created via
  `mkdir` on the host), OCR itself can succeed, but the "move the original into the archive"
  step then fails with a permission error — which can then trigger the silent-thread-death issue
  above, so the *first* successful run is also the *last* one that works. Fix folder ownership
  before the first run, not after diagnosing a mysterious one-shot stop.
- **A folder under constant automated file churn can cause Windows SMB clients to hang or
  crash.** If your intake/delivery folders are shared over SMB and see files created/moved every
  few seconds while OCR runs, a Windows client with the folder open in Explorer can get flooded
  with SMB2 lease-break notifications and stop refreshing (or crash). Disabling opportunistic
  locking on the relevant Samba shares (`oplocks = no`, `level2 oplocks = no`) trades some
  browsing performance on those specific shares for stability, and is worth doing proactively on
  any share you know will see this kind of churn.

## Related

<None yet.>

## Sources

- [Legacy wiki.js: PDF scan OCR document manager pipeline](../../../../../sources/administration/services/2026-09-27-pdf-scan-ocr-pipeline.md) — private source (real configuration and fragility notes)
