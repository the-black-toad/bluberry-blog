---
layout: post
title: "Image Hosting for the Blog: Cloudflare R2 and a Processing Script"
date: 2026-05-30
categories: [Linux]
tags: [linux, bash, imagemagick, cloudflare, workflow]
---

![](https://play-lh.googleusercontent.com/t2xwoWAJPoIHZlYiw82J31fZl40kj962j5DVHohn-Pgn7ZiuoXCl-2_NMyMERa7cCFw)

![](https://upload.wikimedia.org/wikipedia/commons/9/94/Cloudflare_Logo.png)

The blog needed a place to host images that wasn't the repo itself. This post covers the setup: Cloudflare R2 for storage, LocalSend for getting photos off the phone, and a script that strips metadata, resizes, and cleans up originals in one step.

## Why Not Just Commit Images

Storing images in the repo works until it doesn't. Git isn't designed for large binaries and a phone camera produces 3–8MB files with EXIF metadata that includes GPS coordinates, device info, and timestamps. Neither of those things belongs in a public git history.

## Cloudflare R2

R2 is Cloudflare's object storage. The relevant difference from S3 is no egress fees — you pay for storage and writes, not for every download. The free tier covers most personal blogs indefinitely: 10GB storage, 1M writes, 10M reads per month.

Setup is straightforward:
1. Create a bucket in the Cloudflare dashboard
2. Enable public access under the bucket's Settings tab
3. Upload files via the dashboard or CLI

The public URL format is `https://pub-xxxx.r2.dev/filename`. No custom domain required, though one can be added later.

Images are organized under a folder structure inside the bucket. URLs end up as `https://pub-xxxx.r2.dev/images/post-name/photo.jpg`, which is clean enough for use in markdown.

## Getting Photos Off the Phone

iPhones shoot HEIC by default and upload everything with full metadata intact. The phone can upload directly to R2 via the Files app, but that bypasses any processing step.

The solution is LocalSend — an open source AirDrop alternative that works over local WiFi with no cloud involved. It has both iOS and Linux apps. On Artix it's available from the AUR:

```bash
yay -S localsend-bin
```

Photos come over to a staging directory, get processed, and the processed versions go to R2.

## The Processing Script

ImageMagick handles resizing and format conversion. `exiftool` strips metadata. The script at `~/projects/blog-images/process-images.sh` takes one or more image files as arguments and outputs to `~/projects/blog-images/processed/`.

```bash
#!/usr/bin/env bash
set -euo pipefail

MAX_WIDTH=1200
QUALITY=85
OUT_DIR="$(dirname "$0")/processed"

if [[ $# -eq 0 ]]; then
  echo "Usage: $0 <image1> [image2] ..."
  exit 1
fi

mkdir -p "$OUT_DIR"

for img in "$@"; do
  if [[ ! -f "$img" ]]; then
    echo "Skipping: $img (not found)"
    continue
  fi

  filename="$(basename "$img")"
  # Convert HEIC to jpg
  if [[ "${filename,,}" == *.heic ]]; then
    filename="${filename%.*}.jpg"
  fi
  output="$OUT_DIR/$filename"

  echo "Processing: $filename"

  magick "$img" -resize "${MAX_WIDTH}x>" -quality "$QUALITY" "jpg:$output"
  exiftool -all= -overwrite_original "$output"
  rm "$img"

  echo "  -> $output"
done

echo "Done."
```

A few things worth noting:

**HEIC conversion:** ImageMagick has HEIC support via libheif. Passing `jpg:$output` explicitly forces JPEG encoding regardless of the output filename — without the prefix, ImageMagick infers format from the extension, which is less reliable. The filename is also changed from `.heic` to `.jpg` since browsers don't support HEIC.

**Resize behavior:** `-resize "${MAX_WIDTH}x>"` only downscales. The `>` flag means "only if larger than this." Images already under 1200px wide are passed through unchanged.

**Metadata stripping:** `exiftool -all= -overwrite_original` removes all EXIF tags in place. The `-overwrite_original` flag skips the backup file exiftool creates by default.

**Deletion:** The original is deleted after both processing steps succeed. `set -euo pipefail` means the script exits on any error, so if `magick` or `exiftool` fails, `rm` never runs.

## Workflow

```
phone → LocalSend → ~/projects/blog-images/ → process-images.sh → processed/ → R2
```

Run the script, verify the output looks right, upload the contents of `processed/` to R2, reference the URLs in the post. The originals are gone and the processed files have no location data and are a reasonable size for web use.

---

*Project repo: [bluberry](https://github.com/the-black-toad/bluberry) — three-camera stereo rig + classical CV + YOLO for autonomous blueberry picking.*
