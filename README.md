# opsops.com

This repository contains the Drupal CMS codebase for [opsops.com](https://opsops.com), managed by OpsOps PBC.

## Getting Started

Local development uses [DDEV](https://ddev.com).

### Prerequisites

- [Docker](https://docs.docker.com/get-docker/)
- [DDEV](https://ddev.com/get-started/) (v1.25.0+)

### Setup

1. Clone the repository and navigate into the project directory:
   ```bash
   git clone git@github.com:OpsOpsPBC/opsops.com.git
   cd opsops.com
   ```

2. Start the DDEV environment:
   ```bash
   ddev start
   ```
   *(Dependencies are automatically installed via Composer post-start hooks).*

3. Launch the site in your browser:
   ```bash
   ddev launch
   ```

4. Log in as an administrator:
   ```bash
   ddev drush uli
   ```

## Deployment & Environments

Full deployment procedures, environment details, CI/CD automation, and sync workflows are documented in **[docs/deployment.md](docs/deployment.md)**.

Key topics covered in the deployment guide:
- **Environments**: Live (`https://opsops.com`) and Staging (`https://staging.opsops.com`) running on DigitalOcean with atomic zero-downtime releases.
- **CI/CD Pipeline**: Automated deployments via GitHub Actions (`.github/workflows/deploy.yml`) on pushes to `main` and `staging`.
- **Configuration Management**: Exporting, reviewing, and committing Drupal configuration (`config/sync/`).
- **Database & Asset Syncing**: Downloading and importing live database dumps and uploaded media into local DDEV environments.
- **Rollback Procedures**: Immediate zero-downtime symlink rollbacks and database restores in case of issues.

## Common Workflows

Run all commands from the project root:

- **Start / stop environment**:
  ```bash
  ddev start
  ddev stop
  ```
- **Install PHP dependencies**:
  ```bash
  ddev composer install
  ```
- **Rebuild Drupal cache**:
  ```bash
  ddev drush cache:rebuild  # or: ddev drush cr
  ```
- **Run database updates**:
  ```bash
  ddev drush update:db -y
  ```
- **Configuration workflow**:
  - Export configuration from local database to files:
    ```bash
    ddev drush config:export -y   # or: ddev drush cex -y
    ```
  - Import configuration from files to database:
    ```bash
    ddev drush config:import -y   # or: ddev drush cim -y
    ```
- **Add a module**:
  ```bash
  ddev composer require drupal/<module_name>
  ddev drush pm:enable -y <module_name>
  ddev drush cache:rebuild
  ```

## Project Structure & Guardrails

- Custom code belongs in `web/modules/custom` and `web/themes/custom`.
- Do not edit Drupal core or contributed modules/themes in place.
- Do not commit machine-local overrides or secrets (`.env`, `settings.local.php`, `.ddev/config.local.yaml`).
- Do not commit `vendor/` or uploaded user files under `web/sites/*/files`.

## References

- [Deployment Guide](docs/deployment.md)
- [Drupal CMS Documentation](https://project.pages.drupalcode.org/drupal_cms/)
- [DDEV Documentation](https://docs.ddev.com/)
- [Drupal Configuration Management Guide](https://www.drupal.org/docs/administering-a-drupal-site/configuration-management/workflow-using-drush)
