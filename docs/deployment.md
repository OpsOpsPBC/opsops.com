# Opsops — Deployment & Local Development Guide

This guide covers deployment workflows, Drupal configuration management, and database synchronization for **OpsOpsPBC/opsops.com**.

Server-wide provisioning, Nginx snippets, and system administration are documented in the **OpsOpsPBC/drupal-multisite-server** repository.

---

## 1. Environments

The site is hosted on a DigitalOcean droplet using an atomic zero-downtime deployment pattern:

| Environment | Branch | URL | Server Path | Document Root |
|---|---|---|---|---|
| **Live** | `main` | `https://opsops.com` | `/var/www/opsops.com/live` | `/var/www/opsops.com/live/current/web` |
| **Staging** | `staging` | `https://staging.opsops.com` | `/var/www/opsops.com/staging` | `/var/www/opsops.com/staging/current/web` |

*Note: The directory name matches the GitHub repository name (`opsops.com`).*

### Directory Layout on the Server

```text
/var/www/opsops.com/{live|staging}/
├── current -> releases/<timestamp>
├── releases/
│   ├── <timestamp_1>/
│   └── <timestamp_2>/
├── shared/
│   ├── files/          # Persistent public uploads (symlinked to web/sites/default/files)
│   ├── private/        # Persistent private uploads
│   └── settings.php    # Database credentials & environment settings
└── backups/
    └── db_<timestamp>.sql.gz
```

---

## 2. CI/CD Deployment Workflow

Deployments run automatically via `.github/workflows/deploy.yml`:

- **Deploy to Live**: Push to `main` (or run manual workflow on `main`).
- **Deploy to Staging**: Push to `staging` (or run manual workflow on `staging`).

### What GitHub Actions Runs on Deploy:

1. **Build Artifact**:
   - Runs `composer install --no-dev --optimize-autoloader`.
   - Packages the site into an archive (excluding git history, tests, and local files).
2. **Transfer**:
   - Securely uploads archive via SCP to a unique remote tarball path in `/tmp`.
3. **Atomic Release on Server**:
   - Extracts archive into `/var/www/opsops.com/{env}/releases/<timestamp>`.
   - Links `shared/files` and `shared/settings.php`.
   - Atomically switches the `current` symlink to the new release.
   - Takes a pre-deployment database backup in `backups/`.
   - Runs `drush deploy -y -v` (runs DB updates, imports `config/sync`, rebuilds caches).
   - If `drush deploy` fails, **automatically rolls back** the `current` symlink to the previous release.
   - Reloads PHP-FPM to flush OPcache.
   - Prunes old releases and DB backups (retaining the 5 most recent).
   - Runs optional health check against `HEALTH_URL`.

### GitHub Secrets Required

Configure these secrets in GitHub under **https://github.com/OpsOpsPBC/opsops.com > Settings > Secrets and variables > Actions**:

| Secret | Value / Description | Scope |
|---|---|---|
| `SSH_HOST` | Droplet IP address (`134.199.244.78`) | Repository secret |
| `SSH_USER` | `deploy` | Repository secret |
| `SSH_PRIVATE_KEY` | Private SSH key for the `deploy` user | Repository secret |
| `SSH_KNOWN_HOSTS` | Output of `ssh-keyscan -H 134.199.244.78` | Repository secret |
| `HEALTH_URL` | Health check URL (e.g. `https://opsops.com`) | Environment secret (`live` / `staging`) |

---

## 3. Drupal Configuration Workflow

Configuration is tracked in Git under `config/sync/`. In `shared/settings.php`, Drupal points to this directory:
```php
$settings['config_sync_directory'] = '../config/sync';
```

### The Standard Workflow:

1. **Make changes locally** in your DDEV environment (e.g. create fields, alter views, configure modules).
2. **Export configuration**:
   ```bash
   ddev drush cex -y
   ```
3. **Review and commit**:
   ```bash
   git status
   git diff config/sync
   git add config/sync
   git commit -m "Configure contact form and block placement"
   ```
4. **Push to deploy**:
   - Push to `staging` to test on the staging server:
     ```bash
     git push origin staging
     ```
   - Push to `main` to deploy to production:
     ```bash
     git push origin main
     ```
5. GitHub Actions executes `drush deploy -y`, which automatically imports the configuration and rebuilds the cache.

> **Warning:** Never edit configuration directly on the live server. It will be overwritten on the next deployment.

---

## 4. Local Database & File Syncing

To sync the live site database and media files down to your local DDEV environment:

### Step 1: Dump and Download the Database

```bash
# Export the database on the live server
ssh deploy@134.199.244.78 "cd /var/www/opsops.com/live/current && ./vendor/bin/drush sql:dump --gzip --structure-tables-key=common --result-file=/tmp/live_dump.sql.gz"

# Download the dump to your machine
scp deploy@134.199.244.78:/tmp/live_dump.sql.gz ./live_dump.sql.gz

# Clean up remote temporary dump
ssh deploy@134.199.244.78 "rm -f /tmp/live_dump.sql.gz"
```

### Step 2: Import into DDEV

```bash
# Import database into DDEV
ddev import-db --src=./live_dump.sql.gz
rm ./live_dump.sql.gz

# Rebuild cache and generate a one-time login link
ddev drush cr
ddev drush uli
```

### Step 3: (Optional) Sync Uploaded Files

Drupal automatically regenerates styles and caches, so you only need public assets:

```bash
rsync -avz --exclude='styles/' --exclude='css/' --exclude='js/' \
  deploy@134.199.244.78:/var/www/opsops.com/live/shared/files/ web/sites/default/files/
```

---

## 5. Refreshing Staging from Live

To reset staging to match live data:

```bash
ssh deploy@134.199.244.78

# 1. Dump live database
cd /var/www/opsops.com/live/current
./vendor/bin/drush sql:dump --gzip --structure-tables-key=common --result-file=/tmp/staging_refresh.sql.gz

# 2. Import into staging
cd /var/www/opsops.com/staging/current
gunzip -c /tmp/staging_refresh.sql.gz | ./vendor/bin/drush sql:cli
rm -f /tmp/staging_refresh.sql.gz

# 3. Apply staging configuration and rebuild cache
./vendor/bin/drush deploy -y
```

---

## 6. Manual Rollback Procedure

If a deployment succeeds in CI but causes unexpected application issues, you can immediately roll back the code symlink:

```bash
ssh deploy@134.199.244.78

# 1. Inspect recent releases
ls -1dt /var/www/opsops.com/live/releases/*/

# 2. Point 'current' back to the previous release
ln -sfn /var/www/opsops.com/live/releases/<PREVIOUS_TIMESTAMP> /var/www/opsops.com/live/current

# 3. Reload PHP-FPM to clear OPcache
sudo systemctl reload php8.4-fpm

# 4. Clear Drupal cache
cd /var/www/opsops.com/live/current && ./vendor/bin/drush cr
```

If database changes need to be rolled back, pre-deploy backups are saved in `/var/www/opsops.com/live/backups/`:

```bash
cd /var/www/opsops.com/live/current
gunzip -c /var/www/opsops.com/live/backups/db_<TIMESTAMP>.sql.gz | ./vendor/bin/drush sql:cli
./vendor/bin/drush cr
```
