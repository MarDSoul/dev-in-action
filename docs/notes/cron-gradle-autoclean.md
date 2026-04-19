# Script: auto-clean Gradle caches

- date: Apr 2026
- link: Inspired by [Eran Boudjnah's post on LinkedIn](https://www.linkedin.com/posts/eranboudjnah_android-developers-heres-a-friendly-reminder-activity-7294989776686927872-Cwhi)

Script for cleaning Gradle cache. It make sense setup it in `cron` as 2 times per month:

```bash
#!/bin/bash

set -eu

LOG_DIR="$HOME/documents/logs"
LOG_FILE="$LOG_DIR/gradle_clean.log"
G_CACHES_DIR="$HOME/.gradle/caches"

mkdir -p "$LOG_DIR"

echo "Date: $(date '+%F'). Current size:" >> "$LOG_FILE"
du -sh "$G_CACHES_DIR" >> "$LOG_FILE"

find "$G_CACHES_DIR" -maxdepth 1 -regextype posix-extended \
    -regex ".*/[0-9]+\.[0-9]+.*" \
    -type d -mtime +30 \
    -exec rm -rf {} +

if [ -d "$G_CACHES_DIR/build-cache-1" ]; then
    find "$G_CACHES_DIR/build-cache-1" -mindepth 1 -type f -mtime +30 -delete
fi

find "$G_CACHES_DIR" -maxdepth 1 -regextype posix-extended \
    -regex ".*/jars-[0-9]+" \
    -type d -mtime +30 \
    -exec rm -rf {} +

G_CACHES_MOD_DIR="$G_CACHES_DIR/modules-2"
if [ -d "$G_CACHES_MOD_DIR" ]; then
    if [ -d "$G_CACHES_MOD_DIR/files-2.1" ]; then
        find "$G_CACHES_MOD_DIR/files-2.1" -mindepth 2 -maxdepth 2 \
            -type d -mtime +30 \
            -exec rm -rf {} +
    fi
    shopt -s nullglob
    for metadata_dir in "$G_CACHES_MOD_DIR"/metadata-*; do
        if [ -d "$metadata_dir/descriptors" ]; then
            find "$metadata_dir/descriptors" -mindepth 2 -maxdepth 2 \
                -type d -mtime +30 \
                -exec rm -rf {} +
        fi
    done
    shopt -u nullglob
fi

find "$G_CACHES_DIR" -depth -type d -empty -delete

echo "After clean size:" >> "$LOG_FILE"
du -sh "$G_CACHES_DIR" >> "$LOG_FILE"
echo -e "--------\n" >> "$LOG_FILE"
```
