---
title: Restoring Bookstack
date: 2026-04-08 12:00:00 +0200
categories: [Informatics]
tags: [bookstack, notes, second brain]     # TAG names should always be lowercase
---

## Prepare necessary secrets for access to Backblaze, restic repo and Bookstack

1. Restore `restic-env` file (entry `secure.backblaze.com`) from password manager as `~/restic-env` for access to restic repo on Backblaze
2. Create file `~/restic-password` with a single line, the restic repo password (from password manager)
3. Restore Bookstack `.env` file from password manager as `~/apps/bookstack/.env` (or fill in `APP_KEY`, `MYSQL_ROOT_PASSWORD`, `MYSQL_PASSWORD` variables)
    - Delete / do not use `APP_URL` variable (instead, use `restore/docker-compose.yml.template` where this variable is pre-set)
4. Install Docker

### Additional notes and links

- [backblaze.com](https://www.backblaze.com/docs/cloud-storage-integrate-restic-with-backblaze-b2)

## Run the restoration script

```sh
#!/usr/bin/bash
set -euf -o pipefail
# trap "rm -f ~/restic-env ~/restic-password" EXIT

# Install restic if required
export PATH="$HOME/bin:$PATH"
if ! command -v restic &> /dev/null; then
    # install restic
    OPSYS=linux
    ARCH="$(uname -m)"
    [ "$(uname -m)" == "aarch64" ] && ARCH="arm64" || ARCH="amd64"
    RESTIC_BASE_URL=https://github.com/restic/restic/releases/download/
    RESTIC_TAG="0.18.1"  # we'll self-update anyway
    VERSION_ARCH=v${RESTIC_TAG}/restic_${RESTIC_TAG}_${OPSYS}_${ARCH}.bz2
    echo "RESTIC DOWNLOAD URL: ${RESTIC_BASE_URL}${VERSION_ARCH}"
    mkdir -p ~/bin
    curl -L --silent ${RESTIC_BASE_URL}${VERSION_ARCH} | bunzip2 > ~/bin/restic

    chmod +x ~/bin/restic
    echo "-----"
    ~/bin/restic self-update
else
    echo "restic is already installed"
fi

# Load variables
[ -f ~/restic-env ] || { echo "Error: ~/restic-env not found"; exit 1; }
[ -f ~/restic-password ] || { echo "Error: ~/restic-password not found"; exit 1; }
chmod 600 ~/restic-env
chmod 600 ~/restic-password
source ~/restic-env

# Check restic repository snapshots
# restic snapshots  # check snapshots present in cloud repository

# Create project directory
APP_DIR=~/apps/bookstack/
mkdir -p $APP_DIR

# The goal is a "minimal restore" to quickly get access to the backup, to be able to
# export the documents (https://www.bookstackapp.com/docs/user/export-import/).
# Therefore, instead of domain name and reverse proxy, we'll use localhost.
cp ./docker-compose.yml.template $APP_DIR/docker-compose.yml
# APP_KEY, MYSQL_ROOT_PASSWORD, MYSQL_PASSWORD variables are automatically
# picked up by docker compose.

# Restore from Backblaze: bookstack_app_data, bookstack_db_data
cd $APP_DIR
mkdir _restored_
restic restore latest --target _restored_  # `latest` = latest snapshot
# Double-check that it's the correct directory structure; we want the directories
# bookstack_app_data and bookstack_db_data.
mv _restored_/userdata/bookstack_* .

# Remove sensitive files
rm ~/restic-env ~/restic-password

# Fire up containers
docker compose pull  # pull/update containers defined in docker-compose.yml
docker compose up  # -d
# docker compose down  # stop and remove containers

# Access Bookstack GUI in the browser, on `http://localhost:6875`
```
